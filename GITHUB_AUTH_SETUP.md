# Setting up GitHub sign-in (via Supabase Auth)

`index.html` gates the admin panel behind GitHub sign-in: a visitor must authenticate with
GitHub through Supabase, and their GitHub username must appear in an allowlist baked into the
page, before the dispatch UI (`<main>`) is shown. This document walks through every step needed
to stand that up from scratch — creating the Supabase project, wiring up GitHub as an OAuth
provider, and filling in the values `index.html` needs.

Nothing here requires a backend of its own: Supabase hosts the auth service, and the GitHub
OAuth App just needs to know Supabase's callback URL. `index.html` remains a static page.

---

## 1. Create a Supabase account and project

1. Go to [supabase.com](https://supabase.com) and sign up (GitHub sign-up is fine and is
   unrelated to the GitHub OAuth App you'll create in step 3 — that one is for *your users*
   signing into the panel, this one is just how *you* log into Supabase's dashboard).
2. Create a new **Organization** if prompted (free tier is enough for this).
3. Click **New Project**:
   - Pick the organization, give the project a name (e.g. `clinic-software-hub-admin`).
   - Set a database password (you won't need it for this auth-only use case, but Supabase
     requires one) — store it somewhere, you likely won't use it again here.
   - Pick a region close to your users.
   - Click **Create new project** and wait (~1-2 minutes) for it to provision.

## 2. Get the Supabase URL and publishable key

1. In the Supabase dashboard, open your project.
2. Go to **Project Settings** (gear icon, bottom of the left sidebar) → **API Keys**.
3. Copy two values:
   - **Project URL** — looks like `https://<project-ref>.supabase.co`.
   - **Publishable key** — looks like `sb_publishable_...` (older projects instead show an
     `anon` `public` key under "Project API keys" — either works the same way here).

   Both of these are meant to be exposed in client-side code (that's the point of the
   publishable/anon key — Supabase's security model relies on Row Level Security, not on
   this key being secret). Do **not** use the `service_role`/secret key anywhere in
   `index.html` — that one must stay server-side only, and this project has no server.

Keep this browser tab open; you'll paste both values into `index.html` in step 6.

## 3. Create a GitHub OAuth App

This is a **GitHub OAuth App**, not a personal access token and not a GitHub App — it's what
lets Supabase ask GitHub "did this person really authenticate as themselves?"

1. First, go back to Supabase: **Authentication → Providers → GitHub** (don't enable it yet) —
   it displays a **Callback URL** of the form:
   ```
   https://<project-ref>.supabase.co/auth/v1/callback
   ```
   Copy this exact URL; you need it in the next step.
2. On GitHub: **Settings → Developer settings → OAuth Apps → New OAuth App**
   (`https://github.com/settings/developers`).
3. Fill in the form:
   - **Application name**: anything descriptive, e.g. `Clinic Software Hub Admin`.
   - **Homepage URL**: the page's public URL, e.g. `https://clinic-software-hub.github.io/`.
   - **Authorization callback URL**: paste the Supabase callback URL from step 1 exactly.
4. Click **Register application**.
5. On the app's page, copy the **Client ID**.
6. Click **Generate a new client secret**, and copy the secret immediately — GitHub only
   shows it once.

## 4. Add the GitHub credentials to Supabase

1. Back in Supabase: **Authentication → Providers → GitHub**.
2. Toggle the provider **enabled**.
3. Paste the **Client ID** and **Client Secret** from step 3.
4. Click **Save**.

## 5. Configure allowed redirect URLs in Supabase

Supabase will only redirect back to URLs you've explicitly allowed — otherwise sign-in silently
falls back to the project's default (`http://localhost:3000`), which looks like a broken
redirect.

1. Go to **Authentication → URL Configuration**.
2. Set **Site URL** to wherever the page is actually served from, e.g.
   `https://clinic-software-hub.github.io`.
3. Under **Redirect URLs**, add every origin you'll open the page from, for example:
   - `https://clinic-software-hub.github.io/*` (production, GitHub Pages)
   - `http://localhost:8000/*` (if testing locally via `python3 -m http.server`)
4. Save.

## 6. Fill in the values inside `index.html`

Open `index.html` and find this block near the top of the `<script>` (currently around
line 380):

```js
var SUPABASE_URL = "REPLACE_ME";             // Project URL from step 2
var SUPABASE_PUBLISHABLE_KEY = "REPLACE_ME"; // Publishable key from step 2
```

Paste in the **Project URL** and **Publishable key** you copied in step 2.

A few lines below, find the allowlist:

```js
var ALLOWED_GITHUB_USERNAMES = ["REPLACE_ME_1", "REPLACE_ME_2"];
```

Replace the placeholders with the GitHub **usernames** (not emails, not display names — the
`github.com/<username>` handle) of everyone who should be allowed past the gate. This list is
case-insensitive and is the actual access control — signing in with Supabase only proves who
someone is on GitHub, this array decides whether they're let in.

## 7. Commit, push, and test

1. Commit the changes to `index.html` and push to `main` (GitHub Pages redeploys
   automatically, usually within a minute or two).
2. Open the deployed page and click **Sign in with GitHub**.
3. Approve the OAuth consent screen (first time only).
4. You should land back on the page, signed in:
   - If your username is in the allowlist, the dashboard appears with "Signed in as
     @username" in the header.
   - If it isn't, you'll see "Access denied" instead.

### Troubleshooting

- **Redirected to `localhost:3000` with a token in the URL**: the page's URL isn't in
  Supabase's Redirect URLs allow-list yet, or Site URL wasn't updated — revisit step 5.
- **GitHub shows "redirect_uri is not associated with this application"**: the callback URL
  in the GitHub OAuth App (step 3) doesn't exactly match what Supabase expects — re-copy it
  from Authentication → Providers → GitHub and compare carefully (including `https://` and no
  trailing slash differences).
- **Signed in but always "Access denied"**: check `ALLOWED_GITHUB_USERNAMES` — it must contain
  your exact GitHub username (the one after `github.com/`), not your display name or email.
