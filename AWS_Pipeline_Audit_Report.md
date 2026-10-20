# Audit Report: Serverless PII-Safe Document Processing Pipeline (AWS)

Project audited: `cloud_aws_review_ready.zip` (31 files, ~3,100 lines of Python/JS, one SAM template).

## How to read the evidence tags

| Tag | Meaning |
|---|---|
| **[S]** | **Static** evidence: read directly from a file (path and line numbers given). |
| **[L]** | **Local run**: I executed the project's own code on a sandbox machine using mocked AWS (`moto` and mocked Textract/Comprehend clients). This proves what the *code* does. It does **not** prove how real AWS behaves. |
| **[W]** | **Web**: checked against AWS documentation. |
| **UNVERIFIED** | Cannot be known without a live AWS deployment. |

**Nothing in this report was verified against a live AWS account.** The zip contains no deployment output, no CloudWatch logs and no screenshots, so every "works on AWS" claim is UNVERIFIED. I did not have credentials and did not ask for any.

**Checks I ran:** `pytest` (16 passed, local), `cfn-lint 1.57.2` on `template.yaml` (no findings), the SAM transform expanded locally with `aws-sam-translator` (16 resources generated), a secret/credential scan (nothing found), plus the experiments cited as [L] below. I could not run `sam validate` or `sam build` because the SAM CLI is not installed in my sandbox.

---

## A. Executive summary

**What is genuinely there.** A coherent SAM project. The S3 trigger, Lambda orchestration, Textract call, Comprehend call, reverse-offset masking, DynamoDB status tracking, S3 encryption/public-access block/lifecycle, retries, SQS failure destination, CloudWatch alarm and SNS topic are all defined in code or template, and the template lints clean. The happy path is plausibly correct. There are no hard-coded credentials.

**What undermines it.** Several things the README advertises are either not true or are contradicted by the code:

1. **The OCR step can invent text.** If Textract raises `SubscriptionRequiredException`, `text_extractor.py` silently substitutes hard-coded sample text chosen by keywords in the *file name* and the pipeline reports `COMPLETED`. [L] A blank image named `scan_0042.jpg` and random bytes named `holiday_video.png` both came out `COMPLETED` with fabricated "driver licence" content. The same file contains what look like real people's personal details. (C1)
2. **The Management API is public and unauthenticated** (`AuthType: NONE`, CORS `*`) and will hand out upload URLs for arbitrary keys. (C2)
3. **Redaction can leak PII.** [L] The overlap resolver leaves the head or tail of a name or number in clear text. The uploader controls which PII types get masked, and the dashboard's default list leaves passwords, access keys, URLs and similar types unmasked. (C3, C4)
4. **Retries create duplicate records.** [L] One upload that fails three times produces three `FAILED` rows. A transient failure followed by success leaves a stale `FAILED` row beside the `COMPLETED` one. (C5)
5. **The dashboard does not use the deployed API.** It talks to a local Python server that re-implements everything with your own AWS credentials. Four of the dashboard's five routes do not exist on the deployed Lambda. [L] (C7)
6. **The browser-upload route (`/upload-url`) probably fails.** [L] It generates legacy SigV2 presigned URLs, and S3 does not accept SigV2 on buckets created after 24 June 2020. [W] (C8)
7. **Tests mostly test the fallback regexes.** 10 of 16 tests force the Comprehend-unavailable path. The Lambda handler, API, DynamoDB writes and template have no tests.

**Scorecard** (section B, 46 rows, static status): COMPLETE 17, PARTIAL 17, NOT IMPLEMENTED 11, UNVERIFIED 1. Every COMPLETE is "complete in code"; live behaviour is UNVERIFIED for all of them.

**Deployment blockers found statically: none.** The template is valid and the SAM expansion is clean. The real unknowns are account-level (Textract/Comprehend activation, SNS confirmation, Function URL access) and are listed in section D.

**Recommended order of work:** C1 → C2 → C3/C4 → C5 → C7/C8 → everything else. Prompts for Antigravity are at the end of section F.

---

## B. Feature-by-feature implementation matrix

Status is **static** status. Severity is the impact of the gap if it is real.

### 1. S3 upload, validation, file types

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 1.1 | S3 raw bucket triggers Lambda on upload | COMPLETE | [S] `template.yaml` L197-202; [L] SAM expansion shows `NotificationConfiguration` + `lambda:InvokeFunction` permission for `s3.amazonaws.com` | Live trigger UNVERIFIED. No prefix/suffix filter (fine, processed output is a separate bucket). |
| 1.2 | JPG/JPEG/PNG/PDF only | PARTIAL | [S] `pipeline_handler.py` L19, L80-97 | Extension check only. No magic-byte/MIME check, so a renamed file passes validation. |
| 1.3 | Unsupported type marked FAILED with reason | COMPLETE | [S] `pipeline_handler.py` L80-97 | Handled without retry, which is correct. |
| 1.4 | Single-page PDF enforcement | PARTIAL | [S] No page-count check anywhere; `pypdf` is only used in the fallback (`text_extractor.py` L92-104). All 4 sample PDFs are 1 page [L] | Multi-page PDFs rely on Textract rejecting them, which triggers the retry/DLQ path (C10). No multi-page test sample exists. |
| 1.5 | File-size limit | NOT IMPLEMENTED | [S] `file_size_bytes` is recorded (L48, L67-74) but never checked | Oversized files error inside Textract and go to the DLQ. |
| 1.6 | Duplicate-event / retry idempotency | NOT IMPLEMENTED | [S] `pipeline_handler.py` L61 uses a new random ID per invocation; [L] experiments H4/H4b | See C5. |
| 1.7 | Browser upload via presigned URL | PARTIAL | [S] `api_handler.py` L94-115; [L] URL uses SigV2 | Likely rejected by S3 (C8). Not used by the dashboard. No key sanitising, no size cap. |

### 2. Textract, Comprehend, masking

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 2.1 | Textract `DetectDocumentText` by S3 reference | COMPLETE | [S] `text_extractor.py` L19-26; IAM at `template.yaml` L190 | Live behaviour UNVERIFIED. |
| 2.2 | OCR output is genuine | NOT IMPLEMENTED | [S] L30-36, L72-301; [L] H1-H3 | Fabricates text from file names (C1). |
| 2.3 | Comprehend `DetectPiiEntities` with chunking | PARTIAL | [S] `pii_redactor.py` L118-187 | `language_code="en"` hard-coded (`pipeline_handler.py` L145). A single line >80,000 bytes is sent as one request over the 100 KB limit [L] F. Silent regex fallback (C6). |
| 2.4 | Masking with `[TYPE]` labels, right-to-left | PARTIAL | [S] L96-109; tests L28-64 pass [L] | Simple cases are correct. Overlapping spans leak PII (C3). |
| 2.5 | Confidence threshold | COMPLETE | [S] L58-61 (0.5, not configurable) | `Score < 0.5` entities are left in clear text [L] E. |
| 2.6 | Selective masking, custom keywords, ASTERISK mode | PARTIAL | [S] `pipeline_handler.py` L123-137, `pii_redactor.py` L44-55, L71-85, L103-104 | Controlled by uploader-supplied S3 metadata (C4). ASTERISK keeps exact length [L] D. |
| 2.7 | Coverage of Comprehend PII types | PARTIAL | [S] type map at `pii_redactor.py` L66-69 | `IN_AADHAAR` is a real Comprehend type [W] but is not in the `ID_NUMBER` map. `DATE_TIME` is relabelled `DOB` for every date [L] C. |

