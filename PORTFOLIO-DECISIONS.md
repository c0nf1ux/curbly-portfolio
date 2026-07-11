# Curbly: Key Decisions & Trade-Offs

## 1. ACCESSIBILITY-FIRST DESIGN (WCAG 2.1 AA)

Decision: Build for WCAG 2.1 AA compliance from day 1.

Why: Your parents are aging. You lived the problem (3rd-floor compactor, mobility barriers). Elderly + disabled = underserved market. Accessibility builds trust and loyalty.

What This Meant: 40 hours upfront work, ESLint a11y plugin, explicit dark colors (#1f2937 for 8:1+ contrast), proper form labels, semantic HTML, colorblind-safe palette.

Trade-Off: Cost 40 hours upfront | Benefit: Zero rework at launch, 93/100 Lighthouse avg, competitive advantage

Result: Production-ready accessibility, no compromises.

## 2. GPS-VALIDATED FRAUD DETECTION (Self-Tuning)

Decision: Require GPS-geotagged photos at job completion; use Haversine distance formula + variance analysis.

Why: Trash removal is inherently untrustworthy. Did the hauler really travel 100 feet, or claim a longer distance for more pay? Without fraud detection, customers won't trust pricing.

What This Meant: Photo requirement, Haversine formula, variance analysis, confidence scoring, bidirectional alerts, self-tuning based on historical data.

Trade-Off: Cost: Complex algorithm, GPS validation, photo storage | Benefit: Fraud detection, customer trust, pricing transparency

Result: "Here's why you paid $X — you were Y feet away, which is Z more than estimated"

## 3. STANDARDIZED ERROR CODES (40+)

Decision: Every API error returns a machine-readable code + human message, not just a string.

Why: Mobile teams need predictable error handling. String-based errors break with i18n and retry logic. "Invalid email or password" doesn't tell you: email doesn't exist vs. password wrong vs. email not verified.

What This Meant: 40+ error codes defined, integrated into all endpoints, enables mobile retry logic and translated messages.

Trade-Off: Cost: Upfront integration effort | Benefit: Scalable error handling, mobile-friendly, better analytics

Result: Mobile teams move faster, users get better error messages.

## 4. AUDIT LOGGING (90-Day Retention)

Decision: Log all sensitive operations (password changes, GDPR deletion, admin access) with timestamps, IP, user agent.

Why: GDPR requires audit trails. You can't prove a user initiated deletion without logs. Security visibility into compromised accounts. Operations debugging.

What This Meant: Immutable logs, 90-day retention, async logging (non-blocking), structured format with resourceType, resourceId, details.

Trade-Off: Cost: Database storage, query performance | Benefit: Legal compliance, security visibility, troubleshooting

Result: Provable audit trail for every sensitive operation.

## 5. JOB STATE MACHINE (Strict Transitions)

Decision: Jobs can only transition through specific states via dedicated endpoints, not arbitrary PUT requests.

Why: If customers can set any job status, they bypass payment/proof validation. Job transitions must be ordered and validated at each step.

What This Meant: Dedicated endpoints (POST /accept, POST /en-route, POST /complete, DELETE /cancel) instead of arbitrary PUT.

Trade-Off: Cost: More endpoints, stricter validation | Benefit: No state machine bypass, payment integrity guaranteed

Result: Cannot cheat the system by manipulating job status.

## 6. DYNAMIC PRICING BY JOB CHARACTERISTICS

Decision: Price varies by carry distance, bag weight, building type, and accessibility—not flat rate.

Why: A 100-foot carry with heavy bags is harder than 20-foot carry with standard bags. Flat pricing doesn't work long-term (haulers won't accept hard jobs, customers won't pay for easy jobs).

What This Meant: Base $6 + distance ($5 for 50ft+) + weight ($3 for heavy) + building ($2 for commercial) + accessibility ($2 for no elevator).

Trade-Off: Cost: Algorithm complexity, customer education | Benefit: Fairer pricing, better hauler retention, higher margins

Result: Haulers stay around for hard jobs. Customers understand pricing.

## 7. VERCEL + RENDER DEPLOYMENT (No Self-Hosting)

Decision: Deploy frontend on Vercel, backend on Render. Use MongoDB Atlas free tier. Don't self-host.

Why: Self-hosting requires DevOps expertise. You're one person. Vercel + Render + MongoDB free tier starts at $0. Vercel auto-deploys on git push. Render has built-in logging.

What This Meant: Frontend: auto-deploy via Vercel. Backend: auto-deploy via Render. Database: MongoDB Atlas backups.

Trade-Off: Cost: Vendor lock-in | Benefit: Zero DevOps effort, 99.5% uptime guarantee, auto-scaling

Result: You focus on features, not infrastructure.

## 8. JWT AUTHENTICATION (Stateless Sessions)

Decision: Use JWT tokens for authentication, not server-side sessions.

Why: Sessions require server memory or database lookups on every request. JWTs are stateless (just verify signature). Mobile-friendly (long-lived tokens). Standard REST API.

What This Meant: Login creates JWT with userId, role, email. On each request, backend verifies signature (no database lookup). 24-hour expiry.

Trade-Off: Cost: Can't revoke tokens immediately (valid for 24 hours) | Benefit: Stateless, scalable, mobile-friendly

Result: No session table, no server memory bloat.

## 9. IN-MEMORY MONGODB FOR LOCAL TESTING

Decision: Use MongoDB Memory Server for Jest tests, not mocked database.

Why: Mocked databases pass tests but fail in production (mocks don't implement transactions, constraints). Real in-memory MongoDB is realistic. Jest setup starts server once, runs 260 tests in 6 minutes.

What This Meant: jest-setup.js starts MongoDB Memory Server before tests, all 260 tests run against real (in-memory) database, cleanup after.

Trade-Off: Cost: Memory usage (MB, not GB), startup time (10s) | Benefit: Realistic tests, no production surprises

Result: Tests that actually test real database behavior.

## 10. SEPARATE GITHUB REPOS FOR PORTFOLIO

Decision: Don't mix portfolio documentation with production code. Use completely separate repos with -portfolio suffix.

Why: If portfolio files sit in same repo as production code, easy to accidentally commit them. Separate repos make confusion impossible. Recruiters see documentation (safe) separate from code (private).

What This Meant: Production (c0nf1ux/curbly, private) vs. Portfolio (c0nf1ux/curbly-portfolio, public, documentation only).

Trade-Off: Cost: Maintain two repos | Benefit: No accidental code publication, clear separation

Result: Safe, clean portfolio for recruiters.

## Summary: Trade-Off Matrix

| Decision | Cost | Benefit | Winner |
|----------|------|---------|--------|
| WCAG AA | 40 hours | Zero rework, competitive advantage | Accessibility |
| GPS Fraud | Complex algorithm | Customer trust, pricing transparency | Trust |
| Error Codes | Upfront integration | Mobile scalability, better analytics | Scalability |
| Audit Logging | Storage, queries | Compliance, security visibility | Compliance |
| State Machine | More endpoints | Payment integrity, no state bypass | Integrity |
| Dynamic Pricing | Algorithm complexity | Fairer pricing, better retention | Revenue |
| Vercel+Render | Vendor lock-in | Zero DevOps, auto-scaling | Productivity |
| JWT Auth | Can't revoke immediately | Stateless, scalable, mobile-friendly | Scalability |
| In-Memory Mongo | Memory usage | Realistic tests, no surprises | Reliability |
| Separate Repos | Maintain two repos | Safe portfolio, clear separation | Safety |

Thesis: Every decision prioritizes production readiness, security, and accessibility over short-term speed.
