# HR Assistant — Design Spec

**Date:** 2026-08-12
**Status:** Approved, ready for implementation planning

## 1. Purpose

An applicant-tracking MVP. HR staff publish job postings; job-seekers apply with a PDF CV; the system scores each CV against the job with OpenAI and shows HR a ranked candidate list. HR either advances a candidate or rejects them, and on rejection the system drafts a rejection message with AI that HR can edit — or HR writes their own and asks AI to enhance it.

A company may have many HR users. All data is scoped to a company, and no user may see another company's jobs, applications, or CVs.

## 2. Existing stack (verified, not assumed)

The application is the Laravel React starter kit, already installed:

- Laravel 13, PHP 8.4
- Inertia v3 (`inertiajs/inertia-laravel ^3.0`, `@inertiajs/react ^3.0`)
- React 19, TypeScript, Vite 8
- Laravel Wayfinder ^0.1.14 (typed route/action imports)
- Laravel Fortify ^1.37 — **registration is currently disabled**; `config/fortify.php` `features` contains only `Features::resetPasswords()`, and there is no `auth/register` page
- shadcn/ui (`components.json` present) + Tailwind v4
- Pest 5, Larastan, Pint
- SQLite, `QUEUE_CONNECTION=database`, `FILESYSTEM_DISK=local`, `MAIL_MAILER=log`

`User` does not implement `MustVerifyEmail`. The `verified` middleware already on the dashboard route is therefore a no-op; it stays harmlessly.

### New dependencies

Resolved cleanly against the installed tree via `composer require --dry-run`:

| Package | Version | Purpose |
| --- | --- | --- |
| `spatie/laravel-permission` | `^8.3` | roles and permissions |
| `openai-php/laravel` | `^0.20` | OpenAI client (`OpenAI::responses()`, `OpenAI::fake()`) |
| `smalot/pdfparser` | `^2.12` | extract text from uploaded PDF CVs |

Front-end additions: shadcn `table`, `textarea`, `progress`, `alert-dialog`.

## 3. Decisions taken

| Decision | Choice | Why |
| --- | --- | --- |
| AI provider | OpenAI, Responses API with strict `json_schema` structured output | returns a validated object, never prose to parse |
| Model | `OPENAI_MODEL` env var, **no default** — app throws a clear exception if unset | the account's available models are the user's call |
| CV ingestion | `smalot/pdfparser` text extraction, PDF only | no file uploads to OpenAI, cheap, deterministic, testable |
| When AI runs | queued on submit | instant response for the candidate; an OpenAI outage never blocks applying |
| Accounts | company self-signup + email invitations; candidates self-register | matches "a company might have many HRs" |
| Tenancy | global spatie roles + `company_id` column, isolation enforced in policies | spatie's teams feature complicates every query for what a column comparison does correctly |
| Rejection delivery | portal only, no email | `MAIL_MAILER` is `log`; email is a later addition |
| Application stages | three statuses only | "move to next step" is `shortlisted`; no multi-stage pipeline in the MVP |

### Naming constraint

Laravel already owns the `jobs` table for the database queue. The domain model is **`JobPosting` / `job_postings`**. Never `Job` / `jobs`.

## 4. Data model

### `companies`
`id`, `name`, `slug` (unique), `timestamps`

### `users` (modify existing table)
Add `company_id` — nullable, foreign key to `companies`, indexed, `nullOnDelete`. Candidates have `null`.

### `company_invitations`
`id`, `company_id`, `email`, `token` (unique, 64 chars), `invited_by` (users), `expires_at`, `accepted_at` (nullable), `timestamps`.
Plain index on (`company_id`, `email`). Duplicate-invite prevention is a Form Request rule ("this email already has a pending invitation"), not a partial unique index — SQLite support for those is awkward and the check needs a friendly message anyway.

### `job_postings`
`id`, `company_id`, `created_by` (users), `title`, `description` (text), `requirements` (text), `employment_type` (enum), `location`, `min_years_experience` (unsigned tinyint, default 0), `status` (enum), `published_at` (nullable), `timestamps`.
Index on (`company_id`, `status`).

### `applications`
`id`, `job_posting_id`, `user_id` (candidate), `cv_path`, `cv_original_name`, `cv_text` (longText, nullable), `status` (enum), `rejection_reason` (text, nullable), `decided_at` (nullable), `decided_by` (users, nullable), `timestamps`.
**Unique on (`job_posting_id`, `user_id`)** — one application per candidate per job.

### `application_screenings`
`id`, `application_id`, `status` (enum), `match_score` (unsigned tinyint, nullable), `summary` (text, nullable), `strengths` (json, nullable), `gaps` (json, nullable), `recommendation` (string, nullable), `model` (string, nullable), `input_tokens`, `output_tokens` (nullable), `error` (text, nullable), `completed_at` (nullable), `timestamps`.

