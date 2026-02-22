# Project - Build **production‑ready, serverless web application**

> **What you’ll build**
>
> *   React SPA on **S3** behind **CloudFront** + **WAF** + **Route 53**
> *   **API Gateway → Lambda → DynamoDB** backend
> *   **SQS/SNS/EventBridge** for async & notifications
> *   **CloudWatch** logs/alarms/dashboards
> *   **Terraform** modules, variables, and outputs
> *   **CI/CD** friendly

***

## 🏗️ Architecture (recap)

    Route 53 (DNS)
         ↓
    CloudFront (CDN) → WAF (Global Web ACL)
         ↓
    S3 (private; OAC-secured)  ← static SPA

    Route 53 (api.yourdomain.com)
         ↓
    API Gateway (HTTP API)
         ↓
    Lambda (Node/Python)
         ↓
    DynamoDB (single-table + GSI)
         ↓
    DynamoDB Streams → Lambda (processor)

    Async:
    Lambda → SQS (order-processing-queue) → Worker Lambda → SNS (topic/email)
    EventBridge (cron) → Lambda (maintenance)

    Observability:
    CloudWatch Logs | Metrics | Alarms | Dashboards
    WAF logs → Kinesis Firehose → S3 → Athena (optional)

***

## 🧩 Repository layout (Terraform)

    infra/
      main.tf
      providers.tf
      variables.tf
      outputs.tf
      backend.tf                  # Terraform state backend (S3 + DynamoDB recommended)
      modules/
        s3_cloudfront/
          main.tf  variables.tf  outputs.tf
        api_lambda_dynamodb/
          main.tf  variables.tf  outputs.tf
        sqs_sns_eventbridge/
          main.tf  variables.tf  outputs.tf
        waf/
          main.tf  variables.tf  outputs.tf
        route53/
          main.tf  variables.tf  outputs.tf
    lambda/
      orders/index.js
      worker/index.js
      maintenance/index.js
    frontend/
      # your SPA build artifacts (deployed via CI to S3)

> You can apply each module independently per environment (`dev`, `prod`) using workspaces or variable files (`-var-file=dev.tfvars`).

***

# 1) Terraform Providers & Remote State

**`infra/providers.tf`**

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

**`infra/backend.tf`** (recommended)

```hcl
terraform {
  backend "s3" {
    bucket         = "tf-state-yourcompany"
    key            = "serverless-capstone/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-locks"
    encrypt        = true
  }
}
```

**`infra/variables.tf`**

```hcl
variable "aws_region" { type = string, default = "us-east-1" }

variable "domain_name"        { type = string }                  # e.g., "yourdomain.com"
variable "subdomain_api"      { type = string, default = "api" } # "api.yourdomain.com"
variable "subdomain_www"      { type = string, default = "www" } # "www.yourdomain.com"

variable "environment"        { type = string, default = "dev" }

# email to subscribe to SNS
variable "alert_email"        { type = string }

# Optional: WAF allowlist IPs or blocked countries, etc.
variable "waf_rate_limit"     { type = number, default = 1000 }  # per 5 minutes per IP
```

**`infra/outputs.tf`**

```hcl
output "cloudfront_domain"    { value = module.s3_cloudfront.cloudfront_domain_name }
output "api_base_url"         { value = module.api_lambda_dynamodb.http_api_endpoint }
output "api_custom_domain"    { value = module.route53.api_domain_fqdn }
output "www_domain"           { value = module.route53.www_domain_fqdn }
```

***

# 2) Frontend: S3 (private) + CloudFront (OAC) + WAF + ACM + Route 53

### Module: `modules/s3_cloudfront`

**`modules/s3_cloudfront/variables.tf`**

```hcl
variable "domain_name"     { type = string }
variable "www_record"      { type = string } # e.g., "www"
variable "env"             { type = string }
variable "hosted_zone_id"  { type = string } # Route53 hosted zone ID
variable "waf_acl_arn"     { type = string } # global scope WAF ACL for CloudFront
```

**`modules/s3_cloudfront/main.tf`**

