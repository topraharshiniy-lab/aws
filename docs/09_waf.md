# AWS WAF

AWS WAF protects your applications from:

*   SQL Injection
*   XSS attacks
*   Bad bots
*   DDoS Layer‑7 attacks
*   Brute force attempts
*   Rate-based abuse
*   IP reputation threats

WAF integrates with:

*   **CloudFront** (most common)
*   **API Gateway (REST & HTTP)**
*   **Application Load Balancer**
*   **AppSync**

This is a **critical** step for production-ready architectures.

***

# ------------------------------------

# 🧠 **SECTION 1: WAF Core Concepts (Simple Explanation)**

## **1. Web ACL (Access Control List)**

A Web ACL is your main security policy.

It contains:

*   Rules
*   Rule groups
*   Default action (Allow or Block)
*   Logging settings

Example Web ACL:

    "block-bad-bots"  
    "limit-500-requests"  
    "AWS Managed Rules"

***

## **2. Rules**

Individual protections, such as:

*   Block certain countries
*   Allow only certain IPs
*   Detect SQL injection
*   Detect XSS
*   Rate-limit traffic
*   Block bot signatures

***

## **3. Rule Groups**

A reusable group of rules.

Types:

*   **AWS Managed Rule Groups** (recommended)
*   **AWS Marketplace Rule Groups**
*   **Custom Rule Groups**

***

## **4. Managed Rule Groups**

AWS provides prebuilt, constantly updated protections:

Examples:

*   *AWSManagedRulesCommonRuleSet*
*   *AWSManagedRulesSQLiRuleSet*
*   *AWSManagedRulesKnownBadInputsRuleSet*
*   *AWSManagedRulesBotControlRuleSet*

Turn these on for enterprise-grade protection.

***

## **5. Rate‑Based Rules**

Limits how many requests a single IP can send.

Example:

    Block if more than 1000 requests per 5 minutes

Essential for API Gateway and CloudFront.

***

## **6. Scope**

WAF can be attached to:

*   CloudFront (global scope)
*   Regional services (API Gateway, ALB)

***

# ------------------------------------

# 🧪 \*\*SECTION 2: Hands‑On Exercise #1

Create a WAF Web ACL and Attach to CloudFront\*\*

### **Goal:**

Secure your CloudFront distribution with AWS Managed Rules + rate limiting.

***

## **Step 1 — Create Web ACL**

AWS Console → WAF → **Create Web ACL**

Name:

    cf-global-web-acl

Scope:

    CloudFront

Default action:

    Allow

***

## **Step 2 — Add AWS Managed Rule Groups (Recommended)**

Add:

*   **AWSManagedRulesCommonRuleSet** (must-have)
*   **AWSManagedRulesKnownBadInputsRuleSet**
*   **AWSManagedRulesSQLiRuleSet**
*   **AWSManagedRulesAnonymousIpList**
*   **AWSManagedRulesBotControlRuleSet (Standard)**

***

## **Step 3 — Add Rate-Based Rule**

Create rule:

    Name: rate-limit-1000
    Type: Rate-based
    Limit: 1000 requests per 5 minutes per IP
    Action: Block

***

## **Step 4 — Attach to CloudFront**

Under Resources → Add resource →  
Select your CloudFront distribution.

***

## **Step 5 — Save and Deploy**

***

### 🎯 **Outcome:**

Your CloudFront distribution is now protected by a global Web ACL.

***

# ------------------------------------

# 🧪 \*\*SECTION 3: Hands‑On Exercise #2

Attach WAF to API Gateway (Regional)\*\*

### **Goal:**

Protect your backend API from attacks.

***

## **Step 1 — Create a New Regional Web ACL**

Scope:

    Regional

Resource type:

    API Gateway

***

## **Step 2 — Add Security Rules**

Add:

*   SQL injection rule group
*   Known bad inputs
*   Anonymous IP blocking
*   Rate-limit rule (e.g., 200 req/5min/IP)

***

