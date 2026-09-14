# Event-Driven PDF Retrieval Comparison on AWS

An independently developed cloud application for uploading PDF documents,
processing them asynchronously, and comparing two text-chunking strategies for
information retrieval.

The project demonstrates an event-driven architecture on AWS. A Flask web
application stores uploaded PDFs in Amazon S3, two independent workers process
each document through separate Amazon SQS queues, and PostgreSQL stores the
resulting chunks, processing metrics, and retrieval results.

## Key Features

- Upload and validate PDF documents through a Flask web interface
- Trigger asynchronous document processing from S3 object-created events
- Compare fixed-size and paragraph-aware chunking strategies
- Extract PDF text and rank relevant chunks using TF-IDF and cosine similarity
- Track processing status, chunk count, average chunk length, and execution time
- Persist documents, processing runs, queries, and ranked results in PostgreSQL
- Retry transient database deadlocks and retain failed SQS messages for recovery

## Architecture

```mermaid
flowchart LR
    U[User] --> W[Flask Web App]
    W --> S3[Amazon S3]
    W --> DB[(PostgreSQL)]
    S3 --> SNS[Amazon SNS]
    SNS --> Q1[SQS: Fixed-size]
    SNS --> Q2[SQS: Paragraph-aware]
    Q1 --> A[Fixed-size Worker]
    Q2 --> B[Paragraph-aware Worker]
    A --> S3
    B --> S3
    A --> DB
    B --> DB
    DB --> W
```

### Processing Flow

1. The web application uploads a PDF to S3 and creates its database record.
2. An S3 object-created notification is published to SNS and fanned out to two
   SQS queues.
3. Each worker downloads the same PDF and applies its assigned chunking
   strategy.
4. Workers save the chunks and processing metrics to PostgreSQL.
5. After both runs complete, the web interface compares the top matching chunks
   for a user query.

## Technology Stack

- **Application:** Python, Flask
- **AWS:** Amazon S3, Amazon SNS, Amazon SQS, Amazon EC2
- **Database:** PostgreSQL / Amazon RDS
- **Document processing:** pypdf
- **Retrieval:** scikit-learn TF-IDF and cosine similarity

## Project Structure

| File | Purpose |
| --- | --- |
| `app.py` | Flask UI, PDF upload, status display, and retrieval comparison |
| `worker_fixed_size.py` | Worker entry point for fixed-size chunking |
| `worker_paragraph_aware.py` | Worker entry point for paragraph-aware chunking |
| `worker_common.py` | Shared SQS polling, PDF processing, retry, and persistence logic |
| `rag.py` | PDF extraction, chunking algorithms, and retrieval ranking |
| `db.py` | PostgreSQL connections and data-access helpers |
| `schema.sql` | Database schema for documents, runs, chunks, and queries |
| `aws_clients.py` | Configured boto3 clients for S3 and SQS |
| `config.py` | Environment-based application configuration |

## Prerequisites

- Python 3.10 or later
- An S3 bucket configured to publish object-created events
- An SNS topic with subscriptions to two SQS queues
- SQS queue policies that allow messages from the SNS topic
- A PostgreSQL database accessible from the web and worker instances
- AWS credentials or IAM roles with the required S3 and SQS permissions

## Configuration

Set the following environment variables on the relevant machines:

```sh
export AWS_REGION=us-east-1
export S3_BUCKET_NAME=your-s3-bucket-name
export FIXED_SIZE_QUEUE_URL=https://sqs.us-east-1.amazonaws.com/YOUR_ACCOUNT/fixed-size-queue
export PARAGRAPH_AWARE_QUEUE_URL=https://sqs.us-east-1.amazonaws.com/YOUR_ACCOUNT/paragraph-aware-queue
export DATABASE_URL='postgresql://app_user:password@your-rds-endpoint:5432/pdf_rag'
```

Optional settings:

```sh
export MAX_FILE_SIZE_MB=5
export SQS_WAIT_TIME_SECONDS=20
export SQS_VISIBILITY_TIMEOUT=300
```

The web application requires `S3_BUCKET_NAME` and `DATABASE_URL`. Workers
require all variables listed above.

> Do not commit credentials or environment files. For AWS deployments, prefer
> EC2 IAM roles over long-lived access keys.

## Installation

Install the dependencies on the web and worker machines:

```sh
python3 -m pip install -r requirements.txt
```

The application initializes the database tables from `schema.sql` when the web
process starts.

## Running the Application

Start the web application:

```sh
source your-env-file.sh
python3 app.py
```

The interface is available at:

```text
http://WEB_INSTANCE_PUBLIC_IP:5000
```

Start each worker in a separate process:

```sh
# Fixed-size chunking worker
source your-env-file.sh
python3 -u worker_fixed_size.py
```

```sh
# Paragraph-aware chunking worker
source your-env-file.sh
python3 -u worker_paragraph_aware.py
```

## Testing the Workflow

1. Open the web interface and upload a PDF.
2. Confirm that the file appears under `uploads/` in the configured S3 bucket.
3. Verify that both workers receive an event and complete processing.
4. Open the document page after both processing runs are complete.
5. Enter a query and compare the ranked chunks returned by each strategy.

Useful AWS CLI checks:

```sh
aws s3 ls "s3://$S3_BUCKET_NAME/uploads/"
aws sqs get-queue-attributes --queue-url "$FIXED_SIZE_QUEUE_URL" --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
aws sqs get-queue-attributes --queue-url "$PARAGRAPH_AWARE_QUEUE_URL" --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

## Troubleshooting

- **Upload fails:** Check the bucket name, AWS credentials or instance role,
  bucket policy, and network connectivity.
- **Workers receive no messages:** Check the S3 notification, SNS subscriptions,
  SQS queue policies, and queue URLs.
- **A document never becomes ready:** Inspect both worker logs and verify that
  each worker is connected to its assigned queue.
- **Database errors occur:** Check `DATABASE_URL`, database credentials, security
  group rules, and connectivity from every instance.
- **Messages are processed repeatedly:** Ensure the SQS visibility timeout is
  longer than the expected PDF processing time.

## Future Improvements

- Add infrastructure as code for reproducible AWS deployment
- Replace TF-IDF retrieval with vector embeddings
- Add authentication and per-user document isolation
- Add automated tests and continuous integration
- Package the services with containers

## Project Status

This is a personal portfolio project focused on event-driven cloud design,
asynchronous processing, and retrieval-strategy evaluation.
