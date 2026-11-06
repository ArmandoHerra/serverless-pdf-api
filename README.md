# Serverless PDF API

A proof of concept that serves [`pdf-tooling`](https://github.com/ArmandoHerra/pdf-tooling) **1.5.0** as an HTTP API on AWS Lambda behind API Gateway, deployed with the Serverless Framework.

Every endpoint wraps one `pdftooling` verb. The API adds transport around the CLI (uploads, downloads, API keys, quotas) and changes nothing else. The CLI's JSON envelope and exit codes are passed through to the client, so the API's semantics are the CLI's semantics.

> **Status: design only.** No code or infrastructure exists yet. The design is in [`docs/`](docs/); the build steps are in [`docs/03-poc-plan.md`](docs/03-poc-plan.md).

## How it works

```
client ──1. POST /v1/uploads──────────▶ presigned S3 POST
client ──2. upload the PDF to S3 directly (never through API Gateway)
client ──3. POST /v1/{verb} {inputs:[upload_id], options:{…}}
            └▶ Lambda: download → run `pdftooling <verb> … -o json` → upload outputs
client ◀──4. CLI envelope + presigned GET URLs for every output file
```

## Documents

| Doc | What it decides |
|---|---|
| [`docs/01-stack.md`](docs/01-stack.md) | AWS resources, Serverless Framework layout, runtime and packaging, security and cost guardrails |
| [`docs/02-api-contract.md`](docs/02-api-contract.md) | The CLI-to-API contract: endpoints, request/response shapes, flag mapping, exit-code-to-HTTP mapping |
| [`docs/03-poc-plan.md`](docs/03-poc-plan.md) | The build phases, each with acceptance criteria, plus risks, cost and teardown |

## Scope of the PoC

The PoC exposes these pdf-tooling verbs: `merge`, `split`, `extract`, `delete`, `rotate`, `reorder`, `compress`, `encrypt`, `decrypt`, `watermark`, `stamp`, `rasterize`, `text`, `info`, `permissions`, `meta get` and `meta set`.

`ocr` and `convert` are out of scope. They need Tesseract and LibreOffice in the image, which would multiply the image size and cold-start time. The plan lists them as a stretch phase.

## License

Apache-2.0, the same licence as pdf-tooling. pdf-tooling keeps AGPL and GPL code off its call graph, and this wrapper adds none.