Screening is a separate table rather than columns on `applications` so a failed call can be retried and re-run without destroying prior results. An application exposes a `latestScreening` relation.

`cv_text` is extracted once during screening and stored on the application, so the rejection writer reuses it without re-parsing the PDF.

### Enums (`app/Enums/`)

- `ApplicationStatus`: `Submitted`, `Shortlisted`, `Rejected`
- `ScreeningStatus`: `Pending`, `Completed`, `Failed`
- `JobPostingStatus`: `Draft`, `Published`, `Closed`
- `EmploymentType`: `FullTime`, `PartTime`, `Contract`, `Internship`
- `RoleName`: `CompanyAdmin`, `Hr`, `Candidate`

All are backed string enums cast on their models.

## 5. Roles, permissions, and tenancy

Roles are **global** (spatie teams feature stays off). Company isolation is a `company_id` comparison in policies.

| Role | `manage jobs` | `review applications` | `manage team` |
| --- | --- | --- | --- |
| `company_admin` | yes | yes | yes |
| `hr` | yes | yes | no |
| `candidate` | no | no | no |

Seeded by a `RolesAndPermissionsSeeder` that is idempotent (`firstOrCreate`) so it can be re-run.

### Policies

Every HR-side check has two halves. The permission half and the tenancy half. Omitting the tenancy half is the classic multi-tenant data leak, so it lives in the policy and never in a controller.

```
JobPostingPolicy::update(User, JobPosting)
    => $user->can('manage jobs') && $user->company_id === $jobPosting->company_id

ApplicationPolicy::review(User, Application)
    => $user->can('review applications')
       && $user->company_id === $application->jobPosting->company_id

ApplicationPolicy::view(User, Application)
    => the above, OR $user->id === $application->user_id   (candidate sees their own)

CompanyPolicy::manageTeam(User, Company)
    => $user->can('manage team') && $user->company_id === $company->id
```

Route groups additionally gate coarsely with spatie's `role:` middleware, so an unauthorised role never reaches a controller at all.

### Account creation flows

1. **Candidate** — Fortify registration is enabled; `CreateNewUser` assigns the `candidate` role and leaves `company_id` null.
2. **Company** — `/register/company` takes company name plus the admin's name/email/password, creates both in a transaction, assigns `company_admin`, logs the user in.
3. **HR invite** — a company admin invites by email; the invitee opens `/invitations/{token}`, sets a password, and gets the `hr` role plus that `company_id`. Tokens expire after 7 days and are single-use.

## 6. Application architecture

Controllers validate through a Form Request, call one action or service, and redirect. No business logic in controllers.

```
app/Contracts/
    CvScreener.php            screen(JobPosting, string $cvText): ScreeningResult
    RejectionWriter.php       draft(Application): string
                              enhance(Application, string $hrText): string

app/Services/Ai/
    OpenAiCvScreener.php
    OpenAiRejectionWriter.php
    Concerns/BuildsPrompts.php

app/Services/Cv/
    CvTextExtractor.php       extract(string $path): string   (smalot/pdfparser)

app/Data/
    ScreeningResult.php       readonly DTO

app/Actions/
    Companies/RegisterCompany.php
    Companies/InviteTeamMember.php
    Companies/AcceptInvitation.php
    Applications/SubmitApplication.php
    Applications/ShortlistApplication.php
    Applications/RejectApplication.php

app/Jobs/ScreenApplicationJob.php
app/Policies/{JobPosting,Application,Company}Policy.php
app/Exceptions/{CvExtractionException,AiResponseException}.php
```

`CvScreener` and `RejectionWriter` are interfaces bound to their OpenAI implementations in `AppServiceProvider`. Feature tests `swap()` in fakes and never reach the network; the OpenAI classes get their own unit tests using `OpenAI::fake()`.

`app/Services`, `app/Jobs`, `app/Policies`, `app/Enums`, `app/Data`, `app/Contracts`, `app/Exceptions` are new directories under the existing `app/` root — standard Laravel layout, no new top-level project folders.

## 7. Screening pipeline