```hcl
locals { bucket_name = "spa-${var.env}-${var.domain_name}" }

resource "aws_s3_bucket" "spa" {
  bucket = local.bucket_name
  force_destroy = true
}

# Block public access
resource "aws_s3_bucket_public_access_block" "spa" {
  bucket                  = aws_s3_bucket_spa.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Origin Access Control (OAC) for CloudFront
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "oac-${var.env}-${var.domain_name}"
  description                       = "OAC for S3"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# ACM cert in us-east-1 for CloudFront
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "cert" {
  provider          = aws.us_east_1
  domain_name       = "${var.www_record}.${var.domain_name}"
  validation_method = "DNS"
  subject_alternative_names = [var.domain_name]

  lifecycle {
    create_before_destroy = true
  }
}

# DNS validation records
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      record = dvo.resource_record_value
    }
  }
  zone_id = var.hosted_zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = 60
  records = [each.value.record]
}

resource "aws_acm_certificate_validation" "cert_validation" {
  provider        = aws.us_east_1
  certificate_arn = aws_acm_certificate.cert.arn
  validation_record_fqdns = [for r in aws_route53_record.cert_validation : r.fqdn]
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "this" {
  enabled             = true
  comment             = "SPA ${var.env}"
  default_root_object = "index.html"
  web_acl_id          = var.waf_acl_arn

  origin {
    domain_name              = aws_s3_bucket.spa.bucket_regional_domain_name
    origin_id                = "s3-spa"
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
  }

  default_cache_behavior {
    target_origin_id       = "s3-spa"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    compress = true

    cache_policy_id = "658327ea-f89d-4fab-a63d-7e88639e58f6" # CachingOptimized managed policy
  }

  price_class = "PriceClass_100"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn            = aws_acm_certificate_validation.cert_validation.certificate_arn
    ssl_support_method             = "sni-only"
    minimum_protocol_version       = "TLSv1.2_2021"
  }

  aliases = ["${var.www_record}.${var.domain_name}", var.domain_name]
}

# Bucket policy to allow CloudFront via OAC
data "aws_caller_identity" "current" {}
resource "aws_s3_bucket_policy" "spa" {
  bucket = aws_s3_bucket.spa.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Sid       = "AllowCloudFrontOAC"
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = ["s3:GetObject"]
      Resource  = ["${aws_s3_bucket.spa.arn}/*"]
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.this.arn
        }
      }
    }]
  })
}

# Route53 alias
resource "aws_route53_record" "www_alias" {
  zone_id = var.hosted_zone_id
  name    = "${var.www_record}.${var.domain_name}"
  type    = "A"
  alias {
    name                   = aws_cloudfront_distribution.this.domain_name
    zone_id                = aws_cloudfront_distribution.this.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "apex_alias" {
  zone_id = var.hosted_zone_id
  name    = var.domain_name
  type    = "A"
  alias {
    name                   = aws_cloudfront_distribution.this.domain_name
    zone_id                = aws_cloudfront_distribution.this.hosted_zone_id
    evaluate_target_health = false
  }
}

output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.this.domain_name
}
```

**`modules/s3_cloudfront/outputs.tf`**

```hcl
output "cloudfront_domain_name" { value = aws_cloudfront_distribution.this.domain_name }
```

***

# 3) Backend: API Gateway (HTTP API) + Lambda + DynamoDB (+ Streams)

### Module: `modules/api_lambda_dynamodb`

**`modules/api_lambda_dynamodb/variables.tf`**

```hcl
variable "env"               { type = string }
variable "table_name"        { type = string, default = null }
variable "create_table"      { type = bool, default = true }
variable "lambda_zip_orders" { type = string }  # path to zip
variable "lambda_zip_worker" { type = string }
variable "lambda_zip_maint"  { type = string }
variable "queue_url"         { type = string }
variable "topic_arn"         { type = string, default = null }
```

**`modules/api_lambda_dynamodb/main.tf`**

