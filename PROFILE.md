
Hi team,

The C2C backend work is complete and the APIs are now available for frontend integration. Below is the endpoint list, expected behaviors, and key notes so you can proceed.

## Base + Auth

* **Base path:** `/api/v2/c2c/{project_id}`
* **Auth:** `Authorization: Bearer <token>` (same as other project endpoints)
* **Permissions:** Admin OR project owner (`projects.created_by`) OR user has an `Access` row for the project.
* **Error conventions:**

  * 401: invalid/expired token or user not found
  * 403: no permission
  * 404: project not found

## Endpoints

### 1) Upload target + reference files

**POST** `/api/v2/c2c/{project_id}/files` ✅

* Uploads both files (target + reference).
* Returns JSON (untyped) with `target` and `reference` file metadata.

### 2) Configure column mappings

**PUT** `/api/v2/c2c/{project_id}/target-file` ✅
**PUT** `/api/v2/c2c/{project_id}/reference-file` ✅

* Sets mapped column names used for parsing/processing.
* Uses typed responses (`TargetFileResponse`, `ReferenceFileResponse`).

### 3) Level mappings (for grade/level alignment)

**POST** `/api/v2/c2c/{project_id}/level-mappings` ✅
**GET** `/api/v2/c2c/{project_id}/level-mappings` ✅
**DELETE** `/api/v2/c2c/{project_id}/level-mappings/{mapping_id}` ✅

* POST is **upsert** on `(project_id, target_level)`.

### 4) Matching jobs (create + poll)

**POST** `/api/v2/c2c/{project_id}/matching-jobs` ✅

* Creates a new job with `status="pending"`.
* Returns **409 Conflict** if there is already a pending/running job for the project.

**GET** `/api/v2/c2c/{project_id}/matching-jobs` ✅

* Returns the **latest** job for the project.

**GET** `/api/v2/c2c/{project_id}/matching-jobs/{job_id}` ✅

* Returns job details including progress (`records_processed`, `records_total`) + error fields if failed.

### 5) Matches (review + edit)

**GET** `/api/v2/c2c/{project_id}/matches` ✅

* Paginated list: supports `page`, `page_size` (max 200), `sort_by`, `sort_order`.

**GET** `/api/v2/c2c/{project_id}/matches/{match_id}/alternatives` ✅

* Returns alternative reference candidates.
* **If target vector is NULL or dimension mismatch:** returns `200` with `alternatives: []` (no error).

**PUT** `/api/v2/c2c/{project_id}/matches/{match_id}` ✅

* Updates match to a new `reference_record_id` and sets `user_changed=true`.

### 6) Download results

**GET** `/api/v2/c2c/{project_id}/results/download` ✅

* Streams CSV:

  * `Content-Type: text/csv`
  * `Content-Disposition: attachment; filename="c2c_results_{project_id}.csv"`

## Notes / Decisions

* **Vector dimension currently = 300** (matches existing model). Alternatives endpoint will safely return empty if vectors are missing/mismatched.
* Upload endpoint still returns an untyped JSON dict; everything else uses response models.

If you want, I can share example request/response payloads for each endpoint (or a Postman collection-style summary).

Thanks,
Edwin
