# DeVotes
DeVotes is a digital voting platform


> A secure, web-based electronic voting and election management platform for organizations, schools, associations, institutions, and other groups that need to conduct structured elections online.

## Overview

**DeVotes** is an electronic voting and election management platform designed to allow organizations to create, manage, and conduct online elections.

The platform allows organizers to:

- Create and manage elections
- Manage candidates or voting options
- Import and manage eligible voters
- Authenticate voters
- Allow eligible voters to cast their ballots
- Prevent duplicate voting
- Monitor authorized election results
- Publish final results
- Track election activity through audit logs
- Track billable votes and organizer usage
- Manage invoices and payments

The platform uses a multi-organization architecture in which each organization manages only its own elections, voters, candidates, results, and related data.

A super administrator has control over the entire platform.

---

# Core Concept

The basic voting model is:

```text
Organization
     │
     ├── Organizer
     │
     ├── Election
     │     ├── Candidates / Options
     │     ├── Eligible Voters
     │     └── Votes
     │
     └── Billing
           ├── Usage
           ├── Payments
           └── Invoices
```

An election consists of candidates or voting options and a defined group of eligible voters.

A voter authenticates with the platform, accesses the appropriate ballot, casts a vote, and the system prevents that voter from voting more than once in the same election.

---

# Technology Stack

## Backend

- Python
- FastAPI
- PostgreSQL
- Alembic migrations

## Frontend

- React
- Vite
- TypeScript
- Tailwind CSS

## Deployment

- Vercel
- Managed PostgreSQL database

The application should use Vercel for the web/API layer while persistent election, voter, billing, and audit data should remain in a proper managed PostgreSQL database.

---

# User Roles

DeVotes uses explicit role-based access control.

| Role | Scope | Capabilities |
|---|---|---|
| `SUPER_ADMIN` | Entire platform | Full platform administration |
| `ORGANIZER` | Own organization | Create and manage elections |
| `ELECTION_MANAGER` | Assigned elections | Manage assigned elections |
| `VOTER` | Assigned election | Cast a vote |

## Super Administrator

The super administrator controls the entire DeVotes platform.

Capabilities include:

- Manage organizations
- Manage users
- Manage elections
- Manage platform billing
- Access platform-wide administration

## Organizer

An organizer operates within their organization.

They can:

- Create elections
- Manage elections
- Manage candidates
- Manage voters
- View authorized results
- Manage election-related billing

An organizer must never be able to access another organization's voters, elections, votes, invoices, or results.

## Election Manager

An election manager can manage elections to which they have been assigned.

## Voter

A voter is associated with an election and can cast a ballot when eligible.

---

# Election Lifecycle

Every election follows a controlled lifecycle:

```text
DRAFT
   ↓
SCHEDULED
   ↓
OPEN
   ↓
CLOSED
   ↓
RESULTS_PUBLISHED
   ↓
ARCHIVED
```

The backend must enforce these states.

A vote must be rejected when:

- The election is not `OPEN`
- The current time is before `starts_at`
- The current time is after `ends_at`

---

# Voting Process

The standard voting flow is:

```text
Voter Authentication
        ↓
Eligibility Check
        ↓
Election / Ballot
        ↓
Candidate Selection
        ↓
Vote Submission
        ↓
Database Transaction
        ↓
Duplicate Vote Protection
        ↓
Vote Recorded
        ↓
Billing Event
        ↓
Confirmation
```

The system must ensure that one eligible voter cannot vote twice in the same election.

Database constraints and transactions should be used rather than relying only on an application-level `has_voted` flag.

---

# Ballot & Vote Privacy

The voting database should be designed carefully so that voter authentication information does not unnecessarily become part of the actual ballot record.

The platform should explicitly define the privacy model of each voting implementation:

- Anonymous voting
- Confidential voting
- Non-anonymous voting

These models require different security and data-handling approaches.

---

# Database Architecture

The main relationships are:

```text
users
 │
 ├── organization_members
 │
 └── organizations
       │
       ├── elections
       │     ├── candidates
       │     ├── voters
       │     └── votes
       │
       └── billing_accounts
             ├── payments
             └── invoices

audit_logs
```

