# AWS Lambda: Cornell Notes & Code Examples

> Study notes on **AWS Lambda** (serverless compute), with working code examples for every major concept.

*Date: 26 Oct 2026 | Subject: Cloud Computing | Topic: AWS Lambda*

---

## Table of Contents

1. [What is AWS Lambda?](#1-what-is-aws-lambda)
2. [Key Features](#2-key-features)
3. [How Does Lambda Work?](#3-how-does-lambda-work)
4. [Common Use Cases](#4-common-use-cases)
5. [Lambda Pricing](#5-lambda-pricing-pay-as-you-go)
6. [How to Create a Lambda Function](#6-how-to-create-a-lambda-function)
7. [Common Event Sources](#7-common-event-sources)
8. [Best Practices](#8-best-practices)
9. [Important Points & Limits](#9-important-points--limits)
10. [Important Terms](#10-important-terms)
11. [Summary](#11-summary)

---

## 1. What is AWS Lambda?

AWS Lambda is a **serverless compute service** that lets you run code without managing servers. You upload your code, and Lambda runs it in response to events and automatically manages the compute resources for you.

---

## 2. Key Features

- **Serverless**: no server management
- **Pay for what you use**
- **Automatic scaling**
- **High availability**
- **Multiple runtimes**: Python, Node.js, Java, .NET, Go, Ruby, Custom Runtime
- **Easy integration** with AWS services
- **Secure and isolated** execution

---

## 3. How Does Lambda Work?

```
Event Source (Trigger) -> Lambda Function -> Runs your code in a secure container -> Process Event -> Return Result
```

### Minimal handler (Python)

```python
# lambda_function.py
import json

def lambda_handler(event, context):
    """
    event   -> the trigger data (dict)
    context -> runtime info (request id, remaining time, memory limit, etc.)
    """
    name = event.get("name", "World")
    print(f"Request ID: {context.aws_request_id}")  # goes to CloudWatch Logs

    return {
        "statusCode": 200,
        "body": json.dumps({"message": f"Hello, {name}!"})
    }
```

### Minimal handler (Node.js)

```javascript
// index.mjs
export const handler = async (event, context) => {
  const name = event.name ?? "World";
  console.log(`Request ID: ${context.awsRequestId}`);

  return {
    statusCode: 200,
    body: JSON.stringify({ message: `Hello, ${name}!` }),
  };
};
```

---

## 4. Common Use Cases

| Use Case | Typical Trigger |
|---|---|
| Real-time file processing | S3 |
| API backend | API Gateway |
| Data processing (logs, ETL) | Kinesis / S3 / DynamoDB Streams |
| Scheduled tasks | CloudWatch / EventBridge Events |
| IoT data processing | IoT Core |
| Chatbots & microservices | API Gateway / SNS / SQS |

### Example: Real-time file processing (S3 trigger)

```python
import urllib.parse

def lambda_handler(event, context):
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
        size = record["s3"]["object"].get("size", 0)
        print(f"New file: s3://{bucket}/{key} ({size} bytes)")
        # ...process the file here (resize image, parse logs, etc.)
    return {"processed": len(event["Records"])}
```

### Example: API backend (API Gateway, proxy integration)

```python
import json

def lambda_handler(event, context):
    method = event["httpMethod"]
    path = event["path"]

    if method == "GET" and path == "/health":
        return respond(200, {"status": "ok"})

    if method == "POST" and path == "/items":
        item = json.loads(event.get("body") or "{}")
        return respond(201, {"created": item})

    return respond(404, {"error": "Not found"})

def respond(status, body):
    return {
        "statusCode": status,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(body),
    }
```

### Example: Scheduled task (CloudWatch / EventBridge)

```python
from datetime import datetime, timezone

def lambda_handler(event, context):
    print(f"Scheduled job ran at {datetime.now(timezone.utc).isoformat()}")
    # e.g. clean up old data, send a daily report, etc.
```

---

## 5. Lambda Pricing (Pay-as-you-go)

- First **1M requests/month: Free**
- **$0.20 per 1M requests** after the free tier
- **$0.0000166667 per GB-second** (duration x memory)
- **No charge when your code is not running**

> Prices shown are those from the notes (x86, a common region). Always check the [official pricing page](https://aws.amazon.com/lambda/pricing/) since prices vary by region and architecture, and a separate compute free tier also applies.

### Example

A function uses **512 MB** memory and runs for **2 seconds**:

```
Cost = 0.5 GB x 2 sec x $0.0000166667
     = $0.0000166667 per request
```

### Pricing calculator (Python)

```python
def estimate_lambda_cost(
    requests: int,
    memory_mb: int,
    duration_sec: float,
    free_requests: int = 1_000_000,
    price_per_million_requests: float = 0.20,
    price_per_gb_second: float = 0.0000166667,
) -> dict:
    """Rough estimate (ignores the compute free tier and per-region differences)."""
    billable_requests = max(0, requests - free_requests)
    request_cost = (billable_requests / 1_000_000) * price_per_million_requests

    gb_seconds = (memory_mb / 1024) * duration_sec * requests
    compute_cost = gb_seconds * price_per_gb_second

    return {
        "request_cost": round(request_cost, 6),
        "compute_cost": round(compute_cost, 6),
        "total": round(request_cost + compute_cost, 6),
    }

# 512 MB, 2 sec -> per-request compute cost from the notes
print(estimate_lambda_cost(requests=1, memory_mb=512, duration_sec=2))
# {'request_cost': 0.0, 'compute_cost': 1.7e-05, 'total': 1.7e-05}

# 5 million requests per month
print(estimate_lambda_cost(requests=5_000_000, memory_mb=512, duration_sec=2))
```

---

## 6. How to Create a Lambda Function

### Option A: AWS Console (basic steps)

1. Go to the **AWS Lambda Console**
2. Click **"Create function"**
3. Choose **Author from scratch** or use a **blueprint**
4. Configure the function: **Name, Runtime, Architecture, Permissions**
5. **Write or upload** your code
6. **Test and Deploy**

### Option B: AWS CLI

**1. Create an execution role**

```bash
# trust-policy.json
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name lambda-basic-role \
  --assume-role-policy-document file://trust-policy.json

# Allows writing logs to CloudWatch (least privilege starting point)
aws iam attach-role-policy \
  --role-name lambda-basic-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**2. Package and create the function**

```bash
zip function.zip lambda_function.py

aws lambda create-function \
  --function-name hello-lambda \
  --runtime python3.12 \
  --architectures x86_64 \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::<ACCOUNT_ID>:role/lambda-basic-role
```

**3. Test it**

```bash
aws lambda invoke \
  --function-name hello-lambda \
  --cli-binary-format raw-in-base64-out \
  --payload '{"name": "Ali"}' \
  response.json

cat response.json
```

**4. Update the code later**

```bash
zip function.zip lambda_function.py
aws lambda update-function-code \
  --function-name hello-lambda \
  --zip-file fileb://function.zip
```

### Option C: AWS SAM (Infrastructure as Code)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 10
    MemorySize: 256
    Runtime: python3.12

Resources:
  HelloFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: lambda_function.lambda_handler
      CodeUri: .
      Environment:
        Variables:
          STAGE: dev
      Events:
        Api:
          Type: Api
          Properties:
            Path: /hello
            Method: get
        Nightly:
          Type: Schedule
          Properties:
            Schedule: rate(1 day)
```

```bash
sam build
sam local invoke HelloFunction --event event.json   # test locally
sam deploy --guided                                  # deploy
```

### Invoke from another service (boto3)

```python
import json
import boto3

lambda_client = boto3.client("lambda")

response = lambda_client.invoke(
    FunctionName="hello-lambda",
    InvocationType="RequestResponse",   # use "Event" for async
    Payload=json.dumps({"name": "Ali"}),
)

print(json.loads(response["Payload"].read()))
```

---

## 7. Common Event Sources

S3 · CloudWatch Events · API Gateway · DynamoDB · Kinesis · SQS · SNS ... and more

### Example: SQS trigger

```python
import json

def lambda_handler(event, context):
    for record in event["Records"]:
        body = json.loads(record["body"])
        print(f"Message {record['messageId']}: {body}")
```

### Example: SNS trigger

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        message = record["Sns"]["Message"]
        print(f"SNS message: {message}")
```

### Example: DynamoDB Streams trigger

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        print(record["eventName"])  # INSERT | MODIFY | REMOVE
        print(record["dynamodb"].get("NewImage"))
```

### Example: Kinesis trigger

```python
import base64
import json

def lambda_handler(event, context):
    for record in event["Records"]:
        payload = base64.b64decode(record["kinesis"]["data"])
        print(json.loads(payload))
```

---

## 8. Best Practices

### Keep functions small and focused

One function = one job. Split large logic into multiple functions or layers.

### Reuse connections (DB, SDK clients)

Create clients **outside** the handler so they are reused across warm invocations:

```python
import boto3

# Initialized once per container (cold start), reused afterwards
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("Items")

def lambda_handler(event, context):
    table.put_item(Item={"id": event["id"], "value": event["value"]})
    return {"status": "saved"}
```

### Use environment variables for config

```python
import os

TABLE_NAME = os.environ["TABLE_NAME"]
STAGE = os.environ.get("STAGE", "dev")
```

```bash
aws lambda update-function-configuration \
  --function-name hello-lambda \
  --environment "Variables={TABLE_NAME=Items,STAGE=prod}"
```

### Handle errors and timeouts properly

```python
import json

def lambda_handler(event, context):
    try:
        # Bail out gracefully before the timeout hits
        if context.get_remaining_time_in_millis() < 2000:
            raise TimeoutError("Not enough time left to process safely")

        result = do_work(event)
        return {"statusCode": 200, "body": json.dumps(result)}

    except ValueError as e:
        return {"statusCode": 400, "body": json.dumps({"error": str(e)})}
    except Exception as e:
        print(f"Unhandled error: {e}")   # visible in CloudWatch Logs
        raise                             # lets Lambda retry / send to a DLQ

def do_work(event):
    if "id" not in event:
        raise ValueError("'id' is required")
    return {"id": event["id"]}
```

### Use appropriate memory allocation

More memory also means more CPU. Test different sizes to find the best cost/performance balance:

```bash
aws lambda update-function-configuration \
  --function-name hello-lambda \
  --memory-size 512 \
  --timeout 30
```

### Monitor with CloudWatch Logs & Metrics

```bash
# Tail logs live
aws logs tail /aws/lambda/hello-lambda --follow
```

Useful metrics: `Invocations`, `Errors`, `Duration`, `Throttles`, `ConcurrentExecutions`.

### Use IAM roles with least privilege

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["dynamodb:PutItem", "dynamodb:GetItem"],
    "Resource": "arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/Items"
  }]
}
```

---

## 9. Important Points & Limits

| Item | Limit |
|---|---|
| Maximum timeout | **15 minutes** |
| Deployment package (zipped) | **50 MB** |
| Deployment package (unzipped) | **250 MB** |
| IAM | Use roles with **least privilege** |
| Cold start | Can occur on the **first invocation** |

---

## 10. Important Terms

| Term | Meaning |
|---|---|
| **Function** | Your code |
| **Handler** | Entry point |
| **Event** | Trigger data |
| **Invocation** | Execution |
| **Cold Start** | First-time delay |
| **Concurrency** | Number of functions running at the same time |

### Seeing a cold start in action

```python
import time

_cold = True  # module-level: runs once per container

def lambda_handler(event, context):
    global _cold
    was_cold = _cold
    _cold = False
    return {"cold_start": was_cold, "ts": time.time()}
```

Invoke twice in a row: the first response shows `cold_start: true`, the next shows `false`.

### Controlling concurrency

```bash
aws lambda put-function-concurrency \
  --function-name hello-lambda \
  --reserved-concurrent-executions 10
```

---

## 11. Summary

AWS Lambda allows you to run code without provisioning or managing servers. It is **event-driven**, **automatically scales**, and you **pay only for the compute time you consume**. It integrates with many AWS services and is ideal for building modern, scalable, and cost-effective applications.

---

## Suggested Repo Structure

```
.
├── README.md
├── lambda_function.py      # Basic handler
├── template.yaml           # AWS SAM template
├── trust-policy.json       # IAM trust policy
├── event.json              # Sample test event
└── examples/
    ├── s3_trigger.py
    ├── api_gateway.py
    ├── sqs_trigger.py
    └── pricing_calculator.py
```

## Resources

- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