1. Candidate submits the apply form. `SubmitApplication` validates uniqueness, stores the PDF on the private `local` disk at `cvs/{company_id}/{application_id}/{uuid}.pdf`, creates the `applications` row with status `Submitted` and an `application_screenings` row with status `Pending`, then dispatches `ScreenApplicationJob`.
2. The candidate is redirected immediately with a success flash. No AI latency in the request.
3. `ScreenApplicationJob` (`ShouldQueue`, `tries = 3`, exponential backoff, `ShouldBeUnique` on the application id):
   - runs `CvTextExtractor` and saves `cv_text`;
   - if the extracted text is shorter than a minimum threshold, marks the screening `Failed` with "Could not read text from this PDF (it may be a scanned image)" and returns without calling OpenAI;
   - otherwise calls `CvScreener::screen()`, writes `match_score`, `summary`, `strengths`, `gaps`, `recommendation`, model and token counts, sets status `Completed`;
   - `failed()` marks the screening `Failed` with the exception message.
4. HR sees the score in the applications list. Rows still `Pending` render a skeleton. The page polls every 5 seconds using Inertia's `usePoll`, started only when the server reports at least one pending screening on that posting and stopped as soon as none remain. `Failed` rows show a "Re-run screening" button that re-dispatches the job.

`ShouldBeUnique` is keyed on the application id and released once the job finishes, so a re-screen after a failure is never blocked by the completed original.

### Structured output contract

One Responses API call per screening, with `text.format` set to `{ type: 'json_schema', name: 'cv_screening', strict: true, schema: … }`:

```json
{
  "type": "object",
  "properties": {
    "match_score":     { "type": "integer", "minimum": 0, "maximum": 100 },
    "summary":         { "type": "string" },
    "strengths":       { "type": "array", "items": { "type": "string" } },
    "gaps":            { "type": "array", "items": { "type": "string" } },
    "recommendation":  { "type": "string", "enum": ["advance", "hold", "reject"] }
  },
  "required": ["match_score", "summary", "strengths", "gaps", "recommendation"],
  "additionalProperties": false
}
```

The system prompt states the assistant is screening a CV against one job posting, must judge only on evidence present in the CV, and must not infer or invent experience. The user message carries the job title, description, requirements, minimum years of experience, and the extracted CV text.

`OpenAiCvScreener` json-decodes `$response->outputText` and hydrates `ScreeningResult`; a decode failure or missing key throws `AiResponseException`, which fails the queue job into the retry path.

## 8. Decision flow

**Shortlist** — `POST /hr/applications/{application}/shortlist` → `ShortlistApplication` sets status `Shortlisted`, `decided_at`, `decided_by`. Redirect back with flash.

**Reject** — a dialog on the application detail page with a textarea, driven by two JSON endpoints and one form post:

| Endpoint | Returns | Purpose |
| --- | --- | --- |
| `POST /hr/applications/{a}/rejection/draft` | `{ "message": string }` | AI writes a rejection from scratch using the job, the screening gaps, and the CV |
| `POST /hr/applications/{a}/rejection/enhance` | `{ "message": string }` | AI rewrites HR's own text — clearer, warmer, professional — without inventing new reasons |
| `POST /hr/applications/{a}/reject` | redirect | persists the final text and sets status `Rejected` |

The two AI endpoints return JSON rather than an Inertia redirect, and React calls them with Inertia v3's `useHttp` hook, so HR stays in the dialog and keeps editing. Both carry `throttle:10,1` because every click costs money.

`rejection_reason` is validated `required|string|min:20|max:2000` on the final post — HR can always overrule the AI, but cannot reject with an empty message.

The candidate sees the reason on their own application page once the status is `Rejected`.

## 9. Files and security

- CVs live on the **private `local` disk**, never `public`. Path: `cvs/{company_id}/{application_id}/{uuid}.pdf`.
- Upload validation: `required|file|mimes:pdf|max:5120` (5 MB).
- Download is a controller action guarded by `ApplicationPolicy::view` that streams the file; the raw path is never exposed to the browser.
- Cross-company access to any HR route returns 403.
- The two AI endpoints and the apply endpoint are rate limited.

## 10. Routes

```
public
  GET  /                          welcome
  GET  /jobs                      published postings across all companies
  GET  /jobs/{jobPosting}         posting detail + apply CTA

auth: candidate
  POST /jobs/{jobPosting}/apply
  GET  /my/applications           list with status and rejection reason

auth: company_admin|hr
  GET    /hr/jobs                             index
  GET    /hr/jobs/create                      create
  POST   /hr/jobs                             store
  GET    /hr/jobs/{jobPosting}/edit           edit
  PUT    /hr/jobs/{jobPosting}                update
  DELETE /hr/jobs/{jobPosting}                destroy
  GET  /hr/jobs/{jobPosting}/applications     ranked candidate list
  GET  /hr/applications/{application}         detail + screening + decision
  GET  /hr/applications/{application}/cv      streamed download
  POST /hr/applications/{application}/shortlist
  POST /hr/applications/{application}/reject
  POST /hr/applications/{application}/rejection/draft      (json)
  POST /hr/applications/{application}/rejection/enhance    (json)
  POST /hr/applications/{application}/rescreen

auth: company_admin
  GET  /hr/team
  POST /hr/team/invitations
  DELETE /hr/team/invitations/{invitation}

guest
  GET|POST /register              candidate (Fortify)
  GET|POST /register/company
  GET|POST /invitations/{token}
```

