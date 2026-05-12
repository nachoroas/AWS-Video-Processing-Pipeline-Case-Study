# Technical Decisions

The pipeline uses AWS managed services to process long-running videos without blocking the backend or keeping compute resources active when there is no work.

## Decision Summary

| Decision                              | Reason                                                    |
| ------------------------------------- | --------------------------------------------------------- |
| Use S3 as the file boundary           | Keep large video files outside the backend and queue      |
| Use SQS between upload and processing | Decouple ingestion from compute execution                 |
| Use Lambda as a dispatcher            | Keep orchestration small and event-driven                 |
| Use AWS Batch for processing          | Run long-running containerized workloads                  |
| Control concurrency                   | Keep compute cost predictable                             |
| Store results in S3                   | Persist outputs and expose them through controlled access |

## Use S3 as the File Boundary

Video files are too large to move through the backend, Lambda or SQS.

The system stores inputs and outputs in S3. Other services pass object keys, metadata and job identifiers instead of transferring video payloads. This reduces memory pressure, timeout risk and payload-size constraints.

This decision also creates a clean boundary between application logic and file storage.

## Use SQS Between Upload and Processing

Video upload and video processing have different execution times.

SQS separates those concerns. The upload path can complete after the file reaches S3, while processing can start later based on available capacity. This avoids coupling user-facing actions to long-running compute tasks.

SQS also gives the system a buffer when several videos arrive close together.

## Use Lambda as a Dispatcher

Lambda handles one responsibility: consume messages from SQS, validate job metadata and submit jobs to AWS Batch.

The function does not process video. This keeps execution short, reduces operational risk and avoids placing heavy dependencies inside a serverless function.

## Use AWS Batch for Video Processing

The video-processing workload can run for several minutes and requires a controlled execution environment.

AWS Batch fits this workload because it runs containerized jobs and allocates compute around batch execution. The system avoids permanent workers and starts compute based on actual processing demand.

This decision gives the pipeline a better fit for heavy jobs than request-response execution or short-lived serverless functions.

## Control Concurrency

Running several video-processing jobs at the same time can increase cost and resource contention.

The architecture limits how many jobs can be dispatched or processed in parallel. This keeps compute usage predictable and protects the system from cost spikes when many files arrive close together.

The goal is controlled throughput within a defined cost envelope.

## Store Results in S3

Generated outputs are written back to S3.

This keeps results durable and separate from application servers. The backend can expose access through application permissions, such as presigned URLs, without making buckets public.

S3 also makes it easier to apply lifecycle policies for old inputs and outputs.

## Keep the Backend Outside the Processing Path

The backend manages jobs, states and result access. It does not execute video analysis.

This reduces timeout risk, avoids heavy resource use in the API layer and keeps the backend focused on application logic.

## Tradeoffs

### Managed services over custom orchestration

Managed AWS services reduce infrastructure code and operational burden. The tradeoff is stronger coupling to AWS primitives.

### Cost control over maximum parallelism

The architecture limits concurrent processing. This can increase total completion time when many jobs arrive, but it keeps infrastructure cost more predictable.

### Explicit job states over implicit execution

The system tracks job states instead of assuming that each event completes. This adds metadata management, but improves observability and recovery.

### S3 object references over direct file transfer

Passing object keys between services avoids large payload transfers. The tradeoff is that each component must validate permissions and object existence with care.

## Alternatives Considered

### Always-on EC2 worker

An always-on worker is simple to reason about, but it keeps compute active even when there are no videos to process. That increases baseline cost.

### Processing inside the backend

Running video analysis inside the backend would reduce the number of services, but it would expose the API layer to long execution times, memory pressure and failure propagation.

### Processing inside Lambda

Lambda works well for short tasks. Video processing can exceed practical limits for execution time, package size and runtime dependencies.

### Direct S3 to Batch execution

A direct path would reduce the dispatcher layer. Lambda gives the system a place to validate metadata, enforce constraints and prepare job parameters before execution.

## Decision Outcome

The chosen architecture separates storage, queueing, dispatch, processing and result access. This keeps the backend small, isolates heavy workloads and gives the system control over cost, retries and operational boundaries.
