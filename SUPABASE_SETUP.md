# Dayflow Supabase setup

Dayflow uses a versioned Supabase migration as the canonical backend definition. It creates profiles, attendance, leave balances and requests, payroll, announcements, employee-document metadata and storage policies, along with the RPCs used by the browser.

The migration is designed for a new Supabase project. For an existing production database, review it and convert the required differences into a separate forward-only migration rather than rerunning it blindly.

## 1. Create the project and apply the migration

1. Create a Supabase project at [database.new](https://database.new/).
2. Open **SQL Editor → New query**.
3. Open [`supabase/migrations/202608220001_dayflow.sql`](supabase/migrations/202608220001_dayflow.sql) from this repository, copy the complete file into the query, and click **Run** once.
4. Keep email/password authentication enabled. Email confirmation should be enabled for production.

The migration is the source of truth; this guide intentionally does not duplicate its SQL. It explicitly enables RLS, removes anonymous table access, grants only the required authenticated operations, locks down function execution, and creates the private `employee-documents` bucket.

If you later adopt the Supabase CLI after applying this file in SQL Editor, first verify the schema, link the project, and mark this already-applied version in migration history:

```sh
supabase migration repair --status applied 202608220001
supabase migration list
```

Use `supabase db push` only for later forward migrations. Do not push this baseline a second time or paste edited fragments into production without recording them as a new migration.

## 2. Bootstrap the first administrator

The Auth trigger never trusts signup metadata for employee IDs or roles:

- Every Auth user initially receives a server-generated employee ID.
- Every new Auth user starts inactive, including a raw Dashboard/Admin API invite.
- Dayflow's trusted Vercel invitation endpoint finalizes the HR-supplied business employee ID and activates the user only after profile provisioning succeeds.
- Every new profile receives a leave-balance row for the current year.

To create the first administrator:

1. Temporarily allow email signup in **Authentication → Providers → Email**.
2. Sign up through Dayflow with your real work email and verify it.
3. Copy the user's UUID from **Authentication → Users**.
4. Run this one-time statement in SQL Editor, replacing the UUID:

```sql
update public.profiles
set role = 'admin',
    employment_status = 'active',
    department = 'People & Culture',
    job_title = 'HR Administrator',
    hire_date = current_date
where id = 'PASTE_AUTH_USER_UUID_HERE';
```

5. Sign out and back in.
6. Disable public signup again. Production employees should be created with Dayflow's **Add employee** invitation flow, not self-registration.

Self-signups remain inactive even if public signup is accidentally left enabled. They cannot use attendance, submit leave, browse the employee directory, or read announcements. An administrator must explicitly activate them.

## 3. Configure Vercel and the invitation API

Add these public build-time values in **Vercel → Project Settings → Environment Variables** for Production, Preview, and Development:

```dotenv
VITE_SUPABASE_URL=https://YOUR_PROJECT_REF.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=sb_publishable_YOUR_KEY
```

Vite embeds every `VITE_` value in the browser bundle. Only a publishable key (or legacy `anon` key) is safe there.

The server-side endpoint at `api/invite-employee.js` needs the service key below. `SUPABASE_URL` is an optional server-only override; when omitted, the endpoint reuses `VITE_SUPABASE_URL`.

```dotenv
SUPABASE_URL=https://YOUR_PROJECT_REF.supabase.co
SUPABASE_SERVICE_ROLE_KEY=YOUR_SERVER_ONLY_SECRET_OR_LEGACY_SERVICE_ROLE_KEY
```

Despite the compatibility-oriented variable name, `SUPABASE_SERVICE_ROLE_KEY` may contain Supabase's current `sb_secret_...` server key or a legacy `service_role` key. It must never be prefixed with `VITE_`, committed to Git, returned to the browser, or exposed in logs. Scope it only to environments where employee invitations should work and redeploy after adding or rotating it.

Use the server key in Production by default. Add it to Preview only when preview deployments are built exclusively from trusted branches; never expose production secrets to untrusted pull-request code.

The invitation endpoint verifies the caller's access token and requires an active HR/admin profile before using the server key. It rejects pre-existing email identities, creates an initially inactive Auth user with a generated temporary employee ID, then updates the trigger-created profile with the administrator-supplied business employee ID and employee details and activates it. If the profile update fails, it deletes the newly created Auth user. The recipient follows the invite link and Dayflow requires them to set a password before entering the workspace. Use this endpoint rather than a raw Supabase Dashboard invite; raw invites intentionally remain inactive until a trusted administrator provisions them.

For local frontend-only development, copy `.env.example` to `.env` and add the two public `VITE_` values. Local employee invitations additionally require running a server environment that exposes the Vercel API route and provides the two server-only variables.

## 4. Configure authentication URLs

After the first Vercel deployment, open **Supabase → Authentication → URL Configuration**:

1. Set **Site URL** to the exact production origin, such as `https://your-project.vercel.app` or the production custom domain.
2. Add `http://localhost:5173/**` for local Vite development.
3. Add the exact production URL with `/**`.
4. If authentication must work on Vercel previews, add `https://*-YOUR_VERCEL_TEAM_OR_ACCOUNT_SLUG.vercel.app/**`.

Use exact production URLs wherever possible and reserve wildcards for previews. Dayflow sends `window.location.origin` for email-confirmation redirects. The server-side invitation uses the configured Site URL, so it must point to the deployed application.

If a confirmation template is customized, keep Supabase's confirmation token/link variables intact and use `{{ .RedirectTo }}` where an environment-specific redirect target is needed. A hardcoded production URL prevents preview/local confirmation callbacks from returning to their originating environment.

## 5. What the hardened database enforces

- Active and on-leave employees see their private records, while `list_employee_directory()` returns only safe directory fields. Inactive users can read only their own profile so the UI can show its activation state.
- Employees can edit only their contact/avatar fields. HR can maintain employee profiles but cannot modify administrators, peer HR roles, or Auth identity fields; administrators may change roles, while identity changes remain server-only.
- Employees record attendance only through `set_attendance_state()`. The database supplies `auth.uid()`, `current_date`, and `now()`, and a completed day cannot be overwritten.
- HR may correct attendance rows directly under RLS.
- Leave duration is calculated from dates by the database. Employee requests remain pending and cannot carry forged decision fields.
- HR decisions go only through `decide_leave_request()`, are pending-only, cannot approve the caller's own request, set the approver/time on the server, and consume paid/sick/unpaid balances atomically.
- Announcement visibility honors `all`, `employee`, `hr`, and `admin` audiences; on-leave HR/admin accounts have employee-level read access until reactivated.
- Document metadata must point into the owning employee UUID folder, and uploader identity is assigned by the database.
- Anonymous requests have no access to HRMS tables or RPCs.

Leave approver references are restrictive so approval history cannot silently lose its actor. Deactivate an HR/admin profile that has made leave decisions instead of deleting its Auth user; a deliberate archival migration is required if permanent deletion is legally necessary.

`dayflow_today()` exposes the database date used for attendance so the frontend does not rely on a browser-local date. Supabase projects normally use UTC. If Dayflow must use a different legal/business timezone, introduce that as a dedicated follow-up migration and update `dayflow_today()`, attendance RPCs, and leave validation together.

Leave duration is inclusive calendar-day arithmetic. Requests cannot cross a calendar year and must be split at year-end; weekends and organization holidays are not excluded unless a later migration adds a business-calendar table.

## 6. Verify before production

Confirm RLS is enabled:

```sql
select schemaname, tablename, rowsecurity
from pg_tables
where schemaname = 'public'
  and tablename in (
    'profiles', 'attendance', 'leave_balances', 'leave_requests',
    'payroll', 'announcements', 'employee_documents'
  )
order by tablename;
```

Every `rowsecurity` value must be `true`.

Confirm the browser RPC grants:

```sql
select routine_schema, routine_name, grantee, privilege_type
from information_schema.routine_privileges
where routine_schema = 'public'
  and routine_name in (
    'dayflow_today', 'list_employee_directory',
    'set_attendance_state', 'decide_leave_request'
  )
order by routine_name, grantee;
```

Apart from the database owner, only `authenticated` should have `EXECUTE` for these RPCs. Trigger/internal functions in `private` must not be callable by `anon`.

Then test with three real accounts: an employee, an HR user, and an administrator.

- A signed-out request cannot read any HRMS table or call an RPC.
- A direct self-signup is inactive and cannot browse the directory or mutate attendance/leave.
- An employee invited through Dayflow's Vercel endpoint is active and receives a current-year leave balance; a raw Supabase invite remains inactive.
- An employee sees safe directory fields but cannot read another employee's private profile or payroll.
- Direct employee attendance inserts/updates fail; clock-in/out through the RPC succeeds once in each direction.
- A second check-in after checkout fails and preserves the original timestamps.
- Browser-supplied leave `days` is ignored and recalculated from the dates.
- An employee cannot set decision fields, approve leave, or exceed paid/sick allowance.
- HR cannot directly change a leave decision or a profile role; the decision RPC succeeds for another employee's pending request.
- HR cannot edit or deactivate an administrator or peer HR account.
- Only an administrator can promote/demote roles.
- Audience-specific announcements are invisible to unauthorized roles.
- Employee document objects and metadata cannot escape the employee UUID folder.
- The Vercel invitation endpoint rejects employees and inactive HR users, but succeeds for active HR/admin callers.

Finally, review **Database → Advisors**, Auth logs, Postgres logs, and the Vercel function logs. Resolve all security findings before production data is imported.

## Current Supabase references

- [Securing the Data API](https://supabase.com/docs/guides/api/securing-your-api)
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Database functions](https://supabase.com/docs/guides/database/functions)
- [Managing user data](https://supabase.com/docs/guides/auth/managing-user-data)
- [Redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls)
- [Supabase API keys](https://supabase.com/docs/guides/getting-started/api-keys)
