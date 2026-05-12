# Security

This document describes how the pipeline protects video inputs, generated outputs, job metadata and AWS resources.

## Security Summary

| Area             | Approach                                                |
| ---------------- | ------------------------------------------------------- |
| S3 access        | Keep videos and results private                         |
| Result access    | Generate temporary access after backend authorization   |
| IAM              | Give each component narrow permissions                  |
| Secrets          | Keep credentials outside the repository                 |
| Input validation | Validate job ids, object keys and metadata              |
| Public exposure  | Expose only the backend boundary                        |
| Logs             | Record security-relevant events without leaking secrets |

## S3 Access

The pipeline stores input videos and generated results in private S3 locations.

Input and output objects should live under separate prefixes. This allows the system to grant narrower permissions to each component. The Batch worker can read input objects and write output objects, while the backend can control result access without exposing the bucket.

The bucket should block public access. Users should not receive direct public object URLs.

## Result Access

The backend controls access to generated results.

Before exposing a result, the backend should verify that the user can access the related job. After that check, it can generate temporary access, such as a presigned URL, for a specific output object.

A result URL should be:

* short-lived;
* scoped to one object;
* generated after authorization;
* tied to an expected output path.

The system should avoid logging full presigned URLs because they can grant temporary access to private files.

## IAM Boundaries

Each component should have only the permissions required for its responsibility.

| Component         | Permissions                                                     |
| ----------------- | --------------------------------------------------------------- |
| Lambda dispatcher | Read SQS messages, check S3 object existence, submit Batch jobs |
| AWS Batch worker  | Read input objects, write output objects, emit logs             |
| Backend           | Create jobs, read job states, generate controlled result access |
| SQS               | Receive upload-driven processing messages                       |
| S3                | Store inputs, store outputs and emit upload events              |

Broad permissions such as full S3, full SQS or full Batch access increase blast radius. The architecture should avoid them unless there is a clear operational reason.

## Secrets

Secrets do not belong in the repository.

The system should keep these values outside source control:

* AWS access keys;
* database credentials;
* webhook secrets;
* API tokens;
* production environment files;
* private infrastructure identifiers when they expose sensitive structure.

A public repository can include a safe `.env.example`, but not real values.

## Input Validation

The pipeline should validate every identifier before using it in AWS calls.

The system should validate:

* job ids;
* S3 object keys;
* expected input prefixes;
* expected output prefixes;
* file type and MIME type when applicable;
* user ownership of the job;
* Batch job parameters.

S3 object keys require care because they can produce path confusion or unauthorized reads and writes if the system accepts them without validation.

## Queue Message Validation

SQS messages should be treated as untrusted input.

The dispatcher should check that:

* required fields exist;
* the job id has the expected format;
* the input key belongs to an allowed prefix;
* the referenced S3 object exists;
* the job has not reached a terminal state;
* the message maps to a known job.

Invalid messages should fail with clear logs and should not trigger processing.

## Backend Authorization

The backend enforces user-level access to jobs and results.

A user should only access:

* jobs they created;
* outputs linked to their jobs;
* valid result files;
* allowed metadata fields.

The backend should not expose internal AWS identifiers unless they are needed for support or debugging.

## Public Exposure

The public boundary should stay small.

The backend API is the only component that clients should interact with directly. S3, SQS and AWS Batch should remain private infrastructure.

When direct upload is required, the backend can issue a presigned upload URL after validating the request. The system should still validate the uploaded object before dispatching processing.

## Logging and Auditability

Logs should help trace security-relevant events without leaking sensitive data.

Useful events to log:

* job creation;
* upload registration;
* dispatch attempt;
* Batch job submission;
* result generation;
* result access request;
* failed authorization;
* invalid queue message;
* permission error.

Avoid logging:

* full presigned URLs;
* secrets;
* credentials;
* raw tokens;
* sensitive user data;
* unnecessary object contents.

## Failure Modes

| Failure                  | Risk                                                |
| ------------------------ | --------------------------------------------------- |
| Public S3 bucket         | Unauthorized access to videos or results            |
| Broad IAM role           | One component can access more resources than needed |
| Unvalidated object key   | Unauthorized read or write path                     |
| Long-lived presigned URL | Result access lasts beyond the intended window      |
| Secrets in repository    | Credential exposure                                 |
| Missing ownership check  | One user can access another user's job              |
| Verbose logs             | Sensitive data leaks through observability tools    |

## Security Outcome

The security model keeps files private, limits permissions and places access control in the backend. S3 stores the data, SQS buffers work, Lambda dispatches jobs and AWS Batch processes videos under scoped permissions.

Each component should access only the resources required for its responsibility.
