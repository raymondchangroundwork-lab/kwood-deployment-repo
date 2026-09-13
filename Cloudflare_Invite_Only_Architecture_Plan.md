# Cloudflare Invite-Only Architecture Plan

## 1. Project Goal

Build a production-ready web application with the following architecture:

```text
Private GitHub Repo
        ↓
Cloudflare Workers / Static Assets
        ↓
Cloudflare D1 Database
        ↓
Invite-only Login System
        ↓
Each User Has a Separate Account and Separate Data
```

Primary requirements:

- Source code must remain in a **private GitHub repository**.
- Website is publicly reachable through a **Cloudflare-managed custom domain**.
- Only invited users can create accounts.
- No public registration page.
- Each user has unique login credentials.
- Each user can only access their own application data.
- Admin can invite, disable, and manage users.
- Passwords must never be stored in plaintext.
- Authentication and authorization must be enforced server-side.
- The website must continue running even when the developer's local computer is offline or powered off.
- Production secrets must never be committed to GitHub.

---

# 2. Recommended Architecture

```text
                         INTERNET
                            │
                            ▼
                  ┌───────────────────┐
                  │ Cloudflare Domain │
                  │ example.com       │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │ Cloudflare Worker │
                  │ + Static Assets   │
                  │                   │
                  │ Frontend          │
                  │ Backend API       │
                  │ Authentication    │
                  │ Authorization     │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │ Cloudflare D1     │
                  │                   │
                  │ users             │
                  │ sessions          │
                  │ invitations       │
                  │ profiles          │
                  │ application data  │
                  │ audit logs        │
                  └───────────────────┘


                  DEVELOPMENT / SOURCE

                       Claude Code
                            │
                            ▼
                  ┌───────────────────┐
                  │ GitHub PRIVATE    │
                  │ Repository        │
                  └─────────┬─────────┘
                            │
                            ▼
                     Cloudflare Deploy
```

For a new project, prefer **Cloudflare Workers + Static Assets** rather than splitting the frontend and backend into separate platforms unless the current framework strongly favors Pages.

---

# 3. GitHub Repository

## Repository Visibility

The main source repository must be:

```text
PRIVATE
```

Example structure:

```text
/src
/components
/routes
/api
/db
/migrations
/public
/tests
package.json
wrangler.jsonc
README.md
```

Do not commit:

```text
.env
.dev.vars
API keys
Cloudflare API tokens
private keys
admin passwords
session secrets
email provider secrets
AI provider secrets
```

Recommended `.gitignore`:

```gitignore
.env
.env.*
.dev.vars
node_modules/
dist/
.wrangler/
.DS_Store
```

---

# 4. Deployment Flow

Recommended deployment lifecycle:

```text
Claude Code
    ↓
Feature Branch
    ↓
Commit
    ↓
Push to Private GitHub
    ↓
Cloudflare Preview Deployment
    ↓
Test
    ↓
Merge into main
    ↓
Production Deployment
    ↓
example.com
```

Recommended branch strategy:

```text
main
  → production

feature/auth
feature/admin
feature/user-dashboard
feature/security
  → preview deployments
```

Do not force-push production branches.

---

# 5. Domain and Hosting

Buy the domain through **Cloudflare Registrar**.

Recommended setup:

```text
Cloudflare Registrar
        ↓
Cloudflare DNS
        ↓
Cloudflare Workers
        ↓
Cloudflare D1
```

Production URL:

```text
https://example.com
```

Optional Cloudflare development/preview URLs can remain enabled separately.

The developer's computer does not need to remain online because the production application runs on Cloudflare infrastructure.

---

# 6. Authentication Model

Use an **invite-only authentication model**.

There must be no public:

```text
/signup
/register
/create-account
```

Public/authentication routes:

```text
/login
/invite/accept
/forgot-password      # optional in V1
/reset-password       # optional in V1
```

Protected routes:

