***

# AWS IAM (Identity & Access Management)

IAM is the foundation of everything you will do in AWS.

***

# 🎯 **Learning Objectives**

✔ What IAM users, groups, and roles are  
✔ What IAM policies are and how to write them  
✔ How authentication & authorization work  
✔ What resource‑based policies are  
✔ How service roles and trust policies work  
✔ Best practices developers must follow  
✔ How to create IAM roles and attach them to services  
✔ How to secure S3 using IAM

And you will complete **3 hands‑on tasks** with real IAM usage scenarios.

***

# 🧠 **SECTION 1: IAM Key Concepts (Simple Explanation)**

## **1. IAM User**

A person or application that logs in using long-term credentials  
(Access key + Secret key).

**When used?**  
Local development from your laptop.

***

## **2. IAM Group**

A collection of users that share permissions.

**When used?**  
Teams (e.g., Developers, DevOps, Auditors).

***

## **3. IAM Role**

*Temporary* credentials that AWS services assume.

Examples:

*   Lambda execution role
*   EC2 instance role
*   CloudFormation service role
*   API Gateway role

***

## **4. IAM Policy**

A JSON document that defines permissions.

Example:

```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject", "s3:GetObject"],
  "Resource": "arn:aws:s3:::mybucket/*"
}
```

***

## **5. Resource-Based Policy**

Attached *directly to a resource*, not a user.

Examples:

*   S3 Bucket Policy
*   Lambda function policy
*   API Gateway resource policy
*   SQS Queue policy

***

## **6. Trust Policy**

Defines **who can assume a role**.

Example (Lambda execution role trust policy):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

***

# **SECTION 2: Hands‑On Exercise #1 — Create IAM User with Least Privilege**

### **Objective:**

Create a user for local development with *minimum* permissions.

***

## **Step‑by‑Step**

### **Step 1 — Go to IAM Console**

AWS Console → **IAM** → Users → "Create User"

***

### **Step 2 — Create user**

Name:

    developer-user

Check: **Access key – programmatic access**

***

### **Step 3 — Assign permissions**

Choose: **Attach policies directly**

Attach only:

*   *AmazonS3ReadOnlyAccess*
*   *CloudWatchReadOnlyAccess*

Do **NOT** attach full-admin permissions.

***

### **Step 4 — Save credentials**

Download `.csv`.

***

### **Step 5 — Configure on your machine**

```bash
aws configure
```

Enter your access key + secret key.

***

### 🎯 **Outcome:**

You now have a safe developer account with least-privilege access.

***

# **SECTION 3: Hands‑On Exercise #2 — Create IAM Role for Lambda**

### **Objective:**

Create a Lambda execution role to allow Lambda to log into CloudWatch.

***

## **Step‑by‑Step**

### **Step 1 — IAM → Roles → Create role**

Choose trusted entity:  
✔ **AWS Service**  
Service: **Lambda**

***

### **Step 2 — Attach policy**

Choose AWS managed policy:  
✔ *AWSLambdaBasicExecutionRole*

This allows:

*   CloudWatch logging
*   Write logs
*   Create log groups

***

### **Step 3 — Name role**

    lambda-basic-execution-role

***

### **Step 4 — Test**

Go to Lambda → Create function → Select this IAM role.

***

### Behind the scenes:

AWS created this trust policy:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "lambda.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

***

# ------------------------------------

# **SECTION 4: Hands‑On Exercise #3 — Create an S3 Bucket Policy That Allows Only Your Role**

### **Objective:**

Restrict an S3 bucket so *only* the Lambda role can access it.

***

## **Step‑by‑Step**

### **Step 1 — Create S3 bucket**

Name:

    secure-app-bucket-123

***

### **Step 2 — Get the IAM Role ARN**

IAM → Roles → select your role → copy ARN.

Example:

    arn:aws:iam::123456789012:role/lambda-basic-execution-role

***

### **Step 3 — Add Bucket Policy**

S3 → Permissions → Bucket policy → Paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/lambda-basic-execution-role"
      },
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::secure-app-bucket-123/*"
    }
  ]
}
```

***

### 🎯 **Outcome:**

Even if someone has S3 permissions, they **cannot** access this bucket unless they assume the Lambda role.

This is real-world, best-practice security.

***

# **SECTION 5: IAM Policies — Real Examples You Should Know**

### **1. Allow Lambda to access DynamoDB**

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:PutItem",
    "dynamodb:GetItem"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
}
```

***

### **2. Allow CloudFront to read S3 via OAC**

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudfront.amazonaws.com"
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::mybucket/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/ABC123XYZ"
    }
  }
}
```

***

### **3. Deny S3 access unless request uses HTTPS**

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::mybucket/*",
  "Condition": {
    "Bool": {
      "aws:SecureTransport": false
    }
  }
}
```

***

# 🔐 **SECTION 6: IAM Best Practices (Developer Focus)**

✔ Use **roles**, not access keys  
✔ Store secrets in **AWS Secrets Manager**  
✔ Use **MFA** for console access  
✔ Enable **AWS Organizations SCP** for guardrails  
✔ Rotate access keys every 90 days  
✔ Restrict S3 access with OAC, not public access  
✔ Always use **least privilege** policies  
✔ Use resource-level constraints (e.g., only allow specific bucket/table)

***

# **SECTION 7: Step‑1 Final Assignment (Mini Project)**

Build a fully secure IAM configuration for a real serverless app.

### **You must create:**

1️⃣ Developer IAM user with restricted permissions  
2️⃣ Lambda execution role  
3️⃣ S3 bucket policy restricting access to Lambda role  
4️⃣ DynamoDB policy for Lambda  
5️⃣ Enable CloudWatch logging

### **End Result:**

You complete all IAM basics needed for:

*   Lambda
*   API Gateway
*   CloudFront
*   Route 53
*   WAF
*   DynamoDB
*   S3

***