HR routes live in a new `routes/hr.php` required from `routes/web.php`, mirroring the existing `routes/settings.php` convention.

`/dashboard` becomes a role-aware redirect: HR roles to `/hr/jobs`, candidates to `/my/applications`.

## 11. Front end

Pages under `resources/js/pages/`:

```
jobs/index.tsx                    public listing
jobs/show.tsx                     detail + apply form
candidate/applications/index.tsx  my applications
hr/jobs/index.tsx                 my company's postings
hr/jobs/create.tsx                \ both render the shared
hr/jobs/edit.tsx                  / components/job-posting-form.tsx
hr/jobs/applications.tsx          ranked candidates, score column
hr/applications/show.tsx          screening detail + shortlist/reject
hr/team/index.tsx                 members + invitations
auth/register.tsx
auth/register-company.tsx
invitations/accept.tsx
```

All navigation and form submission goes through Wayfinder imports from `@/actions/…` and `@/routes/…`. No hardcoded URL strings.

Shared components:

- `MatchScoreBadge` — 0–100 with fixed colour bands: 75+ green, 50–74 amber, below 50 red.
- `ScreeningPanel` — summary, strengths, gaps, recommendation.
- `RejectDialog` — textarea plus Generate and Enhance buttons wired to `useHttp`; Enhance is disabled while the textarea is empty, and both buttons show a spinner and stay disabled during a request.

Inertia shared props gain `auth.user.roles` and `auth.user.company` so the sidebar renders the right navigation per role.

## 12. Testing

Pest feature tests, one file per slice:

- company registration creates company + admin with the right role
- invitation accept creates an `hr` user on the right company; expired and reused tokens are rejected
- job posting CRUD, and **a user from company B gets 403 on company A's posting** — the tenancy test that matters most
- publishing a posting makes it visible on the public listing; drafts are not
- applying stores the file (`Storage::fake`), creates the screening row, and dispatches the job (`Queue::fake`)
- applying twice to the same posting fails validation
- `ScreenApplicationJob` with a swapped fake screener writes a completed screening; a screener that throws leaves the screening `Failed`
- shortlist and reject transition status and record the decider
- rejecting with a blank message fails validation
- rejection draft and enhance endpoints return JSON from a swapped fake writer
- a candidate sees their own rejection reason and cannot see another candidate's application

Unit tests: `CvTextExtractor` against a fixture PDF; `OpenAiCvScreener` and `OpenAiRejectionWriter` against `OpenAI::fake()`, asserting the request carries the configured model and the strict schema.

Factories for `Company`, `JobPosting`, `Application`, `ApplicationScreening`, with states such as `published()`, `screened()`, `rejected()`.

## 13. Build order

Each step ends green — migrations run, tests pass, Pint clean.

1. Install packages; publish spatie migrations; add `OPENAI_MODEL` and `OPENAI_API_KEY` to `.env` and `.env.example`; add `config/hr.php`.
2. Enums, migrations, models, relations, factories, roles seeder.
3. Auth: enable Fortify registration, `CreateNewUser` with the candidate role, register page; company registration; invitations; role-aware dashboard redirect.
4. Policies and route groups; sidebar navigation per role.
5. Job posting CRUD (HR) + public listing and detail.
6. Apply flow: form request, `SubmitApplication`, file storage, candidate applications page.
7. AI screening: contracts, DTO, `CvTextExtractor`, `OpenAiCvScreener`, `ScreenApplicationJob`, binding, config.
8. HR candidate list with scores, polling, re-screen; application detail page.
9. Decisions: shortlist, reject, `OpenAiRejectionWriter`, draft/enhance endpoints, `RejectDialog`.
10. Full test pass, `vendor/bin/pint`, `composer types:check`, seeder for demo data.

## 14. Explicitly out of scope

Email notifications; multi-stage interview pipelines; CV formats other than PDF; OCR for scanned PDFs; candidate profile with a reusable CV; job search and filtering beyond a simple list; analytics; billing; company branding; resume of applications after a posting is closed.

## 15. Operational notes

- The queue must be running for screening to happen: `php artisan queue:work` (or `composer run dev`, which the starter kit wires up).
- With no `OPENAI_MODEL` set, `OpenAiCvScreener` throws immediately with an explicit message rather than calling the API with a null model.
- Costs scale with applications; the rejection endpoints are the only user-triggered AI calls and are throttled.