```hcl
# DynamoDB
resource "aws_dynamodb_table" "app" {
  count          = var.create_table ? 1 : 0
  name           = var.table_name != null ? var.table_name : "AppTable-${var.env}"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "PK"
  range_key      = "SK"
  attribute {
    name = "PK"
    type = "S"
  }
  attribute {
    name = "SK"
    type = "S"
  }

  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "ALL"
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
}

locals {
  table_name = var.create_table ? aws_dynamodb_table.app[0].name : var.table_name
}

# IAM role & Lambda packages
resource "aws_iam_role" "lambda_exec" {
  name = "lambda-exec-${var.env}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{ Effect = "Allow", Principal = { Service = "lambda.amazonaws.com" }, Action = "sts:AssumeRole" }]
  })
}

resource "aws_iam_role_policy_attachment" "basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_policy" "ddb_rw" {
  name   = "ddb-rw-${var.env}"
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect: "Allow",
      Action: ["dynamodb:*"],
      Resource: "*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ddb" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = aws_iam_policy.ddb_rw.arn
}

# Orders Lambda
resource "aws_lambda_function" "orders" {
  function_name = "orders-${var.env}"
  role          = aws_iam_role.lambda_exec.arn
  runtime       = "nodejs18.x"
  handler       = "index.handler"
  filename      = var.lambda_zip_orders
  timeout       = 10
  environment {
    variables = {
      TABLE_NAME = local.table_name
      QUEUE_URL  = var.queue_url
    }
  }
}

# Worker Lambda
resource "aws_lambda_function" "worker" {
  function_name = "worker-${var.env}"
  role          = aws_iam_role.lambda_exec.arn
  runtime       = "nodejs18.x"
  handler       = "index.handler"
  filename      = var.lambda_zip_worker
  timeout       = 30
  environment {
    variables = {
      TABLE_NAME = local.table_name
      TOPIC_ARN  = var.topic_arn
    }
  }
}

# Maintenance Lambda
resource "aws_lambda_function" "maintenance" {
  function_name = "maintenance-${var.env}"
  role          = aws_iam_role.lambda_exec.arn
  runtime       = "nodejs18.x"
  handler       = "index.handler"
  filename      = var.lambda_zip_maint
  timeout       = 30
  environment {
    variables = {
      TABLE_NAME = local.table_name
    }
  }
}

# API Gateway HTTP API
resource "aws_apigatewayv2_api" "http" {
  name          = "http-api-${var.env}"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "orders" {
  api_id                 = aws_apigatewayv2_api.http.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.orders.invoke_arn
  integration_method     = "POST"
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "orders_post" {
  api_id    = aws_apigatewayv2_api.http.id
  route_key = "POST /orders"
  target    = "integrations/${aws_apigatewayv2_integration.orders.id}"
}

resource "aws_apigatewayv2_route" "orders_get" {
  api_id    = aws_apigatewayv2_api.http.id
  route_key = "GET /orders"
  target    = "integrations/${aws_apigatewayv2_integration.orders.id}"
}

resource "aws_lambda_permission" "apigw_orders" {
  statement_id  = "AllowAPIGWInvokeOrders"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.orders.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.http.execution_arn}/*/*"
}

# CloudWatch log groups (optional explicit)
resource "aws_cloudwatch_log_group" "lambda_orders" {
  name              = "/aws/lambda/${aws_lambda_function.orders.function_name}"
  retention_in_days = 7
}

# Outputs
output "http_api_endpoint" { value = aws_apigatewayv2_api.http.api_endpoint }
output "table_name_out"    { value = local.table_name }
```

> **Zipping lambdas**: Build your code in `lambda/orders/index.js`, `lambda/worker/index.js`, etc., then zip them in CI:
>
> ```bash
> zip -j ./build/orders.zip ./lambda/orders/index.js
> zip -j ./build/worker.zip ./lambda/worker/index.js
> zip -j ./build/maintenance.zip ./lambda/maintenance/index.js
> ```

**Example Lambda code**

`lambda/orders/index.js`

```js
const AWS = require('aws-sdk');
const db = new AWS.DynamoDB.DocumentClient();
const sqs = new AWS.SQS();
const TABLE = process.env.TABLE_NAME;
const QUEUE_URL = process.env.QUEUE_URL;

exports.handler = async (event) => {
  const method = event.requestContext?.http?.method || 'GET';

  if (method === 'POST') {
    const body = JSON.parse(event.body || '{}');
    const userId = body.userId || 'U1001';
    const orderId = 'O' + Date.now();
    const item = {
      PK: `USER#${userId}`,
      SK: `ORDER#${orderId}`,
      GSI1PK: 'ORDERSTATUS#PENDING',
      GSI1SK: new Date().toISOString(),
      status: 'PENDING',
      total: body.total || 0,
      createdAt: new Date().toISOString()
    };

    await db.put({ TableName: TABLE, Item: item }).promise();

    await sqs.sendMessage({
      QueueUrl: QUEUE_URL,
      MessageBody: JSON.stringify({ userId, orderId })
    }).promise();

    return { statusCode: 200, body: JSON.stringify({ orderId, status: 'PENDING' }) };
  }

  // GET list by user
  const qs = event.queryStringParameters || {};
  const userId = qs.customerId || 'U1001';

  const result = await db.query({
    TableName: TABLE,
    KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues: { ':pk': `USER#${userId}`, ':sk': 'ORDER#' }
  }).promise();

  return { statusCode: 200, body: JSON.stringify(result.Items) };
};
```

`lambda/worker/index.js`

```js
const AWS = require('aws-sdk');
const db = new AWS.DynamoDB.DocumentClient();
const sns = new AWS.SNS();
const TABLE = process.env.TABLE_NAME;
const TOPIC_ARN = process.env.TOPIC_ARN;

