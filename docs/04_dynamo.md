# STEP 4 — Amazon DynamoDB

DynamoDB is AWS’s fully managed NoSQL database designed for **massive scale, low-latency, serverless applications**.

You will use DynamoDB with:

*   Lambda
*   API Gateway
*   S3
*   EventBridge
*   CloudWatch

***

# 🧠 **SECTION 1: DynamoDB Core Concepts (Simple Explanation)**

## **1. Tables**

Equivalent to a NoSQL “collection.”

Each table has:

*   **Primary key** (partition key or partition + sort key)
*   Items (rows)
*   Attributes (columns)

***

## **2. Partition Key (HASH Key)**

Determines **where** the item is stored.

Example:

    userId = "123"

***

## **3. Sort Key (RANGE Key)**

(Optional) Determines **ordering** inside a partition.

Example:

    orderId = "A-1001"

Together:

    PK: userId
    SK: orderId

***

## **4. Access Patterns (MOST IMPORTANT)**

DynamoDB performance depends on designing tables **based on how you query the data**, not how you store it.

Examples:

*   Get order by orderId
*   List all orders by user
*   Get all items by date

***

## **5. Capacity Modes**

*   **On‑Demand** (recommended early)
*   **Provisioned** (cheaper at scale)

***

## **6. GSIs (Global Secondary Indexes)**

Indexes you create to support additional queries.

Example:
Query orders by status:

    GSI1PK = status

***

## **7. DynamoDB Streams**

Change-log of modifications.

Used for:

*   Triggering Lambda
*   Event tracking
*   Replication

***

# ------------------------------------

# 🧪 \*\*SECTION 2: Hands‑On Exercise #1

Create DynamoDB Table (Partition Key Only)\*\*

###

## **Step 1 — Go to DynamoDB → Create Table**

Table name:

    Users

Partition key:

    userId (String)

Capacity:

    On-demand

***

## **Step 2 — Insert an Item (Manual)**

Click **Explore Table Items → Create Item**

Example item:

```json
{
  "userId": "U1001",
  "name": "Naveen",
  "role": "Director",
  "active": true
}
```

***

### 🎯 **Outcome:**

You’ve created your first DynamoDB table with a simple key.

***

# ------------------------------------

# 🧪 \*\*SECTION 3: Hands‑On Exercise #2

Create Table With Partition + Sort Key (For Real Apps)\*\*

### **Use Case:** Orders per customer.

***

## **Step 1 — Create Table**

Table name:

    Orders

Partition key:

    customerId (String)

Sort key:

    orderId (String)

***

## **Step 2 — Insert Multiple Items**

```json
{
  "customerId": "C101",
  "orderId": "O5001",
  "amount": 199,
  "status": "PENDING"
}
```

```json
{
  "customerId": "C101",
  "orderId": "O5002",
  "amount": 450,
  "status": "SHIPPED"
}
```

```json
{
  "customerId": "C102",
  "orderId": "O9001",
  "amount": 80,
  "status": "PENDING"
}
```

***

## **Step 3 — Query Items**

Choose:

    customerId = C101

You get:

*   O5001
*   O5002

***

### 🎯 **Outcome:**

You now know how to design and query relational-style data in DynamoDB.

***

# ------------------------------------

# 🧪 \*\*SECTION 4: Hands‑On Exercise #3

CRUD Using Lambda → DynamoDB\*\*

This is the **most common serverless pattern**.

We'll implement:

*   Create item
*   Read item
*   Update item
*   Delete item

Using Node.js or Python.

***

## **Step 1 — Create Lambda Function**

Name:

    orders-crud-handler

Attach IAM Policy:

    AmazonDynamoDBFullAccess (training)

***

## **Step 2 — Add CRUD Example Code**

### **Node.js Example**

```javascript
const AWS = require("aws-sdk");
const db = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
    const action = event.action;

    if (action === "create") {
        const item = event.item;
        await db.put({
            TableName: "Orders",
            Item: item
        }).promise();
        return { message: "Order created" };
    }

    if (action === "read") {
        const result = await db.get({
            TableName: "Orders",
            Key: {
                customerId: event.customerId,
                orderId: event.orderId
            }
        }).promise();
        return result.Item;
    }

    if (action === "delete") {
        await db.delete({
            TableName: "Orders",
            Key: {
                customerId: event.customerId,
                orderId: event.orderId
            }
        }).promise();
        return { message: "Order deleted" };
    }

    return { error: "Invalid action" };
};
```

***

### **Test Events**

#### Create order

```json
{
  "action": "create",
  "item": {
    "customerId": "C101",
    "orderId": "O4444",
    "amount": 250
  }
}
```

#### Read order

```json
{
  "action": "read",
  "customerId": "C101",
  "orderId": "O4444"
}
```

#### Delete order

```json
{
  "action": "delete",
  "customerId": "C101",
  "orderId": "O4444"
}
```

***

### 🎯 **Outcome:**

You just built an API-compatible CRUD layer for DynamoDB.

***

# ------------------------------------

# 🔍 **SECTION 5: DynamoDB Streams + Lambda**

Streams capture table changes.

***

## **Step-by-Step**

### **Step 1 — Enable Streams**

DynamoDB → Table → Exports & streams  
Enable:

    New and old images

***

### **Step 2 — Add Lambda Trigger**

Choose:

    Lambda function: orders-stream-reader

***

### **Step 3 — Add Code to Process Stream Events**

```javascript
exports.handler = async (event) => {
    event.Records.forEach(record => {
        console.log("Operation:", record.eventName);
        console.log("New item:", record.dynamodb.NewImage);
        console.log("Old item:", record.dynamodb.OldImage);
    });
};
```

***

### 🎯 **Outcome:**

You now have an event-driven system that reacts to database updates.

***

# ------------------------------------

# 📘 **SECTION 6: Global Secondary Index (GSI) Example**

### **Use Case:**

Query all orders by status.

***

## **Step 1 — Create GSI**

Index name:

    GSI1

Partition key:

    status (String)

Sort key:

    orderId (String)

***

## **Step 2 — Query Using Lambda**

```javascript
const result = await db.query({
    TableName: "Orders",
    IndexName: "GSI1",
    KeyConditionExpression: "status = :s",
    ExpressionAttributeValues: { ":s": "PENDING" }
}).promise();
```

***

### 🎯 **Outcome:**

You can now use GSIs to support additional access patterns.

***

# ------------------------------------

# 🔐 **SECTION 7: DynamoDB Best Practices (Developer‑Focused)**

✔ Always design tables using **access patterns first**  
✔ Prefer **On-Demand mode** early on  
✔ Use **GSIs** to support extra queries  
✔ Never scan full table — use Query instead  
✔ Use DynamoDB Streams for event-driven pipelines  
✔ Use TTL (Time To Live) for auto-expiring data  
✔ Keep items small (< 400 KB)  
✔ Use **single‑table design** for large applications

***

# ------------------------------------

# 🎓 **SECTION 8: Step‑4 Final Assignment (Mini Project)**

Build a fully functional order management backend using:

### **Architecture**

    API Gateway → Lambda → DynamoDB → DynamoDB Streams → Lambda

***

## **Required Features**

✔ Create Orders  
✔ Read Orders  
✔ List Orders by customer  
✔ List Orders by status (via GSI)  
✔ Process order updates using Streams  
✔ Send logs to CloudWatch

***