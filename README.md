# AWS Video Processing Pipeline Case Study

Technical case study of an AWS-based pipeline for asynchronous sports video processing.

The original project is private. This repository documents the architecture, decisions and operational concerns without exposing proprietary code, sensitive data or business logic.

## Overview

The system processes long sports videos outside the request-response path. Users upload videos to S3, the system queues processing work through SQS, Lambda dispatches jobs, and AWS Batch runs the containerized workload.

```txt
S3 upload → SQS queue → Lambda dispatcher → AWS Batch worker → S3 results
```

The backend does not process videos. It creates jobs, tracks states and exposes access to generated results.

## Problem

Sports video processing can take several minutes and requires more resources than a regular API request. Running the workload inside the backend would increase timeout risk, memory pressure and operational coupling.

The system needed to:

* receive large video files;
* process them after upload;
* avoid always-on compute;
* control concurrent processing;
* store generated outputs;
* expose job status and results through the backend.

## Solution

The architecture separates storage, queueing, dispatching, processing and result access.

S3 stores input videos and generated outputs. SQS buffers processing requests. Lambda validates and dispatches jobs. AWS Batch runs the heavy processing task in containers. The backend manages job state and result access.

## Main Components

| Component | Responsibility                           |
| --------- | ---------------------------------------- |
| S3        | Store video inputs and generated outputs |
| SQS       | Buffer processing requests               |
| Lambda    | Dispatch jobs to AWS Batch               |
| AWS Batch | Run long-running processing containers   |
| Backend   | Manage job state and result access       |

## My Role

* Designed the AWS processing pipeline.
* Connected upload events with asynchronous job execution.
* Defined the boundaries between backend, storage, queue and processing layers.
* Added concurrency control to keep compute cost predictable.
* Structured the system around explicit job states and S3-based result storage.

## Documents

* `architecture.md`: system flow and component responsibilities.
* `decisions.md`: technical decisions and tradeoffs.
* `operations.md`: job lifecycle, failures, retries and monitoring.
* `security.md`: access control, S3 boundaries, IAM and result access.

## Public Boundary

This repository documents the design of a private project. It does not include production code, proprietary algorithms, datasets, credentials or real infrastructure configuration.