### 3. DynamoDB

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 3.1 | `PROCESSING` record on arrival | COMPLETE | [S] `dynamodb_manager.py` L30-65 | No conditional write. |
| 3.2 | `COMPLETED` / `FAILED` transitions with reason and step | COMPLETE | [S] L67-143 | No `ConditionExpression`, so a late write can overwrite a terminal state. |
| 3.3 | Recovery of stuck `PROCESSING` records | NOT IMPLEMENTED | [S] only `except` handler at `pipeline_handler.py` L220-239 | Timeout (120 s, `template.yaml` L164) or out-of-memory never reaches the handler, so the row stays `PROCESSING` forever. |
| 3.4 | Audit metadata | PARTIAL | [S] timestamps, duration, counts, S3 keys stored | No retry count, Lambda request ID, detector/OCR provenance or stage timeline. |
| 3.5 | "No PII persisted in DynamoDB/metadata" (README L177) | PARTIAL | [S] text is never stored, only counts | File names and S3 keys are stored and embedded in IDs (C14). |

### 4. Buckets, encryption, lifecycle

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 4.1 | Raw + processed buckets, SSE-S3 | COMPLETE | [S] `template.yaml` L35-67 | |
| 4.2 | Public access blocked | COMPLETE | [S] L43-47, L68-72 | |
| 4.3 | Raw lifecycle expiry | COMPLETE | [S] L54-58, default 1 day | Day-granular, so objects can persist up to ~2 days. No `AbortIncompleteMultipartUpload` rule. |
| 4.4 | Processed-bucket retention | NOT IMPLEMENTED | [S] none defined | Grows forever and blocks `sam delete` (C20). |
| 4.5 | Restricted CORS | NOT IMPLEMENTED | [S] `AllowedOrigins: ["*"]`, raw allows PUT/POST (L48-53, L73-78) | See C17. |
| 4.6 | TLS-only bucket policy | NOT IMPLEMENTED | [S] none | |

### 5. Retries, DLQ, alarm, SNS

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 5.1 | 2 async retries | COMPLETE | [S] `template.yaml` L165-166; [L] expansion shows `MaximumRetryAttempts: 2` | Live UNVERIFIED. |
| 5.2 | Failure destination to SQS | COMPLETE | [S] L167-170; [L] role gets `sqs:SendMessage` | This is an on-failure *destination*, not a classic DLQ. Only receives events after all retries fail. |
| 5.3 | Handler re-raises unexpected errors | COMPLETE | [S] `pipeline_handler.py` L220-239 | Also re-raises *bad-document* errors, which should not alert (C10). |
| 5.4 | CloudWatch alarm on DLQ | PARTIAL | [S] L134-151 | Fires once, never resets while messages sit in the queue (C11). |
| 5.5 | SNS topic + email subscription | PARTIAL | [S] L120-132; `AlertEmail` defaults to `""`; `samconfig.toml` L22 does not pass it | A plain `sam deploy` creates **no** subscriber. Email needs a confirmation click. |
| 5.6 | Email actually delivered | UNVERIFIED | | Needs live test (section G). |
| 5.7 | Failure-test script | PARTIAL | [S] `scripts/simulate_failure.py` L97-99 | Always prints "complete" even if nothing arrived (C23). |

### 6. Management API and dashboard

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 6.1 | Function URL API deployed | COMPLETE | [S] `template.yaml` L207-242, L279-281; [L] expansion creates `AWS::Lambda::Url` plus both public-invoke permissions (`lambda:InvokeFunctionUrl` and `lambda:InvokeFunction` with `InvokedViaFunctionUrl: true`) | Live UNVERIFIED. |
| 6.2 | Authentication | NOT IMPLEMENTED | [S] `AuthType: NONE` L217; no auth code in `api_handler.py` | C2. |
| 6.3 | CORS | PARTIAL | [S] wildcard in template L218-221 **and** in handler L30-35 | Wildcard, duplicated; browser behaviour with doubled headers UNVERIFIED. |
| 6.4 | Dashboard calls the deployed API | NOT IMPLEMENTED | [S] `app.js` L2, L120, L180, L222, L411; [L] H5 | C7. |
| 6.5 | List/stats correctness | PARTIAL | [S] `api_handler.py` L57, L65; `dashboard_server.py` L55 | Wrong beyond 50/80 rows (C12). |
| 6.6 | Dashboard upload | PARTIAL | [S] `app.js` L165-203; `dashboard_server.py` L214-261 | Works only through the local server; error-handling bug at `app.js` L192; path traversal (C9). |
| 6.7 | Live polling of result | PARTIAL | [S] `app.js` L110-162 | Compares client clock to server timestamps; after 4 s falls back to matching by file name, so it can show an older record with the same name (C21). |
| 6.8 | Redacted text, PII breakdown, failure reason + step | COMPLETE | [S] `app.js` L367-435 | Rendering escapes HTML correctly for text content. |
| 6.9 | Dashboard hostable without your laptop | NOT IMPLEMENTED | [S] needs `dashboard_server.py` + local credentials | |
| 6.10 | Planned dashboard features: status dropdown/filter, document selector, "Processing" and "Total PII" cards, stage timeline, DLQ count display | NOT IMPLEMENTED | [S] `index.html` L47-63 shows only Total/Completed/Failed; `/stats` returns `dlq_message_count` but the UI never shows it | These were in the feature list you described earlier. |

### 7. IAM, security, cost

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 7.1 | Lambda execution roles | COMPLETE | [S] SAM `Policies`; [L] expansion | |
| 7.2 | Least privilege | PARTIAL | [L] expanded role policies | Broader than needed (C16). |
| 7.3 | No hard-coded credentials | COMPLETE | [L] scan for key patterns, `.env`, `.aws`, `.git`, `.pem`: none | Real-looking personal data is hard-coded instead (section E). |
| 7.4 | PII-safe logging | NOT IMPLEMENTED | [S] `pipeline_handler.py` L34, L224-228; `api_handler.py` L43 | C15. |
| 7.5 | Cost guards | NOT IMPLEMENTED | [S] no reserved concurrency, size cap, log retention, or budget resource | C18. |

### 8. Tests

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 8.1 | Unit tests exist and pass | COMPLETE | [L] 16 passed in 0.23 s | |
| 8.2 | Tests exercise the handler, API, DynamoDB writes, template | NOT IMPLEMENTED | [S] none import `pipeline_handler` or `api_handler` | 10 of 16 tests force `SubscriptionRequiredException` and test the regex fallback only. |
| 8.3 | Integration tests (moto or live) | NOT IMPLEMENTED | [S] none | |

### 9-10. Configuration and consistency

| # | Feature | Status | Evidence | Gap |
|---|---|---|---|---|
| 9.1 | Parameters, env vars, outputs | PARTIAL | [S] `Environment` param is declared (L10-14) but never referenced; `DLQ_URL` set on the processor (L176) but never read | |
| 9.2 | `deploy.ps1` | PARTIAL | [S] | No credential check, no `sam validate`, hard-codes `Environment=dev` (L27). |
| 10.1 | README/docs match code | PARTIAL | [S] see C24 | |

