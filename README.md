# AWS Video Processing Pipeline Case Study

Technical case study of an AWS-based asynchronous pipeline for long-running sports video processing.

The original project is private. This repository documents the architecture and technical decisions without exposing proprietary code, sensitive data or business logic.

## Architecture

The architecture separates file upload from heavy processing. The backend does not process videos and does not keep workers active. Its responsibility is to create jobs, expose job states and provide access to results.

S3 upload → SQS queue → Lambda dispatcher → AWS Batch worker → S3 results

## Flow
1. A user uploads a video to S3.
1. S3 publishes an event to SQS.
1. Lambda consumes the message from the queue.
1.Lambda validates the job and dispatches an execution to AWS Batch.
1. AWS Batch runs a processing container.
1. The container reads the video from S3.
1. The processing task generates the output files.
1. The results are written back to S3.
1. The backend exposes job status and result access.
## Components

# S3

S3 acts as the input and output boundary. The system stores original videos and generated results as objects, avoiding large file transfers through the backend.

# SQS

SQS decouples video upload from processing. The queue absorbs events, controls the dispatch flow and prevents direct dependency between upload and execution.

# Lambda

Lambda acts as a lightweight dispatcher. It consumes messages from SQS, prepares job parameters and submits executions to AWS Batch.

# AWS Batch

AWS Batch runs the heavy processing workload inside containers. This avoids keeping compute resources active when there are no pending videos.

## Backend

The backend manages job creation, job states and result access. It does not execute compute-intensive video processing.

# Design Principles
- Separate upload, dispatch, processing and result storage.
- Keep the backend outside the heavy processing path.
- Use managed AWS services for queueing, storage and batch execution.
- Control concurrency to reduce infrastructure cost.
- Keep each component responsible for one part of the pipeline.

## Repository Scope

This repository does not include production code, proprietary algorithms, datasets, credentials or real infrastructure configuration.

It focuses on architecture, responsibilities and technical decisions.
