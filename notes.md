Requirement: Matching Job Status by Project ID Endpoint (USERSTORY-492971)
Summary (business)
Provide an API endpoint that allows clients (UI, automation, or other services) to retrieve the current status of a project’s MatchingJob using the Project ID. This endpoint enables consistent job monitoring, progress display, and troubleshooting by returning a standardized MatchingJobResponse payload for the job associated with the given project.
This is primarily used to:

show users whether matching has started / is running / has completed,
surface failure reasons when matching fails,
support polling-based UX patterns (e.g., “refresh status every N seconds”).


Business goals

Visibility: Users can see the current state of matching for a project without requiring backend logs or DB access.
Reliability: Standardizes job status retrieval across clients and environments.
Supportability: Improves troubleshooting via consistent error messages and correlation IDs.
Extensibility: Enables future enhancements (e.g., progress %, step breakdown, job history) without breaking the core contract.


User story
As an API developer, I need an endpoint that can be used to track the status of MatchingJobs by their related Project ID, so that clients can monitor progress and outcomes (success/failure) and present accurate job status to the end user.

Acceptance criteria

Endpoint accepts a single project_id path parameter (int).
Endpoint finds the MatchingJob related to that Project ID.
Endpoint returns a MatchingJobResponse on success (HTTP 200).
If no project exists for project_id, return 404 Not Found with a clear message.
If project exists but no MatchingJob exists for the project, return 404 Not Found with a clear message (or 204 No Content, if that’s the team standard—pick one and enforce it consistently).
If multiple MatchingJobs exist for a project, the endpoint returns the most recent job (by created_at or id descending) unless the system guarantees one job per project.
Endpoint does not modify any state (read-only).
Endpoint uses standard auth + project permission checks (same pattern as other project-scoped endpoints).
Endpoint logs:

request path and project_id,
authenticated user_id,
correlation ID,
job_id and job status when found,
not-found conditions as warnings.




API contract
Method

GET

Path

/api/v2/c2c/{project_id}/matching-job

(If your existing routing uses /api/v1/ or /api/v2/ differently, align to the repo convention.)



Path parameter

project_id (int) — required

Success response

200 OK
Body: MatchingJobResponse


The endpoint must return the schema as defined in the repo. If the schema already exists, do not invent fields—return exactly what MatchingJobResponse expects.

Example (illustrative only — align to your real schema):
JSON{  "id": 123,  "project_id": 456,  "status": "RUNNING",  "created_at": "2026-02-01T20:50:31Z",  "updated_at": "2026-02-01T20:55:12Z",  "started_at": "2026-02-01T20:50:40Z",  "completed_at": null,  "progress_pct": 65,  "message": "Matching in progress",  "error_detail": null}Show more lines
Error responses

401 Unauthorized — invalid/expired token
403 Forbidden — user lacks permission to access the project
404 Not Found

project does not exist
or no matching job exists for the project (depending on your chosen behavior)


500 Internal Server Error — unexpected DB/runtime error


Technical checklist (implementation)
Data retrieval

 Authenticate user (bearer token) using repo-standard helper (e.g., get_authenticated_user)
 Validate user permission for the project:

Admin role OR
project creator OR
user has Access record for project


 Fetch project:

db.query(ProjectMetadata).filter_by(id=project_id).first()
If missing → 404


 Fetch MatchingJob by project_id

If 1 job per project:
db.query(MatchingJob).filter_by(project_id=project_id).first()
If multiple jobs possible:
db.query(MatchingJob).filter(MatchingJob.project_id == project_id).order_by(MatchingJob.created_at.desc()).first()


 If job missing → 404 with message like:

"MatchingJob not found for project_id: {project_id}"



Response shaping

 Return MatchingJobResponse using existing Pydantic schema
 Ensure date/time fields serialize correctly (UTC recommended)
 Ensure enum/string status values match schema expectations

Logging & observability

 Log at INFO on success:

project_id, job_id, status, user_id, correlation ID


 Log at WARN for 404 not found cases
 Log exceptions with logger.exception(...) and include correlation ID

Error handling

 Fail fast on auth/permission issues (401/403)
 Consistent 404 messaging
 Wrap unexpected exceptions → 500 with generic message (don’t leak internals)


Implementation notes / design decisions
1) Handling multiple jobs per project (history)
If your system can create multiple matching jobs for a single project over time (e.g., reruns):

Return the latest job by default.
(Optional future enhancement) Add a query parameter:

?job_id=... or ?include_history=true
But USERSTORY-492971 only requires project_id—keep it simple for this iteration.



2) Status normalization
If internal job status values differ (e.g., Celery states, internal enums), normalize them to the API contract:

QUEUED, RUNNING, SUCCEEDED, FAILED, CANCELED
…and ensure MatchingJobResponse.status matches exactly.

3) Data consistency
If job exists but fields like progress or message can be null, ensure schema supports nullability or set defaults.

Test plan
Unit tests (pytest recommended)

 Permission logic tests

admin role user → allowed
creator user → allowed
access record user → allowed
unrelated user → 403


 Job retrieval tests

project exists + job exists → 200
project exists + job missing → 404
project missing → 404


 Multiple jobs (if applicable)

create 2 jobs, ensure endpoint returns newest by created_at/id



Integration tests

 Create project + matching job row → call GET endpoint → validate MatchingJobResponse
 Validate 401 when token missing/invalid
 Validate 403 when user lacks project permission

Manual (curl) example
Shellcurl -X GET "http://localhost:8000/api/v2/c2c/456/matching-job" \  -H "Authorization: Bearer <token>"Show more lines
Expected:

200 with MatchingJobResponse JSON


Logging & observability

INFO:

“MatchingJob status retrieved”
include: project_id, job_id, status, user_id, correlation_id


WARN:

“Project not found”
“MatchingJob not found for project”


ERROR/EXCEPTION:

Unexpected DB/session errors




Security & permissions

Endpoint must enforce the same project-scoped authorization rules as other endpoints:

Admin OR creator OR explicit project access.


Do not include sensitive internal details in error messages.
Ensure correlation ID is included in logs for tracing.


Definition of done (DoD)

 Endpoint implemented at agreed route and returns MatchingJobResponse
 Auth + permission checks applied consistently
 All acceptance criteria met
 Unit tests + integration tests added and passing
 OpenAPI docs updated (FastAPI auto docs reflect schema + responses)
 Logging includes project_id + correlation_id and supports debugging


Quick dev checklist (single PR)

 Add GET endpoint route + handler
 Add DB query logic: project lookup, job lookup
 Add response mapping to MatchingJobResponse
 Add logging for success/not-found/error
 Add tests (unit + basic integration)
 Run test suite + lint
