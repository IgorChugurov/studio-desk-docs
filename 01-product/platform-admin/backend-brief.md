# Backend brief — Platform admin

> **Ownership.** This brief is written by the project owner from the canon of the topic and the approved product decisions. The backend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `backend-brief-feedback.md` in the same folder. Only the owner changes canon.

**Derived from:** `01-product/platform-admin/overview.md`

## Mission

Build the backend of the platform admin: the single place where the platform administrator signs in by e-mail and code, creates and edits studios (tenants), deactivates and activates them, and enters a studio's admin as its owner. When this works, a studio can exist with its own address and owner, and its owner can later sign in to the studio admin.

## Sequencing

No dependencies on other briefs. Order inside this brief: first the API contract (`api-contract.md`, phase 1), then implementation. The frontend brief is issued after the contract is agreed and the design is accepted.

## Source-of-truth order

1. `01-product/platform-admin/overview.md` and `01-product/profile.md`
2. `03-architecture/` (`backend-structure.md`, `api-conventions.md`, `authentication.md`) and `04-engineering-rules/backend.md`
3. The approved design, only where it affects the API
4. This brief, for sequencing

Canon wins over this brief. Where they appear to disagree, raise it in feedback instead of choosing.

## Current backend baseline

Verified 2026-10-03: the Nest application foundation exists — three APIs (`src/apis/platform|studio|public`) with their guards, the common error format, Zod validation, OpenAPI, `/api/health`, PostgreSQL users per API with a grants test — and automatic deploy to the server. No platform admin features: no tables, endpoints, sign-in, or studios.

## Product behaviour (mandatory)

Product terms and the full flows are in the canon. These are the invariants a plausible-looking implementation could break silently.

1. **One platform administrator.** Defined by e-mail in the environment and created when the database is created. No password. No API or UI to manage administrators.
2. **Sign-in is the same for the platform administrator and studio owners:** e-mail, code sent by e-mail, token. No passwords. Codes live in a separate table with creation time and expiry.
3. **No account enumeration.** The response to a code request is the same whether or not the e-mail has access. It always contains the code expiry time and the time at which resending becomes available.
4. **The API lets the client tell apart:** wrong code, expired code, too many attempts, resend requested too early. The canon has a separate fixed message for each.
5. **A studio** has a name, a unique subdomain, an optional unique custom domain, and a state: Active or Deactivated. Nothing else. A studio is a tenant. There is no physical delete.
6. **Format and uniqueness.** The backend always validates the name, the e-mail, the subdomain, and the custom domain, whatever the form checks. A name is 2 to 100 characters. A subdomain is 3 to 20 characters: lowercase Latin letters (a-z), digits, and hyphens, and it must start with a letter (for example `yoga-2`, `yoga-22`). A custom domain is lowercase Latin letters, digits, hyphens, and dots, with at least one dot (for example `yogaspace.com`), without `https://`, up to 253 characters, and not on the platform domain `studio-desk.axondigital.xyz`. The backend trims spaces at the edges and lowercases e-mails; it does not otherwise normalize: case and characters are not corrected, an invalid value is rejected. Subdomain and custom domain are unique across all studios, including Deactivated ones (they stay taken). Reserved subdomains cannot be used. Each failure is distinguishable: name invalid, subdomain invalid, subdomain taken, subdomain reserved, domain invalid, domain used by another studio, e-mail invalid.
7. **One owner per studio:** e-mail only. One e-mail may own several studios.
8. **Creation** is atomic: studio and owner together. A new studio is Active.
9. **Only the platform administrator** can create studios and change subdomain, custom domain, and owner. Editing works in any studio state.
10. **Owner change:** the previous owner's tokens stop working immediately. The mechanism: `03-architecture/authentication.md`.
11. **A Deactivated studio:** its owner and staff cannot sign in, and its public site is not served (tenant lookup by host does not resolve it as an active studio). Data is kept. Activation reverses this.
12. **Log in as studio:** only the platform administrator, immediately, for a studio in any state including Deactivated. The result is access as the studio owner. How the studio admin opens in the browser is not part of this brief.
13. **Studio list:** fields name, subdomain, custom domain, owner e-mail, status, created. Search by name, subdomain, custom domain, owner e-mail. Filter by state. Newest first by default. Server-side pagination.

