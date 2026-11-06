# 03 — PoC plan

Each phase ends with acceptance criteria that a command can check. A phase is done when its criteria pass, not when its code exists. Phases 1–4 run entirely on a laptop. Nothing is deployed to AWS before Phase 6.

## Planned repository layout

```
serverless-pdf-api/
├── serverless.yml
├── Dockerfile                    # FROM public.ecr.aws/lambda/python:3.13 (arm64)
├── requirements.txt              # pdf-tooling==1.5.0 + boto3, hash-pinned
├── pyproject.toml                # dev tooling only (pytest, ruff, moto), managed by uv
├── src/api/
│   ├── registry.py               # the verb table from 02-api-contract.md, as data
│   ├── argv.py                   # request → argv list (allowlist, --flag=value, server flags)
│   ├── runner.py                 # subprocess with deadline, scratch dir, cleanup
│   ├── envelope.py               # exit-code→HTTP table, path rewriting, response shape
│   ├── storage.py                # presigned POST/GET, key namespacing
│   └── handlers/{uploads,worker}.py
├── tests/
│   ├── unit/                     # argv building, injection cases, mapping table, rewriting
│   ├── contract/                 # registry vs `pdftooling <verb> --help` of the pinned build
│   └── integration/              # the image under the Lambda Runtime Interface Emulator + moto S3
├── scripts/
│   ├── client.sh                 # upload → call → download, for demos
│   └── smoke.sh                  # every verb against a deployed stage
└── .github/workflows/ci.yml      # lint, tests, arm64 image build (no deploy)
```

## Phase 0 — Prerequisites (operator)

- Install Serverless Framework v4 (`npm i -g serverless`) and run `serverless login`, or decide to use `osls` instead.
- An AWS profile with deploy rights in the target account; region `us-east-1`.
- Create an AWS Budgets alarm at $5/month to your e-mail (manual, once).
- Docker with arm64 build support (`docker buildx`; QEMU emulation if the build host is x86_64).

**Done when:** `serverless --version` reports 4.x, `aws sts get-caller-identity` succeeds, and `docker buildx ls` lists `linux/arm64`.

## Phase 1 — Lambda image runs pdf-tooling

- Write the `Dockerfile` and the hash-pinned `requirements.txt`.
- Build for `linux/arm64`, then run `pdftooling doctor -o json` and `pdftooling version -o json` inside the image.
- Measure the image size, plus the CLI's cold and warm startup time inside the container (`time pdftooling version -o json`, 10 runs).

**Done when:**
- All PDF engines report available in `doctor`, and `version` reports `1.5.0`. Only `tesseract`/`soffice` are expected to be missing.
- The arm64 build installs only wheels, with no source build of pikepdf, pypdfium2 or Pillow (check `pip install --only-binary=:all:`).
- Image size and startup times are written in `docs/measurements.md`.
- **Decision recorded:** if warm CLI startup is above roughly 1 s, consider calling the CLI entry point in-process. Subprocess stays the default because it guarantees exit-code fidelity.

## Phase 2 — Core library: registry, argv, envelope

- `registry.py` encodes the verb table from `02-api-contract.md`.
- `argv.py` turns a validated request into an argument list.
- `envelope.py` holds the exit-code→HTTP table, path rewriting and output naming.
- `runner.py` creates the scratch directory, writes password files with `0600`, runs the CLI with a deadline (`timeout = remaining_ms/1000 − 2`), and always removes the directory.

**Done when** the unit tests prove:
- Every server-controlled flag in `options` is rejected with `400`, including `--in-place`, `-O`, `--password-file` and `output_format`.
- An option value of `"--in-place"` reaches the CLI as `--pages=--in-place` (one argument), and the CLI rejects it with exit 2.
- A password never appears in `argv`, the logs, or the response, and the password file's mode is `0600`.
- Every row of the exit-code table maps to the documented HTTP status, and an unknown exit code maps to `500`.
- Path rewriting changes only the documented fields. Every other key of a recorded real envelope passes through unchanged.
- The scratch directory is gone after success, failure and timeout alike.

## Phase 3 — Contract test against the pinned CLI

- For every registry verb, run `pdftooling <verb> --help` (from the installed 1.5.0) and assert that every allowlisted option exists as a flag.
- Assert that `pdf_tooling.cli.exit_codes.ALL_EXIT_CODES == (0,1,2,3,4,5,6)`, so a new code fails this test until `02-api-contract.md` maps it.
- Golden tests: run each verb once on fixture PDFs through `runner.py` (no AWS), store the rewritten response, and compare.