```text
/dashboard
/account
/reports
```

Admin-only routes:

```text
/admin
/admin/users
/admin/invitations
```

---

# 7. Invite-Only User Lifecycle

## Admin Flow

```text
Admin Login
    ↓
Admin Dashboard
    ↓
Invite User
    ↓
Enter Email + Role
    ↓
Server Generates Secure One-Time Token
    ↓
Store Only Token Hash in D1
    ↓
Generate Invitation URL
    ↓
Admin Sends Link to User
```

Example invitation link:

```text
https://example.com/invite/accept?token=RANDOM_SECURE_TOKEN
```

V1 can avoid SMTP entirely by letting the admin copy the invitation URL and send it manually through email, WhatsApp, Signal, etc.

## User Flow

```text
Receive Invitation
    ↓
Open Invitation Link
    ↓
Server Validates Token
    ↓
User Sets Password
    ↓
Password Is Hashed
    ↓
Account Is Activated
    ↓
Invitation Becomes Invalid
    ↓
Login
    ↓
Dashboard
```

Invitation requirements:

- Cryptographically random token.
- Store only the token hash in D1.
- Single-use.
- Expiration time.
- Invalid after acceptance.
- Invalid if account is disabled.

Suggested expiry:

```text
24–72 hours
```

---

# 8. Password Security

Never store:

```text
password = "userPassword123"
```

Store only a strong password hash.

Recommended approach:

```text
Password
   ↓
Established password hashing implementation
   ↓
Salted password hash
   ↓
D1
```

Do not invent a custom encryption or hashing algorithm.

Use a mature authentication/password library compatible with the Cloudflare Workers runtime.

Minimum password policy can begin around:

```text
10–12+ characters
```

Optionally require stronger password rules later.

---

# 9. Session Architecture

Prefer **server-side sessions** for V1.

Login process:

```text
Email + Password
      ↓
Worker verifies credentials
      ↓
Generate cryptographically random session token
      ↓
Store session token hash in D1
      ↓
Send session token through secure cookie
```

Cookie requirements:

```text
HttpOnly
Secure
SameSite=Lax or SameSite=Strict
Path=/
```

JavaScript in the browser should not be able to read the authentication cookie.

Recommended session behavior:

```text
Default session: 7 days
Optional Remember Me: longer session
```

Admin must be able to revoke sessions.

---

# 10. Database Schema

## users

```text
id
email
password_hash
role
status
created_at
updated_at
last_login_at
```

Suggested values:

```text
role:
- admin
- user

status:
- invited
- active
- disabled
```

## sessions

```text
id
user_id
token_hash
created_at
expires_at
revoked_at
```

## invitations

```text
id
email
token_hash
role
expires_at
accepted_at
created_by
created_at
```

## profiles

```text
id
user_id
display_name
avatar_url
created_at
updated_at
```

## application-specific data

Example `reports` table:

```text
id
user_id
title
content
created_at
updated_at
```

Example AI/chat tables:

```text
conversations
- id
- user_id
- title
- created_at

messages
- id
- conversation_id
- user_id
- role
- content
- created_at
```

## audit_logs

```text
id
actor_user_id
action
target_type
target_id
metadata
created_at
```

Useful actions include:

```text
USER_LOGIN
USER_LOGOUT
USER_INVITED
USER_DISABLED
PASSWORD_CHANGED
SESSION_REVOKED
INVITE_ACCEPTED
```

---

# 11. User Data Isolation

Every user-owned record must contain a server-controlled `user_id`.

Example:

```text
reports

id    user_id     title
1     USER-A      Report A1
2     USER-A      Report A2
3     USER-B      Report B1
```

User A must never be able to access User B's records.

Incorrect design:

```http
GET /api/reports?user_id=USER-A
```

The client must never be trusted to decide which user it is.

Correct design:

