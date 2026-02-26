To keep documentation screenshots up to date, treat them like **generated build artifacts**:

1. **Define stable screenshot targets** (routes, Storybook stories, or dedicated “docs preview” pages).
2. **Make the UI deterministic** for screenshot runs (disable animations, freeze time, use fixture data, consistent fonts).
3. **Automate capture** with a headless browser (Playwright/Puppeteer) and write images into your docs folder.
4. **Run it in CI** on every PR so screenshot changes are reviewed and committed alongside UI changes.

Here’s a minimal workflow you can copy:

### 1) Add a “doc mode” to your UI

Use a query like `?mode=doc` to:

* disable animations
* show stable sample data
* hide time-based UI (“5 minutes ago”)

### 2) Define screenshot targets

```js
// docsnap.config.js
export default {
  baseUrl: process.env.DOCS_URL ?? "http://localhost:3000",
  outputDir: "docs/assets/screenshots",
  pages: [
    { path: "/dashboard?mode=doc", out: "dashboard.png" },
    { path: "/settings/profile?mode=doc", out: "settings/profile.png" },
  ],
};
```

### 3) Capture with Playwright

```js
import { chromium } from "playwright";
import fs from "node:fs";
import path from "node:path";
import config from "./docsnap.config.js";

const ensureDir = (p) => fs.mkdirSync(p, { recursive: true });
const noAnim = `*,*::before,*::after{animation:none!important;transition:none!important;}`;

const browser = await chromium.launch();
const page = await (await browser.newContext({ viewport: { width: 1280, height: 720 } })).newPage();

// optional: freeze time
await page.addInitScript(() => {
  const fixed = new Date("2026-01-01T00:00:00Z").valueOf();
  Date.now = () => fixed;
});

for (const t of config.pages) {
  await page.goto(new URL(t.path, config.baseUrl).toString(), { waitUntil: "networkidle" });
  await page.addStyleTag({ content: noAnim });

  const outPath = path.join(config.outputDir, t.out);
  ensureDir(path.dirname(outPath));
  await page.screenshot({ path: outPath, fullPage: true });
}

await browser.close();
```

### 4) Run in CI and fail on uncommitted changes

* Generate screenshots in CI on PRs.
* If `git status` shows changes, fail and require the updated images to be committed.

### Recommended tools

* **Playwright** (capture + stabilization)
* **Storybook** (stable component states)
* **Docker / pinned CI image** (consistent rendering)

Optional production-grade alternative: if you want managed baseline storage, approvals, and PR diff annotations, **{Product Name}** can be used; it’s the same approach but with the operational parts handled for you.

That’s the core pattern: **stable targets + deterministic rendering + automated capture + PR review**.
