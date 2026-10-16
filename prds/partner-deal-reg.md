# Pixee Partner Deal Registration Portal — Design Specification

**Status:** Implemented (MVP and Phase 2 partner views). HubSpot configured; Microsoft Entra External ID sign-in pending admin setup. Open items in §16.  
**Product:** Pixee Partner Deal Registration  
**Preferred URL:** `https://partners.pixee.ai`  
**Primary constraint:** **HubSpot CRM is the single source of truth for all business data. Do not provision or write to an application database.**

## 1. Objective

Build a secure, branded, responsive partner-facing web application where authorized Pixee channel partners can submit new customer opportunities and, in a subsequent phase, check registration status. Pixee employees review and approve/reject submissions within HubSpot. All partner identities/authorizations (as business records), company and contact relationships, registration records, sales deals, decisions, and status history must be represented in HubSpot.

### Core requirements

- Match the visual language of https://pixee.ai (capture the actual color, typography, and layout tokens from the live site during implementation; do not invent brand assets).
- Fast mobile-friendly registration workflow with accessible validation and confirmation.
- HubSpot is the only persistent **business-data** store; no Supabase, Firebase, PostgreSQL, MongoDB, or replicated partner/deal database.
- Never expose HubSpot credentials to browsers; all CRM access occurs in server-side endpoints.
- Restrict partner access by **verified identity AND HubSpot association/authorization**, never by an unverified email supplied in a form.
- Preserve a human approval step: unreviewed submissions enter the sales pipeline only in the **Registered** stage (excluded from forecast and pipeline reports) and become active opportunities only when Pixee approves them.
- Handle duplicate submissions, account conflicts, failures, retries, and auditing.

## 2. Stack

| Layer | Decision | Notes |
| --- | --- | --- |
| Web framework | Next.js (App Router), React, TypeScript | Server-side rendering/server actions or API route handlers |
| Styling | Tailwind CSS | Extract theme tokens from Pixee website |
| Components | shadcn/ui | Forms, dialogs, tables, notifications |
| Validation | Zod + React Hook Form | Shared schemas, server-side validation mandatory |
| Integration | HubSpot CRM APIs | Private app token for single-account internal integration; server-side only |
| Business data | **HubSpot only** | Companies, Contacts and Deals |
| Hosting | Vercel | Production/staging, secrets via environment variables |
| Domain | `partners.pixee.ai` | HTTPS and DNS configuration |
| Email/approval | HubSpot workflows **where supported by plan** | Otherwise internal CRM tasks and lightweight application-triggered notifications as approved |
| Observability | Vercel logs/monitoring | No sensitive submitted data in logs; operational telemetry only |

**Not included:** external application database, CRM-to-database synchronization, partner data persistence in local storage, API token in browser code.

## 3. HubSpot Record Model (decided)

Each partner registration is a **Deal in the existing Pixee sales pipeline**, so one record follows the opportunity from registration to close. Two stages are added to that pipeline:

| Stage | Position | Probability | Purpose |
| --- | --- | --- | --- |
| **Registered** | Before Discovery | 1%, open | Every new registration lands here, awaiting Pixee review |
| **Closed Rejected** | After Closed Lost | 0%, closed | Registrations Pixee declines |