## **Step 3 — Attach to API Gateway**

Resource → choose your API → Stage: `prod`

***

### 🎯 **Outcome:**

Your API Gateway is now protected against attacks & abuse.

***

# ------------------------------------

# 🧪 \*\*SECTION 4: Hands‑On Exercise #3

Create Custom WAF Rules\*\*

## **1. Block a specific country**

Example:

    Country: Russia (RU)
    Action: Block

***

## **2. Allow only traffic from US**

    Allow: US
    Block: Everyone else

***

## **3. Block specific IPs**

Create IP Set:

    203.0.113.10
    198.51.100.22

Rule:

    If IP in IPSet → Block

***

## **4. Block requests with suspicious user-agent**

Custom rule:

    If header 'User-Agent' contains "curl" → Block

***

### 🎯 **Outcome:**

You now know how to write custom tailored WAF rules.

***

# ------------------------------------

# 🧪 **SECTION 5: Logging WAF Traffic (Critical for Real Apps)**

WAF logs give you:

*   attack details
*   blocked IPs
*   bot activity
*   threat patterns

***

## **Step 1 — Create S3 Bucket or Firehose Stream**

For example:

    waf-logs-myapp

***

## **Step 2 — Enable Logging**

WAF → Web ACL → Logging

Destination:

    Kinesis Firehose → S3

***

## **Step 3 — Analyze Logs Using Athena**

Use SQL queries to find patterns.

Example:

    SELECT httprequest.clientip, count(*) 
    FROM waf_logs 
    WHERE action = 'BLOCK'
    GROUP BY httprequest.clientip
    ORDER BY 2 desc;

***

### 🎯 **Outcome:**

You can monitor attackers, bots, and blocked requests.

***

# ------------------------------------

# 🧠 **SECTION 6: Important WAF Rule Examples (Copy/Paste)**

### **Limit POST requests (anti‑spam)**

    If Method = POST and
    Rate > 100 requests/min → Block

***

### **Block requests without User-Agent**

    If missing header "User-Agent" → Block

***

### **Block URL paths**

Block these patterns:

    /admin  
    /wp-login  
    /phpmyadmin  

***

### **Block common SQLi patterns**

AWS Managed Rules already covers:

*   `' OR '1'='1`
*   `UNION SELECT`
*   injection attempts

***

# ------------------------------------

# 🔐 **SECTION 7: WAF Best Practices (Production-Grade)**

✔ Use AWS Managed Rules ALWAYS  
✔ Enable full logging using Firehose  
✔ Add rate-limiting for APIs  
✔ Put WAF in front of **CloudFront**, not API Gateway alone  
✔ Add bot control  
✔ Use allow‑list rules for admin endpoints  
✔ Test blocking rules in “Count” mode first  
✔ Don’t rely only on WAF → validate inputs in Lambda

***

# ------------------------------------

# 🎓 **SECTION 8: Step‑9 Final Assignment (Mini Project)**

Build a production-ready WAF security layer:

***

## **Requirements**

### **1. Create Web ACL for CloudFront**

*   AWS managed rule groups
*   Bot Control
*   SQLi rule
*   Known bad IP rule
*   Rate-limit rule (1000 req/5 min)

### **2. Create separate Web ACL for API Gateway**

*   Lower rate limits
*   Block anonymous IPs
*   Block countries
*   Logging enabled

### **3. Build a Kinesis Firehose → S3 pipeline**

*   Store WAF logs
*   Run Athena queries

### **4. Report metrics in CloudWatch Dashboard**

*   Requests blocked
*   Rate-limit triggers
*   Top attack sources

***

## **Expected Final Architecture**

    Route 53
         ↓
    CloudFront → WAF (Global)
         ↓
          SPA on S3

    Route 53
         ↓
    API Gateway → WAF (Regional)
         ↓
    Lambda → DynamoDB

***

### 🎯 **End Result:**

You now have an enterprise-grade WAF security posture for your serverless architecture.

***