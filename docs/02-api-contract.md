# 02 — CLI to API contract

## Principle

**The API is a transport for `pdftooling`, not a reinterpretation of it.** pdf-tooling's public API is its `-o json` envelope (`schema_version: 1`) and its exit-code table (0–6). This API:

1. maps a JSON request onto one `pdftooling` invocation, using a per-verb allowlist;
2. runs it with `-o json` in a fresh scratch directory;
3. returns the CLI's envelope **verbatim** under `result`, changing only the path strings (see "Path rewriting");
4. picks the HTTP status from the CLI's exit code using a fixed table;
5. uploads exactly the files named in `result.items[].output` and returns a presigned URL for each.

If the CLI and this document disagree about a verb's behaviour, the CLI is right and this document has a bug.

Pinned version: **pdf-tooling 1.5.0**. A contract test (see `03-poc-plan.md`, Phase 3) checks every allowlisted option against `pdftooling <verb> --help` from the pinned build.

## Authentication and limits

- Every route requires the header `x-api-key: <key>`. A missing or invalid key gets API Gateway's own `403 {"message":"Forbidden"}`.
- The usage plan allows 5 requests/s (burst 10) and 1000 requests/day per key; over that, API Gateway returns `429`.
- Maximum upload: 50 MB per file. Maximum inputs per request: 1, except `merge` (2–20).

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/health` | Liveness plus the deployed versions: the `pdftooling version -o json` envelope. |
| `GET` | `/v1/verbs` | The allowlist itself: each verb, how many inputs it takes, what it outputs, and its options with types. Generated from the same registry the handler uses. |
| `POST` | `/v1/uploads` | Get a presigned S3 POST for one input file. |
| `POST` | `/v1/{verb}` | Run a verb: `merge`, `split`, `extract`, `delete`, `rotate`, `reorder`, `compress`, `encrypt`, `decrypt`, `watermark`, `stamp`, `rasterize`, `text`, `info`, `permissions`. |
| `POST` | `/v1/meta/get`, `/v1/meta/set` | The `meta` group's two subcommands. |

### `POST /v1/uploads`

Request:
```json
{ "filename": "report.pdf", "content_type": "application/pdf" }
```
`filename` is used only for display and in download names. It is never used as a path.

Response `201`:
```json
{
  "upload_id": "upl_01JB7Z6Q4N9X8V2K3M5P7R1T0W",
  "url": "https://<bucket>.s3.amazonaws.com/",
  "fields": { "key": "in/…", "policy": "…", "x-amz-signature": "…", "Content-Type": "application/pdf" },
  "max_bytes": 52428800,
  "expires_at": "2026-11-05T12:15:00Z"
}
```
The client then sends `multipart/form-data` to `url` with every entry of `fields` followed by `file`. S3 rejects anything over `max_bytes`.

### `POST /v1/{verb}` — request

```json
{
  "inputs": ["upl_…"],
  "options": { "pages": "1-3", "angle": 90 },
  "dry_run": false,
  "secrets": { "password": "…" }
}
```

| Field | Type | Rule |
|---|---|---|
| `inputs` | array of upload ids | Required. Each id must belong to the caller and point at an uploaded object, or the API returns `404`. |
| `options` | object | Keys are the CLI's long flags in `snake_case` (`--image-dpi` becomes `image_dpi`). An unknown key returns `400`. Booleans become bare flags, and `false` omits the flag. Numbers and strings become `--flag=value`. A list becomes one comma-joined value (used by `encrypt`'s `allow`). |
| `dry_run` | boolean | Default `false`. Maps to `--dry-run`. Returns the CLI's plan, uploads no outputs, and leaves `outputs` as `[]`. |
| `secrets` | object | Optional: `password`, `owner_password`, `user_password`. Each is written to a `0600` file and passed as `--password-file`, `--owner-password-file` or `--user-password-file`. Never logged, never echoed back. |

**Flags the server always controls, which the client can never set:** `-o/--output-format` (always `json`), `-O/--output`, `--out-dir`, `--name`, `-f/--force`, `--in-place`, `--no-backup`, `-y/--yes`, `--threads`, and every `--*-password-file`. Sending one in `options` returns `400` with `kind: "usage"`.

### Response envelope

The same shape on every `POST /v1/{verb}`, successful or not:

```json
{
  "api_version": "1",
  "request_id": "req_01JB7Z…",
  "pdftooling_version": "1.5.0",
  "verb": "split",
  "exit_code": 0,
  "result": { "schema_version": 1, "verb": "split", "dry_run": false, "items": [ … ], "warnings": [], "duration_ms": 0, "exit_code": 0 },
  "outputs": [
    { "name": "report-001.pdf", "bytes": 1213, "url": "https://…presigned…", "expires_at": "2026-11-05T12:15:00Z" }
  ],
  "error": null
}
```

| Field | Meaning |
|---|---|
| `exit_code` | The `pdftooling` process's exit code, or `null` if the API rejected the request before running the CLI. |
| `result` | The CLI's stdout parsed as JSON: the operation envelope, or the CLI's error envelope `{"schema_version":1,"error":{"code","kind","message","path"}}`. `null` if the CLI never ran. |
| `outputs` | One entry per non-null `result.items[].output`, in item order. `[]` for read-only verbs and for dry runs. |
| `error` | `null`, or an API-level error `{"source":"api"\|"cli","kind","message"}`. For a CLI failure it repeats the CLI's `kind`/`message` with `source: "cli"`, so a client can check one field. |

**Rule for clients:** a request succeeded iff `exit_code == 0`. In a multi-item run (`merge`, `split`, `rasterize`), also read each `result.items[].ok`: an item that succeeded still has its file in `outputs` even when the run as a whole exits `1`.

### Path rewriting

The CLI reports paths inside the Lambda's scratch directory. Before returning the envelope, the API rewrites exactly three kinds of string, and nothing else:

| CLI field | Becomes |
|---|---|
| `items[].input`, `pages[].source` | The upload's display `filename` |
| `items[].output` | The `outputs[].name` it corresponds to |
| `error.path` | The display filename, or `null` |

Every other key and value is passed through unchanged, including keys a later pdf-tooling release adds.

**Output names.** On disk, inputs are named `input-<n>.pdf`. The display stem is the upload's `filename` without its extension, reduced to `[A-Za-z0-9._-]` and capped at 80 characters (`document` if nothing is left). For each output:
- `-O` verbs: the server writes `out/input-1.pdf`, and the name becomes `<display-stem>.<verb>.pdf` (e.g. `report.rotate.pdf`).
- `dir` verbs: the CLI names its files after the input stem (`input-1-001.pdf`, `input-1-0001.png`), and the server replaces the leading `input-1` with the display stem (`report-001.pdf`).

The S3 key is `out/{api_key_id}/{request_id}/<name>`. The presigned GET sets `Content-Disposition: attachment; filename="<name>"`.

## Exit code to HTTP status

| CLI exit | CLI `kind` | HTTP | Notes |
|---|---|---|---|
| 0 | — | **200** | Includes dry runs and empty-but-valid reports. |
| 1 | `failure` | **422** | Corrupt input, an engine error, or one or more items failing. Outputs of successful items are still returned. |
| 2 | `usage` | **400** | The CLI rejected the flag combination, e.g. `rotate` without `pages`, or `split` with two modes. |
| 3 | `engine_missing` | **500** | A server misconfiguration (an engine absent from the image). Should never happen for in-scope verbs; alarmed. |
| 4 | `no_input` | **422** | Valid but nothing to act on, e.g. a page selection that resolves to zero pages. |
| 5 | `refused` | **409** | A safety gate declined. Fresh scratch directories make this rare; an encrypted input without `allow_decrypted_output` is the usual case. |
| 6 | `auth` | **422** | The PDF's password is missing, wrong, or of the wrong kind. Deliberately not `401`/`403`, which mean the *API key*. |
| — | `timeout` (api) | **504** | The CLI did not finish within the deadline (26 s) and was killed. |
| — | `not_found` (api) | **404** | An unknown upload id, or one owned by another key. |
| — | `usage` (api) | **400** | API-level validation: unknown verb or option, a server-controlled flag, a wrong type, a wrong number of inputs. |
| — | `too_large` (api) | **413** | The uploaded object exceeds `max_bytes` (a backstop to S3's policy check). |
| — | `internal` (api) | **500** | CLI stdout that is not JSON, an S3 failure, or any other unexpected error. Logged with `request_id`. |

A new CLI exit code (pdf-tooling only ever *adds* codes) maps to `500` with `kind` passed through until this table is updated. The contract test fails when `ALL_EXIT_CODES` grows.

## Verb registry (PoC scope)

"Output" says how the server sets the destination: `-O` means one file (`-O out/<stem>.pdf`), `dir` means `--out-dir out/`, and `json` means the result is the envelope alone, with no file.

| Endpoint | Inputs | Output | Client options (`snake_case` → CLI flag) | Secrets |
|---|---|---|---|---|
| `merge` | 2–20 | `-O` | `bookmarks` (per-file\|preserve\|none), `allow_decrypted_output` | `password` |
| `split` | 1 | `dir` | exactly one of `every` (int), `ranges` (str), `each_page`, `at_bookmarks`; `allow_decrypted_output` | `password` |
| `extract` | 1 | `-O` | `pages` (str, required), `allow_decrypted_output` | `password` |
| `delete` | 1 | `-O` | `pages` (str, required), `allow_decrypted_output` | `password` |
| `rotate` | 1 | `-O` | `pages` (str, required), `angle` (90\|180\|270\|-90), `absolute`, `allow_decrypted_output` | `password` |
| `reorder` | 1 | `-O` | `pages` (str, required), `allow_decrypted_output` | `password` |
| `compress` | 1 | `-O` | `lossless`, `images` (keep\|downsample\|recompress), `image_dpi` (float), `image_quality` (int), `pages` (needs `images`), `allow_decrypted_output` | `password` |
| `encrypt` | 1 | `-O` | `allow` (list of permission tokens), `legacy` | `owner_password`, `user_password` |
| `decrypt` | 1 | `-O` | — | `password` |
| `watermark` | 1 | `-O` | `text` (str, required), `position` (overlay\|underlay), `opacity` (0–1), `rotate` (float), `font_size` (float), `pages`, `allow_decrypted_output` | `password` |
| `stamp` | 1 | `-O` | `from` (**upload id**, resolved to a server path), `from_page` (int), `position`, `pages`, `allow_decrypted_output` | `password` |
| `rasterize` | 1 | `dir` | `pages`, `dpi` (float) **or** `width` (int), `format` (png\|jpeg\|tiff\|webp), `quality` (1–100), `grayscale` | `password` |
| `text` | 1 | `json` | `pages`, `layout` | `password` |
| `info` | 1 | `json` | `fonts`, `pages_detail` | `password` |
| `permissions` | 1 | `json` | — | `password` |
| `meta/get` | 1 | `json` | `xmp` | `password` |
| `meta/set` | 1 | `-O` | `title`, `author`, `subject`, `keywords`, `creator` (str), `clear_producer`, `clear_all`, `allow_decrypted_output` | `password` |

The API checks only types, the number of inputs, and the allowlist. Rules about how flags combine (mutually exclusive modes, required flags, value ranges) stay the CLI's job and come back as `400 usage` from exit code 2, so the API never duplicates rules that could drift from the CLI.

## Worked example

```bash
# 1. upload
curl -s -H "x-api-key: $KEY" -d '{"filename":"report.pdf"}' "$API/v1/uploads" > up.json
curl -s $(jq -r '.fields | to_entries[] | "-F \(.key)=\(.value)"' up.json) -F file=@report.pdf "$(jq -r .url up.json)"

# 2. rotate page 1 by 90 degrees
curl -s -H "x-api-key: $KEY" -d "{\"inputs\":[\"$(jq -r .upload_id up.json)\"],\"options\":{\"pages\":\"1\",\"angle\":90}}" \
  "$API/v1/rotate" | tee res.json | jq '{exit_code, error, outputs: [.outputs[].name]}'

# 3. download
curl -s -o rotated.pdf "$(jq -r '.outputs[0].url' res.json)"
```

The server runs this command for step 2:

```
pdftooling rotate --pages=1 --angle=90 -o json -O /tmp/req-…/out/input-1.pdf /tmp/req-…/in/input-1.pdf
```

## Versioning of this API

- `/v1` and `api_version: "1"` follow the same rule as pdf-tooling's `schema_version`: **additions never bump it.** A new verb, option or response key is additive. A rename, a removal, a type change or a meaning change requires `/v2`.
- Upgrading pdf-tooling within `1.x` is additive by pdf-tooling's own contract. The pinned version moves only after the contract test passes against the new release.
