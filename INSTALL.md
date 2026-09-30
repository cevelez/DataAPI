# Data API for Gibbon — Installation Guide

**Module version 1.0.5** · OneRoster 1.1 read API + write-capable User Provisioning for Gibbon SIS

This guide is for a school's own Gibbon administrator installing this module on their **own
hosting** (shared hosting / cPanel-style, or any standard LAMP/Apache setup) — not the
Railway/Docker environment this module was originally developed on. Nothing below assumes Docker,
Railway, or SSH access; everything is done through your normal hosting file manager (or SFTP) and
the Gibbon admin UI in your browser.

---

## Before you start

- You need **admin access to Gibbon** and either SFTP/SSH or your host's file manager (e.g.
  cPanel's File Manager) to upload files.
- This is a genuinely **fresh install** of this module — if you've never had this module
  installed on this Gibbon instance before, installing it requires **zero manual SQL**.
  Gibbon's own "Install" button does everything.
- Three things below matter more than the rest of this guide — read the callout boxes even if
  you skim everything else.

---

## Step 1 — Upload the module folder

1. Unzip `DataAPI-1.0.5.zip` on your own computer. You should get one folder named `DataAPI`
   containing `manifest.php`, `api.php`, and everything else.
2. Upload that whole `DataAPI` folder into your Gibbon installation's `modules/` directory, so the
   result is:
   ```
   <your Gibbon root>/modules/DataAPI/manifest.php
   <your Gibbon root>/modules/DataAPI/api.php
   <your Gibbon root>/modules/DataAPI/src/...
   ...
   ```