- New registrations are created in **Registered** with status `pending_review`. The partner's estimated value and expected close date are stored only in the deal's standard **Amount** (with its currency) and **Close date**; the form offers only currencies enabled for deals in HubSpot. Partners see the deal's current Amount and Close date, including later edits by Pixee. Because Registered deals carry amounts, exclude Registered from forecast and pipeline reports and from reps' default views, and exclude Closed Rejected from win-rate and loss reports.
- The deal's **Next step** is set to `M/D: Approve/Reject the deal registration` (submission date, account time zone).
- A **Note** on the deal records the customer use case and the partner's notes for Pixee.
- Associate each registration with the partner Company and the submitting partner Contact, each with the **Partner** association label, and with the **customer's Company as the deal's Primary company**: the Company matching the website domain or exact name (never modified), or a new prospect Company. When several Companies match, Pixee sets the Primary company during review.
- Associate the **customer contact** the partner provides: the existing HubSpot Contact with that email (never modified by partner input), or a new Contact with only first name, last name and email. Customer contacts are never associated with the partner Company, so they cannot sign in to the portal.
- **On approval**, Pixee moves the deal from Registered to Discovery, confirms the amount and close date, and sets the owner. The registration deal itself becomes the commercial deal; no second deal is created.
- **On rejection**, the deal moves to Closed Rejected.
- **Duplicates:** if a registration duplicates an existing open deal, Pixee merges it into that deal or moves it to Closed Rejected and records the existing deal in `pixee_related_deal_id`.
- Name registration deals in Pixee's convention: `Prospect | New | Year | Partner`.

HubSpot plan features and quotas change. **Verify current account entitlements for pipelines, unique-value properties, workflows, required API scopes, and approval automation before building.**

## 4. Data Model (HubSpot)

### 4.1 Contact

Use HubSpot Contacts for partner users. Required/illustrative properties:
- `email` (normalized lower case), name, company association.
- Custom `pixee_partner_user_enabled` (boolean): empty until the contact's first sign-in, when the portal sets it to `true`; Pixee sets `false` to block a contact, or `true` to admit a contact whose email is not at the partner's domain.
- Optional `pixee_partner_role` (`partner_submitter`, `partner_admin`); **do not infer authorization solely from email domain**.
- Optional `pixee_partner_id` or association to approved partner Company.

### 4.2 Company

Use HubSpot Companies for both partner organizations and prospect companies.
- Partner designation via controlled custom property `pixee_company_role` (e.g., `partner`, `prospect`, `customer`; account for companies playing more than one role).
- Website and normalized domain for tentative matching; do not use domain alone as a globally unique key.
- Review company-association labels to identify a user's **partner organization** separately from other associated companies.

### 4.3 Registration (Deal properties)

Stored as properties on the registration Deal. At minimum:

| Property (illustrative internal name) | Type | Notes |
| --- | --- | --- |
| `pixee_registration_id` | unique string | Human-readable external reference, generated server-side; apply uniqueness if HubSpot supports it |
| `pixee_registration_status` | enumeration | `pending_review`, `info_needed`, `approved`, `rejected`, `withdrawn` |
| `pixee_partner_company_id` | HubSpot ID/reference | Source partner organization; associations preferred |
| `pixee_partner_contact_id` | HubSpot ID/reference | Verified submitting user; associations preferred |
| `pixee_prospect_name` | text | Submitted legal/customer name |
| `pixee_prospect_domain` | text | Normalize and validate, optional when unknown |
| `pixee_prospect_country` | enum/text | Controlled country list |
| `amount` + `deal_currency_code` (standard) | number + currency | Partner's estimated value; currency must be enabled in HubSpot |
| `closedate` (standard) | date-time | Partner's expected close date; optional |
| `pixee_product_interest` | controlled selection | Confirm Pixee's current product offerings |
| `pixee_customer_requirements` | multi-line text | Sanitize and length-limit |
| `pixee_partner_notes` | multi-line text | No passwords, secrets, or unnecessary customer personal data |
| `pixee_review_reason` | multi-line text | Internal; redact before partner view |
| `pixee_reviewed_by` / `pixee_reviewed_at` | identifier/date-time | Audit fields |
| `pixee_related_deal_id` | HubSpot ID/reference | After review: the deal's own ID, or the existing deal it duplicates |
| `pixee_partner_message` | multi-line text | Partner-facing message from the reviewer; shown verbatim to the partner |
| `pixee_submission_key` | unique string | Idempotency token / retry deduplication |

Use existing HubSpot properties where semantically appropriate; do not blindly create duplicates. Property *display labels* and *internal names* can differ. Map them in one typed server-side configuration module.

