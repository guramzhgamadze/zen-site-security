# Zen Site Security

**Activate SSL in one click, then harden what sits behind it — HTTPS migration, certificate monitoring, security headers, a strict CSP, and attack-surface reduction.**

A WordPress plugin that migrates your site to HTTPS safely and keeps it there. Getting a
certificate is the easy part; moving WordPress onto it without breaking the site, noticing before
it expires, and hardening the platform behind it is the work this plugin does.

- **Website:** https://guramzhgamadze.github.io/zen-site-security/
- **WordPress.org:** https://wordpress.org/plugins/zen-site-security/
- **Requires:** WordPress 6.5+ · PHP 8.0+
- **Licence:** GPL-2.0-or-later
- **Current version:** 1.14.0

---

## One-click activation

The plugin verifies that a valid SSL certificate is actually installed for your domain before it
does anything — activation is blocked until one is found, so you can never lock yourself out by
accident. Activation then:

- switches your WordPress Address and Site Address to https,
- 301-redirects every HTTP request (pages and REST API) to HTTPS,
- fixes mixed content on your pages on the fly.

The certificate is probed directly and parsed, so the dashboard shows the real issuer, expiry date,
and whether it covers your domain — wildcards included.

### HTTP to HTTPS redirect, your way

- **PHP 301 redirect** (default) — works on every server and disappears automatically when the
  plugin is deactivated.
- **.htaccess 301 redirect** (advanced; Apache/LiteSpeed) — redirects at server level before
  WordPress loads, with the PHP redirect kept as a safety net. The rules are placed above the
  WordPress block, wrapped in clear markers, and removed on deactivation.
- On nginx the plugin shows you the exact server snippet to copy instead.

The .htaccess rules skip requests that already arrive with `X-Forwarded-Proto: https`, so sites
behind a proxy or load balancer do not hit a redirect loop.

### Mixed content fixer

Insecure `http://` references to your own site — including www/non-www variants and JSON-escaped
URLs — plus common `src`, `href`, `action`, `og:image`, `url()` and `srcset` patterns are rewritten
to `https://` just before the page is sent to the browser. Feeds, sitemaps and JSON responses are
left untouched. An optional fixer for the WordPress admin is available too.

### Certificate monitoring

A daily background check (WP-Cron) re-probes the certificate independently of admin visits and
emails the site administrator once when fewer than 15 days remain, and again if it expires — so an
expiring certificate is caught even when nobody logs in that week.

**Certificate quality (TLS) grading** gives a letter grade with specific findings for the
negotiated protocol, key strength, signature algorithm, cipher and certificate chain, shown on the
settings page and the Dashboard widget, with a link to the SSL Labs deep test for a full
protocol/cipher audit.

### Emergency recovery

If anything goes wrong, add one line to `wp-config.php`:

```php
define( 'ZENSS_DISABLE_SSL', true );
```

On the next visit the plugin reverts your site to http, disables the redirect, and removes its
.htaccess rules. No admin access required.

---

## Hardening (all opt-in)

Every header stands down automatically if another plugin already sends it, so nothing is ever
double-sent.

| Control | What it does |
|---|---|
| **Security headers** | `X-Content-Type-Options: nosniff`, `X-Frame-Options` (clickjacking), `Referrer-Policy` (keeps tokens out of cross-site referrers), a conservative `Permissions-Policy`, and CSP `upgrade-insecure-requests`. Optionally written into `.htaccess` (mod_headers) so static files and non-WordPress responses are covered too. |
| **Strict CSP** | An advanced Content-Security-Policy that stamps a per-request nonce on WordPress-rendered scripts and ships `script-src 'nonce-…' 'strict-dynamic'`. Report-Only by default, with a violations panel and a one-click **Allow** button that adds each recognised source to the policy. A "ready to enforce?" nudge appears once report-only has been quiet for two weeks. |
| **HSTS** | Opt-in `Strict-Transport-Security`. Max-age starts at one day for safe testing; the preload-eligible configuration (1 year + includeSubDomains) requires an explicit second opt-in, because it is hard to undo. |
| **SameSite login cookies** | The WordPress auth cookies are re-issued with an explicit `SameSite=Lax` attribute, so CSRF protection no longer depends on browser defaults — a second layer next to WordPress nonces. |
| **Web cache deception protection** | Logged-in pages and authenticated REST responses are marked `Cache-Control: no-store, private`, and a dynamic response is never allowed to be cached under a static-looking URL — the origin-side defence recommended by OWASP and PortSwigger. Common page-cache plugins are signalled to bypass logged-in responses. |
| **Attack-surface reduction** | Disable the wp-admin file editors (`DISALLOW_FILE_EDIT`), block PHP execution in the uploads directory (an uploaded webshell becomes a dead file), deny web access to sensitive files (logs, database dumps, backup copies, wp-config variants), disable directory listings, disable XML-RPC and pingbacks, and hide the WordPress version and PHP `X-Powered-By` header. |
| **security.txt** | Optional RFC 9116 responsible-disclosure file served on the fly at `/.well-known/security.txt` with your security contact, policy URL and preferred languages. Nothing is written to disk, so it never clashes with Let's Encrypt / ACME challenges. |
| **Score & Site Health** | A Dashboard security-score widget (0–100) with the TLS grade, how many protections are active, your top recommended next steps, and a one-click "enable recommended protections" — plus the same posture as tests under Tools → Site Health. One shared engine drives both, so the score, the recommendations and the Site Health tests always agree. |

