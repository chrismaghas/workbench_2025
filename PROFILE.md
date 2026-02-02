
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
Chris

Conventions used in examples

Headers (all endpoints)

Authorization: Bearer <token>
Content-Type: application/json


Base URL

/api/v2/c2c/{project_id}

1) Upload target + reference files
POST /api/v2/c2c/{project_id}/files
Request (multipart/form-data)

Form-data keys (typical):

target_file: (file) target CSV

reference_file: (file) reference CSV

cURL

curl -X POST "http://localhost:8000/api/v2/c2c/123/files" \
  -H "Authorization: Bearer <token>" \
  -F "target_file=@./tests/fixtures/c2c_target_sample.csv" \
  -F "reference_file=@./tests/fixtures/c2c_reference_sample.csv"

Response 200 (current untyped JSON contract)
{
  "status": "success",
  "message": "C2C target + reference files uploaded for project 123",
  "project_id": 123,
  "target": { "id": 1, "path": "…", "filename": "c2c_target_sample.csv" },
  "reference": { "id": 2, "path": "…", "filename": "c2c_reference_sample.csv" }
}

Common errors

401: invalid token / user not found

403: no permission

404: project not found

2) Configure column mappings
PUT /api/v2/c2c/{project_id}/target-file
Request
{
  "title_colname": "Job Title",
  "department_colname": "Department",
  "level_colname": "Level",
  "function_colname": null,
  "description_colname": null
}

Response 200
{
  "id": 1,
  "project_id": 123,
  "column_names": ["Job Title", "Department", "Level"],
  "title_colname": "Job Title",
  "department_colname": "Department",
  "level_colname": "Level",
  "function_colname": null,
  "description_colname": null
}

PUT /api/v2/c2c/{project_id}/reference-file
Request
{
  "title_colname": "Position",
  "department_colname": "Team",
  "level_colname": "Grade",
  "function_colname": null,
  "description_colname": null
}

Response 200
{
  "id": 2,
  "project_id": 123,
  "column_names": ["Position", "Team", "Grade"],
  "title_colname": "Position",
  "department_colname": "Team",
  "level_colname": "Grade",
  "function_colname": null,
  "description_colname": null
}

Common errors

400: missing/empty title_colname

401/403/404 as usual

3) Level mappings
POST /api/v2/c2c/{project_id}/level-mappings
Request
{
  "target_level": "P3",
  "reference_level": "L4"
}

Response 201 (create) OR 200 (update) — depending on implementation
{
  "id": 10,
  "project_id": 123,
  "target_level": "P3",
  "reference_level": "L4"
}

Errors

400: "target_level is required" / "reference_level is required"

401/403/404

GET /api/v2/c2c/{project_id}/level-mappings
Response 200
{
  "project_id": 123,
  "mappings": [
    { "id": 10, "project_id": 123, "target_level": "P3", "reference_level": "L4" },
    { "id": 11, "project_id": 123, "target_level": "P4", "reference_level": "L5" }
  ]
}

DELETE /api/v2/c2c/{project_id}/level-mappings/{mapping_id}
Response 204 (No Content)

(no body)

Errors

404: "Level mapping not found for id: {mapping_id}"

4) Matching jobs
POST /api/v2/c2c/{project_id}/matching-jobs
Request

(no body)

Response 201
{
  "id": 50,
  "project_id": 123,
  "status": "pending",
  "started_at": null,
  "ended_at": null
}

Errors

400: "Target file column mappings not configured"

400: "Reference file column mappings not configured"

409: "A matching job is already pending or running for this project"

401/403/404

GET /api/v2/c2c/{project_id}/matching-jobs (latest job)
Response 200
{
  "id": 50,
  "project_id": 123,
  "status": "running",
  "started_at": "2026-02-01T10:10:00Z",
  "ended_at": null
}


If no job exists yet, implementations usually return 404 or 200 with null; if yours returns something specific, align UI to that.

GET /api/v2/c2c/{project_id}/matching-jobs/{job_id}
Response 200
{
  "id": 50,
  "project_id": 123,
  "status": "running",
  "started_at": "2026-02-01T10:10:00Z",
  "ended_at": null,
  "records_processed": 120,
  "records_total": 500,
  "error_class": null,
  "error_traceback": null
}