### 4.4 Status transitions

```text
Draft (browser only; not saved)
    -> Pending Review
    -> Info Needed -> Pending Review
    -> Approved -> deal moves from Registered to Discovery
    -> Rejected -> deal moves to Closed Rejected
    -> Withdrawn (policy-dependent)
```

Approval and rejection are Pixee-authorized actions only. Record actor, timestamp, and rationale in HubSpot. Do not silently overwrite an existing decision.

## 5. User Experience

### Phase 1 — Register Deal

**Public entry point:** `partners.pixee.ai/register-deal`.

**Identity:** Partner initiates verified login prior to submitting a registration. If Pixee opts for a public form instead, submissions must be treated as **unverified leads requiring manual review**, with no confidential duplicate/deal-status disclosure.

**Suggested form sections:**
1. **Partner:** Partner company and submitter contact automatically derived from verified HubSpot identity, displayed read-only.
2. **Prospect:** Customer company, website/domain, country, and the customer contact's first name, last name and work email (required; covered by Pixee's privacy policy).
3. **Opportunity:** Estimated value + currency, expected close date, product interest, description/customer requirements.
4. **Attestation:** Accuracy, and agreement to Pixee's Terms of Service and Privacy Policy (pixee.ai/policies).

**Behavior:**
- Required field indicators and inline validation; keyboard navigation and WCAG-conscious focus styles.
- Submit button disabled while in flight; request includes an idempotency key.
- Confirmation displays *registration reference*, not sensitive CRM information.
- Responsive layouts and intentional empty/error states.
- Prevent inadvertent resubmission on refresh or network retry.

### Phase 2 — Partner Portal

- `/dashboard`: registrations submitted by the authenticated partner (or, for partner admins, their whole organization): customer, reference, deal value, expected close, submitted (and by whom, for admins), last updated and status. Filters: status, and for admins the submitter (limited to people at their own partner company). Every column is sortable in both directions (server-side via HubSpot; submitter name, which HubSpot cannot sort, is sorted in the app over a bounded set of at most 1,000 registrations).
- `/registrations/[id]`: sanitized partner-visible registration detail: the partner's own submission (use case, notes), submitter, decision date and the reviewer's partner-facing message; never expose internal sales notes, the internal review reason, unrelated company/deal data, or other partners' records.
- Optional additional-information response flow with field-level permissions and audit trail.
- Pagination, filtering, and no unbounded CRM list reads.

### Pixee internal workflow

- Review pending registrations directly within HubSpot using saved views, ownership, notifications/tasks, and workflows where available.
- Mark duplicate/conflict, request more information, approve, or reject.
- On approval move the deal from Registered to Discovery; if it duplicates an existing open deal, merge it into that deal or close it as Closed Rejected.

## 6. Authentication & Authorization

### Design requirement

**HubSpot is the authority for whether a verified identity belongs to an enabled partner and which partner Company they can access.** HubSpot CRM is not, by itself, a turnkey login/session provider for a bespoke Next.js portal.

Choose one authentication implementation during technical discovery:
- **Preferred where supported:** a managed, standards-based identity provider (OIDC/passwordless) to establish identity, with HubSpot as the sole source for roles, account relationships, and business data. The identity service may persist credential/session metadata but **must not store or replicate deal/partner business records**.
- **Strict no-other-persistent-store policy:** investigate whether Pixee's existing identity infrastructure can issue verified, short-lived OIDC tokens; otherwise document that a secure, abuse-resistant email one-time-code service typically needs transient challenge/rate-limit state. Do not build a stateless OTP flow without replay protection.

After authentication, the Next.js server:
1. Validates the signed identity token/session (issuer, audience, expiry, CSRF and session controls).
2. Finds the corresponding HubSpot Contact by verified email or an explicitly linked immutable subject identifier.
3. Verifies the Contact has an authorized association to exactly one partner Company and is not blocked (`pixee_partner_user_enabled` is not `false`). On first sign-in (flag not yet set) the verified email must also be at the partner Company's domain, additional domains or a subdomain; the domain is an additional check, never sufficient alone. The portal then sets `pixee_partner_user_enabled = true`.
4. For **every** read/write, enforces that the requested registration belongs to that company (server-side). Never trust `partnerCompanyId` from a request body.

