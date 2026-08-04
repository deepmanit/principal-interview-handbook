# API Design

> "A well-designed API is a contract between clients and services. Good APIs are simple, consistent, evolvable, and resilient."

---

# Why API Design Matters

Every distributed system starts with APIs.

Poor APIs lead to:

- Tight coupling
- Breaking changes
- Difficult integrations
- Poor performance
- Security vulnerabilities

Principal Engineers design APIs that can evolve for years without breaking clients.

---

# API Design Framework

For every API, think in this order:

```
Requirements
        │
        ▼
Resource Modeling
        │
        ▼
HTTP Method
        │
        ▼
Request
        │
        ▼
Response
        │
        ▼
Error Handling
        │
        ▼
Authentication
        │
        ▼
Pagination
        │
        ▼
Versioning
        │
        ▼
Rate Limiting
```

---

# Step 1 – Model Resources

Think in terms of **nouns**, not verbs.

❌ Bad

```
POST /createUser

GET /getTweets
```

✅ Good

```
POST /users

GET /tweets
```

---

# Step 2 – Choose the Correct HTTP Method

| Method | Purpose | Idempotent |
|---------|----------|------------|
| GET | Read | ✅ |
| POST | Create | ❌ |
| PUT | Replace | ✅ |
| PATCH | Partial Update | ❌ |
| DELETE | Delete | ✅ |

Example

Create Tweet

```
POST /tweets
```

Delete Tweet

```
DELETE /tweets/{id}
```

Update Profile

```
PATCH /users/{id}
```

---

# Step 3 – Request Example

```
POST /tweets
```

Request

```json
{
    "text":"Hello World",
    "visibility":"PUBLIC"
}
```

---

# Step 4 – Response Example

```json
{
    "tweetId":"12345",
    "createdAt":"2026-08-04T10:00:00Z",
    "status":"SUCCESS"
}
```

Always return meaningful metadata.

---

# HTTP Status Codes

| Code | Meaning |
|------|----------|
| 200 | OK |
| 201 | Created |
| 202 | Accepted |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 409 | Conflict |
| 429 | Too Many Requests |
| 500 | Internal Server Error |

---

# Idempotency

One of the most frequently asked Principal Engineer topics.

Definition

Calling the same request multiple times should produce the same result.

Example

```
PUT /users/123
```

Calling it twice

Produces the same state.

Good.

---

Payment APIs

Never rely only on POST.

Use

```
Idempotency-Key
```

Example

```
POST /payments

Headers

Idempotency-Key:
6b7f0d1c...
```

If the client retries after a timeout,

the server returns the original result

instead of charging twice.

---

# Pagination

Avoid returning millions of rows.

---

## Offset Pagination

```
GET /tweets?page=3&size=20
```

Pros

Simple

Cons

Slow on large datasets.

---

## Cursor Pagination

```
GET /tweets?cursor=abc123
```

Pros

- Fast
- Stable
- Preferred for social media feeds

Examples

- Twitter
- Instagram
- Facebook

---

# Filtering

```
GET /users?country=India

GET /orders?status=DELIVERED

GET /products?category=Laptop
```

---

# Sorting

```
GET /products?sort=price

GET /users?sort=name
```

---

# Versioning

Avoid breaking clients.

Common approaches

```
/v1/users

/v2/users
```

or

```
Accept:
application/vnd.company.v2+json
```

---

# Authentication

Never send tokens in query parameters.

Use

```
Authorization:

Bearer <JWT>
```

---

# Rate Limiting

Protect services.

Strategies

- Token Bucket
- Leaky Bucket
- Sliding Window
- Fixed Window

Example

```
100 Requests / Minute
```

If exceeded

Return

```
429 Too Many Requests
```

---

# Error Response

```json
{
  "errorCode":"USER_NOT_FOUND",
  "message":"User does not exist",
  "timestamp":"2026-08-04T10:00:00Z",
  "requestId":"abc123"
}
```

Always include

- Error Code
- Human-readable Message
- Request ID
- Timestamp

---

# API Evolution

Never remove fields abruptly.

Instead

```
Old Field

↓

Deprecated

↓

Monitor Usage

↓

Remove in Next Version
```

---

# Common Interview Mistakes

❌ Using verbs in URLs

❌ Returning 200 for every request

❌ No pagination

❌ No idempotency

❌ No authentication

❌ Breaking backward compatibility

---

# Principal Engineer Communication

Instead of saying

> "I'll create a REST API."

Say

> "I'll expose resource-oriented REST endpoints with consistent naming conventions, cursor-based pagination for large collections, idempotency keys for retryable write operations, JWT-based authentication, structured error responses, and explicit versioning to support backward-compatible evolution."

---

# API Design Checklist

Before finalizing an API, verify:

- Resource-oriented URLs
- Correct HTTP methods
- Consistent status codes
- Idempotency where required
- Pagination
- Filtering
- Sorting
- Authentication
- Authorization
- Rate limiting
- Structured error responses
- Versioning strategy

---

# Key Takeaways

Good APIs are not just functional.

They are:

- Easy to understand
- Safe to retry
- Backward compatible
- Secure
- Scalable
- Observable
- Designed to evolve