---

## Threat coverage (honest scope)

Mapped against the PortSwigger Web Security Academy topics. A platform plugin defends transport and
configuration; it cannot patch vulnerable code inside another plugin or theme — nothing can, and
any plugin claiming a regex "SQLi/XSS firewall" is selling something bypass-prone.

| Topic | Coverage here |
|---|---|
| **Web cache deception** | **Direct defence** — `Cache-Control: no-store, private` on authenticated and static-looking dynamic responses, REST auth responses, and cache-plugin bypass signals |
| **CSRF** | SameSite=Lax auth cookies plus optional CSP `frame-ancestors` (WordPress nonces remain the primary control) |
| **XSS** | Exploit-blunting headers: `nosniff`, `object-src 'none'`, `base-uri 'self'`, `frame-ancestors`, Referrer-Policy — and, opt-in, a real nonce-based `script-src` via Strict CSP |
| **XXE** | Out of scope — PHP 8 disables XML external entities by default; a plugin cannot harden another plugin's parser |
| **NoSQL injection** | Not applicable — WordPress core is MySQL |
| **SQL injection** | Attack-surface reduction only (no file editors, sensitive-file blocking); the fix belongs in the vulnerable code |
| **Web LLM attacks** | Out of scope — these are application-layer, belonging in the code that calls the model, not a transport plugin |

---

## Zen family boundary