exports.handler = async (event) => {
  for (const rec of event.Records) {
    const { userId, orderId } = JSON.parse(rec.body);
    await db.update({
      TableName: TABLE,
      Key: { PK: `USER#${userId}`, SK: `ORDER#${orderId}` },
      UpdateExpression: 'SET #s = :s, GSI1PK = :g',
      ExpressionAttributeNames: { '#s': 'status' },
      ExpressionAttributeValues: { ':s': 'CONFIRMED', ':g': 'ORDERSTATUS#CONFIRMED' }
    }).promise();

    if (TOPIC_ARN) {
      await sns.publish({
        TopicArn: TOPIC_ARN,
        Message: JSON.stringify({ userId, orderId, status: 'CONFIRMED' })
      }).promise();
    }
  }
  return {};
};
```

`lambda/maintenance/index.js`

```js
exports.handler = async () => {
  console.log('Daily maintenance');
  return {};
};
```

***

# 4) Messaging & Scheduling: SQS + SNS + EventBridge

### Module: `modules/sqs_sns_eventbridge`

**`modules/sqs_sns_eventbridge/variables.tf`**

```hcl
variable "env"        { type = string }
variable "email"      { type = string }
variable "worker_lambda_arn" { type = string }
variable "maintenance_lambda_arn" { type = string }
```

**`modules/sqs_sns_eventbridge/main.tf`**

```hcl
# SQS
resource "aws_sqs_queue" "orders" {
  name                      = "orders-processing-${var.env}"
  visibility_timeout_seconds = 60
  message_retention_seconds  = 345600
}

# Lambda event source mapping
resource "aws_lambda_event_source_mapping" "worker" {
  event_source_arn = aws_sqs_queue.orders.arn
  function_name    = var.worker_lambda_arn
  batch_size       = 1
}

# IAM permissions for worker to consume SQS
data "aws_iam_policy_document" "sqs_consume" {
  statement {
    actions   = ["sqs:ReceiveMessage", "sqs:DeleteMessage", "sqs:GetQueueAttributes"]
    resources = [aws_sqs_queue.orders.arn]
  }
}

resource "aws_iam_policy" "sqs_consume" {
  name   = "sqs-consume-${var.env}"
  policy = data.aws_iam_policy_document.sqs_consume.json
}

# SNS Topic
resource "aws_sns_topic" "order_events" {
  name = "order-events-${var.env}"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "email"
  endpoint  = var.email
}

# EventBridge schedule → maintenance Lambda
resource "aws_cloudwatch_event_rule" "daily" {
  name                = "daily-cron-${var.env}"
  schedule_expression = "cron(0 0 * * ? *)" # every day at 00:00 UTC
}

resource "aws_cloudwatch_event_target" "maint_target" {
  rule      = aws_cloudwatch_event_rule.daily.name
  target_id = "maintenance"
  arn       = var.maintenance_lambda_arn
}

# Permission for EventBridge to invoke Lambda
resource "aws_lambda_permission" "allow_events" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = var.maintenance_lambda_arn
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily.arn
}

