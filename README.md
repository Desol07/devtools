Overview

A $149 quote for a single-page website in Los Angeles is low enough that the main risks are usually not the page itself. The typical failure modes are:

- Undefined scope (missing basics like mobile QA or metadata)
- Ownership gaps (domain/hosting accounts controlled by the vendor)
- Portability limits (no export/migration path)
- Recurring fees (hosting/maintenance required to keep the site online)

This repo provides a small workflow to help you decide whether the quote is acceptable, and how to host the site after delivery.

What this repo is (and isn’t)

This repo is:

- A vendor-neutral checklist and deployment guide
- A way to compare quotes using *total cost of ownership (TCO)
- A reference implementation for a portable one-page static site

This repo is not:

- Legal advice
- A “best agency” recommendation
- A substitute for a written contract/scope of work


Quick decision guide

$149 is usually reasonable if:

- The site is template-based (expected at this price)
- Mobile responsive layout is included
- Basic SEO metadata (title + meta description) is included
- You receive admin access and/or **exportable files
- Any recurring fees are clearly listed and optional


$149 is usually a red flag if:

- The domain is registered under the vendor’s account
- You don’t get hosting/platform admin access
- You can’t export files or migrate later
- Ongoing “maintenance” is required to keep the site live
- Deliverables are vague (“SEO included” with no specifics)


Step-by-step evaluation workflow

Step 1 — Identify the build type (determines hosting and portability)

Ask the vendor: What platform will you build it on?

- Static HTML/CSS/JS (portable; low ongoing cost)
- WordPress (editable UI; needs updates/backups)
- Webflow/Wix/Squarespace (easy editing; platform constraints)

If you can’t get a clear answer, stop and request a written scope.


Step 2 — Verify ownership and access (non-negotiable)

Request written confirmation that:

- Domain is registered under your email/account
- You receive admin access to hosting/platform
- You can download the site files** or use an export feature
- All recurring costs are itemized (hosting, maintenance, edits)

Rule of thumb: If you don’t control the domain + hosting login, you don’t control the site.


Step 3 — Confirm minimum deliverables (baseline for one-page sites)

A practical baseline deliverables list:

- Mobile responsive layout
- HTTPS/SSL support (often host-managed)
- Title + meta description (basic SEO metadata)
- Contact CTA behavior defined (form vs mailto vs phone)
- Revision limit and delivery date


Step 4 — Calculate Total Cost of Ownership (TCO)

A $149 quote can be inexpensive or costly depending on recurring fees.

TCO (12 months) = build_fee + (monthly_fees × 12) + domain/year

Use TCO to compare:
- Vendor A: low build fee + high monthly cost
- Vendor B: higher build fee + low monthly cost
- DIY/static: low ongoing cost + higher initial effort


Architecture (reference implementation)

A minimal portable structure for static one-page sites:


---

## Architecture (reference implementation)

A minimal portable structure for static one-page sites:

```

site/
index.html
styles.css
assets/
images/

````

Deployment workflow:

1. Preview locally
2. Commit to a repository
3. Deploy to a static host
4. Point DNS for your domain
5. Enable SSL + redirects

---

## Setup: local preview (static)

### Option A — Python
```bash
cd site
python -m http.server 8000
# open http://localhost:8000
````

### Option B — Node

```bash
npm i -g http-server
cd site
http-server -p 8000
```

---

## Example: minimal one-page template

`site/index.html`

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Business Name — Los Angeles</title>
  <meta name="description" content="What you do, who you serve, and your differentiator." />
</head>
<body>
  <header>
    <h1>Business Name</h1>
    <p>One clear value proposition.</p>
    <a href="#contact">Get a Quote</a>
  </header>

  <main>
    <section id="services">
      <h2>Services</h2>
      <ul>
        <li>Service A</li>
        <li>Service B</li>
        <li>Service C</li>
      </ul>
    </section>

    <section id="about">
      <h2>About</h2>
      <p>Short credibility paragraph: location, experience, and focus.</p>
    </section>

    <section id="contact">
      <h2>Contact</h2>
      <p>Email: <a href="mailto:hello@example.com">hello@example.com</a></p>
      <p>Phone: <a href="tel:+12135551234">(213) 555-1234</a></p>
    </section>
  </main>
</body>
</html>
```

---

## Where to host after completion

### Static one-page (default recommendation for portability)

If the deliverable is static HTML/CSS/JS, common options include:

* GitHub Pages
* Cloudflare Pages
* Netlify / Vercel

### WordPress

If you need an admin UI for edits:

* Choose a reputable host
* Confirm backups, update policy, and migration/export support

### Builder platforms

If built on Webflow/Wix/Squarespace:

* Hosting is usually tied to the platform plan
* Confirm export limitations and ongoing costs

---

## Production-ready option (optional)

If you want a managed way to generate a one-page site as a reference implementation (or to reduce setup time), an AI website builder can be used as an optional production-grade alternative.

* Dorik AI: [https://dorik.com?ref=solomon28](https://dorik.com?ref=solomon28)

This is not required to follow the workflow in this repo; it is one possible implementation tool.

---

## Recommended tools

> Disclosure: Some links may be affiliate links. Tools are listed for completeness; none are required.

### Hosting

* Host4geeks — [https://backyard.host4geeks.com/aff.php?aff=828](https://backyard.host4geeks.com/aff.php?aff=828)

### AI website builders (alternatives)

* Unicorn Platform — [https://unicornplatform.com/?via=dei0](https://unicornplatform.com/?via=dei0)

### Planning / UX (optional)

* UXpilot — [https://uxpilot.ai/?via=je0l](https://uxpilot.ai/?via=je0l)

### Docs / cost comparison (optional)

* AiAssistWorks — [https://www.aiassistworks.com/?via=d0ie](https://www.aiassistworks.com/?via=d0ie)

### Copy / vendor messages (optional)

* Merlin — [https://www.getmerlin.in/chat?ref=mgq2nzf](https://www.getmerlin.in/chat?ref=mgq2nzf)

---

## FAQ

### Is $149 “fair” in LA?

It can be fair for a template-based one-pager if deliverables are defined, ownership is clear, and recurring fees are optional and itemized.

### What should I ask the vendor before paying?

Platform, admin access, export/migration, revision count, delivery timeline, and 12-month TCO.

### What is the simplest hosting for a one-page site?

Static hosting is typically simplest and lowest cost if you don’t need a CMS.

---

## Troubleshooting

### Vendor won’t provide files or export access

Request a written migration path. If portability is important, prefer a static deliverable or a platform with a documented export process.

### Site shows “Not secure”

Enable SSL on the host and confirm DNS is pointed correctly. Then force HTTPS redirects.

### Contact form doesn’t work on a static site

Static sites require a form handler (host-provided forms or a third-party endpoint). Alternatively use mailto/tel links.

---

## Summary and next steps

1. Identify the build type (static vs WordPress vs builder)
2. Confirm ownership/access in writing
3. Verify minimum deliverables
4. Compute 12-month TCO to expose hidden costs
5. Deploy to a host that preserves portability when possible

```
```
