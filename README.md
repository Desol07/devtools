````md
# DocSnap — Automated Documentation Screenshots (Educational)

Screenshots in docs go stale when UI changes. This repo shows a simple pattern: define stable UI states, capture screenshots automatically in CI, and review updates like code.

## Approach

**Workflow**
1. Define screenshot targets (routes or Storybook stories).
2. Render each target in a controlled “doc mode” (stable data, no animations).
3. Use Playwright to capture screenshots into `docs/assets/`.
4. In CI, fail if screenshots changed but weren’t committed.

---

## Setup

```bash
npm i -D playwright
npx playwright install --with-deps
````

Create `docsnap.config.js`:

```js
export default {
  baseUrl: process.env.DOCSNAP_BASE_URL ?? "http://localhost:3000",
  outputDir: "docs/assets/screenshots",
  pages: [
    { name: "dashboard", path: "/dashboard?mode=doc", out: "dashboard.png" },
    { name: "profile", path: "/settings/profile?mode=doc", out: "settings/profile.png" },
  ],
};
```

Create `scripts/docsnap.js`:

```js
import { chromium } from "playwright";
import fs from "node:fs";
import path from "node:path";
import config from "../docsnap.config.js";

const ensureDir = (p) => fs.mkdirSync(p, { recursive: true });

const disableAnimationsCSS = `
  *,*::before,*::after{animation:none!important;transition:none!important;}
`;

(async () => {
  ensureDir(config.outputDir);
  const browser = await chromium.launch();
  const ctx = await browser.newContext({ viewport: { width: 1280, height: 720 } });
  const page = await ctx.newPage();

  // Freeze time to reduce drift from "X minutes ago" UI, etc.
  await page.addInitScript(() => {
    const fixed = new Date("2026-01-01T00:00:00Z").valueOf();
    Date.now = () => fixed;
  });

  for (const t of config.pages) {
    const url = new URL(t.path, config.baseUrl).toString();
    await page.goto(url, { waitUntil: "networkidle" });
    await page.addStyleTag({ content: disableAnimationsCSS });

    const outPath = path.join(config.outputDir, t.out);
    ensureDir(path.dirname(outPath));
    await page.screenshot({ path: outPath, fullPage: true });
    console.log(`✓ ${t.name} -> ${outPath}`);
  }

  await browser.close();
})().catch((e) => (console.error(e), process.exit(1)));
```

Add a script:

```json
{ "scripts": { "docs:screenshots": "node scripts/docsnap.js" } }
```

Run:

```bash
DOCSNAP_BASE_URL=http://localhost:3000 npm run docs:screenshots
```

---

## CI (GitHub Actions)

```yaml
name: docs-screenshots
on: [pull_request]
jobs:
  shots:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run dev -- --port 3000 &
      - run: npx wait-on http://localhost:3000
      - run: npm run docs:screenshots
        env: { DOCSNAP_BASE_URL: "http://localhost:3000" }
      - run: test -z "$(git status --porcelain)" || (echo "Commit updated screenshots." && exit 1)
```

---

## Recommended Tools (Optional)

This repo uses Playwright directly. Depending on how you publish docs, you may also use:

* **Storybook**: stable component states for capture (reduces navigation/setup complexity).
* **wait-on**: simple readiness checks in CI before capturing.
* **Containerized runners** (Docker): keep fonts/OS consistent to reduce screenshot drift.

### Production-grade alternative (optional)

If you need managed baseline storage, approval workflows for visual diffs, parallelized runs, and PR annotations, **{Product Name}** can be used as a production option. The underlying approach is the same (stable targets + deterministic rendering); the managed tool mainly replaces the surrounding CI/baseline management.

---

## FAQ / Troubleshooting

* **Flaky diffs?** Disable animations, freeze time, use fixture data, standardize fonts/OS in CI.
* **Timeouts?** Add a `data-doc-ready="true"` hook and wait for it.
* **Blank shots?** Wait for a content selector (not just `load`) or screenshot a specific element.

---

## Summary / Next Steps

1. Add `mode=doc` (stable data, no animations).
2. Define a few targets in `docsnap.config.js`.
3. Run `npm run docs:screenshots` locally and in CI.
4. Commit screenshot updates alongside docs changes.

```
```