---

## C. Critical bugs and risks, ranked

Fix IDs (F1, F2...) point to section F.

### CRITICAL

**C1. The OCR step fabricates text from file names.**
- **Where:** `src/text_extractor.py` L30-36 (trigger), L129-137 (`"blank"` in name returns empty), L139-291 (canned documents by keyword; L201 matches the substring `"id"`), L291 (generic "Uploaded content parsed"), L295-301 (reports 99% confidence).
- **Evidence [L]:** with Textract mocked to raise `SubscriptionRequiredException`: a blank PNG named `scan_0042.jpg` → `COMPLETED`, output `Document: scan_0042.jpg [NAME]`; random bytes named `holiday_video.png` → `COMPLETED` with a full invented driver-licence record (`"video"` contains `"id"`); random bytes named `my_invoice_photo.jpg` → `COMPLETED` with an invented income certificate.
- **Impact:** Every result produced on an account where Textract is not activated is fake but looks real. The only marker (`fallback_mode`) goes into the S3 audit JSON, never into DynamoDB or the dashboard. The canned text also contains what look like real individuals' names, phone and Aadhaar-format numbers and a home address. This is a review-integrity risk as well as a privacy one.
- **Fix:** F1.

**C2. The Management API is public and unauthenticated.**
- **Where:** `template.yaml` L216-221 (`AuthType: NONE`, CORS `*`); `api_handler.py` L55-61, L95-115, L117-137.
- **Evidence:** [S] no auth check anywhere; [L] SAM expansion creates a `Principal: "*"` invoke permission. `/documents` lists file names, raw keys and PII counts; `/document/{id}` returns the redacted text; `/upload-url` returns a signed PUT for **any key the caller names** (`../../etc/x.txt` accepted [L]).
- **Impact:** Anyone who learns the URL can read results, and can push files into the raw bucket, which triggers Lambda, Textract and Comprehend at your cost. A Function URL is not secret.
- **Fix:** F2.

**C3. The overlap resolver leaves PII visible.**
- **Where:** `src/pii_redactor.py` L427-447.
- **Evidence [L]:** `"Reach John Alexander Smith ..."` with spans 6-20 and 11-26 → `Reach John [NAME] ...` (head leaks). Spans 6-20 and 15-26 → `Reach [NAME] Smith ...` (tail leaks). A card number with spans 5-15 and 10-24 → `Card 4111 [ID_NUMBER] ...`. The existing overlap test (`tests/test_pipeline_unit.py` L66-82) only covers two spans with the *same start*.
- **Impact:** Partially overlapping detections are normal with Comprehend and very common with the regex fallback, so fragments of names and numbers survive redaction. For a PII project this is the worst correctness bug.
- **Fix:** F3 (merge overlapping spans instead of picking one).

**C4. The uploader decides what gets masked, and the dashboard default leaves types unmasked.**
- **Where:** `pipeline_handler.py` L123-137 reads `mask-types`, `custom-keywords`, `mask-mode` from S3 object metadata; `pii_redactor.py` L71-85 turns `mask-types` into an allow-list; `dashboard/index.html` L126-156 ticks six types by default; `dashboard_server.py` L237-243 sends them.
- **Evidence [L]:** `allowed_types=["NOTHING"]` → output identical to the input, status `COMPLETED`, 0 PII. With the dashboard's default six types, a `PASSWORD` and an `AWS_ACCESS_KEY` entity were left in clear text while `EMAIL` was masked. The CLI scripts send no metadata, so the CLI masks everything and the dashboard does not: different behaviour from the same pipeline.
- **Impact:** The "PII-safe" guarantee depends on a value the uploader supplies. Comprehend returns many types the six checkboxes never cover (password, URL, age, username, IP address, bank/card types and others), and `IN_AADHAAR` is not in the `ID_NUMBER` map.
- **Fix:** F4.

### HIGH

**C5. Retries and duplicate events create duplicate records.**
- **Where:** `pipeline_handler.py` L61 (`doc_{uuid4}_{file_name}`), L66-78; `dynamodb_manager.py` L59-60 (`put_item` with no condition).
- **Evidence [L]:** one upload failing 3 times → 3 `FAILED` rows (`doc_25583c54_resume.png`, `doc_7e263e00_...`, `doc_d9e618f8_...`). A throttle then success → one stale `FAILED` + one `COMPLETED`. `scripts/simulate_failure.py` leaves three junk `FAILED` rows each run.
- **Impact:** Failed/total counts on the dashboard are inflated, successful redactions look like failures, and S3 gets duplicate processed objects.
- **Fix:** F5.

**C6. Comprehend failure silently switches to a regex detector that is tuned to the developer's own samples.**
- **Where:** `pii_redactor.py` L176-187 (switch), L189-390 (regex), L362 (hard-coded place names `OOTHUMEDU`, `RAMANAYACKENPALAYAM`, `Pethanaickenpalayam`, `Attur`), L355 (bare `OR|WA|CA` treated as address context).
- **Evidence:** [L] the regex missed mixed-case `Oothumedu, Ramanayackenpalayam` in the invented income certificate (pattern L362 is case-sensitive). [S] 10 of the 16 tests (`tests/test_pipeline_unit.py` L110, 125, 143, 167, 187, 201, 230, 271, 318, 356) test only this fallback, using the same personal details the regex was tuned against.
- **Impact:** Output is labelled as Comprehend-based while some runs used regex, and nothing records which detector ran. Passing tests say little about real documents.
- **Fix:** F1, F6.

**C7. The dashboard bypasses the deployed API; the two backends have diverged.**
- **Where:** `dashboard/app.js` L2 (`API_BASE = window.location.origin`), L120, L180, L222, L411; `api_handler.py` L55, L64, L95, L118; `scripts/dashboard_server.py` implements the dashboard's routes locally.
- **Evidence [L]:** against `api_handler`: `POST /upload-direct` → 404; `GET /document?id=` → 404; `GET /documents` returns only `documents` (no `stats`, which `app.js` L124/L298 expects). `fetchStats()` (`app.js` L280) is never called.
- **Impact:** The deployed ManagementFunction (and its public URL) does nothing for the dashboard, and the dashboard works only on a machine that holds your AWS credentials. README L7/L70 describes the API as the dashboard's backend.
- **Fix:** F7.

**C8. Presigned upload URLs use the retired SigV2 scheme.**
- **Where:** `api_handler.py` L101-109 (`boto3.client("s3")` with defaults).
- **Evidence [L]:** the generated URL carries `AWSAccessKeyId`, `Signature`, `Expires` (SigV2), not `X-Amz-Algorithm`. [W] AWS states new buckets created after 24 June 2020 do not accept SigV2 requests, and the failure message is "authorization mechanism ... not supported. Please use AWS4-HMAC-SHA256". Your buckets are brand new.
- **Impact:** `/upload-url` very likely returns URLs S3 will reject. UNVERIFIED live. Also, metadata is not signed, so `mask-*` options could not be sent this way, and there is no size cap.
- **Fix:** F8.

