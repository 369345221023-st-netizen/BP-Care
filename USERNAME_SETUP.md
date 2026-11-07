# BP Care — Username + Password (no email field)

This version replaces the public **Login** and **Register** forms with **Username + Password**. It also removes the Google OAuth button. It preserves the current BP Care dashboard and the Admin Dashboard.

## Important: what this implementation does

Supabase Auth cannot authenticate by username alone. The `bp-username-auth` Edge Function maps a username to a Supabase account on the server, then uses the Supabase password-based authentication API. For username-only newly registered demo accounts, it creates an internal placeholder email under the reserved `example.com` domain. No user email is requested in the UI. This is **not** a verified email address and cannot receive password recovery mail. Please do not use this pilot approach for real patients' data without designing account recovery, account security and privacy review first.

**Existing Super Admin accounts are not deleted**, and their passwords do not change. To log in with their username, you must assign a unique username in the `profiles` table (step 2).

## 1. Prepare SQL

In Supabase **SQL Editor → Database → New query**, run:

`supabase/migrations/20261107_username_auth.sql`

This adds `profiles.username` if missing, a unique case-insensitive index, and server-side request-throttling storage and RPC. No existing users are removed.

## 2. Give your existing Super Admin a username

Go to **Supabase → Authentication → Users**, copy the correct **User UID** of your Super Admin and run this in SQL Editor, with a username you choose:

```sql
update public.profiles
set username = 'my_admin'
where id = 'REPLACE_WITH_REAL_SUPER_ADMIN_UUID';
```

If the Super Admin has no row in `profiles`, run this first (replace the ID):

```sql
insert into public.profiles (id, full_name)
values ('REPLACE_WITH_REAL_SUPER_ADMIN_UUID', 'System Administrator')
on conflict (id) do nothing;
```

Check that the edit affected exactly one row. Then use the **same password as before**, but enter the new username on Login.

If you have other *existing* email accounts you want to keep, assign each a distinct username in `profiles` the same way.

## 3. Deploy the new Edge Function

In Supabase **Edge Functions → Deploy a new function → Via Editor**:

- Function name: `bp-username-auth` (exact spelling)
- Source: copy all of `supabase/functions/bp-username-auth/index.ts`
- Under function **Settings**, set **Verify JWT with legacy secret = OFF**. This is a public login/register endpoint, so login JWTs don't exist *before* the user signs in. The function has username/password verification server-side, constrained CORS origins and an SQL-backed rate limit. This does not by itself protect the endpoint from all automated abuse.
- Check that the Edge Function environment has `SUPABASE_URL`, `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY`, available in standard Supabase hosted projects. **Never put service role / secret keys in Vue or `.env.local`.**
- If your Netlify hostname changes, set `BP_CARE_SITE_URL` under **Edge Functions → Secrets** to the new full URL (for example `https://your-site.netlify.app`). The local Vite origin 5173, 5174 and 4173 are already allowed.

**Security:** On a public site, implement production-grade bot protection, stronger rate limiting at the gateway, password recovery / MFA policy and audit logging *before collecting real health information*. In particular, the demo registration endpoint is publicly callable and the per-IP throttle depends on networking headers.

## 4. Local testing

Copy your existing `.env.local` into the new project directory. It must contain only your public project URL and publishable key:

```env
VITE_SUPABASE_URL=https://YOUR_PROJECT.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=YOUR_PUBLISHABLE_KEY
```

Then in the project root:

```bash
npm install
npm run dev
```

Test: register a **new username** → automatically login → logout → login again. Test the Super Admin with its newly assigned username. Check that ordinary users cannot open Admin Dashboard.

## 5. Deploy Netlify again

When testing works:

```bash
npm run build
```

Upload the updated `dist` folder to **Deploys on the existing Netlify project**. Do not create a new Netlify project. No need to upload `.env.local` to Netlify: Vite reads its `VITE_` values during your local build.

## Scope and limitations

- Public Login and Register now use **Username + Password only**. The Google button is removed.
- Supabase Auth still uses internal email identities (including existing email accounts). Do not manually replace real admin email IDs in Auth.
- The Admin Dashboard now creates Username + Password accounts through the updated `bp-admin` function. Re-deploy **bp-admin** with the new code; keeping the previously deployed version will not work. A Super Admin can create User and Admin; an Admin can only create User. The initial password is known to the creating administrator, so for production add a secure first-login password-change process.
- Username-only pilot accounts **have no automatic password recovery**. Do not store real patient data until that is solved.
- Health record persistence is still the demo behavior of the existing website; this release does not connect blood-pressure readings to Supabase.


## 6. New: Admin creates Username + Password (no email)

The `AdminDashboard.vue` form now collects **display name, username, initial password, password confirmation and role**. No email invitation is sent.

**Mandatory:** go to Supabase Edge Functions → `bp-admin` → Code, replace the deployed `index.ts` with `supabase/functions/bp-admin/index.ts` from this ZIP and deploy the new revision. The old deployed function expects an email and will not understand the new form. The function uses the server-side `SUPABASE_SERVICE_ROLE_KEY` only; never expose that key to the browser. `bp-username-auth` also needs to be deployed, and the `20261107_username_auth.sql` migration run, for members to log in with their username.

Existing Super Admin still needs their `profiles.username` assigned (see step 2). Test creating one **dummy member account** first and sign in with that username in a separate private window. Do not use patient records in this pilot; username-only accounts have no email recovery, and initial passwords known to administrators need a first-login reset design.