```text
Browser
   ↓
Secure Session Cookie
   ↓
Worker Validates Session
   ↓
Worker Determines Authenticated user_id
   ↓
D1 Query Uses Server-Derived user_id
```

Example conceptual query:

```sql
SELECT *
FROM reports
WHERE user_id = ?;
```

The bind value must come from the validated server session, not from an arbitrary request parameter.

---

# 12. Authorization

Authentication answers:

```text
Who is the user?
```

Authorization answers:

```text
What is the user allowed to do?
```

Every protected backend route must verify authentication.

Every admin backend route must also verify:

```text
role == admin
```

Do not rely on hiding buttons in the frontend.

Bad:

```javascript
if (role !== "admin") hideAdminButton();
```

This is only UI behavior, not security.

The Worker must reject unauthorized requests with an appropriate `401` or `403` response.

---

# 13. Recommended Routes

## Public

```text
/
/login
/invite/accept
```

## Authenticated User

```text
/dashboard
/account
/reports
```

## Admin

```text
/admin
/admin/users
/admin/invitations
```

## API

```text
/api/auth/login
/api/auth/logout
/api/auth/me

/api/invite/accept

/api/user/profile
/api/user/reports

/api/admin/users
/api/admin/invitations
/api/admin/invite
/api/admin/disable-user
/api/admin/revoke-sessions
```

---

# 14. Admin Features

V1 admin dashboard should support:

- View users.
- View account status.
- Invite a new user.
- Regenerate an invitation if the previous one expired.
- Disable a user.
- Re-enable a user.
- Revoke all sessions for a user.
- View invitation status.
- View basic audit history.

Example user table:

| Email | Status | Role | Last Login | Actions |
|---|---|---|---|---|
| user-a@example.com | Active | User | 2026-09-12 | Disable |
| user-b@example.com | Invited | User | — | Reissue Invite |
| admin@example.com | Active | Admin | 2026-09-12 | — |

---

# 15. Disable User Behavior

When an admin disables a user:

```text
status = disabled
```

The system should also revoke all active sessions.

After disablement:

```text
New login → denied
Existing session → denied
Protected API → denied
```

---

# 16. Password Reset

V1 can use an admin-generated password reset/setup link.

```text
Admin
   ↓
Generate Reset Link
   ↓
Secure One-Time Token
   ↓
User Sets New Password
   ↓
Existing Sessions Revoked
```

A later phase can add automated reset emails through an email provider.

---

# 17. Secrets Management

Store production secrets in Cloudflare secret/environment management.

Examples:

```text
SESSION_SECRET
EMAIL_API_KEY
AI_API_KEY
OTHER_SERVER_SECRET
```

Never expose these through client-side code.

Never prefix a sensitive secret in a way that causes it to be bundled into the frontend.

Never commit secrets to GitHub.

---

# 18. Database Migrations

All schema changes should be versioned in GitHub.

Recommended directory:

```text
/migrations
```

Example:

```text
0001_users.sql
0002_sessions.sql
0003_invitations.sql
0004_profiles.sql
0005_reports.sql
0006_audit_logs.sql
```

Do not make undocumented production database changes manually unless necessary for emergency recovery.

---

# 19. Security Controls

## Rate Limiting

Rate-limit at least:

```text
POST /api/auth/login
POST /api/invite/accept
POST /api/password-reset
POST /api/admin/invite
```

Examples:

- Limit repeated failed logins.
- Apply temporary delay/block after multiple failures.
- Consider Cloudflare Turnstile later if abuse appears.

## SQL Injection

Use parameterized queries/bind parameters.

Never concatenate raw user input into SQL.

Bad:

```javascript
`SELECT * FROM users WHERE email = '${email}'`
```

Good:

```text
Prepared / bound query
```

## XSS

- Escape user-controlled output.
- Avoid unsafe raw HTML rendering.
- Use CSP where practical.

## CSRF

If using cookie-based authentication, protect state-changing requests appropriately.

