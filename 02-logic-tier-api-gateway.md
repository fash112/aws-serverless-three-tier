# Logic Tier — REST API with Lambda + API Gateway

**Stage 2 of 3** · Services: Amazon API Gateway, AWS Lambda
**Build time:** ~1.5 hours including project extension

---

## Goal

Build the backend of the three-tier application — the layer that receives requests from the front end, decides whether to act on them, and runs the code that does the work.

---

## Architecture

```
   Front end (presentation tier)
   "Get User Data" button
         │
         │  GET /users?userId=…
         ▼
   API Gateway — REST API
   ├─ Resource:  /users
   ├─ Method:    GET
   ├─ Integration: Lambda proxy
   └─ Stage:     prod  →  invoke URL
         │
         │  event
         ▼
   Lambda function
   └─ queries DynamoDB (data tier — stage 3)
```

---

## Design decisions

**Why REST rather than HTTP or WebSocket API.** API Gateway supports REST, HTTP, and WebSocket API types. REST was chosen because it uses standard HTTP methods and is broadly compatible with most programming languages and with Lambda. WebSocket exists for persistent bidirectional connections, which this application does not need.

**Resources and methods.** A resource is a path within the API — one API might have separate paths for user data, messaging data, and product data. Each resource carries methods, which are the actions available on it. For a REST API those map to HTTP verbs: `GET` to retrieve, `POST` to create, `PUT` to edit, `DELETE` to remove.

A `GET` method was created on the users resource and wired to the Lambda function. When a request arrives at something like `api.com/users`, the API knows to pass that event to Lambda if the method is `GET`.

**What API Gateway is actually doing.** It sits in front of the function as a traffic manager: it reduces load on the backend, determines whether a request is authorized, and passes it through to Lambda only if it is. The function is never directly exposed.

**Lambda proxy integration** passes the entire request through to the function rather than mapping fields individually, which keeps the API definition simple and puts request parsing in the function code.

---

## Deployment and stages

An API is deployed to a **stage**, which is a snapshot of the API used to manage versions. This one was deployed to `prod` — the stage carrying live traffic that users can reach. Each stage has its own invoke URL.

---

## What broke

Visiting the prod invoke URL returned an error. Two causes, both real:

1. **The Lambda function had no permission to reach DynamoDB.** An unauthorized call to the data layer surfaces as a failure at the API.
2. **The DynamoDB table did not exist yet.** The function was querying a table that had not been created.

Both were resolved in stage 3, where the table was created and a scoped inline IAM policy was written for the function. The sequencing lesson is straightforward: the logic tier cannot be validated end to end until the data tier it depends on exists.

---

## Project extension: API documentation

This went beyond the guided steps.

Documentation was written directly in the API Gateway console, because an API without instructions is an API other developers cannot use — and one that likely goes unused entirely.

Documentation publishes to a specific stage, for the same reason the API itself does: different stages hold different versions of the API, so the explanations and requirements attached to each will differ.

The published and downloaded output combined the manually written descriptions with documentation API Gateway generates automatically. That export is in OpenAPI form and can be loaded into Swagger or Redoc to render browsable API reference pages.

---

## What I'd add before this carried real traffic

- **Authentication** — Cognito user pool or a Lambda authorizer. The endpoint is currently open.
- **Throttling and usage plans** — an open endpoint that invokes Lambda on every call is a cost exposure as much as an availability one.
- **Request validation** at the gateway, rejecting malformed requests before they reach the function.
- **CORS configuration** scoped to the CloudFront domain rather than left permissive.
- **A dev or staging stage** alongside prod, so changes are tested before they reach live traffic.
- **Structured access logging** and CloudWatch alarms on 4xx and 5xx rates.