output "queue_url"  { value = aws_sqs_queue.orders.id }
output "topic_arn"  { value = aws_sns_topic.order_events.arn }
```

***

# 5) WAF (Global for CloudFront) + Logging (optional)

### Module: `modules/waf`

**`modules/waf/variables.tf`**

```hcl
variable "env"           { type = string }
variable "rate_limit"    { type = number }
```

**`modules/waf/main.tf`**

```hcl
resource "aws_wafv2_web_acl" "global_cf" {
  name        = "cf-global-acl-${var.env}"
  description = "Global WAF for CloudFront"
  scope       = "CLOUDFRONT"

  default_action { allow {} }

  rule {
    name     = "AWS-AWSManagedRulesCommonRuleSet"
    priority = 1
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "commonRules"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "RateLimit"
    priority = 2
    action { block {} }
    statement {
      rate_based_statement {
        limit              = var.rate_limit
        aggregate_key_type = "IP"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "rateLimit"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "cfGlobalWebAcl"
    sampled_requests_enabled   = true
  }
}

output "web_acl_arn" { value = aws_wafv2_web_acl.global_cf.arn }
```

> You can extend with Bot Control, SQLi, IP sets, and logging via Kinesis Firehose → S3 if needed.

***

# 6) Route 53 (Hosted Zone + Records for www & api)

### Module: `modules/route53`

**`modules/route53/variables.tf`**

```hcl
variable "domain_name" { type = string }
```

**`modules/route53/main.tf`**

```hcl
data "aws_route53_zone" "main" {
  name         = var.domain_name
  private_zone = false
}

output "hosted_zone_id" { value = data.aws_route53_zone.main.zone_id }
output "zone_name"      { value = data.aws_route53_zone.main.name }
```

> We’ll use this data source to feed hosted zone id to the S3/CloudFront module and the API custom domain module (below).

**API Custom Domain (optional)**  
You can add an additional module or extend `api_lambda_dynamodb` to create **ACM (regional)** + **API Gateway custom domain** + **Route53 alias**. For brevity, here’s a quick inline pattern you can adapt:

```hcl
# In your root main.tf (regional certificate for API Gateway custom domain)
resource "aws_acm_certificate" "api_cert" {
  domain_name       = "${var.subdomain_api}.${var.domain_name}"
  validation_method = "DNS"
}

resource "aws_route53_record" "api_cert_val" {
  zone_id = module.route53.hosted_zone_id
  name    = aws_acm_certificate.api_cert.domain_validation_options[0].resource_record_name
  type    = aws_acm_certificate.api_cert.domain_validation_options[0].resource_record_type
  records = [aws_acm_certificate.api_cert.domain_validation_options[0].resource_record_value]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "api_cert_validation" {
  certificate_arn = aws_acm_certificate.api_cert.arn
  validation_record_fqdns = [aws_route53_record.api_cert_val.fqdn]
}

resource "aws_apigatewayv2_domain_name" "custom" {
  domain_name = "${var.subdomain_api}.${var.domain_name}"
  domain_name_configuration {
    certificate_arn = aws_acm_certificate_validation.api_cert_validation.certificate_arn
    endpoint_type   = "REGIONAL"
    security_policy = "TLS_1_2"
  }
}

resource "aws_apigatewayv2_api_mapping" "map" {
  api_id      = module.api_lambda_dynamodb.api_id         # expose api_id in that module if needed
  domain_name = aws_apigatewayv2_domain_name.custom.id
  stage       = "$default"
}

resource "aws_route53_record" "api_alias" {
  zone_id = module.route53.hosted_zone_id
  name    = "${var.subdomain_api}.${var.domain_name}"
  type    = "A"
  alias {
    name                   = aws_apigatewayv2_domain_name.custom.domain_name_configuration[0].target_domain_name
    zone_id                = aws_apigatewayv2_domain_name.custom.domain_name_configuration[0].hosted_zone_id
    evaluate_target_health = false
  }
}

output "api_domain_fqdn" { value = aws_apigatewayv2_domain_name.custom.domain_name }
```

***

# 7) Root Composition: `infra/main.tf`

This is where you wire everything together.

```hcl
module "route53" {
  source      = "./modules/route53"
  domain_name = var.domain_name
}

module "waf" {
  source      = "./modules/waf"
  env         = var.environment
  rate_limit  = var.waf_rate_limit
}

# Build Lambdas as zip files outside Terraform (CI), then pass file paths here
module "sqs_sns_eventbridge" {
  source                 = "./modules/sqs_sns_eventbridge"
  env                    = var.environment
  email                  = var.alert_email
  worker_lambda_arn      = module.api_lambda_dynamodb.worker_lambda_arn
  maintenance_lambda_arn = module.api_lambda_dynamodb.maintenance_lambda_arn
}

module "api_lambda_dynamodb" {
  source              = "./modules/api_lambda_dynamodb"
  env                 = var.environment
  lambda_zip_orders   = "./build/orders.zip"
  lambda_zip_worker   = "./build/worker.zip"
  lambda_zip_maint    = "./build/maintenance.zip"
  queue_url           = module.sqs_sns_eventbridge.queue_url
  topic_arn           = module.sqs_sns_eventbridge.topic_arn
  create_table        = true
}

module "s3_cloudfront" {
  source         = "./modules/s3_cloudfront"
  domain_name    = var.domain_name
  www_record     = var.subdomain_www
  env            = var.environment
  hosted_zone_id = module.route53.hosted_zone_id
  waf_acl_arn    = module.waf.web_acl_arn
}

# Optional: API custom domain
# (inline resources shown earlier)
```

**Expose extra outputs in `api_lambda_dynamodb/outputs.tf` as needed:**

```hcl
output "http_api_endpoint"         { value = aws_apigatewayv2_api.http.api_endpoint }
output "api_id"                    { value = aws_apigatewayv2_api.http.id }
output "worker_lambda_arn"         { value = aws_lambda_function.worker.arn }
output "maintenance_lambda_arn"    { value = aws_lambda_function.maintenance.arn }
output "table_name"                { value = local.table_name }
```

***

# 8) CI/CD (GitHub Actions example)

**Backend (Terraform)**

```yaml
name: deploy-infra
on:
  push:
    branches: [ main ]
    paths: [ 'infra/**', 'lambda/**' ]
jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Zip Lambdas
        run: |
          mkdir -p build
          zip -j ./build/orders.zip ./lambda/orders/index.js
          zip -j ./build/worker.zip ./lambda/worker/index.js
          zip -j ./build/maintenance.zip ./lambda/maintenance/index.js

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        working-directory: infra
        run: terraform init

      - name: Terraform Plan
        working-directory: infra
        run: terraform plan -out=tfplan -var="domain_name=yourdomain.com" -var="alert_email=you@company.com"

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        working-directory: infra
        run: terraform apply -auto-approve tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1
```

**Frontend**

```yaml
name: deploy-frontend
on:
  push:
    branches: [ main ]
    paths: [ 'frontend/**' ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '18' }

      - name: Build SPA
        run: |
          cd frontend
          npm ci
          npm run build

      - name: Upload to S3
        run: |
          aws s3 sync frontend/dist s3://spa-dev-yourdomain.com --delete
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation --distribution-id YOUR_DIST_ID --paths "/*"
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1
```

***

# 9) Security, Observability & Ops

**Security**

*   S3 private + OAC
*   WAF (managed rules + rate limit; add Bot Control/SQLi as needed)
*   API throttling (configure staged throttling on HTTP API if required via `aws_apigatewayv2_stage`)
*   IAM: least‑privilege (narrow the `dynamodb:*` to CRUD on your table)
*   Secrets: use **SSM Parameter Store** / **Secrets Manager** (avoid hardcoding)

**Observability**

*   CloudWatch Logs for all Lambdas
*   Alarms:
    *   Lambda `Errors > 0` (1 min)
    *   API Gateway 5xx spikes
    *   DynamoDB `ThrottledRequests > 0`
    *   SQS `ApproximateNumberOfMessagesVisible` threshold
*   Dashboard: Key graphs from Lambda/API/DynamoDB/SQS/CloudFront/WAF

**Ops**

*   Runbooks for: API errors, queue backlog, WAF false positives, cache invalidations
*   Cost controls: S3 lifecycle to Glacier for logs, budgets & alerts, CloudFront caching policies

***

# ✅ Test Plan

**Functional**

1.  `POST /orders` → returns `{ orderId, status: 'PENDING' }`
2.  Worker updates → status becomes `CONFIRMED`; SNS email triggers
3.  `GET /orders?customerId=...` returns list

**Load**

*   Use `k6`/`artillery` to POST orders at RPS 20–100
*   Watch Lambda concurrency, API latency, DDB throttles, SQS backlog

**Security**

*   Confirm S3 is not public
*   Verify WAF blocks rate‑limited bursts and logs in CloudWatch
*   Check CloudFront uses HTTPS and OAC policy is correct

***

## 🎓 You’re Done When…

*   [ ] `www.yourdomain.com` serves SPA via CloudFront (HTTPS)
*   [ ] `api.yourdomain.com` hits API Gateway (HTTPS)
*   [ ] Orders flow: API → SQS → Worker → SNS email
*   [ ] EventBridge cron invokes maintenance Lambda
*   [ ] WAF attached to CloudFront; logs/metrics visible
*   [ ] CloudWatch Alarms & Dashboard in place
*   [ ] Terraform state remote, plans reviewed, apply succeeds in CI

***