Security standards: httpOnly Secure SameSite cookies; CSRF protection on cookie-authenticated mutations; rate limiting; anti-bot safeguards; short sessions and logout; no account enumeration; avoid exposing HubSpot IDs unnecessarily.

## 7. HubSpot Integration

- Implement typed HubSpot client under `src/lib/hubspot/` with bearer token supplied **only** from Vercel server env.
- Least-privilege HubSpot private app scopes: read Contacts and Companies; write Deals; write Contacts (first sign-in flag, customer contacts, registration Note); write Companies (only to create customer companies that do not exist yet).
- Use supported CRM search, objects, associations, and batch endpoints as appropriate; account for pagination and rate limits.
- Retry transient `429`/`5xx` with bounded exponential backoff and jitter; do not blindly retry non-idempotent creates.
- Maintain a durable `pixee_submission_key` in HubSpot, check before creating a registration, and add a server-side concurrency strategy. Search-then-create alone is **not atomic**; when possible use a HubSpot-enforced unique identifier or upsert. Otherwise explicitly accept/document residual race risk and reconcile duplicates.
- Use request timeouts, redact PII/secrets from logs, and return generic browser error messages.
- Validate both input schema and allowable property values before writing.

### Duplicate and conflict checks

Perform a privileged, server-side pre-review across relevant HubSpot Companies and Deals:
- Normalize prospect website/domain and company name.
- A Company or Contact that already exists in HubSpot is **not** a conflict on its own.
- A conflict is an **open deal** associated with the customer's Company (any pipeline, including another registration still in Registered): **MEDIUM**; **HIGH** when the customer contact is already on that deal.
- Return generic `submitted_for_review` to partners; **do not reveal whether Pixee already has the prospect or which partner owns it**.
- Mark the conflict for Pixee internal review (`pixee_potential_conflict`, and the level plus each open deal in `pixee_conflict_summary`). Matching is advisory, not automatic denial, because subsidiaries, common domains, and naming variations are possible.

## 8. API Contract (illustrative)

| Method | Path | Principal | Description |
| --- | --- | --- | --- |
| `GET` | `/api/me` | Authenticated partner | Sanitized identity, enabled flag, and partner company label |
| `POST` | `/api/registrations` | Authenticated authorized partner | Validate and submit registration; include idempotency key |
| `GET` | `/api/registrations` | Authenticated partner (Phase 2) | Paginated list limited to their partner organization |
| `GET` | `/api/registrations/:id` | Authenticated partner (Phase 2) | Sanitized detail after ownership check |
| `POST` | `/api/hubspot/webhook` | Verified HubSpot webhook (optional) | React to approved status or notifications; validate signatures and freshness |

Example create payload (partner identification is **not** accepted from the browser):

```json
{
  "idempotencyKey": "uuid-v4-from-client",
  "prospect": {
    "companyName": "Example Industries",
    "website": "https://example.com",
    "country": "US"
  },
  "customerContact": {
    "firstName": "Casey",
    "lastName": "Customer",
    "email": "casey.customer@example.com"
  },
  "opportunity": {
    "estimatedValue": 75000,
    "currency": "USD",
    "expectedCloseDate": "2027-03-31",
    "productInterest": ["sast_triage_fix"],
    "requirements": "Application security program evaluation"
  },
  "notes": "Interested in a technical evaluation",
  "acceptedTerms": true
}
```

Success response `201` (or `200` for idempotent replay):

```json
{
  "registrationId": "PXR-2026-000123",
  "status": "pending_review",
  "message": "Your registration was received for review."
}
```

