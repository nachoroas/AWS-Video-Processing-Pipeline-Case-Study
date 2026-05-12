# operations.md

# Operations

This document describes how the pipeline handles job states, failures, retries, logging, monitoring and cost control.

## Operational Summary

| Area          | Approach                                                |
| ------------- | ------------------------------------------------------- |
| Job lifecycle | Track each job through explicit states                  |
| Dispatch      | Use Lambda to move validated jobs from SQS to AWS Batch |
| Processing    | Run containerized workloads in AWS Batch                |
| Failures      | Preserve job context and mark failed states             |
| Retries       | Retry transient failures with limits                    |
| Cost control  | Limit concurrency and avoid always-on compute           |
| Recovery      | Reprocess from S3 using stored job metadata             |

## Job Lifecycle

Each video-processing request should move through explicit states.

| State        | Meaning                                             |
| ------------ | --------------------------------------------------- |
| `PENDING`    | The job was created, but processing has not started |
| `QUEUED`     | The upload produced a processing request            |
| `DISPATCHED` | Lambda submitted the job to AWS Batch               |
| `RUNNING`    | AWS Batch started the processing container          |
| `SUCCEEDED`  | The worker finished and wrote results to S3         |
| `FAILED`     | The job failed during dispatch or processing        |

These states make the pipeline easier to debug. They also let the backend expose meaningful progress to users.

## Dispatch Operation

Lambda handles dispatch from SQS to AWS Batch.

For each message, Lambda should:

1. read the SQS message;
2. validate required metadata;
3. check that the input object exists in S3;
4. check concurrency limits;
5. submit the job to AWS Batch;
6. store the Batch job identifier;
7. update the job state to `DISPATCHED`.

The dispatcher stays small. It does not run video-processing logic.

## Processing Operation

AWS Batch runs the processing container.

For each job, the container should:

1. read job parameters;
2. download or stream the input video from S3;
3. run the video-processing task;
4. write output files to S3;
5. emit logs for each relevant step;
6. update the job state to `SUCCEEDED` or `FAILED`.

The worker should fail with clear errors when required inputs are missing.

## Failure Handling

The main failure cases are:

| Failure                | Handling                                               |
| ---------------------- | ------------------------------------------------------ |
| Missing S3 object      | Mark job as `FAILED` and log the missing key           |
| Invalid job metadata   | Reject dispatch and mark job as `FAILED`               |
| Batch submission error | Retry if transient, fail if persistent                 |
| Worker crash           | Mark job as `FAILED` through Batch status handling     |
| Missing output file    | Mark job as `FAILED` after validation                  |
| Permission error       | Fail fast and inspect IAM policies                     |
| Timeout                | Mark job as `FAILED` and allow controlled reprocessing |

Failures should preserve enough context to support debugging: job id, input key, Batch job id, state transition and error message.

## Retries

Retries should be bounded.

The system should:

* retry transient AWS errors;
* avoid retrying invalid metadata;
* avoid retrying missing input files without new evidence;
* cap retry count per job;
* preserve the original error history;
* avoid uncontrolled automatic reprocessing.

A failed job should be recoverable from stored metadata and the original S3 input, as long as the input object still exists.

## Logging

Each component should emit structured logs.

Recommended log fields:

* `job_id`
* `input_key`
* `output_key`
* `batch_job_id`
* `state`
* `component`
* `event`
* `error_code`
* `error_message`

Useful log events:

* job created;
* video uploaded;
* SQS message received;
* job dispatched;
* Batch job started;
* processing completed;
* output written;
* job failed.

Logs should let an operator reconstruct the full path of a job from upload to result.

## Monitoring

The pipeline should monitor both correctness and cost.

| Metric                  | Why it matters                             |
| ----------------------- | ------------------------------------------ |
| SQS queue depth         | Shows backlog                              |
| Oldest SQS message age  | Shows dispatch delay                       |
| Batch jobs by status    | Shows processing health                    |
| Failed jobs count       | Shows reliability issues                   |
| Average processing time | Shows workload cost and performance        |
| S3 storage growth       | Shows retention and cost risk              |
| Lambda errors           | Shows dispatch problems                    |
| Retry count             | Shows unstable workloads or infrastructure |

The key operational question is whether the system can process new videos without silent failures or uncontrolled cost growth.

## Concurrency Control

Concurrency should be controlled before jobs reach heavy compute.

The system can limit:

* number of jobs dispatched per interval;
* number of active Batch jobs;
* number of jobs per user or project;
* maximum retry count;
* maximum video duration or input size.

This keeps processing predictable. It may increase completion time, but it protects the system from sudden cost spikes.

## Cost Control

Main cost drivers:

* Batch compute runtime;
* number of concurrent jobs;
* S3 storage size;
* repeated failed processing;
* CloudWatch logs volume;
* data transfer between services.

Cost controls:

* avoid always-on workers;
* limit concurrent Batch jobs;
* store only required outputs;
* apply S3 lifecycle policies;
* avoid unbounded retries;
* compress or clean temporary files;
* monitor average runtime per job.

The system should optimize for controlled throughput, not maximum parallelism.

## Recovery

Recovery starts from job metadata and S3 objects.

A job can be reprocessed when:

* the input object still exists;
* the previous failure cause is understood;
* retry limits allow a new attempt;
* the system can write outputs to a clean destination.

Reprocessing should create a visible state transition. Operators should know whether a result came from the first attempt or a retry.

## Runbook

### A job is stuck in `PENDING`

1. Check whether the video was uploaded to S3.
2. Check whether S3 emitted an event.
3. Check whether an SQS message exists.
4. Check Lambda trigger configuration.
5. Inspect dispatcher logs.

### A job is stuck in `DISPATCHED`

1. Check the Batch job id.
2. Inspect AWS Batch status.
3. Check whether the compute environment has capacity.
4. Inspect container startup logs.
5. Confirm IAM permissions for S3 access.

### A job failed during processing

1. Inspect worker logs.
2. Confirm the input object exists.
3. Check video format and size.
4. Check runtime dependencies.
5. Confirm output path permissions.
6. Decide whether the job can be retried.

### Results are missing

1. Check whether the Batch job finished.
2. Inspect the expected output key.
3. Confirm the worker wrote files to S3.
4. Check backend result mapping.
5. Confirm download permissions or presigned URL generation.

## Operational Outcome

The operational model keeps the pipeline inspectable. Each job has a state, each component has a narrow responsibility and each failure leaves enough context for recovery.

The system favors predictable processing, bounded retries and controlled cost over maximum throughput.