> ⚠️ **No spaces — anywhere in the path.**
> Keep **both** your Gibbon install path *and* this module's folder name completely free of
> spaces. Don't install Gibbon into something like `/My School SIS/`, and don't rename this
> module's folder from `DataAPI` to something friendlier like `Data API`.
>
> This isn't a generic best-practice warning — it's a specific, previously-real bug in this exact
> module. Early in development this module was almost named with a space in its own folder name,
> and that choice caused a genuine URL-encoding regression (the API's own URL routing broke). It
> was deliberately renamed to a no-space `DataAPI` folder to fix it, and that fix only works if the
> folder name — and everything in the path leading up to it — stays space-free. A GUI file manager
> makes it easy to accidentally create a spaced folder without thinking about it; double-check both
> before moving on.

## Step 2 — Confirm nothing in front of your server blocks the API

Two separate layers can silently block this module's endpoints before they ever run — check both.

### 2a. Apache: honor the module's `.htaccess`

This module ships its own `.htaccess` (inside the `DataAPI` folder) that turns on two Apache
settings its API endpoints depend on:

- `AcceptPathInfo On` for `api.php` — the REST API reads the resource path (e.g. `/orgs`,
  `/users/123`) off the URL itself, and needs this to parse it correctly.
- `CGIPassAuth On` for `api.php` and `token.php` — some shared-hosting PHP setups (particularly
  PHP-FPM via `mod_proxy_fcgi`) strip the `Authorization` header before PHP ever sees it; this
  un-strips it so OAuth bearer tokens actually reach the API.

For either of these to take effect, your hosting account's Apache config needs
**`AllowOverride All`** (or at least enough to permit these two directives) on the directory
Gibbon is installed in. This is a **real requirement worth double-checking** before you go live:

- Most shared/cPanel hosting has this on by default — you likely don't need to do anything.
- If you administer your own Apache config (a VPS, a dedicated server), confirm the `<Directory>`
  block covering your Gibbon install has `AllowOverride All`, not `AllowOverride None` or a
  restricted subset.
- If you're not sure, the simplest check is to finish the install below and try a real API call
  (Step 6) — a `404`/routing error on a URL that should resolve, or a `401` even with a correct
  bearer token, both point back to this setting.

### 2b. CDN / WAF / bot-protection: exempt the API paths

> ⚠️ **If your domain sits behind Cloudflare (or a similar CDN/WAF/bot-protection service),
> its bot-challenge features WILL block every API call to this module** — including a real
> OneRoster consumer's own connection attempts — unless you explicitly exempt the API paths.
> This isn't hypothetical: it's a real issue hit during this module's own pilot rollout. A
> correctly-configured, correctly-credentialed request came back as a `403` with an HTML "Just a
> moment..." challenge page instead of a JSON response — the request never reached Apache or PHP
> at all, so nothing about this module's own configuration (Step 2a, credentials, mappings) could
> have fixed it. Cloudflare's automated bot-detection can't distinguish a legitimate OAuth2 API
> client from a scraping bot; it will treat both the same unless told not to.

If your domain uses Cloudflare, add a Configuration Rule (or WAF custom rule) in your Cloudflare
dashboard, before generating credentials or testing:

- **Path:** match `<your domain>/modules/DataAPI/*` (or narrower: just `api.php` and `token.php`
  specifically, if you'd rather scope it tightly)
- **Setting:** set Security Level to **"Essentially Off"** for that path, or explicitly bypass
  **Managed Challenge** / **Bot Fight Mode** there

This endpoint is already protected by its own OAuth2 client credentials (Step 4) — a bot
challenge on top of it doesn't add security, it just makes the API unreachable by any real
integration. If you're behind a different CDN/WAF provider (not Cloudflare), look for the
equivalent "bypass"/"allowlist path" feature — the underlying problem and fix are the same
regardless of provider.

If you're not sure whether your domain uses one of these services, the simplest check is the same
as 2a: try a real API call (Step 6) and see what comes back. An HTML page mentioning "checking
your browser," "just a moment," or a CAPTCHA — instead of a JSON `401`/`404`/actual data — is the
tell that something in front of your server, not this module, is intercepting the request.

## Step 3 — Install the module through Gibbon

1. Log in to Gibbon as an administrator.
2. Go to **System Admin → Manage Modules**.
3. Find **Data API** in the list of available modules and click **Install**.
4. Gibbon will create this module's database tables and settings automatically — for a genuinely
   fresh install (no prior installation of this module on this instance), this is the entire
   database step. You do not need to run any SQL by hand.
5. Once installed, you'll see seven sections in the sidebar under Data API: **Status**,
   **OneRoster 1.1**, **Users**, **Demographics**, **User Provisioning**, **Access Log**, and
   **Branding**.

## Step 4 — Generate your credentials

Each connector (the read-only **OneRoster 1.1** API and the write-capable **User Provisioning**
connector) has its own client ID/secret pair, generated independently so a compromised read-only
credential and a compromised write credential are two different, separately-rotatable blast
radii.

1. **OneRoster 1.1 → Generate Credentials** — generates the client ID/secret your roster
   consumer (e.g. ClassLink's Roster Server) will use to pull data. Enable the connector and save.
2. **User Provisioning → Generate Credentials** — generates the separate client ID/secret for the
   write-capable provisioning connector, if you're using it. This has its own enable toggle and,
   on the **User Provisioning** permissions matrix, its own per-field/per-role Read / Write /
   Clear / Create permissions — nothing is writable by default until you explicitly grant it.
3. Store both secrets somewhere safe immediately — like the OneSync/User Provisioning secret,
   they're not necessarily easy to retrieve in plain text again later depending on how you've
   configured secret storage.

## Step 5 — Review the self-healing Role and Grade Mapping suggestions

Before handing credentials to a real consumer, check two pages under **Users**:

- **Role Mapping** — every Gibbon Role on your instance gets a suggested OneRoster role
  (`administrator`/`aide`/`parent`/`proctor`/`student`/`teacher`) the first time this page loads,
  pre-filled to reproduce sensible default behavior. Review each suggestion and adjust any that
  don't fit your school's roles (e.g. a "Counsellor" role you'd rather map to `teacher` than the
  generic `proctor` fallback).
- **Grade Mapping** — every Year Group gets a suggested OneRoster grade code the first time this
  page loads. Confirm these match your naming convention, especially if you use a non-American
  (e.g. British `Y7`–`Y13`) year-numbering scheme.

Suggested-but-unconfirmed mappings are **not** used by the live API until you save/confirm them —
until then, the API ships the raw Gibbon value instead. Confirm each mapping you want active.

Optionally, also review **Demographics → Demographic Mapping** (keyword-driven ethnicity/race
mapping rules) and **Users → Metadata Mapping** if you want to expose additional Gibbon fields
through the OneRoster `metadata` extension point.

## Step 6 — Test before going live

1. Request a token from your token endpoint (`token.php` for OneRoster, `provisionToken.php` for
   User Provisioning) using the credentials from Step 4.
2. Make one real read call, e.g. `GET /modules/DataAPI/api.php/ims/oneroster/v1p1/orgs`, with the
   returned bearer token, and confirm you get real data back rather than a `401`/`404`.
3. Only then hand the credentials to your actual OneRoster/provisioning consumer.

---

## Updating an existing installation to a newer version

**This section is for you if this module is already installed and working, and you've been given
a newer version to apply** (e.g. you're currently on 1.0.0 and were sent 1.0.1). It is a
different, much smaller process than Steps 1–6 above, which are for an instance that has never
had this module on it before — skip straight here instead of repeating those.

> ⚠️ **Do NOT uninstall and reinstall to update.** Gibbon's own "Install" flow seeds fresh, blank
> settings and database tables the instant it doesn't recognize an existing install of this
> module — exactly what an Uninstall-then-Install cycle would trigger. That would silently wipe
> your real credentials, Role/Grade Mapping, IP Allowlist entries, and Access Log history, not
> just update the code. There is no supported "reset and reinstall" path for applying an update to
> this module — always update the files in place instead, per the steps below.

A point release like a bug fix normally changes only some of this module's own PHP files — there
is usually **no database change at all**, and Gibbon's Manage Modules screen doesn't need to be
touched for the fix itself to take effect. (If a specific release genuinely does add a new setting
or table, that will be called out explicitly in what you're sent — don't assume it unless told.)

1. **Back up first.** Copy or zip the current `DataAPI` folder on your server aside somewhere,
   before changing anything. Cheap insurance; costs nothing if the update goes cleanly.
2. **Get the new files.** Depending on what you're sent, this is either one or two specific
   changed files, or a full `DataAPI-X.Y.Z.zip` covering the whole module.
3. **Overwrite in place, same path as before** (`modules/DataAPI/`):
   - If you were sent individual file(s), upload each one directly over the existing file at the
     exact same path (e.g. `modules/DataAPI/src/Security/TokenIssuer.php` replaces the file of the
     same name already there).
   - If you were sent a full zip, unzip it and upload the whole `DataAPI` folder over the existing
     one, overwriting every file. This is safe: it replaces code files only and never touches
     `gibbonSetting`/your other database tables, since none of that data lives in the module
     folder itself — your credentials, mappings, and Access Log history are untouched.
4. **Update the displayed version number, and let Gibbon apply any pending database change
   automatically.** System Admin → Manage Modules → Data API → **Upgrade**. As of v1.0.4, this
   module ships a real `CHANGEDB.php` (Gibbon core's own native mechanism for this — previously
   this module didn't have one, so this button only ever updated the displayed number, nothing
   else; that's fixed now). What actually happens when you click it: Gibbon compares your
   currently-installed version to the version in the files you just uploaded, runs any SQL
   `CHANGEDB.php` defines for versions in that range, and only then updates the displayed number
   — so for most releases (ones with no database change at all) this is still purely cosmetic,
   exactly as before, but for a release that DOES need one, clicking this now applies it for you
   instead of you copying `INSERT`/`UPDATE` statements by hand. It's still safe to click even if
   you already ran a version's SQL by hand from an older copy of this guide (as ISU did for 1.0.3,
   before `CHANGEDB.php` existed) — the statements it runs are written to be safe to run twice
   (e.g. `INSERT IGNORE` rather than a plain `INSERT`), so re-running one that already applied is
   a harmless no-op, not an error.
5. **If the fix doesn't seem to take effect** after the files are genuinely overwritten (confirm
   via your file manager's own "last modified" timestamp on the file you just replaced), ask your
   host whether PHP's opcache is running with `opcache.validate_timestamps=0`. Some locked-down
   hosting setups disable automatic file-change detection entirely, so PHP keeps running the old
   code from memory until PHP-FPM is restarted or the opcache is manually cleared — even though
   the new file is genuinely sitting on disk.

**Worked example (the real 1.0.0 → 1.0.1 update):** this release fixed the OneRoster token
endpoint to also accept `Authorization: Basic` credentials, not just body parameters (some
OneRoster consumers send credentials this way regardless of their own settings) — a symptom that
looks like "my consumer's connection test fails with an auth error no matter what I try." The fix
touches exactly one file, `src/Security/TokenIssuer.php`, with no database change at all — replace
that one file (or the whole folder from the 1.0.1 zip) per the steps above, and it takes effect on
the very next request.

**Worked example, a release that DOES need a database change (1.0.2 → 1.0.3):** this release added
two new admin toggles (Users → Status Inclusion — whether people with a Gibbon status of `Left`
or `Pending Approval` appear in the `users` feed). New toggles need somewhere to store their
value, so this release adds two rows to `gibbonSetting` that a fresh install's own "Install" button
would create automatically, but an in-place file swap never touches. After overwriting the files
as usual, also run this once against your own database (phpMyAdmin, Adminer, or any MySQL client
your host provides) — skip it and the two new toggles will show as blank/broken rather than
defaulting cleanly to "Off":

```sql
INSERT INTO `gibbonSetting` (`gibbonSettingID`, `scope`, `name`, `nameDisplay`, `description`, `value`)
VALUES (NULL, 'DataAPI', 'includeLeftStatusUsers', "Include 'Left' status users", 'If Yes, people whose Gibbon status is Left are included in the users feed, alongside Full and Expected. Defaults to No.', 'N');

INSERT INTO `gibbonSetting` (`gibbonSettingID`, `scope`, `name`, `nameDisplay`, `description`, `value`)
VALUES (NULL, 'DataAPI', 'includePendingApprovalStatusUsers', "Include 'Pending Approval' status users", 'If Yes, people whose Gibbon status is Pending Approval are included in the users feed, alongside Full and Expected. Defaults to No.', 'N');
```

Both default to `'N'` — running this doesn't change your feed's behavior at all, it just makes the
two new toggles on the Users page actually work. This is also the general pattern for any future
release that mentions a database change: the exact `INSERT`/`UPDATE` statements needed will be
given the same way, to run once, manually, after the file swap.

**Worked example (1.0.3 → 1.0.4):** this release does two things. First, it adds a real
`CHANGEDB.php` to the module (see Step 4 above) — the same `INSERT IGNORE` statements from the
1.0.3 example above now live there permanently, so clicking **Upgrade** applies them for you on
any instance that hasn't already run them by hand, and safely no-ops on one that has (like ISU's).
Second, it adds a character-encoding safety check to every OneSync/User Provisioning write — see
"A note on character encoding for User Provisioning writes" below for what that means and why. No
SQL to run by hand for this release either way: whether you run it via **Upgrade** or skip that
step entirely, the actual fix (both the `CHANGEDB.php` mechanism and the encoding check) is active
as soon as the files are overwritten.

**Worked example (1.0.4 → 1.0.5):** if you applied 1.0.4 and clicked **Upgrade**, you may have
noticed Manage Modules correctly showed `v1.0.4` while this module's own **Status** page still
showed an old, stuck version number — that was a real bug in this module, not something you did
wrong. Two separate places store this module's version, and until 1.0.5, nothing kept them in
sync: `gibbonModule.version` (what Manage Modules reads, and what Upgrade actually updates) and a
separate row this module used to keep for its own Status page, which Gibbon's native Upgrade
mechanism has no knowledge of and never touches. Fixed by having the Status page read the same
value Manage Modules already does, instead of maintaining a second copy — no SQL to run for this
one either; overwriting the files is enough, and the Status page will immediately show whatever
Manage Modules already says.

---

## A note on character encoding for User Provisioning writes

This module's own database tables, and every database connection it opens, use `utf8mb4` — the
real, complete UTF-8 implementation, capable of storing any Unicode character, including 4-byte
ones like most emoji. **Core Gibbon's own tables are a different story.** As of this module's own
testing (checked directly via `information_schema`, on every real instance seen so far, including
this one's Railway sandbox and a real school's live server), Gibbon core's `gibbonPerson` table —
specifically the columns User Provisioning writes to (`firstName`, `surname`, `preferredName`,
`email`, `username`) — uses `utf8mb3`, MySQL's older, historically misleadingly-named `"utf8"`
charset, which tops out at 3 bytes per character.

This is **not** a MySQL-vs-MariaDB difference, or anything to do with which SQL engine a school
runs — it's a MySQL-family-wide historical accident (the real, complete `utf8mb4` implementation
shipped in 2010 under a new name because the incomplete original `utf8` name was already in use
across millions of tables) that predates this module entirely and affects core Gibbon's own schema
the same way on MySQL or MariaDB alike, on any version checked so far.

**Practical effect:** if a OneSync/User Provisioning write contains a genuine 4-byte character —
most emoji, or a handful of rare CJK/historic characters — in `givenName`, `familyName`, `email`,
or `username`, this module rejects it with a clear `422 unsupported_character` error rather than
letting it silently corrupt or fail unpredictably against a core column that can't store it.
Ordinary accented text in any language (é, ü, ñ, ç, and the vast majority of real names in any
script) is well within the 3-byte range and is never affected — this only catches the genuinely
rare 4-byte case. There is nothing to configure for this; it checks your instance's real, live
column charset automatically, every time, rather than assuming.

---

## Quick troubleshooting

| Symptom | Likely cause |
|---|---|
| Every page fatals immediately after upload | File permissions from the upload, or PHP version mismatch — confirm your host meets Gibbon's own core requirements first; this module doesn't add any beyond what Gibbon core already needs. |
| API always 404s, even on a resource you know exists | `AcceptPathInfo` isn't taking effect — check `AllowOverride` (Step 2a). |
| API always 401s, even with a freshly-issued token | `Authorization` header being stripped before PHP sees it — check `AllowOverride`/`CGIPassAuth` (Step 2a). |
| Every API call returns an HTML page instead of JSON — "Just a moment...", "checking your browser," a CAPTCHA, or a `403` with no JSON body at all | This isn't coming from the module — a CDN/WAF/bot-protection service (Cloudflare, etc.) in front of your domain is intercepting the request before it reaches Apache or PHP. See Step 2b: add a path exemption for `/modules/DataAPI/*` in that service's dashboard. |
| "Install" doesn't appear, or errors, in Manage Modules | Confirm the `DataAPI` folder itself (not a subfolder, not a zip) sits directly under `modules/`, and that its name and full path have no spaces (see the callout in Step 1). |
| A mapped Role/Grade doesn't show up in the API | Confirm the mapping was actually saved/confirmed on the Role Mapping / Grade Mapping page — an unconfirmed suggestion doesn't count (see Step 5). |
| Your OneRoster consumer's own "test connection" fails with a generic auth error, even with the right client ID/secret entered | As of v1.0.1, the token endpoint accepts credentials both ways — as `client_id`/`client_secret` body parameters, or as an RFC 6749 `Authorization: Basic base64(client_id:client_secret)` header. Some consumers (ClassLink's own generic Roster Server connector, confirmed) always send Basic Auth regardless of any "include credentials in token body" toggle they expose — that toggle may not do what its label suggests. If you're still stuck, check your consumer's own connection/sync log (not just this module's Access Log) for the literal request it made. |
| A OneSync/User Provisioning write fails with `422 unsupported_character` | As of v1.0.4, this is expected behavior, not a bug — see "A note on character encoding for User Provisioning writes" above. The value being written contains a genuine 4-byte Unicode character (most likely an emoji) that a core Gibbon column on this instance can't store. Remove that character from the source value, or check with whoever operates this Gibbon instance whether they can migrate the affected core column(s) to `utf8mb4` themselves — that's outside this module's own tables and not something it can change for you. |
