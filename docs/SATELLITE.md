# DocuSeal — OWeb satellite

`app_id`: **docuseal**  
Activation scope: **workspace**  
Hosting: **Docker / self-hosted** (upstream DocuSeal); Vercel project **`docuseal-oweb`** holds constellation secrets and future edge entrypoints.  
Auth host: **https://auth.oweb.one**  
Control plane: **https://oweb.one**

DocuSeal joins the OWeb constellation for **e-signatures and document flows** inside workspaces. Identity, workspaces, and billing stay on One OS — this fork does not run a second Supabase Auth project.

Canonical contracts (OWeb repo):

- [`SATELLITE_ONBOARDING_KIT.md`](https://github.com/OneWeb-LLC/oweb/blob/main/docs/SATELLITE_ONBOARDING_KIT.md)
- [`SATELLITE_AUTH_RETURN.md`](https://github.com/OneWeb-LLC/oweb/blob/main/docs/SATELLITE_AUTH_RETURN.md)

## Auth flows (target)

| Path | Entry | Notes |
|------|--------|--------|
| Native | `/login` (product) | Same OneID email/password against shared Auth |
| OWeb return | `https://oweb.one/login?launch=docuseal` | App Store mints token → `/sso?launch_token=…` |
| SSO redeem | `/sso` | Redeem via OWeb `POST /api/v1/ecosystem/redeem-launch-token` — no service role in the app image |

Post-auth: record app + workspace activation (`/api/v1/oneid/activate*`), upsert namespaced profile (`ds_*` / `docuseal_*`).

## Multi-tenant isolation

| Layer | Isolation |
|-------|-----------|
| Identity / billing | One OS (`auth.oweb.one`, `ao_org_members`, `ao_orgs`) |
| DocuSeal domain data | Namespaced tables or org-scoped rows keyed by `workspace_id` / `org_id` |
| File storage | Disk/S3 per deployment; scope uploads by workspace when integrated |

## Environment

Production/runtime secrets live in the team Vercel project **`docuseal-oweb`** (shared One OS keys copied from **`oweb`**).

| Variable | Where | Required |
| --- | --- | --- |
| `SUPABASE_URL` | Runtime | Yes — One OS project |
| `SUPABASE_PUBLISHABLE_KEY` | Runtime / client | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Server only (SSO redeem, admin bridges) | Yes when SSO is enabled |
| `SUPABASE_PROJECT_ID` | Tooling | Yes |
| `VITE_SUPABASE_*` | Any Vite-side tooling | Mirror of client Supabase config |
| `NEXT_PUBLIC_OWEB_URL` | Config | `https://oweb.one` |
| `OWEB_APP_ID` | Config | `docuseal` |
| `PUBLIC_APP_URL` | Config | Public DocuSeal origin (update when domain is set) |
| `DATABASE_URL` | Docker / host | DocuSeal Postgres (separate from One OS) |
| `SECRET_KEY_BASE` | Docker / host | Rails secret |

See [`OWEB_SATELLITE_ENV.example`](./OWEB_SATELLITE_ENV.example).

## Isolation checklist

- [ ] Supabase client points at One OS only
- [ ] No duplicate signup/password store in DocuSeal
- [ ] Domain tables prefixed (`docuseal_*` or `ds_*`)
- [ ] SSO path does not embed `SUPABASE_SERVICE_ROLE_KEY` in client bundles

## Auth-return smoke checklist

1. Product login with OneID → dashboard + activations  
2. Continue with OWeb → SSO → dashboard  
3. New user via `oweb.one/signup?launch=docuseal` → SSO return  
4. Direct `oweb.one/login?launch=docuseal`  
5. Failed mint shows actionable error (not silent marketing redirect)

## Control-plane registration

Register in OWeb `src/lib/ecosystem-apps.ts` with production `launchUrl` matching `PUBLIC_APP_URL` before marking `status: live` in the App Store.