**C9. Local dashboard server: path traversal plus wildcard CORS.**
- **Where:** `dashboard_server.py` L269-271 (`os.path.join(SAMPLE_DIR, file_name)` on a query parameter), L77-79 and L85-87 (CORS `*`), L119-125 (POST with no body or custom header, so no CORS pre-flight).
- **Evidence [L]:** `file=../README.md` and `file=../src/pii_redactor.py` resolve outside `sample_docs` and pass the `exists` check.
- **Impact:** The server binds to 127.0.0.1, but any web page open in your browser can POST to it and make it upload local files to the raw bucket, and the `*` CORS header lets that page read replies. Files without a supported extension are uploaded and fail, but remain in the raw bucket until expiry.
- **Fix:** F9.

**C10. Bad documents are treated as operational crashes.**
- **Where:** `pipeline_handler.py` L99-105 and L220-239; `text_extractor.py` L27-38 (only `SubscriptionRequiredException` is special-cased).
- **Evidence:** [S] a multi-page PDF, corrupt image, zero-byte file or oversized file makes Textract raise a `ClientError`; the handler re-raises; Lambda retries twice; the event lands in the queue and the alarm emails you. Textract's synchronous API accepts only single-page PDFs/TIFFs [W: verify against current Textract limits].
- **Impact:** An ordinary user mistake produces 3 `FAILED` rows, a DLQ message and an alert, instead of one clean `FAILED / VALIDATION` row.
- **Fix:** F10.

**C11. The alarm alerts on the first failure only.**
- **Where:** `template.yaml` L134-151 (`ApproximateNumberOfMessagesVisible >= 1`, no `OKActions`); nothing consumes or purges the queue (14-day retention, L117).
- **Impact:** Once one message sits in the queue the alarm stays in `ALARM`. Later failures send no new email until the queue is drained. Also, `ReceiveMessage` on the queue (granted at L240-241) would hide messages and could clear the alarm incorrectly.
- **Fix:** F11.

**C12. List and stats are wrong once the table grows.**
- **Where:** `api_handler.py` L57 (`scan(Limit=50)`), L65 (`scan()` unpaginated); `dashboard_server.py` L55 (`scan(Limit=80)`).
- **Evidence:** [S] `Limit` on a scan caps items *evaluated* in key order, so it returns an arbitrary 50/80 rows, not the newest, and the code then sorts only that subset. The `StatusIndex` GSI (`template.yaml` L98-106) is never used.
- **Impact:** After ~80 documents the "Total" card stops growing and a fresh upload may never appear, so the dashboard's wait loop times out.
- **Fix:** F12.

### MEDIUM

**C13. Stuck `PROCESSING` rows.** `pipeline_handler.py` L220-239 handles only exceptions; a Lambda timeout (120 s) or memory kill leaves the row `PROCESSING`. Fix: F17.

**C14. File names leak into IDs, keys and logs.** `pipeline_handler.py` L61 (ID), L155-158 (processed key), L176-179 (audit JSON), L34 (logged event); `dynamodb_manager.py` L46-48. Names like `Priya_Raman_resume.pdf` become PII in places the README says contain none (README L177). Fix: F13.

**C15. Logs and error messages expose detail.** `pipeline_handler.py` L34 logs the full S3 event and L224-232 logs and stores `str(e)`; `api_handler.py` L43 logs full requests and L142-143 returns `str(e)` to the caller (can include table names and ARNs). No log-retention setting exists (no `RetentionInDays` in the template), so logs persist indefinitely. Fix: F14.

**C16. IAM is broader than needed.** [L] Expanded roles: processor gets `S3CrudPolicy` on processed (includes `PutObjectAcl`, `PutLifecycleConfiguration`) and `DynamoDBCrudPolicy` (`DeleteItem`, `BatchWriteItem`, `Scan`...). The (public) API role gets read on the **raw** bucket, `PutLifecycleConfiguration`/`PutObjectAcl` on raw, full DynamoDB CRUD, and an unused `sqs:ReceiveMessage`. `comprehend:ContainsPiiEntities` is unused. Textract/Comprehend `Resource: "*"` is unavoidable. Fix: F15.

**C17. Bucket CORS and transport.** `template.yaml` L48-53 allows `PUT, POST` from any origin on the raw bucket; L73-78 `GET` from anywhere on processed; no `aws:SecureTransport` deny policy. Fix: F16.

**C18. No cost guard-rails.** No file-size check, no `ReservedConcurrentExecutions`, no log retention, no budget, and an open upload route (C2). Textract and Comprehend bill per page/unit. Fix: F2, F10, F14.

**C19. Comprehend behaviour assumptions.**
- Language is fixed to `en` (`pipeline_handler.py` L145) while the samples and fallback handle Tamil. [W] The current API reference lists `en, es, fr, de, it, pt, ar, hi, ja, ko, zh, zh-TW`; older pages still say English or Spanish only; Tamil is not listed.
- A single line over 80,000 bytes is sent whole and exceeds the 100 KB limit [L] F; [W] the limit is 100 KB.
- Offsets are zero-based character offsets [W]; behaviour on text with combining/Indic characters is untested.
- `DATE_TIME` → `DOB` (L68-69) labels invoice dates and validity dates as dates of birth [L] C.
- ASTERISK mode (L103-104) keeps the exact length of the hidden value [L] D.
- Fix: F4, F6.

**C20. Teardown and retention.** Fixed bucket names with no `DeletionPolicy` (`template.yaml` L38, L63); the processed bucket has no lifecycle, so it fills up and a later `sam delete` fails on the non-empty bucket. DynamoDB PITR (L107-108) adds cost and no TTL is set. Fix: F16.

### LOW

**C21. Dashboard UI bugs.** `app.js` L192: a `throw` inside the `FileReader.onload` callback is not caught by the surrounding `try`, so the spinner never clears and `isUploading` stays `true`, which pauses background polling (L52-56). L127-130: falls back to matching by file name only. L114/L127: compares browser time to server timestamps. `index.html` L200 accepts `.txt` while L210 advertises "PDF, PNG, JPG". L35 hard-codes "AWS us-east-1".

**C22. Hard-coded stack/region.** `dashboard_server.py` L14-15, script defaults, `samconfig.toml` L20-21, `deploy.ps1` L3-4. Deploying with a different `-StackName`/`-Region` breaks the dashboard and scripts.

**C23. `simulate_failure.py` cannot fail.** L65/L78 track `message_arrived` but L97-99 prints "Failure handling verification complete!" regardless; the alarm is checked once, immediately (L82-95), before it has had time to evaluate; SNS delivery is not checked. With the fallback active, its payload (`corrupt_document_crash_test.png`, nonexistent bucket) matches no keyword and would take the generic fabricated-text branch (L291), so it may *succeed* instead of failing (reasoned from the code; not run).

**C24. Documentation mismatches.**
- README L175/L177 ("least privilege", "never persist unredacted PII") overstate; see C14, C16.
- README L138, `upload_and_monitor.py` L128/L152: "5 sample documents"; the folder has 11 files.
- `docs/ARCHITECTURE.md` L98 quotes a failure message different from `pipeline_handler.py` L82-85; its Step 7 says the "wider, higher-confidence span" wins, but the resolver is greedy by start offset and length (see C3).
- `docs/AWS_VS_GCP_COMPARISON.md` names Cloud Vision for GCP OCR; your Review 1 plan used Document AI. Its free-tier/pricing figures (L81-85) are unsourced and the AWS free-tier model changed in 2025; remove or link to official pricing pages.
- README L5 links a `LICENSE` file that does not exist.
- README L71 says `requirements.txt` has boto3/botocore only; it also lists `pypdf`.

