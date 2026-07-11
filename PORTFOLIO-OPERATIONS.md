# Curbly: Operations & Deployment

## Deployment Architecture

Frontend (Vercel): React app, auto-deploy on git push, CDN caching, cost $0
Backend (Render): Node.js/Express, auto-deploy, environment vars, cost $7/mo
Database (MongoDB Atlas): Free M0 tier, auto-backups 24h, connection pool 10-25, cost $0

## Environment Variables

Backend (Render):
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/curbly
JWT_SECRET=<32-char random key>
STRIPE_SECRET_KEY=sk_test_xxx (test) / sk_live_xxx (prod)
STRIPE_WEBHOOK_SECRET=whsec_xxx
ADMIN_EMAIL=onepunchllc@outlook.com
NODE_ENV=production

Frontend (Vercel):
REACT_APP_API_URL=https://api.render.com
REACT_APP_STRIPE_KEY=pk_test_xxx (public key)

## Deployment Process

1. Developer pushes to main branch
2. GitHub Actions trigger: Run 260 backend tests, run frontend lint + build, run E2E tests
3. If all tests pass: Vercel auto-deploys frontend, Render auto-deploys backend
4. Database migrations run automatically

## RTO & RPO Targets

RTO (Recovery Time Objective):
Most incidents: 1 hour
Critical data loss: 15 minutes

RPO (Recovery Point Objective):
Data loss: 15 minutes (latest backup)
Transaction loss: 0 (Stripe webhook idempotency)

## Backup Strategy

MongoDB Atlas Backups: Automated daily at 2 AM UTC, 7-day retention, geographically redundant

Restore Procedure:
1. Navigate to MongoDB Atlas console
2. Click "Backup" → "Restore"
3. Select backup time
4. Create new cluster or restore to existing
5. Update connection string in Render
6. Test connectivity before resuming traffic

## Disaster Recovery Runbook

Scenario 1: Database Corruption
Symptoms: Queries failing with document validation error
Response: Assess scope, check last good backup, restore to new cluster, update MONGODB_URI, monitor 30 min

Scenario 2: Render Service Outage
Symptoms: API returns 503 Service Unavailable
Response: Check Render logs, upgrade instance if out of memory, revert to previous commit, manually trigger redeploy

Scenario 3: Stripe Webhook Failures
Symptoms: Payments authorized but not charged
Prevention: Idempotency keys built in, webhook handler checks if payment already processed
Recovery: Query Payments for orphaned records, verify Stripe has charge, update DB or manually create via API

Scenario 4: Cascading Failures
Symptoms: Frontend won't load, API unreachable, database slow
Response: Check status page, check GitHub Actions, check Render logs, try connecting via MongoDB CLI

## Monitoring & Alerting

### What We Monitor

Frontend (Vercel):
Build success/failure rate
Page load times (Lighthouse)
404 errors
User agent errors

Backend (Render):
API response times (<500ms p95)
Error rates (<0.1%)
Database query times (<100ms p95)
Active connections (< 25)

Database (MongoDB):
Connection pool usage
Query performance
Storage usage (free tier: 512 MB limit)
Index hit ratio (> 95%)

Payments (Stripe):
Webhook processing lag (< 30s)
Failed charges (investigate immediately)
Refund requests (audit trail)

### Alert Thresholds

CRITICAL (immediate action):
API error rate > 5%
Database connection pool maxed out
MongoDB storage > 90%
Stripe webhook lag > 5 minutes

HIGH (investigate within 1 hour):
API response time p95 > 2 seconds
Build failing for 2+ hours
Distance validation failing

MEDIUM (investigate within 24 hours):
Unhandled 500 errors
Test coverage < 95%
Audit logs corrupted

## Incident Response

Severity Levels:
CRITICAL: Payment processing failing, data corruption, security breach, user data exposed
HIGH: API down 30+ min, large number failed jobs (10+), GPS validation broken
MEDIUM: Minor API bug, performance degradation, error logging issues
LOW: Typo in error message, minor UI bug, analytics discrepancy

Response Procedure (CRITICAL):

0. STOP & NOTIFY: Email onepunchllc@outlook.com
1. ASSESS: Check dashboard, recent commits, estimate impact
2. CONTAIN: Switch to backup, revert commit, scale up instance
3. FIX: Root cause analysis, fix, test locally, deploy
4. COMMUNICATE: "Issue detected at X time, resolved at Y time, root cause Z"
5. POSTMORTEM: Write it down, document prevention steps

## Maintenance Windows

Weekly Maintenance (Monday 2 AM UTC):
Run database optimization
Check backup integrity
Review error logs
Check disk usage

Monthly Maintenance (First Sunday, 3 AM UTC):
Disaster recovery drill
Update dependencies
Review access logs
Audit user permissions
Check Stripe reconciliation

Quarterly Maintenance (Every 3 months):
Full security audit
Performance analysis
Capacity planning
Compliance review

## Scaling Plan (Future)

Phase 1 (Current): Single Node
Vercel: Free tier (~50 concurrent)
Render: Small instance (~100 concurrent)
MongoDB: Free tier (512 MB, ~100k documents)

Phase 2 (If 1,000+ jobs/month): Horizontal Scaling
Vercel: Automatic
Render: Multiple instances behind load balancer
MongoDB: Upgrade to M2 ($19/mo), replicas, sharding by zip code

Phase 3 (If international): Global Distribution
Vercel: Automatic CDN
Render: Multiple regions
MongoDB: Multi-region replication

## Cost Analysis (Launch Phase)

Monthly Costs:
Vercel: $0 (hobby)
Render: $7 (small backend)
MongoDB: $0 (M0 free)
Stripe: 2.9% + $0.30 per transaction

At 100 jobs/month ($1,000 revenue):
Stripe: $59
Infrastructure: $7
Total: ~$66 (6.6% of revenue)

At 200 jobs/month ($2,000 revenue):
Stripe: $118
Infrastructure: $7
Total: ~$125 (6.2% of revenue)

Upgrade Path:
If exceeding limits: MongoDB → $19/mo (M2), Render → $25/mo (medium instance)
Target: Keep infrastructure costs < 10% of revenue