**Done when:** `pytest tests/contract` passes against 1.5.0. Deliberately misspelling an option in the registry makes it fail (prove it fails once, then revert).

## Phase 4 — Storage and handlers, locally

- `storage.py`: presigned POST with a `content-length-range` condition and server-generated keys `in/{api_key_id}/{upload_id}`; output upload under `out/{api_key_id}/{request_id}/`; presigned GET with 15-minute expiry and `Content-Disposition`.
- `handlers/uploads.py` and `handlers/worker.py` (API Gateway proxy events).
- Integration test: the built image under the Runtime Interface Emulator, S3 mocked with moto or a local MinIO.

**Done when:**
- Upload, rotate and download work end to end locally, and the downloaded PDF has page 1 rotated (checked with `pdftooling info --pages-detail`).
- An upload id from key A, used with key B's identity, returns `404`.
- A `dry_run: true` request returns `outputs: []` and writes nothing to the `out/` prefix.

## Phase 5 — `serverless.yml`

- Configure the ECR image, the two functions, the REST API routes with `private: true`, the API key and usage plan, the bucket (public access block, SSE-S3, 1-day lifecycle on `in/` and `out/`), least-privilege IAM, 14-day log retention, `reservedConcurrency: 5`, and API Gateway data tracing off.
- Check every key against the Serverless v4 docs, starting with those flagged in `01-stack.md`.

**Done when:** `serverless package --stage dev` succeeds, and the generated CloudFormation template (`.serverless/`) is reviewed for public access, IAM wildcards and the usage plan.

## Phase 6 — Deploy `dev` and smoke-test (needs operator go-ahead; creates billable resources)

- Run `serverless deploy --stage dev`, fetch the API key value into a local `.env`, and run `scripts/smoke.sh`.
- Record in `docs/measurements.md`: the p50/p95 latency per verb on a 1 MB and a 20 MB PDF, the Lambda cold start, and the duration and memory that CloudWatch reports. Then tune `memorySize`.

**Done when:**
- `smoke.sh` returns the expected status for every verb, plus one failure case for each of exit codes 1, 2, 4, 5 and 6.
- A request without `x-api-key` gets `403`. The 11th request in a burst gets `429`.
- A file uploaded to `in/` is readable only through presigned URLs, and the bucket has no public access.
- The budget alarm exists.

## Phase 7 — Stretch (pick after Phase 6)

| Item | Why |
|---|---|
| **Async jobs**: `POST /v1/jobs/{verb}` → `202 {job_id}`, a worker invoked asynchronously, status in DynamoDB, `GET /v1/jobs/{id}` | Removes the 29 s API Gateway limit for large `rasterize`/`compress` runs. |
| **OCR image variant** (Tesseract in a second image, `ocr` verb) | The first verb with an external engine, which exercises exit code 3 and pdf-tooling's engine carve-out on `--dry-run`. |
| **Batch inputs** (`--out-dir` multi-input runs) | pdf-tooling reports every input of a batch by name, so the API can return partial success per item. |
| **OpenAPI 3.1 document** generated from `registry.py` | A machine-readable contract and client generation. |
| **CI deploy via GitHub OIDC** to a `dev` stage, manual approval | No long-lived AWS keys in GitHub. |
| **Zip-packaged `uploads` function** | Faster cold start for the presign path. |

## Teardown

```bash
aws s3 rm s3://<bucket> --recursive   # the stack cannot delete a non-empty bucket
serverless remove --stage dev
# then delete the ECR repository Serverless created, if it remains
```

**Done when:** `aws cloudformation describe-stacks` no longer lists the stack, and the ECR repository and bucket are gone.

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| A pikepdf or pypdfium2 arm64 wheel is missing for the chosen Python | Low | Phase 1 checks it first. Fall back to `x86_64`, which costs about 20% more per GB-second. |
| CLI startup dominates latency | Medium | Phase 1 measures it; in-process invocation is the escape hatch. |
| Large PDFs exceed 29 s | Medium for `rasterize` | Return `504 timeout` cleanly; the async jobs in Phase 7 are the fix. |
| Serverless v4 login or licence friction | Low | `osls` (v3 fork) as a drop-in; the design uses only v3-compatible features. |
| A pdf-tooling 1.x upgrade adds an option or exit code | Expected | The Phase 3 contract test fails on the new exit code and the pin moves deliberately. Additions are non-breaking by pdf-tooling's contract. |
| Account details leaking in a public repo | Low | `.gitignore` covers `.env`; no ids are hard-coded; add a gitleaks step to CI in Phase 2. |
