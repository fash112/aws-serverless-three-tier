# Data Tier — Data Retrieval with Lambda + DynamoDB

**Stage 3 of 3** · Services: Amazon DynamoDB, AWS Lambda, IAM
**Build time:** ~1 hour 20 minutes

---

## Goal

Build the data layer of the three-tier application: a database holding user records, and a function that retrieves the right record when the logic tier asks for it.

---

## Architecture

```
   API Gateway (logic tier)
         │
         │  event containing userId
         ▼
   Lambda function
   ├─ extracts userId from the triggered event
   ├─ queries DynamoDB by partition key
   ├─ handles errors
   └─ execution role → custom inline IAM policy
         │
         ▼
   DynamoDB table
   └─ partition key: userId
      example item: { userId, name, email }
```

---

## Design decisions

**Why DynamoDB.** It is a NoSQL database — fast at data retrieval and flexible in how data is stored.

**Partition key.** `userId` was chosen as the partition key, making it the identifier that uniquely locates each record. Because the application's only access pattern is "fetch one user by ID", a direct key lookup is exactly the right shape. Every read hits a single partition.

**Schemaless storage.** DynamoDB does not enforce a fixed schema, so new attributes can be added to a record without affecting any other record. Each item carries its own set of attributes independently. That flexibility is useful for evolving user profiles, but it moves the burden of consistency into the application layer — nothing in the database prevents one record from having fields another lacks.

---

## The function

Lambda runs code without any servers to manage — the reason it is called serverless, or function as a service.

The function does three things:

1. Takes the `userId` from the triggering event — in practice, a user submitting an ID through a form or field on the website
2. Queries DynamoDB for the item matching that `userId`
3. Handles errors, so a failure returns something meaningful rather than an unhandled exception

Testing the function directly returned the expected record — `userId`, `name`, and `email` for the test user — confirming the query and the permissions were both correct before the API was pointed at it.

---

## The hardest part: writing a custom inline IAM policy

This was new, and it was the most challenging piece of the build.

The choice sits between AWS managed policies, which are pre-written and broad, and inline policies, which are written specifically for one identity. Managed policies are convenient but grant more than a single-purpose function needs. Writing an inline policy meant deciding exactly which DynamoDB actions the function's execution role could perform and on which table.

That principle — grant the narrowest permission that makes the thing work — is the same one that appeared in the presentation tier, where the S3 bucket stayed private and CloudFront was given precisely one path in via OAC. Different service, identical reasoning.

Getting this right also resolved the error seen at the end of stage 2, where the API failed because the function had no permission to reach the database.

---

## What I'd add before this carried real traffic

- **Least-privilege review** of the inline policy, scoped to the specific table ARN rather than any broader resource pattern
- **Encryption at rest** with a customer-managed KMS key, if the data warranted it
- **Point-in-time recovery** enabled on the table
- **Input validation** on `userId` before it reaches the query
- **Access pattern design** — the current model supports exactly one query. Any second access pattern (list users, search by email) would need a global secondary index, and that design should happen before the table grows, not after.
- **Structured logging** of query latency and failures, rather than default output