## Cross-cutting constraints

Stack, the three APIs, and database users: `03-architecture/backend-structure.md`. Errors, responses, lists and pagination, OpenAPI: `03-architecture/api-conventions.md`. Sign-in, tokens, sessions, revocation, and log in as studio: `03-architecture/authentication.md`. Raise conflicts in feedback.

## Decisions left to the backend

Schema and naming, endpoints and methods, DTOs, error codes, indexes and constraints, transaction strategy, module structure. Also: code lifetime, attempt and resend limits, one-time use and hashing; the reserved subdomain list; the remaining technical constraints of a DNS label for the subdomain and the custom domain (for example a hyphen at the end of a label). Product behaviour above must not change because of these decisions.

## Required research before implementation

1. E-mail delivery for sign-in codes: provider, cost, free tier. The project is non-commercial for now, so the cheapest reliable option is preferred.

Record the recommendation in feedback before implementing the affected part.

## API contract

`api-contract.md` in this folder, written by the backend in phase 1 and agreed by the owner. This brief states the requirements it must satisfy and does not define endpoints or payloads.

## Phases

1. API contract for the platform admin. Needs nothing.
2. Platform administrator and sign-in: codes, tokens, revocation. Needs phase 1.
3. Studios: create, list, update, uniqueness, owner change. Needs phase 2.
4. Deactivate, activate, log in as studio (including the handoff code exchange in the studio API), host resolution for Deactivated studios. Needs phase 3.

## Required tests

Each invariant above is proven by a test, in particular: no enumeration (identical response for known and unknown e-mail); code expiry, attempt limit, and resend limit; subdomain format boundaries (2, 3, 20, 21 characters, leading digit, uppercase, special characters); uniqueness including Deactivated studios; creation atomicity; previous owner's token rejected immediately after owner change; sign-in rejected for a Deactivated studio; log in as studio works for a Deactivated studio; non-administrators cannot call administrator endpoints.

## Acceptance criteria

1. The administrator in the environment can sign in by e-mail and code and receives a token; nobody else can use administrator endpoints.
2. The response to a code request does not reveal whether the e-mail has access.
3. The four code failures from invariant 4 are distinguishable by the client.
4. A studio can be created with a valid unique subdomain and optional unique custom domain, and a failing format, uniqueness, or reserved-name check names the failing field.
5. The list supports search, state filter, newest-first order, and pagination.
6. After owner change the previous owner's token is rejected on the next request.
7. A Deactivated studio rejects owner and staff sign-in, is not served on its host, and keeps its subdomain and domain taken.
8. Log in as studio works for Active and Deactivated studios and only for the platform administrator.
9. There is no way to delete a studio or to manage administrators through the API.
10. `api-contract.md` exists and is agreed before implementation of phase 2 starts.

## Required feedback

`backend-brief-feedback.md` contains:
1. Answers to the research points above.
2. Deviations from this brief and why.
3. Behaviour decided in the backend because neither canon nor this brief defined it.
4. Which items the implementer believes change the canon (opinion only; canon is not edited).
5. Which documents were updated.

## Out of scope

- The studio admin, including the e-mail to the owner after creation, first-login settings, and studio staff and roles.
- The public site and client sign-in.
- The platform administrator action log.
- Custom domain DNS and certificates.
- All interface texts and form behaviour, including how the form proposes a subdomain from the studio name.
- How the studio admin opens after log in as studio.

## Quality checklist

- [ ] Mission is understandable without reading the code
- [ ] Baseline is verified against the code
- [ ] Mandatory behaviour and decisions left to the backend are clearly separated
- [ ] Every acceptance criterion is checkable
- [ ] No canon is restated