**C25. Dead code.** `DynamoDBManager.get_document_record` (never called); `DLQ_URL` env var on the processor; `ContainsPiiEntities` permission; `StatusIndex` GSI; the `.txt` branch in the fallback (unreachable because `.txt` is rejected at validation); the `Environment` parameter.

---

## D. AWS credentials and configuration audit

**Credentials in the project: none found.** [L] I scanned all text files for access-key patterns, `aws_secret_access_key`, private keys and `password=`; I also checked the zip for `.env`, `.aws`, `.git`, `.pem` and `__pycache__`. Nothing matched, and no 12-digit account ID is hard-coded (the template uses `AWS::AccountId`). You must never put access keys in the source, the zip or the dashboard, and I am not asking for any.

| Item | Configured in project? | Where | What you must supply / where to get it |
|---|---|---|---|
| Local AWS authentication | No (by design) | n/a | Use `aws configure sso` (IAM Identity Center) or a named profile in `aws configure --profile name`; credentials come from the AWS console (IAM Identity Center or your IAM user's security-credentials page). Set `$env:AWS_PROFILE = "name"` because the scripts have no `--profile` option. |
| Deployer permissions | No | n/a | In a personal sandbox account, an admin/PowerUser-plus-IAM role is simplest. SAM needs to create IAM roles, S3, DynamoDB, SQS, SNS, CloudWatch, Lambda, and a CloudFormation stack. |
| Region | Yes: `us-east-1` | `samconfig.toml` L20; `deploy.ps1` L4; scripts; `dashboard_server.py` L15; `index.html` L35 | Keep `us-east-1` everywhere or change all of them. Confirm Textract and Comprehend are available in the region you choose. |
| Stack name | Yes: `pii-safe-pipeline` | `samconfig.toml` L7/L21; `deploy.ps1` L3; scripts; `dashboard_server.py` L14 | Alarm name `pii-safe-pipeline-DLQ-Messages-Alarm` is derived from it (`simulate_failure.py` L110). |
| `Environment` parameter | Yes (`dev`) | `template.yaml` L10-14 | Unused by any resource. |
| `AlertEmail` parameter | **Not passed by default** | default `""` (L16-19); `samconfig.toml` L22 omits it | Supply via `.\deploy.ps1 -AlertEmail you@example.com`. Then **click the confirmation link** AWS emails you. Until then the subscription is `PendingConfirmation`. |
| `RawFileRetentionDays` | Yes (`1`) | `samconfig.toml` L22 | Min 1 (template L23). |
| Lambda env vars (processor) | Yes | `template.yaml` L172-177: `RAW_BUCKET_NAME`, `PROCESSED_BUCKET_NAME`, `TABLE_NAME`, `DLQ_URL`, `LOG_LEVEL` | Set by the stack; nothing to enter. `DLQ_URL` is unused. |
| Lambda env vars (API) | Yes | L223-227 | Set by the stack. No `LOG_LEVEL` (handler hard-codes INFO). |
| Lambda execution roles | Yes (SAM-generated) | `Policies` L178-196, L228-242 | Permission list expanded in C16. |
| Textract / Comprehend activation | **Unknown** | n/a | The fallback code exists because `SubscriptionRequiredException` was apparently hit. That error normally points to an account/service-activation issue. Check in the console (Textract and Comprehend in the target region) and the Billing/Account pages; complete account verification or plan upgrade if prompted. Test with the two CLI commands in section G.1 before deploying. |
| SNS email confirmation | Not applicable until deployed | n/a | See `AlertEmail`. |
| Function URL access | Created by the stack | `template.yaml` L216-221, L279-281 | UNVERIFIED live. The URL appears in the `ManagementApiUrl` output. Nothing in the dashboard uses it (C7). |
| Dashboard server needs | n/a | `dashboard_server.py` | Your profile needs: `cloudformation:DescribeStacks`, `dynamodb:Scan` and `GetItem` on the table, `s3:PutObject` on the raw bucket, `s3:GetObject` on the processed bucket. |
| Scripts need | n/a | `scripts/*.py` | `lambda:InvokeFunction`, `sqs:GetQueueAttributes`, `cloudwatch:DescribeAlarms`, `s3:PutObject`, `dynamodb:Scan`, `cloudformation:DescribeStacks`. |
| Local tooling | No | n/a | AWS CLI v2, SAM CLI, Python 3.12 (3.10+ per README L99), `pip install pytest moto`. |
| `.gitignore` | **Missing** | n/a | Add one covering `.aws/`, `.env`, `__pycache__/`, `.aws-sam/`. |

---

## E. Security vulnerabilities and unnecessary files

**Vulnerabilities** (details in section C):
1. C2: public unauthenticated API; arbitrary-key upload URLs; wildcard CORS.
2. C3, C4: redaction leaks PII; masking policy controlled by the uploader.
3. C9: path traversal and CORS `*` on the local dashboard server; no request-size limit (`dashboard_server.py` L116-117, L229).
4. C14, C15: PII in file names, keys, logs and error messages; logs never expire.
5. C16, C17: over-broad IAM and bucket CORS; no TLS-only policy.
6. C1: fabricated results presented as real.
7. **Real-looking personal data committed to the repo.** `text_extractor.py` (20 matching lines), `tests/test_pipeline_unit.py` (28), `scripts/generate_sample_docs.py` (7), `pii_redactor.py` (1), and the generated sample documents contain what appear to be real people's names, phone numbers, Aadhaar-format numbers and a home address. Replace all of it with obviously synthetic data and treat any copy already shared as exposed.

**Unnecessary or removable files and code**

| File / code | Why |
|---|---|
| `AI_REVIEW_PROMPTS.md` | Internal working note; should not ship in a deliverable. |
| `sample_docs/sample_document.png` | Byte-identical to `sample_id_card.png` [L] (same checksum). |
| `text_extractor._fallback_text_extraction` (L72-301) | Fabricates output (C1). Delete. |
| `pii_redactor._fallback_regex_detect_pii` (L189-390) | Tuned to one person's documents; leaks (C6). Delete or move to test-only utilities. |
| `docs/AWS_VS_GCP_COMPARISON.md` pricing table | Unsourced, stale. Fix or remove. |
| `src/requirements.txt` `boto3`/`botocore` | The Lambda runtime provides them; pinning them enlarges both functions (both use `CodeUri: ./src`). Keep `pypdf` only if page counting is added (F10). |
| `samconfig.toml` `sync` / `local_*` sections | Unused. |
| `DynamoDBManager.get_document_record`, `DLQ_URL` on processor, `StatusIndex` (unless F12 uses it), `Environment` parameter, `comprehend:ContainsPiiEntities` | Unused. |
| `raw_dynamodb_record` field in `dashboard_server.py` L209 | Duplicate of `document`. |

---

## F. Exact fixes and tests

Do these in order. Each fix lists the tests that must pass before it counts as done. Use `moto` (`from moto import mock_aws`) for S3/DynamoDB and `MagicMock` for Textract/Comprehend.

### F1. Remove all fabricated output (C1, C6)
- Delete `_fallback_text_extraction` and `_fallback_regex_detect_pii`. Replace both `SubscriptionRequiredException` branches with a raise:
```python
if error_code == "SubscriptionRequiredException":
    raise RuntimeError("Textract is not activated for this account/region") from e
```
- Record provenance on every record and in the audit JSON: `ocr_engine: "textract"`, `pii_engine: "comprehend"`.
- Move any regex logic you still want to a clearly named test helper that cannot be reached from the Lambda handler.
- **Tests:** (a) Textract mock raising `SubscriptionRequiredException` → handler raises, DynamoDB row `FAILED/EXECUTION_ERROR`, processed bucket empty. (b) Same for Comprehend. (c) A blank PNG with any file name never reaches `COMPLETED`. (d) Assert no hard-coded personal data remains: `grep` the repo for the old names/numbers in CI.

### F2. Lock down the Management API (C2, C18)
Decide first whether you need it at all.
- **If the dashboard stays local** (simplest, and what `dashboard_server.py` already does): delete `ManagementFunction`, `api_handler.py` and the `ManagementApiUrl` output, or set `FunctionUrlConfig: AuthType: AWS_IAM` so only signed requests work.
- **If you want a hosted dashboard:** put an API Gateway HTTP API with a Cognito or Lambda authorizer in front of it, restrict CORS to the dashboard origin, serve the static files from S3 + CloudFront, and validate every input.
- Either way: allow-list extensions, never accept a client-chosen key, return a generic error plus a request ID, and never return `str(e)`.
- **Tests:** unauthenticated `curl.exe -i "$api/documents"` returns 401/403; a unit test that a missing credential returns 401; `/upload-url` rejects `../x`, `.exe`, and oversize requests.

### F3. Merge overlapping spans (C3)
Replace `_resolve_overlapping_spans` with a union merge:
```python
@staticmethod
def _resolve_overlapping_spans(entities):
    spans = sorted(entities, key=lambda e: (e["BeginOffset"], -e["EndOffset"]))
    merged = []
    for e in spans:
        if merged and e["BeginOffset"] < merged[-1]["EndOffset"]:
            last = merged[-1]
            if e["EndOffset"] > last["EndOffset"]:
                last["EndOffset"] = e["EndOffset"]
            if PRIORITY.get(e["Type"], 0) > PRIORITY.get(last["Type"], 0):
                last["Type"] = e["Type"]
        else:
            merged.append(dict(e))
    return merged
```
- **Tests (must fail today, pass after):** spans (6,20)+(11,26) on `"Reach John Alexander Smith ..."` → neither `John` nor `Smith` survives; spans (6,20)+(15,26) likewise; card spans (5,15)+(10,24) → no digit fragment survives; nested spans; adjacent spans; identical spans; property test with random span pairs asserting every character covered by any input span is hidden.

### F4. Make masking default-deny and server-controlled (C4, C19)
- Remove `mask-types` from the handler (`pipeline_handler.py` L123-137). Always mask every entity type Comprehend returns above the threshold.
- Keep `custom-keywords` as **additive only**. Drop ASTERISK mode, or output a fixed-length mask.
- Map `IN_AADHAAR` (and the other Comprehend ID types) to `ID_NUMBER` or keep their own label; stop relabelling `DATE_TIME` as `DOB` (use `DATE`, or `DOB` only when a "birth" keyword precedes it).
- If you need "leave these types visible", make it an admin setting (stack parameter or SSM), never an upload header.
- Remove the masking checkboxes from `index.html`, or turn them into "extra things to mask".
- **Tests:** `allowed_types` can no longer reduce masking; PASSWORD, AWS keys, URLs, IP addresses and `IN_AADHAAR` entities are masked; `DATE_TIME` is not labelled DOB.

### F5. Idempotent processing (C5)
```python
import hashlib
seq = record["s3"]["object"].get("sequencer") or record["s3"]["object"].get("eTag", "")
document_id = hashlib.sha256(f"{bucket}/{object_key}/{seq}".encode()).hexdigest()[:24]
# put_item(..., ConditionExpression="attribute_not_exists(document_id)")
# on ConditionalCheckFailedException: get_item; if status in (COMPLETED, FAILED) -> skip;
# if PROCESSING -> continue and ADD attempts :1
# mark_completed / mark_failed: ConditionExpression="#st = :processing"
```
- **Tests:** same S3 event twice → exactly one row; transient Textract failure then success → one row, `COMPLETED`, `attempts=2`; three failures → one `FAILED` row; `COMPLETED` is never overwritten by a late `FAILED`.

### F6. Test the real detector path (C6, C19)
- Add tests for: confidence threshold boundary, chunk boundaries (text just over 80,000 bytes), a single line over 100 KB (split it), Unicode offsets (Tamil/Devanagari text, emoji), `IN_AADHAAR` mapping.
- Remove or relabel the 10 fallback-only tests once F1 deletes the fallback.

### F7. One backend for the dashboard (C7)
- Choose one: (a) local server only (delete the Lambda API), or (b) make `api_handler.py` serve exactly the routes `app.js` calls and set `API_BASE` from a small `config.js`.
- Route contract to implement for (b): `GET /documents?status=` → `{documents, stats}`; `GET /document?id=` (URL-decode the id); `POST /upload-url` (F8). Return `stats.total`, `completed`, `failed`, `processing`, `total_pii_redacted`, `dlq_message_count`.
- Call `fetchStats` or delete it; show Processing and DLQ cards.
- **Tests:** a route-contract test that parses every `fetch(` in `app.js` and asserts a handler exists for each; an API test with a file name containing spaces and `()`.

### F8. Fix presigned uploads (C8)
```python
from botocore.config import Config
s3 = boto3.client("s3", region_name=os.environ["AWS_REGION"],
                  config=Config(signature_version="s3v4"))
post = s3.generate_presigned_post(
    Bucket=raw_bucket, Key=f"uploads/{uuid.uuid4().hex}{ext}",
    Conditions=[["content-length-range", 1, 10_000_000], {"Content-Type": ctype}],
    ExpiresIn=300)
```
- Allow-list `ext`; generate the key server-side.
- **Tests:** [L] URL/fields contain `AWS4-HMAC-SHA256`; live: upload with `curl.exe` succeeds, and an 11 MB file is rejected.

### F9. Harden the local server (C9)
- Use `os.path.basename(file_name)` and require membership in `os.listdir(SAMPLE_DIR)`.
- Reject requests whose `Origin`/`Host` is not `localhost:8000` or `127.0.0.1:8000`; remove `Access-Control-Allow-Origin: *` (the dashboard is same-origin).
- Cap `Content-Length` (for example 10 MB); catch JSON/base64 errors and return 400; switch to `ThreadingHTTPServer`; read `STACK_NAME`/`REGION` from env or arguments.
- **Tests:** `POST /upload-sample?file=../README.md` → 400; oversize body → 413; foreign `Origin` → 403.

### F10. Validate before calling Textract, and classify errors (C10)
- Check extension **and** magic bytes (`\xff\xd8`, `\x89PNG`, `%PDF`); check size (say ≤ 5 MB); for PDFs count pages with `pypdf` (first read from S3) and reject more than 1. Each rejection is `FAILED/VALIDATION` with no re-raise.
- Treat `UnsupportedDocumentException`, `BadDocumentException`, `DocumentTooLargeException` and `InvalidParameterException` as `FAILED/TEXT_EXTRACTION` (no re-raise). Re-raise only throttling and 5xx errors.
- **Tests:** multi-page PDF, corrupt PNG, zero-byte file and 15 MB file → one `FAILED` row, queue stays empty; throttling error → retried, queue receives the event only after retries.

### F11. Alarm that fires for every failure (C11)
```yaml
DLQAlarm:
  Properties:
    MetricName: NumberOfMessagesSent
    Namespace: AWS/SQS
    Statistic: Sum
    Period: 60
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    TreatMissingData: notBreaching
    AlarmActions: [!Ref FailureNotificationTopic]
    OKActions:    [!Ref FailureNotificationTopic]
```
- Add a short runbook: inspect then purge/redrive the queue.
- **Tests:** two failures five minutes apart produce two emails; `aws cloudwatch describe-alarm-history` shows `ALARM → OK → ALARM`.

### F12. Correct list and stats (C12)
- List: `Query` `StatusIndex` per status (or add a GSI with a constant partition key and `upload_timestamp` as sort key) with `ScanIndexForward=False`, `Limit=50`, and a pagination token.
- Stats: paginated `Query` with `Select="COUNT"` per status, or maintain counters.
- **Tests:** seed 200 rows; the newest upload always appears; totals equal 200.

### F13. Opaque IDs and names (C14)
- `document_id` from F5; processed key `redacted/{document_id}.txt`; metadata key likewise.
- Store the original file name only if policy allows it; otherwise store a hash, and drop it from the audit JSON and logs.
- **Tests:** upload `Priya_Raman_resume.pdf`; assert the name appears in no DynamoDB attribute, no processed-bucket key and no captured log line.

### F14. Logging and log retention (C15, C18)
- Log `document_id`, step and duration only; no raw events, no `str(e)` (log the exception class and a truncated, scrubbed message).
- Add log groups with `RetentionInDays: 14` for both functions (`AWS::Logs::LogGroup` named `/aws/lambda/<FunctionName>`).
- **Tests:** a `caplog` test asserts no S3 key or file name appears; `aws logs describe-log-groups` shows retention.

### F15. Least-privilege IAM (C16)
Replace the SAM policy templates with explicit statements:
- Processor: `s3:GetObject` on `RawBucket/*`; `s3:PutObject` on `ProcessedBucket/*`; `dynamodb:PutItem`, `UpdateItem`, `GetItem` on the table; `textract:DetectDocumentText` and `comprehend:DetectPiiEntities` on `*`.
- API (if kept): `dynamodb:GetItem`, `Query` on the table and index; `s3:GetObject` on `ProcessedBucket/redacted/*` only; no raw-bucket read, no `ReceiveMessage`.
- **Tests:** `aws iam simulate-principal-policy` denies `s3:GetObject` on the raw bucket for the API role and `dynamodb:DeleteItem` for both.

### F16. Bucket hardening and teardown (C17, C20)
- Restrict CORS origins (or remove CORS if uploads go through the local server).
- Add a bucket policy denying `aws:SecureTransport = false`; add lifecycle rules for the processed bucket (for example 30 days) and `AbortIncompleteMultipartUpload` for both.
- Decide on `DeletionPolicy` for the table; document that `sam delete` requires emptying the processed bucket first.
- **Tests:** `aws s3api get-bucket-policy`, `get-bucket-cors`, `get-bucket-lifecycle-configuration`; an HTTP (non-TLS) request is denied.

### F17. Stuck-record protection (C13)
- Write `processing_started_at`, `lambda_request_id` and `attempts` on creation.
- In the handler, check `context.get_remaining_time_in_millis()` before each stage and mark `FAILED/TIMEOUT` when under ~10 s remain.
- Optionally add an EventBridge schedule that marks `PROCESSING` rows older than 5 minutes as `FAILED/TIMEOUT`.
- **Tests:** a simulated slow Textract call produces `FAILED/TIMEOUT`, not a permanent `PROCESSING`.

### F18. Provenance and stage timeline (planned feature, NOT IMPLEMENTED)
Add a `stages` list to each record:
```
{stage: "uploaded"|"validated"|"ocr_done"|"pii_done"|"stored", at: ISO8601, service: "S3"|"Textract"|"Comprehend", ms: int, items?: int}
```
- Append each stage as it completes; show the timeline in the dashboard's detail panel, together with `ocr_engine`/`pii_engine`. This is the in-product evidence that Textract and Comprehend actually ran.
- **Tests:** a happy-path handler test asserts the five stages appear in order with non-decreasing timestamps.

### F19. Dashboard gaps (C21, section 6.10)
- Catch errors inside `reader.onload`; always reset `isUploading`.
- Match the new record by a server-returned `document_id`/`sequencer`, not file name and client time.
- Add status filter, Processing and Total-PII cards, DLQ count, and a failed-stage line.
- **Tests:** a Playwright or jsdom smoke test uploading a stub that returns 500 clears the spinner and re-enables polling.

### F20. Cleanup and docs (C22-C25)
- Delete the files listed in section E; add `.gitignore` and a `LICENSE` or remove the badge.
- Replace sample data with synthetic data, and add a multi-page PDF, a corrupt image and an oversize sample.
- Make `deploy.ps1` run `aws sts get-caller-identity` and `sam validate --lint`, take `-Environment`, and pass `AlertEmail` through `samconfig.toml` or warn when it is empty.
- Make `simulate_failure.py` exit non-zero unless the message arrived, the alarm reached `ALARM`, and the SNS subscription is `Confirmed`; clean up its junk rows.
- Correct README/ARCHITECTURE/COMPARISON claims (C24).
- **Tests:** CI runs `cfn-lint`, `pytest`, and a grep that fails on `AuthType: NONE` and on any personal-data fixture.

### Prompts for Antigravity (paste one at a time, check the result before the next)
1. "Apply F1 only (remove the fabricated fallbacks and add provenance fields). Show the diff and run the tests."
2. "Apply F3 and add the listed failing-then-passing overlap tests."
3. "Apply F4 (server-controlled masking) and the tests."
4. "Apply F5 (idempotent IDs and conditional writes) and the tests."
5. "Apply F2: remove the public Function URL unless I say I need a hosted dashboard. Ask me first."
6. "Apply F7 and F9 together so the dashboard and backend routes match and the local server is hardened."
7. "Apply F10, F11 and F17."
8. "Apply F12 through F16 and F18, one at a time, running `cfn-lint` and `pytest` after each."
9. "Re-audit against this report's matrix and list what is still PARTIAL, NOT IMPLEMENTED or UNVERIFIED. Do not claim AWS tests passed unless they were run against AWS."

---

## G. Final post-deployment verification checklist

Set once in PowerShell (`$S` and `$R` must match your deploy):
```powershell
$S = "pii-safe-pipeline"; $R = "us-east-1"
$o = aws cloudformation describe-stacks --stack-name $S --region $R --query "Stacks[0].Outputs" --output json | ConvertFrom-Json
$raw  = ($o | Where-Object OutputKey -eq 'RawBucketName').OutputValue
$proc = ($o | Where-Object OutputKey -eq 'ProcessedBucketName').OutputValue
$tbl  = ($o | Where-Object OutputKey -eq 'DocumentTrackingTableName').OutputValue
$dlq  = ($o | Where-Object OutputKey -eq 'PipelineDeadLetterQueueUrl').OutputValue
$topic= ($o | Where-Object OutputKey -eq 'FailureNotificationTopicArn').OutputValue
$api  = ($o | Where-Object OutputKey -eq 'ManagementApiUrl').OutputValue
```

### G.1 Before deploying
| # | Check | Command | Pass looks like |
|---|---|---|---|
| 1 | Who am I | `aws sts get-caller-identity` | Your account ID and the expected role/user. |
| 2 | Profile and region | `aws configure list` | Region matches `$R`. |
| 3 | Local tests | `python -m pytest tests -v` | All pass (16 today; more after F-fixes). |
| 4 | Template lint | `cfn-lint template.yaml` (or `sam validate --lint`) | No findings. |
| 5 | Comprehend works on this account | `aws comprehend detect-pii-entities --language-code en --text "My name is Jane Doe, jane@example.com" --region $R` | Returns `Entities`, **not** `SubscriptionRequiredException`. |
| 6 | Textract works (needs a file in S3) | `aws textract detect-document-text --document "S3Object={Bucket=$raw,Name=<key>}" --region $R` after the first upload | Returns `Blocks`. If it errors with a subscription message, fix the account before relying on any result. |

### G.2 Deploy and static resource checks
| # | Check | Command | Pass looks like |
|---|---|---|---|
| 7 | Deploy | `.\deploy.ps1 -AlertEmail you@example.com` | Stack `CREATE_COMPLETE` / `UPDATE_COMPLETE`. |
| 8 | Outputs present | the `$o` query above | Bucket names, table, queue URL, topic ARN (and API URL if kept). |
| 9 | Encryption | `aws s3api get-bucket-encryption --bucket $raw` (and `$proc`) | `AES256`. |
| 10 | Public access | `aws s3api get-public-access-block --bucket $raw` | All four settings `true`. |
| 11 | Lifecycle | `aws s3api get-bucket-lifecycle-configuration --bucket $raw` | Expiration = your retention days. |
| 12 | S3 → Lambda trigger | `aws s3api get-bucket-notification-configuration --bucket $raw` | One `LambdaFunctionConfigurations` entry for `s3:ObjectCreated:*`. |
| 13 | Retries + destination | `aws lambda get-event-invoke-config --function-name "$S-processor" --region $R` | `MaximumRetryAttempts: 2` and the SQS destination. |
| 14 | Lambda env vars | `aws lambda get-function-configuration --function-name "$S-processor" --region $R --query Environment` | Table/bucket names present. |
| 15 | SNS subscription | `aws sns list-subscriptions-by-topic --topic-arn $topic --region $R` | `SubscriptionArn` is a real ARN, not `PendingConfirmation` (click the email link first). |
| 16 | Alarm | `aws cloudwatch describe-alarms --alarm-names "$S-DLQ-Messages-Alarm" --region $R` | Exists, action = the topic. |
| 17 | API is locked (after F2) | `curl.exe -i "$api/documents"` | 401/403. A `200` with data **confirms C2**. |
| 18 | IAM | `aws iam simulate-principal-policy ...` for the API role | Denied on the raw bucket and on `DeleteItem` (after F15). |

### G.3 Functional tests (upload with the CLI script or the dashboard)
| # | Test | Expected |
|---|---|---|
| 19 | `sample_id_card.png`, `sample_medical_intake.jpg`, `sample_application.pdf` | Row `COMPLETED`; redacted `.txt` exists in `$proc` (`aws s3 ls s3://$proc/redacted/`); open it and confirm **no** name, phone, e-mail or ID fragments remain. |
| 20 | `blank_document.png` | `FAILED` at `TEXT_EXTRACTION`. |
| 21 | **A blank PNG renamed `scan1.png`** | Must **not** be `COMPLETED` (this is the C1 test). |
| 22 | `unsupported_document.txt` | `FAILED` at `VALIDATION`, no DLQ message. |
| 23 | Multi-page PDF (after F10) | `FAILED/VALIDATION`, one row, queue stays empty. |
| 24 | Corrupt PNG, zero-byte file (after F10) | One `FAILED` row each, no retries. |
| 25 | Same file uploaded twice (after F5) | Two separate objects but no duplicate rows for one event; check `aws dynamodb scan --table-name $tbl --max-items 20`. |
| 26 | File name with spaces and brackets, e.g. `my resume (1).pdf` | Visible in the dashboard and opens correctly. |
| 27 | Text with overlapping detections (an address next to a name) | No head/tail fragments survive. |
| 28 | Dashboard default settings | Passwords/keys/URLs in a test document are masked (F4). |

### G.4 Evidence that the AWS services were really used
| # | Check | Command | Pass looks like |
|---|---|---|---|
| 29 | Lambda logs | `aws logs tail /aws/lambda/$S-processor --since 15m --region $R` | Step logs; **no** "fallback" warnings. |
| 30 | API calls recorded | `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DetectDocumentText --region $R` (repeat for `DetectPiiEntities`) | Events for your uploads (may lag up to ~15 minutes). |
| 31 | Per-record provenance (after F1/F18) | `aws dynamodb get-item ...` | `ocr_engine=textract`, `pii_engine=comprehend`, stage timestamps in order. |

### G.5 Failure path
| # | Check | Command | Pass looks like |
|---|---|---|---|
| 32 | Force a real failure (after F20 makes the script assert) | `python scripts/simulate_failure.py` | Exits 0 only if the message arrived; non-zero otherwise. |
| 33 | Queue depth | `aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages` | Increases by 1 per failed event. |
| 34 | Alarm state | `aws cloudwatch describe-alarms --alarm-names "$S-DLQ-Messages-Alarm"` | `ALARM`. Allow a couple of minutes. |
| 35 | Email | Check the inbox you confirmed | Alert arrives. Trigger a second failure after the alarm returns to `OK` and confirm a **second** email (C11). |
| 36 | Clean up the test | `aws sqs purge-queue --queue-url $dlq` and delete junk rows | Queue empty, alarm back to `OK`. |

### G.6 Lifecycle, cost, teardown
| # | Check | Pass looks like |
|---|---|---|
| 37 | Raw expiry | After 24-48 hours, `aws s3 ls s3://$raw/` no longer lists the test uploads. |
| 38 | Processed-bucket retention (after F16) | Lifecycle rule present. |
| 39 | Log retention (after F14) | `aws logs describe-log-groups --log-group-name-prefix /aws/lambda/$S` shows `retentionInDays`. |
| 40 | Budget alert | One exists in the Billing console before heavy testing. |
| 41 | Teardown | `aws s3 rm s3://$proc --recursive`, then `sam delete --stack-name $S --region $R`; stack deletes cleanly. |

**Do not report success on code generation alone.** The items that remain UNVERIFIED until you run G.2-G.5 are: S3 trigger, Textract/Comprehend access, Function URL access, presigned upload, retries, destination delivery, alarm firing, SNS email, lifecycle expiry, and every dashboard-against-AWS behaviour.