Failure example
{
  "id": 50,
  "project_id": 123,
  "status": "failed",
  "started_at": "2026-02-01T10:10:00Z",
  "ended_at": "2026-02-01T10:12:10Z",
  "records_processed": 120,
  "records_total": 500,
  "error_class": "ValueError",
  "error_traceback": "Traceback (most recent call last): …"
}

5) Matches
GET /api/v2/c2c/{project_id}/matches?page=1&page_size=50&sort_by=id&sort_order=asc
Response 200
{
  "page": 1,
  "page_size": 50,
  "total": 2,
  "items": [
    {
      "id": 100,
      "target_record_id": 2001,
      "target_norm_title": "software engineer",
      "reference_record_id": 3001,
      "reference_norm_title": "sr developer",
      "confidence": 0.87,
      "user_changed": false
    },
    {
      "id": 101,
      "target_record_id": 2002,
      "target_norm_title": "product manager",
      "reference_record_id": 3002,
      "reference_norm_title": "pm lead",
      "confidence": 0.83,
      "user_changed": true
    }
  ]
}

GET /api/v2/c2c/{project_id}/matches/{match_id}/alternatives
Response 200 (normal)
{
  "match_id": 100,
  "target_record_id": 2001,
  "target_norm_title": "software engineer",
  "current_reference_record_id": 3001,
  "alternatives": [
    { "reference_record_id": 3001, "reference_norm_title": "sr developer", "confidence": 0.87 },
    { "reference_record_id": 3005, "reference_norm_title": "software engineer ii", "confidence": 0.81 },
    { "reference_record_id": 3010, "reference_norm_title": "backend engineer", "confidence": 0.79 }
  ]
}

Response 200 (vectors missing/mismatch → empty alternatives)
{
  "match_id": 100,
  "target_record_id": 2001,
  "target_norm_title": "software engineer",
  "current_reference_record_id": 3001,
  "alternatives": []
}

PUT /api/v2/c2c/{project_id}/matches/{match_id}
Request
{
  "reference_record_id": 3005
}

Response 200
{
  "id": 100,
  "target_record_id": 2001,
  "target_norm_title": "software engineer",
  "reference_record_id": 3005,
  "reference_norm_title": "software engineer ii",
  "confidence": 0.81,
  "user_changed": true
}

6) Download results CSV
GET /api/v2/c2c/{project_id}/results/download
Response 200

Headers:

Content-Type: text/csv
Content-Disposition: attachment; filename="c2c_results_123.csv"


Body example (first rows):

target_record_id,target_title,target_department,target_level,target_norm_title,reference_record_id,reference_title,reference_department,reference_level,reference_norm_title,confidence,user_changed
2001,Software Engineer,Engineering,P3,software engineer,3001,Sr. Developer,Tech,L4,sr developer,0.87,false


If no matches yet → headers only.

Postman-style collection summary (quick copy/paste)

C2C

POST {{baseUrl}}/api/v2/c2c/{{project_id}}/files

PUT {{baseUrl}}/api/v2/c2c/{{project_id}}/target-file

PUT {{baseUrl}}/api/v2/c2c/{{project_id}}/reference-file

POST {{baseUrl}}/api/v2/c2c/{{project_id}}/level-mappings

GET {{baseUrl}}/api/v2/c2c/{{project_id}}/level-mappings

DELETE {{baseUrl}}/api/v2/c2c/{{project_id}}/level-mappings/{{mapping_id}}

POST {{baseUrl}}/api/v2/c2c/{{project_id}}/matching-jobs

GET {{baseUrl}}/api/v2/c2c/{{project_id}}/matching-jobs

GET {{baseUrl}}/api/v2/c2c/{{project_id}}/matching-jobs/{{job_id}}

GET {{baseUrl}}/api/v2/c2c/{{project_id}}/matches?page=1&page_size=50&sort_by=id&sort_order=asc

GET {{baseUrl}}/api/v2/c2c/{{project_id}}/matches/{{match_id}}/alternatives

PUT {{baseUrl}}/api/v2/c2c/{{project_id}}/matches/{{match_id}}

GET {{baseUrl}}/api/v2/c2c/{{project_id}}/results/download