Errors: `400` invalid input, `401` unauthenticated, `403` unauthorized, `409` idempotency misuse, `429` too many requests, `502/503` unavailable CRM. All error details must avoid leaking customer/deal information.

## 9. HubSpot Workflows and Notifications

- Trigger on new registration → assign internal reviewer, set status `pending_review`, create task/notification.
- On approval/rejection/request-more-info → partner confirmation/update notification **only after verified recipient association**.
- On approval → move the deal to Discovery (it becomes the commercial deal); on rejection → move it to Closed Rejected.
- Provide fallback manual HubSpot views/tasks when the Pixee plan does not support the desired workflow features.
- Avoid exposing rejection/conflict details that reveal competitor/other partner records.
- Confirm which HubSpot email features can send transactional operational emails on Pixee's subscription before relying on them; use a vetted transactional mail service only if explicitly approved (without copying persistent business records into it).

## 10. Project Layout

```text
src/
  app/
    (public)/login/page.tsx
    (partner)/register-deal/page.tsx
    (partner)/dashboard/page.tsx              # Phase 2
    api/me/route.ts
    api/registrations/route.ts
    api/registrations/[id]/route.ts           # Phase 2
  components/
    registration-form.tsx
    layout/
    ui/
  lib/
    auth/
    hubspot/client.ts
    hubspot/properties.ts
    hubspot/registrations.ts
    hubspot/partners.ts
    authorization.ts
    idempotency.ts
    validation.ts
    errors.ts
  types/
    registration.ts
.env.example
README.md
Design.MD
```

## 11. Environment Variables

The complete, commented list is in [`.env.example`](.env.example). The essentials:

```dotenv
# Server-only; never prefix secrets with NEXT_PUBLIC_
HUBSPOT_PRIVATE_APP_TOKEN=
HUBSPOT_REGISTRATION_PIPELINE_ID=default            # the Pixee sales pipeline
HUBSPOT_REGISTRATION_PENDING_STAGE_ID=1459095634    # its "Registered" stage
HUBSPOT_SUBMISSION_KEY_UNIQUE=true
HUBSPOT_DEAL_PARTNER_CONTACT_LABEL=Partner          # contact-deal label for the submitting partner
HUBSPOT_DEAL_PARTNER_COMPANY_LABEL=Partner          # company-deal label for the partner company
HUBSPOT_DEAL_CUSTOMER_CONTACT_LABEL=                # optional label for the customer contact
HUBSPOT_DEAL_CURRENCIES=USD                         # currencies offered in the form (enabled for deal Amount)
HUBSPOT_TIME_ZONE=America/New_York                  # dates in deal text (Next step)

# Microsoft Entra External ID (OIDC)
AUTH_ISSUER=
AUTH_CLIENT_ID=
AUTH_CLIENT_SECRET=
AUTH_SESSION_SECRET=

NEXT_PUBLIC_APP_ORIGIN=https://partners.pixee.ai
```

Do not invent production HubSpot IDs or property internal names. Generate them from inspected HubSpot configuration and document provisioning steps. Include secret rotation and separate staging credentials.

## 12. Reliability, Security, and Compliance

- All mutations validate session, partner entitlement, request schema, and allowed lifecycle transition.
- HubSpot API tokens restricted to minimum scopes; rotate and audit access.
- Use HTTPS/HSTS, CSP, secure cookie policy, dependency scanning, and SAST in CI.
- Do not expose internal CRM search results or existence checks to partners.
- Do not log registration descriptions, access tokens, contact email addresses, or arbitrary HubSpot payloads.
- Limit input size; sanitize content for display; avoid raw HTML injection.
- Privacy notice, retention, and business contact collection reviewed by Pixee legal/security.
- Document how a disabled partner instantly loses access despite an otherwise valid login session.
- Account for HubSpot API outages: clear retry guidance, no fictitious success and no duplicate writes.
- Assess HubSpot API rate limits and concurrency for projected partner volume.

## 13. Deployment