## Login Enumeration

Return a generic error such as:

```text
Invalid email or password.
```

Do not reveal whether a particular email address exists.

---

# 20. Audit Logging

Recommended events:

```text
Successful login
Failed login threshold reached
Logout
Password changed
Invitation created
Invitation accepted
Invitation expired
User disabled
User re-enabled
Sessions revoked
Admin action
```

Audit logs should be append-oriented and not casually editable through the normal UI.

---

# 21. Backup and Recovery

GitHub backs up source history, but it does **not** back up the production D1 database.

Production plan must include:

- D1 backup/export strategy.
- Restore procedure.
- Periodic verification that backups are usable.
- Separate handling for uploaded files if the project later adds file storage.

Priority data:

```text
users
profiles
reports
application records
critical audit records
```

---

# 22. Monitoring

At minimum monitor:

```text
Worker errors
5xx API responses
Authentication failures
D1 errors
Deployment failures
Unexpected traffic spikes
Rate limit events
```

Do not log passwords, raw session tokens, raw invitation tokens, API keys, or other secrets.

---

# 23. V1 Implementation Phases

## Phase 0 — Infrastructure

- Create/confirm private GitHub repository.
- Create Cloudflare project.
- Bind D1 database.
- Configure development and production environments.
- Configure custom domain later when app is stable.

## Phase 1 — Database

Create migrations for:

```text
users
sessions
invitations
profiles
audit_logs
```

## Phase 2 — Authentication

Implement:

```text
login
logout
session validation
password hashing
protected routes
```

## Phase 3 — Invite Only

Implement:

```text
admin invite creation
one-time invite token
invite acceptance
set-password flow
invite expiration
```

## Phase 4 — User Dashboard

Implement:

```text
/dashboard
/account
user-owned data
```

## Phase 5 — Admin Dashboard

Implement:

```text
/admin/users
/admin/invitations
disable/re-enable users
revoke sessions
```

## Phase 6 — Data Isolation

Audit every user-owned endpoint to guarantee that `user_id` is derived from the validated server session.

## Phase 7 — Security Review

Test:

```text
SQL injection
XSS
CSRF
session fixation
session revocation
authorization bypass
IDOR
brute-force protection
invite-token replay
expired token behavior
```

## Phase 8 — Production Domain

- Buy/configure Cloudflare domain.
- Enable HTTPS.
- Verify redirects.
- Verify cookies use `Secure`.
- Test production login flow.

## Phase 9 — Backup / Audit / Monitoring

- Add backup procedure.
- Add audit logs.
- Add monitoring/error visibility.

---

# 24. Acceptance Tests

The V1 is not complete until all of these pass.

## Authentication

- Invited user can create a password.
- Invitation cannot be reused.
- Expired invitation is rejected.
- Non-invited user cannot register.
- Wrong password fails.
- Disabled user cannot log in.
- Logout invalidates the current session.

## Authorization

- Normal user cannot access `/admin`.
- Normal user cannot call admin APIs directly.
- User A cannot view User B's data.
- User A cannot update/delete User B's data.
- Changing request parameters does not bypass ownership restrictions.

## Sessions

- Session cookie is `HttpOnly`.
- Session cookie is `Secure` in production.
- Expired session is rejected.
- Revoked session is rejected.
- Disabling a user invalidates active sessions.

## Secrets

- `.env` and `.dev.vars` are not in Git.
- No secret keys exist in frontend bundles.
- No plaintext passwords exist in database or logs.

## Deployment

- Private GitHub repo remains private.
- Preview build succeeds.
- Production build succeeds.
- Custom domain works with HTTPS.
- Website continues running while the developer machine is shut down.

---

# 25. Suggested V1 User Experience

## Admin

```text
/login
   ↓
/admin
   ↓
/admin/users
   ↓
Invite User
   ↓
Copy Invitation Link
```

## User