## Elections

An election contains information such as:

```text
id
organization_id
title
description
status
starts_at
ends_at
created_at
```

## Candidates

```text
id
election_id
name
description
sort_order
```

## Voters

```text
id
election_id
external_reference
display_name
email
voting_token_hash
has_voted
created_at
```

## Votes

```text
id
election_id
candidate_id
ballot_reference
cast_at
```

The final database implementation should also use appropriate unique constraints and transactions to prevent duplicate ballots.

---

# Billing

DeVotes separates the voting transaction from billing.

```text
VOTE CAST
    ↓
Successful Vote Transaction
    ↓
Billing Event
    ↓
billable_votes += 1
    ↓
Organizer Usage
    ↓
Invoice / Payment
```

A failed or duplicate vote must not become a billable vote.

## Pricing Models

The platform can support:

### Pay Per Vote

```text
₦X per vote
```

### Vote Packages

```text
10,000 vote package
20,000 vote package
50,000 vote package
```

> Pricing examples above are taken from the original system specification and should be configured according to the actual market and deployment.

The billing subsystem should maintain an immutable usage record rather than calculating billing simply from the current number of votes.

This makes:

- Refunds
- Disputes
- Invoices
- Auditing

easier to manage.

---

# Security

Security is a core requirement of DeVotes.

The platform should implement:

- Password hashing using Argon2 or bcrypt
- Short-lived authentication tokens or secure session cookies
- Role-based authorization
- Organization-level authorization
- Rate limiting
- CSRF protection where applicable
- Strict input validation
- Database transactions for vote casting
- Unique database constraints preventing duplicate ballots
- Administrator audit logs
- Server-side permission checks
- Protection against modification of closed-election votes
- Separation of voter authentication and ballot data where required
- Encrypted database connections
- Environment variables for secrets
- Database backups
- Restoration testing
- Monitoring
- Error logging
- Automated security tests

Client-side permission checks must never be treated as the primary security mechanism.

---

# Project Structure

```text
devotes/
│
├── README.md
├── LICENSE
├── .gitignore
├── .env.example
├── vercel.json
├── docker-compose.yml
├── Makefile
│
├── backend/
│   ├── pyproject.toml
│   ├── uv.lock
│   │
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   │
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   ├── security.py
│   │   │   ├── permissions.py
│   │   │   ├── database.py
│   │   │   └── logging.py
│   │   │
│   │   ├── models/
│   │   │   ├── user.py
│   │   │   ├── organization.py
│   │   │   ├── election.py
│   │   │   ├── candidate.py
│   │   │   ├── voter.py
│   │   │   ├── vote.py
│   │   │   ├── payment.py
│   │   │   ├── invoice.py
│   │   │   └── audit_log.py
│   │   │
│   │   ├── schemas/
│   │   │   ├── auth.py
│   │   │   ├── organization.py
│   │   │   ├── election.py
│   │   │   ├── candidate.py
│   │   │   ├── voter.py
│   │   │   ├── vote.py
│   │   │   ├── payment.py
│   │   │   └── common.py
│   │   │
│   │   ├── api/
│   │   │   ├── router.py
│   │   │   ├── auth.py
│   │   │   ├── admin.py
│   │   │   ├── organizations.py
│   │   │   ├── elections.py
│   │   │   ├── candidates.py
│   │   │   ├── voters.py
│   │   │   ├── voting.py
│   │   │   ├── results.py
│   │   │   ├── billing.py
│   │   │   └── health.py
│   │   │
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── middleware/
│   │   └── utils/
│   │
│   ├── migrations/
│   │   └── versions/
│   │
│   └── tests/
│
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── index.html
│   │
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── index.css
│       ├── api/
│       ├── components/
│       ├── pages/
│       ├── hooks/
│       ├── context/
│       ├── types/
│       └── lib/
│
├── scripts/
│   ├── create_admin.py
│   ├── seed_demo.py
│   └── backup_database.py
│
├── docs/
│   ├── architecture.md
│   ├── api.md
│   ├── database.md
│   ├── security.md
│   ├── deployment.md
│   └── billing.md
│
└── .github/
    └── workflows/
        ├── tests.yml
        └── deploy.yml
```

