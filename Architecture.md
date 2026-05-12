# architecture.md

# Architecture

This architecture processes long-running sports videos through an asynchronous AWS pipeline. The backend creates and tracks jobs, while the processing workload runs outside the request-response path.

```txt
S3 upload → SQS queue → Lambda dispatcher → AWS Batch worker → S3 results
```

## Overview

The system separates upload, dispatch, processing and result storage.

A video enters the system through S3. The upload event creates a message in SQS. Lambda consumes the message and submits a job to AWS Batch. AWS Batch runs a processing container that reads the video from S3 and writes the generated results back to S3.

The backend does not process videos. It manages metadata, job states and result access.

## Flow

1. A user uploads a video to S3.
2. S3 emits an event to SQS.
3. SQS stores the processing request.
4. Lambda consumes the message from SQS.
5. Lambda validates the job metadata.
6. Lambda submits a job to AWS Batch.
7. AWS Batch starts a processing container.
8. The container reads the input video from S3.
9. The container runs the video-processing task.
10. The container writes results to S3.
11. The backend exposes job status and result access.

## Components

### S3

S3 stores input videos and generated results.

It defines the storage boundary of the system. Large files do not pass through the backend, Lambda or SQS. Each component references objects through keys instead of moving video payloads directly.

### SQS

SQS stores processing requests after upload events.

It decouples file ingestion from compute execution. Uploads can complete even when processing capacity is limited. The queue also gives the system a place to control dispatch rate and retries.

### Lambda

Lambda works as a dispatcher.

It reads messages from SQS, validates required metadata and submits jobs to AWS Batch. It does not run video-processing logic.

### AWS Batch

AWS Batch runs the processing workload.

It executes a containerized task with the required dependencies for video analysis. The task reads the input object from S3 and writes outputs back to S3.

### Backend

The backend manages the application-level job lifecycle.

Its responsibilities include job creation, job state tracking, result access and integration with the user-facing application. It does not execute heavy processing.

## Data Boundaries

The system avoids sending large video files through services designed for small messages or short-lived execution.

* S3 stores video inputs and outputs.
* SQS stores job references and metadata.
* Lambda passes job parameters.
* AWS Batch reads and writes objects from S3.
* The backend stores job state and exposes access to results.

## Responsibility Boundaries

| Component | Responsibility                      |
| --------- | ----------------------------------- |
| S3        | Store input and output files        |
| SQS       | Buffer processing requests          |
| Lambda    | Dispatch jobs                       |
| AWS Batch | Execute video-processing containers |
| Backend   | Manage job state and result access  |

## Failure Points

The main failure points are:

* invalid upload events;
* missing or malformed job metadata;
* failed Batch submissions;
* failed processing containers;
* missing output files;
* inconsistent job states;
* permission errors between AWS services.

The architecture handles these cases through explicit job states, structured logs, bounded retries and restricted IAM permissions.

## Concurrency Control

Video processing can become expensive when several jobs run at the same time.

The system controls concurrency before dispatching jobs to AWS Batch. This keeps compute usage predictable and prevents the architecture from scaling beyond the intended cost envelope.

## Result Access

Generated results are written to S3.

The backend exposes access to those results through application-level permissions. One practical option is to generate presigned URLs after validating that the user can access the job.

## Design Summary

The architecture keeps the backend focused on application logic and moves heavy processing to AWS Batch. S3 handles large files, SQS buffers work, Lambda dispatches jobs and AWS Batch executes the long-running workload.
