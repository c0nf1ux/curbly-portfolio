# Curbly Architecture

## System Overview

Frontend (React, Vercel) → Backend (Node.js, Render) → Database (MongoDB, Atlas)

## Data Models

### Users
{
  _id: ObjectId,
  email: String (unique),
  passwordHash: String (bcrypt),
  firstName: String,
  lastName: String,
  role: Enum (customer | hauler | admin),
  profile: { phone, bio, photoUrl, accessibility },
  subscription: { tier, status, billedAt },
  createdAt: Date
}

### Jobs
{
  _id: ObjectId,
  customerId: ObjectId,
  haulerId: ObjectId,
  status: Enum (created | posted | accepted | en_route | completed | cancelled),
  address: String,
  description: String,
  estimatedCarryDistance: Number,
  estimatedBagWeight: Enum,
  calculatedPrice: Number,
  beforePhotoLocation: { latitude, longitude },
  afterPhotoLocation: { latitude, longitude },
  paymentAuthorized: Boolean,
  completedAt: Date,
  createdAt: Date
}

### DistanceHistory
{
  _id: ObjectId,
  jobId: ObjectId (unique),
  customerId: ObjectId,
  estimatedDistance: Number,
  actualDistance: Number,
  variance: Number,
  confidence: Number (0-1),
  sampleCount: Number,
  createdAt: Date
}

### Payments
{
  _id: ObjectId,
  jobId: ObjectId (unique),
  customerId: ObjectId,
  haulerId: ObjectId,
  amount: Number,
  status: Enum (authorized | charged | failed | refunded),
  stripePaymentIntentId: String (unique),
  stripeChargeId: String,
  createdAt: Date
}

### AuditLogs
{
  _id: ObjectId,
  userId: ObjectId,
  action: Enum (password_changed, gdpr_deletion, admin_access, etc.),
  resourceType: Enum (user | job | payment | admin),
  details: { previousValue, newValue, ipAddress, userAgent },
  createdAt: Date,
  expiresAt: Date (TTL: 90 days)
}

## API Contracts

POST /api/register
{ email, password (8+ upper/lower/digit/special), firstName, lastName, role }
Response 201: { user, token }
Response 400: { errorCode, message }

POST /api/login
{ email, password }
Response 200: { user, token }
Response 401: { errorCode: "AUTH_CREDENTIALS_INVALID", message }

POST /api/jobs/:id/accept
Response 200: { job } (status: accepted)

POST /api/jobs/:id/complete
{ beforePhotoLocation, afterPhotoLocation }
Response 200: { job }
Validation: Photos exist, GPS calculated, distance validated

DELETE /api/jobs/:id
Response 200: { message }
Validation: Only customer, only before accepted

## GPS Distance Validation & Fraud Detection

Algorithm:
1. Hauler provides GPS coordinates in photos
2. Haversine formula calculates actual distance
3. Query DistanceHistory for this address
4. Calculate variance (actual vs. estimated)
5. Confidence scoring: (positive_samples / total) * 0.9 + 0.1
6. Bidirectional alerts: flag under/overpayment

Example:
Job 1: Estimate 100ft, actual 95ft (variance -5ft, good)
Job 2: Estimate 80ft, actual 120ft (variance +40ft, underestimated)
Job 3: Estimate 90ft, actual 85ft (variance -5ft, good)

After 3 samples: Confidence = (2/3) * 0.9 + 0.1 = 0.7
Next job: If variance > 50ft, alert triggered

## Authentication & Authorization

JWT Token:
Header: { alg: "HS256", typ: "JWT" }
Payload: { userId, email, role, iat, exp }
Signature: HMAC_SHA256(header.payload, secret)

Authorization:
- POST /api/jobs/:id/accept: Must be hauler, job status "posted"
- POST /api/jobs/:id/complete: Must be accepting hauler, job status "en_route"
- DELETE /api/jobs/:id: Must be customer owner, job not accepted

## Error Handling

Standardized Error Codes (40+):
AUTH_EMAIL_ALREADY_REGISTERED
AUTH_CREDENTIALS_INVALID
AUTH_PASSWORD_INVALID
JOB_NOT_FOUND
JOB_ALREADY_ACCEPTED
JOB_INVALID_STATUS_TRANSITION
PAYMENT_CARD_DECLINED
PAYMENT_INSUFFICIENT_FUNDS
PAYMENT_ALREADY_AUTHORIZED
ADMIN_UNAUTHORIZED

Response Format:
{
  "errorCode": "AUTH_CREDENTIALS_INVALID",
  "message": "Email or password is incorrect.",
  "errorId": "err_1234567890abcdef",
  "timestamp": "2026-07-11T15:30:00Z"
}

## Deployment Architecture

Frontend (Vercel):
- React build, auto-deploy on git push
- CDN caching, cost: $0

Backend (Render):
- Node.js/Express, auto-deploy
- Environment: MONGODB_URI, JWT_SECRET, STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, ADMIN_EMAIL
- Cost: $7/mo

Database (MongoDB Atlas):
- Free M0 tier, auto-backups daily
- Connection pool: 10–25 connections, 5-min idle timeout
- Cost: $0

## Security

Password Storage: Bcrypt with 10 salt rounds, never plaintext
Payment Data: Never store CC data, Stripe tokens only, idempotency keys prevent duplicates
GDPR: Audit logs auto-delete 90 days, self-service deletion endpoint
Access Control: PUT /api/jobs/:id restricted to cancellation, all validation server-side

## Performance

Database: Indexed queries (email, customerId, haulerId, jobId), TTL index on AuditLog
API: Request validation before DB, pagination (20 default, max 100), async audit logging
Connection Pool: Sized 10–25, 5-min idle timeout

## Scalability (Future)

Redis for sessions and user data
MongoDB replicas for analytics
Sharding by geography if needed
Message queue for async processing
