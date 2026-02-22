
# AWS Lambda

Complete Step‑by‑Step Guide + Examples

AWS Lambda is the core of serverless development.  
You write code → AWS runs it → you never manage servers.

***

# 🧠 **SECTION 1: Lambda Core Concepts (Simple Explanation)**

## **1. Lambda Function**

Your code + runtime environment.

Supported runtimes (most common):

*   Python 3.x
*   Node.js
*   Java
*   .NET
*   Go

For serverless apps, **Python or Node.js** is recommended.

***

## **2. Lambda Execution Role**

The IAM role Lambda assumes each time it runs.

Examples:

*   Access S3
*   Read/write DynamoDB
*   Write CloudWatch logs

***

## **3. Lambda Trigger**

Events that invoke your function.

Common triggers:

*   S3 upload
*   API Gateway request
*   DynamoDB stream
*   EventBridge cron
*   CloudFront (Lambda\@Edge)
*   SNS / SQS

***

## **4. Environment Variables**

Config for your function. Example:

    DB_NAME=Orders
    LOG_LEVEL=debug

***

## **5. Timeout & Memory**

Lambda has:

*   max timeout: 15 minutes
*   memory: 128 MB → 10 GB

More memory = more CPU = faster execution.

***

## **6. Concurrency**

How many Lambdas can run in parallel.

Types:

*   Unreserved concurrency
*   Reserved concurrency
*   Provisioned concurrency

***
# SECTION 2: Hands‑On Exercise #1

Create Your First Lambda Function\*\*

***

## **Step 1 — Go to Lambda Console → Create Function**

Choose:

    Author from scratch

Name:

    hello-lambda

Runtime:

    Node.js or Python (recommended)

Execution role:

    Create a new role with basic Lambda permissions

***

## **Step 2 — Add Sample Code**

### **Python Example**

```python
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "body": "Hello from Lambda!"
    }
```

### **Node.js Example**

```javascript
exports.handler = async (event) => {
  return {
    statusCode: 200,
    body: "Hello from Lambda!"
  };
};
```

***

## **Step 3 — Test the Lambda**

Click **Test** → Create test event (any JSON).

Example:

```json
{
  "name": "Naveen"
}
```

Output:

    Hello from Lambda!

***

### 🎯 **Outcome:**

You have created and executed your first Lambda function.

***
# SECTION 3: Hands‑On Exercise #2

Access S3 from Lambda\*\*

This teaches you how Lambda interacts with other AWS services.

***

## **Step 1 — Create S3 Bucket**

    lambda-demo-bucket-12345

Upload a file:

    sample.txt

***

## **Step 2 — Add S3 Read Permissions to Lambda Role**

Attach this policy:

    AmazonS3ReadOnlyAccess

***

## **Step 3 — Update Lambda Code to Read S3 File**

### **Python Example**

```python
import boto3

s3 = boto3.client("s3")

def lambda_handler(event, context):
    bucket = "lambda-demo-bucket-12345"
    key = "sample.txt"

    response = s3.get_object(Bucket=bucket, Key=key)
    content = response["Body"].read().decode("utf-8")

    return {
        "file_content": content
    }
```

***

### 🎯 **Outcome:**

Your Lambda now reads objects from S3 — a very common serverless pattern.

***
# SECTION 4: Hands‑On Exercise #3

Trigger Lambda When a File Is Uploaded (Event‑Driven)\*\*

This is one of the most common architectures:  
**S3 → Lambda** workflow.

***

## **Step 1 — Open S3 Bucket → Properties → Event Notifications**

Name:

    on-upload-trigger

Event:

    PUT (ObjectCreated)

Destination:

    Lambda function → hello-lambda

***

## **Step 2 — Update Lambda Code to Log the Event**

### **Node.js Example**

```javascript
exports.handler = async (event) => {
  console.log("New file uploaded:");
  console.log(JSON.stringify(event));
};
```

***

### 🎯 **Outcome:**

Your Lambda is now event-driven — key serverless skill.

***
# 📘 **SECTION 5: Lambda With DynamoDB (CRUD Example)**

You’ll use this heavily with API Gateway later.

***

## **Example: Write Item to DynamoDB**

### **Step 1 — Create Table**

Name:

    Orders

Partition key:

    orderId (String)

***

## **Step 2 — Give Lambda DynamoDB Permissions**

Attach:

    AmazonDynamoDBFullAccess (for testing)

In production → use least‑privilege CRUD policies.

***

## **Step 3 — Lambda Code**

### **Node.js Example**

```javascript
const AWS = require("aws-sdk");
const db = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
  const item = {
    orderId: "1001",
    product: "Laptop",
    amount: 1200
  };

  await db.put({
    TableName: "Orders",
    Item: item
  }).promise();

  return { statusCode: 200, body: "Order saved" };
};
```

***

### 🎯 **Outcome:**

You now know how to integrate Lambda with DynamoDB.

***
# 🧰 **SECTION 6: Lambda Environment Variables**

### Step 1 — Configure Environment Variables

Go to:

    Configuration → Environment variables

Add:

    STAGE=dev
    LOG_LEVEL=debug

***

### Step 2 — Use in Lambda

Python:

```python
import os

def lambda_handler(event, context):
    stage = os.environ["STAGE"]
    return {"stage": stage}
```

***
# 🔧 **SECTION 7: Lambda Performance Optimization**

✔ Increase memory for better CPU & speed  
✔ Use **Provisioned Concurrency** for low-latency APIs  
✔ Use AWS Lambda Power Tuning (optional)  
✔ Keep dependencies minimal  
✔ Cache clients outside handler (very important!)

Example:

```python
s3 = boto3.client("s3")  # outside handler
```

***
# 🔐 **SECTION 8: Developer Security Best Practices**

✔ Never hardcode secrets → use Secrets Manager  
✔ Assign least-privilege IAM permissions  
✔ Enable CloudWatch logging  
✔ Use Lambda Layers for shared libraries  
✔ Remove unused permissions  
✔ Use VPC only when necessary (keeps cold starts low)

***
# 🎓 **SECTION 9: Step‑3 Final Assignment (Mini Project)**

Build a production-ready serverless workflow:

***

## **Requirements**

1️⃣ Create Lambda function  
2️⃣ Trigger Lambda via S3 upload  
3️⃣ Read file from S3  
4️⃣ Parse content  
5️⃣ Insert data into DynamoDB  
6️⃣ Log everything to CloudWatch  
7️⃣ Restrict bucket access to Lambda role only  
8️⃣ Use environment variables for config

***

## **Expected Architecture**

    S3 → Lambda → DynamoDB → CloudWatch

***

# 🎯 **Next Step Ready**

You now understand how Lambda works with:

*   S3
*   DynamoDB
*   Events
*   Environment configs
*   IAM

***