Designed to pair with [Zen Login & Authentication](https://github.com/guramzhgamadze/zen-login-authentication).
Each plugin is standalone, but together they split the work with exactly one owner per control:

| | Zen Login & Authentication | Zen Site Security |
|---|---|---|
| **Scope** | Identity & authentication | Transport & platform |
| **Owns** | Login/registration forms, rate limiting, 2FA/TOTP, passkeys, breached-password checks, Turnstile, sessions & device alerts, user enumeration (`?author=`, REST `/users`), generic login errors, **XML-RPC**, auth-page headers | HTTPS migration & redirects, mixed content, certificate monitoring, HSTS, **site-wide security headers**, SameSite cookie attributes, file editors, PHP-in-uploads, sensitive files, directory listings, version disclosure |

When both are active, this plugin's XML-RPC switch stands down in favour of its sibling, and its
site-wide headers skip any header the sibling already queued.

---

## Privacy

Zen Site Security makes no external network requests. It does not phone home, has no licence
server, collects no telemetry, and loads no remote assets. The only outbound connection it ever
makes is a TLS handshake with your own domain, to read your own certificate.

---

## Installation

1. Install through **Plugins → Add New** (search for "Zen Site Security"), or from
   [WordPress.org](https://wordpress.org/plugins/zen-site-security/).
2. Activate it through the Plugins screen.
3. Go to **Zen Site Security**.
4. If a valid certificate is detected, click **Activate SSL & HTTPS redirect**.
5. Optionally enable HSTS and the hardening controls later, once everything runs smoothly.

---

## FAQ

**Does the plugin generate SSL certificates?**
No. Your hosting provider installs the certificate — most offer free Let's Encrypt certificates in
their control panel. This plugin detects it, migrates WordPress to https, and keeps the site
redirected and mixed-content free.

**What happens when I deactivate the plugin?**
The .htaccess rules are removed and the PHP redirect stops. Your site URLs stay on https on
purpose — deactivating a plugin should never push a working https site back to http. Use **Revert
site to HTTP** on the settings page first if you really want to go back.

**I activated SSL and now I cannot reach my site.**
Add `define( 'ZENSS_DISABLE_SSL', true );` to `wp-config.php` via FTP or your hosting file manager.
The plugin reverts everything on the next request. Remove the line once the certificate is fixed.

**Does it work behind a proxy or load balancer?**
Yes. The .htaccess rules skip requests that already arrive with `X-Forwarded-Proto: https`,
preventing redirect loops. If your proxy terminates SSL and WordPress does not detect it, your
proxy setup needs the standard `HTTP_X_FORWARDED_PROTO` handling in `wp-config.php`.

**Can this plugin stop SQL injection or XSS in my other plugins?**
No plugin can patch vulnerable code in another plugin — be wary of any that claims to. What this
one does is defence in depth: headers that make XSS harder to exploit, SameSite cookies that blunt
CSRF, and attack-surface reduction that turns many exploit paths into dead ends. Keeping
WordPress, plugins and themes updated remains essential.

**Is multisite supported?**
Version 1.x targets single-site installs. Multisite support is planned.

---

## Changelog

### 1.14.0
New: **Allow location on this site** under Browser APIs. The `Permissions-Policy` header denies
geolocation with an empty allowlist, and per the specification an empty list disables a feature "in
top-level and nested browsing contexts" — your own pages included, not just third-party frames. A
site with a store locator or a "find nearest" control found it dead: the browser refused with no
prompt at all, while its own site settings still showed location as allowed. Switching this on sends
`geolocation=(self)`, which keeps every cross-origin frame and injected script blocked and opens the
feature to your pages alone. Off by default; with it off the header is byte-for-byte unchanged, and
camera, microphone and payment stay denied either way.

### 1.13.1
Fix: enabling Strict CSP in Report-Only mode no longer removes the enforcing
`Content-Security-Policy` from your front end, and no longer drops CSP from wp-admin. The enforcing
baseline policy stays in place until Strict CSP is actually switched to Enforce.

### 1.13.0
Renamed from "Zen HTTPS & SSL" to **Zen Site Security** (new slug: `zen-site-security`). The plugin
covers far more than SSL — security headers, hardening, Content-Security-Policy, certificate
monitoring and security.txt — so the broader name reflects what it actually does.

### 1.12.0 – 1.12.1
Optional **security.txt** (RFC 9116) served on the fly at `/.well-known/security.txt`, with
configurable Contact, Policy URL and Preferred-Languages. No file is written to disk. 1.12.1
corrects the version shown on the settings page.

### 1.11.0
The Dashboard security widget lists the certificate quality findings (protocol, key, signature)
beneath the TLS grade, matching the settings page.

### 1.10.0
**CSP report → allowlist workflow.** The strict CSP now also governs styles, images, fonts,
connections, frames and media, so the violations panel shows exactly what would be blocked — and
each recognised source gets a one-click **Allow** button. A "ready to enforce?" nudge appears once
report-only has been quiet for two weeks. The Dashboard widget gains the TLS grade.

### 1.9.0
**Certificate quality (TLS) grading** — a letter grade with specific findings for the negotiated
protocol, key strength, signature algorithm, cipher and certificate chain, plus a link to the SSL
Labs deep test.

### 1.8.0
**Dashboard security-score widget** — a score out of 100, how many protections are active, your top
recommended next steps, and a one-click "Enable recommended protections". **Site Health
integration** surfaces the same protections as tests under Tools → Site Health, driven by one
shared posture engine.

### 1.7.0
**Certificate expiry email alerts** — a daily background check re-probes the certificate
independently of admin visits and emails the administrator when fewer than 15 days remain.

### 1.6.0
**Strict CSP (script nonces)** — an advanced, opt-in policy that stamps a per-request nonce on
WordPress-rendered scripts and ships `strict-dynamic`. Report-Only by default with a violations
panel. Fix: the .htaccess CSP no longer emits `upgrade-insecure-requests` before SSL is active.

### 1.5.0
**Server-level headers** — write the security headers into .htaccess (mod_headers) so static files
and non-WordPress responses are covered, not just pages WordPress renders. Uses `Header always
set`, so PHP-rendered pages are never double-headed, and is removed cleanly on deactivation.

### 1.4.0
Translation template added; the settings page moved to its own top-level menu and adopted the Zen
family admin theme (card layout, toggle switches, clearer grouping).

### 1.3.0
**Web cache deception protection** and opt-in CSP anti-XSS directives (`object-src 'none'`,
`base-uri 'self'`, and `frame-ancestors` mirroring your clickjacking setting), assembled into a
single header.

### 1.2.0
**Sibling awareness** — when Zen Login & Authentication is active, the XML-RPC switch defers to it
and the settings page shows the division of responsibilities. Deny web access to sensitive files;
disable directory listings; remove PHP's `X-Powered-By`. Fix: managed .htaccess markers are
line-anchored so the redirect and hardening blocks can never swallow each other.

### 1.1.0
The **security hardening module** (all opt-in): security headers with automatic stand-down,
`SameSite=Lax` auth cookies, disabled file editors, blocked PHP execution in uploads, XML-RPC and
pingback disabling, and version hiding. Fix: SSL activation could not persist, because the
registered-settings sanitizer reverted the flag on every click.

### 1.0.0
Initial release: one-click SSL activation with certificate verification, PHP and .htaccess
redirects, the mixed content fixer, the certificate dashboard, opt-in HSTS, and the
`ZENSS_DISABLE_SSL` emergency recovery constant.

---

## About this repository

This repository hosts the **documentation and project page** for Zen Site Security. The plugin
source is developed privately; the released, installable plugin is distributed through the
[WordPress plugin directory](https://wordpress.org/plugins/zen-site-security/) under the
GPL-2.0-or-later licence, which includes its complete source.

Other plugins in the Zen family:

- [Zen Login & Authentication](https://github.com/guramzhgamadze/zen-login-authentication) — identity and two-factor auth
- [Zen MCP Bridge](https://github.com/guramzhgamadze/zen-mcp-bridge) — read-only MCP server for WordPress
- [Zen GEO](https://github.com/guramzhgamadze/zen-geo) — generative engine optimization
- [Zen Blogger](https://github.com/guramzhgamadze/zen-blogger) — accessible Elementor blog widgets

---

Built by [Guram Zhgamadze](https://github.com/guramzhgamadze).