1. Inspect existing Pixee site and derive design tokens, permitted logo assets, spacing, and responsive patterns.
2. Confirm HubSpot plan entitlements; create the registration properties, the Registered and Closed Rejected stages, associations, and permissions.
3. Provision scoped private app credentials and an approved identity provider; configure allowlisted callback URLs.
4. Build/validate API integration and authorization checks before UI polish.
5. Configure Vercel preview/staging/production; bind `partners.pixee.ai` DNS and TLS.
6. Add CRM workflow, notification, telemetry, and rate-limit monitoring.
7. Run acceptance tests with two unrelated partner companies and real HubSpot test records; document operational ownership and runbook.

## 14. Acceptance Criteria

- [ ] No application database exists and no partner/deal business data is persisted outside HubSpot.
- [ ] Partner can verify identity and submit complete registration successfully.
- [ ] Partner organization/contact are derived from verified server-side HubSpot associations.
- [ ] Submission is visible in HubSpot with reference ID, fields, associations, and status `pending_review`.
- [ ] Repeat submissions with the same idempotency key do not create additional records.
- [ ] Unauthorized user, disabled partner, and Partner A requesting Partner B's registration are all rejected.
- [ ] Duplicate account/deal checks notify Pixee without disclosing private account intelligence to partner.
- [ ] Approval/rejection process stores decision, actor, and timestamp in HubSpot.
- [ ] Approved registration corresponds to exactly one commercial Deal: the registration deal itself, or the existing deal it duplicates.
- [ ] HubSpot API errors, throttling, timeouts, and retries are tested.
- [ ] UI is responsive and keyboard-accessible; styling aligns with live Pixee site.
- [ ] No HubSpot token/PII is exposed in browser bundles or logs.

## 15. Delivery Phases

**MVP:** branded secure login, verified partner access, registration form, HubSpot creation/associations, idempotency, conflict flagging, HubSpot review process, submission acknowledgment.

**Phase 2:** authenticated partner dashboard with filtered status history; follow-up information requests; partner-admin visibility within company.

**Phase 3:** partner analytics, deal protection windows and expiration rules, automated ownership policies, program agreements, CRM reporting, and optional multi-region/localization.

## 16. Open Decisions / Questions for Pixee

1. **HubSpot tier:** Starter, Professional, or Enterprise? What pipeline, workflow, unique-value property, and transactional email entitlements are enabled?
2. **Partner access:** login and dashboard from day one, or initial form-only registration?
3. **Authentication:** can Pixee use an existing IdP, or may a managed external auth provider store identity/session metadata (but not business records)?
4. **Approval policy:** Who approves; when does deal protection start; expiration window; dispute resolution?
5. **Deal linkage:** *Decided* — registrations are deals in the Pixee pipeline, starting in Registered (§3).
6. **Notifications:** Which email templates and sender domain are approved?
7. **Partner program:** Which countries, currencies and reseller/distributor hierarchy are needed? *Products decided:* AI-SAST (Beacon), Design Review (Foresight), SAST (Triage & Fix), SCA (Triage & Fix).

## 17. Instructions to Coding Agent

- **Do not begin by adding a database/ORM** (no Prisma schema, Supabase, Firebase, Postgres, Redis persistence, etc.). If an infrastructure component needs temporary state (rate limiting/OTP), first surface the constraint and propose a no-business-data design consistent with Pixee's policy.
- Start by implementing typed HubSpot interfaces against **actual account object properties, scopes, and pipeline IDs**; avoid assumed internal names.
- Never treat a form-supplied email/domain as authorization. Enforce server-side ownership checks on every CRM read and write.
- Deliver small, testable units: CRM adapter, auth guard, Zod schema, create endpoint, form, HubSpot approval workflow documentation, then dashboard.
- Add automated tests for happy paths, retries, validation errors, conflicts, access isolation, and API outage.
- If account entitlements/identity-provider configuration are unknown, make the implementation configurable and document the decision required instead of silently choosing a feature that may not exist.
