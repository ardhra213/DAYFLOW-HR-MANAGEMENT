# GitHub, Vercel, and Supabase deployment

This is the production deployment contract for Dayflow. GitHub stores the source and runs CI, Vercel builds the Vite app and hosts the `/api` function, and Supabase provides authentication and HRMS data.

## 1. Verify the project locally

Use Node.js 22, as declared in `.nvmrc` and `package.json`:

```bash
npm ci
npm run typecheck
npm run lint
npm run build
```

For local Vite development, copy `.env.example` to `.env`. Leaving both Supabase client values blank opens the local demo. To use a real Supabase project, fill in the project URL and publishable key. Never commit `.env` or any other `.env*` file.

## 2. Push the source to GitHub

Create an empty GitHub repository, then run these commands from the Dayflow project root:

```bash
git init -b main
git add .
git commit -m "Deploy Dayflow"
git remote add origin https://github.com/YOUR_ACCOUNT/YOUR_REPOSITORY.git
git push -u origin main
```

The included GitHub Actions workflow runs `npm ci`, type checking, linting, and the production build on pull requests and pushes to `main`. In GitHub branch protection, require the **Typecheck, lint, and build** check before merging to `main`.

## 3. Create and configure Supabase

1. Create the Supabase project.
2. Follow `SUPABASE_SETUP.md` and apply `supabase/migrations/202608220001_dayflow.sql` in the Supabase SQL Editor.
3. From the Supabase Connect/API Keys screen, copy:
   - the Project URL;
   - the publishable key (`sb_publishable_...`);
   - a server secret (`sb_secret_...`) or legacy `service_role` key for the Vercel invitation function.
4. Create the first administrator as described in `SUPABASE_SETUP.md`.

The publishable key is intentionally safe for browser use when Row Level Security is enabled. The server secret bypasses RLS and must never be sent to the browser, committed to GitHub, placed in a `VITE_` variable, or printed in logs.

## 4. Import GitHub into Vercel

1. In Vercel, choose **Add New → Project** and import the GitHub repository.
2. Keep **Root Directory** set to the repository root (`.`).
3. Confirm these settings, which are also enforced by `vercel.json`:
   - Framework Preset: **Vite**
   - Install Command: `npm ci`
   - Build Command: `npm run build`
   - Output Directory: `dist`
   - Node.js: **22.x**
4. Set the production branch to `main` and leave automatic Git deployments enabled.

The SPA rewrite sends client-side application paths to `index.html`. Vercel checks filesystem routes before rewrites, so files such as `api/invite-employee.js` remain Vercel Functions.

## 5. Add Vercel environment variables

Add these in **Project Settings → Environment Variables** before the production deployment:

| Variable | Value | Vercel environments | Exposure |
| --- | --- | --- | --- |
| `VITE_SUPABASE_URL` | `https://YOUR_PROJECT_REF.supabase.co` | Production, Preview, Development | Browser-visible; client-safe |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | `sb_publishable_...` | Production, Preview, Development | Browser-visible; client-safe with RLS |
| `VITE_ENABLE_DEMO_MODE` | `false` | Production, Preview, Development | Browser-visible feature flag |
| `VITE_ALLOW_SELF_SIGNUP` | `false` | Production, Preview, Development | Browser-visible feature flag |
| `SUPABASE_SERVICE_ROLE_KEY` | `sb_secret_...` or legacy service-role key | Production | Server-only secret |

The four client variable names should exist in all three scopes, but their Supabase values do not have to be identical. Prefer the production Supabase project only in Production and a separate staging project in Preview and Development.

Keep the production service key out of Preview. If invitation testing is required in previews, use the staging project's server key and restrict access to those deployments. GitHub Actions does not need any Supabase secret.

All `VITE_` values are embedded at build time. Vercel Functions also receive the environment snapshot from their deployment. **Redeploy after adding, changing, or rotating any environment variable.**

## 6. Configure Supabase authentication URLs

After Vercel assigns the production domain, open **Supabase → Authentication → URL Configuration** and set:

- **Site URL:** the exact production origin, for example `https://dayflow.vercel.app` or the exact custom domain;
- **Redirect URLs:**
  - `https://YOUR_PRODUCTION_DOMAIN/**`
  - `https://*-YOUR_VERCEL_TEAM_OR_ACCOUNT_SLUG.vercel.app/**`
  - `http://localhost:5173/**` for local development.

Use the exact production origin rather than a wildcard for the Site URL. The Vercel wildcard is only for Preview deployments. If authentication email templates are customized, preserve Supabase's redirect target rather than hardcoding a different site.

Changing Supabase URL Configuration takes effect in Supabase and does not rebuild the frontend. Changing Vercel environment variables always requires a redeploy.

## 7. Verify the connected deployment

1. Confirm GitHub CI is green and the Vercel deployment log shows `npm ci` and a successful Vite build.
2. Open the production URL and verify it shows the sign-in screen, not the demo workspace or a configuration error.
3. Open `/favicon.svg` and `/site.webmanifest` and confirm both return their static files.
4. Send `GET /api/invite-employee`; it should return a JSON `405 Method not allowed`, not `index.html`. This confirms the API namespace bypasses the SPA rewrite.
5. Sign in as the first administrator and invite an employee. The invitation must be handled by the Vercel Function without exposing the server key; the invite recipient should be prompted to set a password before entering the workspace.
6. Sign in as an employee and verify RLS limits profile, attendance, leave, and payroll access as described in `SUPABASE_SETUP.md`.
7. Push a test branch and confirm Vercel creates a Preview deployment; merge through `main` to create Production.

## Troubleshooting

- **Production says Supabase is not configured:** check both client variables in the correct Vercel environment, then redeploy.
- **Employee invitation returns 503:** add `SUPABASE_SERVICE_ROLE_KEY` to the deployed environment and redeploy.
- **An `/api/*` request returns HTML:** confirm Vercel's Root Directory is the repository root and that `api/invite-employee.js` appears in the deployment's Functions list.
- **Confirmation or invitation link is rejected:** add the exact production URL or matching Preview wildcard to Supabase Redirect URLs.
- **Preview cannot invite employees:** this is expected unless that Preview environment has a separate staging server key.
