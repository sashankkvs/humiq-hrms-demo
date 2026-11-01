# Humiq prototype architecture

This prototype is a front-end demonstration of a modular-monolith HRMS for a 11–50 person company. It is intentionally self-contained and uses seeded data so the architecture and workflows can be demonstrated without a backend.

## Modules represented

- **Identity & access** — workspace switcher, role-aware navigation, and settings.
- **People** — employee directory and organization data.
- **Work** — projects, standalone tasks, assignees, priority, due dates, and status.
- **Projects** — universal cross-team project view with summaries, progress, members, and member-only resource links.
- **Time & approvals** — weekly timesheet submissions, review, approve/return workflow.
- **Leave** — requests, balances, and manager approval.
- **Payroll prep** — approved-time inputs, compensation readiness, exception checks, and immutable export history.
- **Reporting** — department hours and data-quality signals.
- **Onboarding** — new-hire pipeline for HR/managers plus a required employee checklist.
- **Appointments / Calendar** — Google Calendar connection state, upcoming events, availability, and booking workflow.

## Role dashboards and onboarding gate

The prototype has an Employer / Employee mode switcher:

- **Employer** can review people, work allocation, timesheets, leave, payroll prep, and a new-hire pipeline showing each person’s current onboarding stage and progress.
- **Employee** gets a personal dashboard with assigned work, hours, leave balance, and onboarding progress.
- **New employees are gated**: work, hours, and leave routes resolve back to the onboarding checklist until all five required steps are complete. Completing the final step unlocks the employee workspace.

In production, this should be enforced server-side with an `OnboardingStatus` / `OnboardingTask` model and an authorization policy such as `RequireOnboardingComplete` for employee self-service endpoints. The UI gate is only a usability layer.

## Google Calendar integration boundary

The prototype shows seeded calendar events and booking actions. A production adapter should:

1. Sign users in with Google OAuth/OIDC and store refresh tokens encrypted server-side.
2. Use the narrowest Calendar scopes required: read-only for viewing, free/busy for availability, and event write access for booking.
3. Read events with `events.list`, check availability with `freebusy.query`, and create appointments with `events.insert` using explicit `start`, `end`, and `timeZone` values.
4. Store the Google event ID, calendar ID, sync metadata, and an audit record so retries are idempotent.

The app uses `Asia/Tokyo` in the demo. Real integration needs domain-admin consent decisions, OAuth consent-screen configuration, token revocation handling, webhook/sync strategy, and privacy review.

## Projects and resource access

The universal Projects page should map to `Projects`, `ProjectMembers`, and `ProjectResources` tables. Resource links should be checked by the API on every request or redirect; hiding a link in the UI is not authorization. A minimal production model is:

```text
Projects
ProjectMembers(ProjectId, EmployeeId, Role)
ProjectResources(ProjectId, ResourceType, Url, Title, Visibility)
ProjectActivity(ProjectId, ActorId, Action, CreatedAt)
```

For private documents, use object-storage keys rather than public URLs and issue short-lived signed URLs only after verifying project membership.

## Recommended production shape

```text
React + TypeScript + Vite
          │ REST / OpenAPI
ASP.NET Core modular monolith
  ├─ Identity & Access
  ├─ People & Organization
  ├─ Work & Projects
  ├─ Time & Leave
  ├─ Payroll Integration
  └─ Reporting / Audit
          │ EF Core
      PostgreSQL
```

Keep one deployable API and one PostgreSQL database initially. Enforce authorization on the API, not just in the UI. Model approval actions as explicit state transitions and append audit events for approvals, returns, payroll exports, and sensitive employee changes.

## Demo flow

1. Start on **Overview** to show headcount, team hours, approvals, and recent activity.
2. Open **People** and search the directory.
3. Open **Work & projects** to show project-based and standalone tasks.
4. Open **Timesheets**, approve a pending submission, and show the confirmation toast.
5. Open **Leave** and review a request.
6. Open **Payroll prep** and create the October export snapshot.

## Next implementation slice

Replace the in-memory arrays in `app.js` with typed API clients, then add PostgreSQL migrations for employees, projects, tasks, time entries, timesheets, leave requests, compensation records, payroll exports, and audit logs. Google Workspace OIDC and Japan payroll-provider integration should be validated against current official documentation before production work begins.