```text
Receive Invitation Link
   ↓
/invite/accept
   ↓
Set Password
   ↓
/login
   ↓
/dashboard
   ↓
Own Data Only
```

---

# 26. Cost Strategy

Initial goal:

```text
GitHub Private Repository     → Free tier where applicable
Cloudflare Workers            → Start on free tier
Cloudflare D1                 → Start on free tier
Cloudflare DNS                → Free
HTTPS                         → Included
Invite system                 → Application feature
Domain                        → Annual registration cost
```

Using manual invitation links in V1 avoids needing a separate transactional email provider initially.

---

# 27. Future Enhancements

After V1 is stable, consider:

- Automated invitation emails.
- Automated password-reset emails.
- MFA.
- Cloudflare Turnstile.
- Fine-grained roles/permissions.
- Staff accounts.
- Billing/subscriptions.
- AI usage quotas.
- User file uploads.
- Additional audit tooling.
- Device/session management UI.
- User self-service password change.
- Data export/delete workflows.
- Offline/PWA functionality if required.

---

# 28. Non-Negotiable Security Rules

1. **GitHub repository remains private.**
2. **No public registration.**
3. **Only admins can create invitations.**
4. **Passwords are never stored in plaintext.**
5. **Authentication cookies are HttpOnly + Secure.**
6. **User identity is derived from a validated server session.**
7. **The browser never decides its own `user_id`.**
8. **Every user-owned database operation enforces ownership server-side.**
9. **Every admin API validates the admin role server-side.**
10. **Secrets never enter GitHub or frontend bundles.**
11. **Invitation/session tokens are stored as hashes where practical.**
12. **Production changes go through preview/testing before main deployment.**
13. **Database migrations are version-controlled.**
14. **Backups are separate from source-control backups.**
15. **Never log raw passwords, secrets, session tokens, or invitation tokens.**

---

# 29. Final Target Architecture

```text
┌──────────────────────────────────────────────┐
│             GitHub PRIVATE Repo              │
│                                              │
│ Source Code                                  │
│ Database Migrations                          │
│ Tests                                        │
│ Cloudflare Configuration                     │
└────────────────────┬─────────────────────────┘
                     │
                     │ Deploy
                     ▼
┌──────────────────────────────────────────────┐
│              Cloudflare Workers              │
│                                              │
│ Frontend                                     │
│ Backend APIs                                 │
│ Authentication                              │
│ Authorization                               │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│                Cloudflare D1                 │
│                                              │
│ users                                        │
│ sessions                                     │
│ invitations                                  │
│ profiles                                     │
│ application data                             │
│ audit logs                                   │
└──────────────────────────────────────────────┘

                       USERS

       Admin ────────────────→ Invite User
                                      │
                                      ▼
                               One-Time Link
                                      │
                                      ▼
                                   User A
                                      │
                                Set Password
                                      │
                                      ▼
                                 Own Account
                                      │
                                      ▼
                                Own Data Only

                                   User B
                                      │
                                      ▼
                                 Own Account
                                      │
                                      ▼
                                Own Data Only
```

---

# 30. Recommended Claude Code Execution Order

When implementation begins, give Claude Code the work in this order:

```text
1. Inspect existing project architecture.
2. Do not break current working functionality.
3. Create/verify private-repo-safe `.gitignore`.
4. Configure Cloudflare Worker + D1.
5. Add versioned D1 migrations.
6. Implement secure invite-only authentication.
7. Implement server-side sessions.
8. Implement admin authorization.
9. Implement per-user data isolation.
10. Implement admin user management.
11. Add security controls and rate limiting.
12. Add tests for authentication and authorization bypasses.
13. Run build/lint/tests.
14. Deploy to a preview environment.
15. Perform acceptance tests.
16. Only after successful testing, merge/deploy to production.
```

Do not permit Claude Code to expose secrets, force-push, make the repository public, or bypass the security requirements in this document.