---

# API

The API uses versioned REST endpoints.

## Authentication

```text
/api/v1/auth/login
/api/v1/auth/logout
/api/v1/auth/me
```

## Administration

```text
/api/v1/admin/organizations
/api/v1/admin/users
/api/v1/admin/elections
/api/v1/admin/billing
```

## Organizations

```text
/api/v1/organizations
/api/v1/organizations/{organization_id}
```

## Elections

```text
/api/v1/elections
/api/v1/elections/{election_id}
/api/v1/elections/{election_id}/candidates
/api/v1/elections/{election_id}/voters
/api/v1/elections/{election_id}/results
/api/v1/elections/{election_id}/ballot
/api/v1/elections/{election_id}/vote
```

## Billing

```text
/api/v1/billing/usage
/api/v1/billing/invoices
/api/v1/billing/payments
```

## Health

```text
/api/v1/health
```

---

# Frontend Areas

The frontend is organized around three primary user experiences.

## Administrator

```text
Admin Dashboard
├── Organizations
├── Users
├── Elections
└── Billing
```

## Organizer

```text
Organizer Dashboard
├── Elections
├── Create Election
├── Election Details
├── Candidates
├── Voters
├── Results
└── Billing
```

## Voter

```text
Election
   ↓
Ballot
   ↓
Vote Confirmation
   ↓
Thank You
```

---

# V1 Features

The first shippable version should include:

- Super-admin dashboard
- Organizer accounts
- Organization isolation
- Election creation
- Candidate/option management
- Voter import
- Voter authentication
- Ballot page
- One-vote enforcement
- Election open/close controls
- Live vote count for authorized users
- Final results
- Audit trail
- Usage/billable-vote tracking
- Invoices and payments
- Responsive Tailwind UI
- REST API
- PostgreSQL migrations
- Automated tests
- Vercel deployment configuration
- Environment configuration
- Seed/demo data
- Documentation

---

# Deployment Architecture

```text
                    VERCEL
                       │
             ┌─────────┴─────────┐
             │                   │
        React/Vite            FastAPI
        Frontend                API
             │                   │
             └─────────┬─────────┘
                       │
                       ▼
              Managed PostgreSQL
```

Vercel should host the application/API layer.

PostgreSQL should be hosted through a proper managed database provider rather than being treated as the primary database directly on Vercel.

---

# Development Principles

DeVotes should prioritize:

1. **Security**
2. **Vote integrity**
3. **Organization isolation**
4. **Reliable election state management**
5. **Clear separation of voting and billing**
6. **Auditability**
7. **Scalability**
8. **Maintainable architecture**
9. **Automated testing**
10. **Clear documentation**

---

# Important Security Principle

The system must never assume that a frontend restriction is sufficient to protect election data.

Every protected API operation must validate:

```text
Authentication
      ↓
Role
      ↓
Organization / Election Scope
      ↓
Permission
      ↓
Requested Operation
```

---

# Testing

Automated tests should specifically cover:

- Authentication
- Permissions
- Organization isolation
- Election creation
- Election lifecycle
- Voting
- Duplicate voting
- Results
- Billing
- Authorization bypass attempts

Particular attention should be given to ensuring that an organizer cannot access another organization's data.

---

# Environment Configuration

Secrets and deployment-specific configuration must be stored through environment variables.

Example:

```env
DATABASE_URL=
SECRET_KEY=
ACCESS_TOKEN_EXPIRE_MINUTES=
ENVIRONMENT=
```

Do not commit production secrets to the repository.

---

# License

This project should include a `LICENSE` file defining the applicable licensing terms.

---

# Project Status

**Version:** V1  
**Project:** DeVotes  
**Type:** Electronic Voting / Election Management Platform  
**Backend:** FastAPI / Python  
**Frontend:** React / Vite / TypeScript / Tailwind CSS  
**Database:** PostgreSQL  
**Deployment:** Vercel + Managed PostgreSQL

---

# Author
**Cimon Abayateye**
Founder / Developer
**Caleb Wodi**  
Cofounder / Developer

Original system design specification dated **October 2, 2026**.
