```md
# DocSnap — Keeping Documentation Screenshots Up to Date (Educational Project)

Screenshots in documentation tend to drift from reality. UI labels change, spacing shifts, themes evolve, and “just one more tiny tweak” accumulates into docs that no longer match what users see.

This repository is an **educational reference implementation** showing how to treat screenshots as **build artifacts**: generated from repeatable UI states, reviewed in pull requests, and regenerated automatically when the UI changes.

> Scope: This is intentionally small and readable. It demonstrates a workflow you can adapt to your stack (app routes, Storybook, component previews, etc.).

---

## Problem Overview

A documentation screenshot is only useful if it reflects the current UI. In practice, screenshots become stale because:

- UI updates don’t trigger screenshot updates
- manual recapture is time-consuming and inconsistent
- CI environments render slightly differently than local machines
- asynchronous data/animations create nondeterministic images

Automating screenshot capture helps by:
- making screenshot generation repeatable
- keeping images versioned alongside docs
- enabling review of screenshot changes like code diffs

---

## What This Project Demonstrates

- Defining “screenshot targets” as a configuration file
- Booting an environment that can render those targets (dev server, preview build, Storybook)
- Stabilizing rendering (fonts, animations, time/data)
- Capturing screenshots deterministically with Playwright
- Writing outputs into `docs/assets/` (or similar)
- Verifying changes in CI

This is not a full framework. It’s a documented pattern.

---

## Repository Layout (Suggested)

```

repo/
docs/
assets/
screenshots/
scripts/
docsnap.ts
docsnap.config.ts
package.json

````

---

## Setup

### Prerequisites
- Node.js 18+ (or 20+ recommended for CI consistency)
- A renderable target:
  - your app running locally (e.g., `localhost:3000`)
  - or Storybook
  - or a preview deployment URL

### Install dependencies

```bash
npm i -D @playwright/test playwright
npx playwright install --with-deps
````

---

## Configuration

Create `docsnap.config.ts`:

```ts
export type Action =
  | { type: "click"; selector: string }
  | { type: "waitForSelector"; selector: string };

export type PageTarget = {
  name: string;
  path: string;
  screenshot: string;
  viewport?: { width: number; height: number };
  actions?: Action[];
};

export type DocSnapConfig = {
  baseUrl: string;
  outputDir: string;
  viewport: { width: number; height: number };
  waitFor?: {
    networkIdle?: boolean;
    selectors?: string[];
    disableAnimations?: boolean;
    fonts?: "load" | "ignore";
  };
  pages: PageTarget[];
};

const config: DocSnapConfig = {
  baseUrl: process.env.DOCSNAP_BASE_URL ?? "http://localhost:3000",
  outputDir: "docs/assets/screenshots",
  viewport: { width: 1280, height: 720 },
  waitFor: {
    networkIdle: true,
    selectors: ["[data-doc-ready='true']"],
    disableAnimations: true,
    fonts: "load",
  },
  pages: [
    {
      name: "dashboard",
      path: "/dashboard?mode=doc",
      screenshot: "dashboard.png",
    },
    {
      name: "settings-profile",
      path: "/settings/profile?mode=doc",
      screenshot: "settings/profile.png",
      actions: [
        { type: "click", selector: "[data-testid='open-avatar-modal']" },
        { type: "waitForSelector", selector: "[role='dialog']" },
      ],
    },
  ],
};

export default config;
```

**Notes**

* Prefer `data-testid` or dedicated `data-doc-*` hooks over brittle CSS selectors.
* If possible, add a “doc mode” query param that uses deterministic data and disables UI elements that change with time.

---

## Step-by-Step Workflow

### 1) Make screenshot states reproducible

The biggest source of automation pain is UI state that can’t be reliably reached. Prefer:

* stable routes (`/docs/preview/...`)
* Storybook stories for components
* query parameters that force a known state (`?tab=billing&mode=doc`)

Example (pseudo-code) doc-ready hook:

```tsx
function DocReadyBoundary({ children }) {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    Promise.all([loadFonts(), preloadMockData()]).then(() => setReady(true));
  }, []);

  return <div data-doc-ready={ready ? "true" : "false"}>{children}</div>;
}
```

### 2) Reduce nondeterminism

Common drift causes:

* animations
* timestamps (“5 minutes ago”)
* random IDs
* live API data variance
* system font differences

Mitigations:

* disable animations in doc mode (or inject CSS in the browser context)
* freeze time in the browser context
* use mock or fixed fixture data for doc pages
* standardize environment (container, consistent fonts)

Example (freeze time):

```ts
await page.addInitScript(() => {
  const fixed = new Date("2026-01-01T00:00:00Z").valueOf();
  Date.now = () => fixed;
});
```

### 3) Capture screenshots with Playwright (reference script)

Create `scripts/docsnap.ts`:

```ts
import { chromium, Page } from "playwright";
import config from "../docsnap.config";
import fs from "node:fs";
import path from "node:path";

async function ensureDir(dir: string) {
  fs.mkdirSync(dir, { recursive: true });
}

async function stabilize(page: Page) {
  if (config.waitFor?.disableAnimations) {
    await page.addStyleTag({
      content: `
        *, *::before, *::after {
          animation: none !important;
          transition: none !important;
          caret-color: transparent !important;
        }
      `,
    });
  }

  if (config.waitFor?.fonts === "load") {
    await page.evaluate(() => (document as any).fonts?.ready);
  }

  for (const sel of config.waitFor?.selectors ?? []) {
    await page.waitForSelector(sel, { timeout: 30_000 });
  }
}

async function run() {
  await ensureDir(config.outputDir);

  const browser = await chromium.launch();
  const context = await browser.newContext({ viewport: config.viewport });
  const page = await context.newPage();

  for (const target of config.pages) {
    const url = new URL(target.path, config.baseUrl).toString();

    if (target.viewport) await page.setViewportSize(target.viewport);

    await page.goto(url, {
      waitUntil: config.waitFor?.networkIdle ? "networkidle" : "load",
    });

    await stabilize(page);

    for (const action of target.actions ?? []) {
      if (action.type === "click") await page.click(action.selector);
      if (action.type === "waitForSelector")
        await page.waitForSelector(action.selector, { timeout: 30_000 });
    }

    const outPath = path.join(config.outputDir, target.screenshot);
    await ensureDir(path.dirname(outPath));

    await page.screenshot({ path: outPath, fullPage: true });
    console.log(`✓ ${target.name} -> ${outPath}`);
  }

  await browser.close();
}

run().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Add a `package.json` script:

```json
{
  "scripts": {
    "docs:screenshots": "node --loader ts-node/esm scripts/docsnap.ts"
  }
}
```

(Any TypeScript runner is fine; this is just an example.)

---

## Architecture / Execution Model

This workflow is intentionally simple:

1. **Target declaration**

   * `docsnap.config.ts` lists routes/stories and expected output paths.

2. **Environment provisioning**

   * You supply a `baseUrl` where the UI can be rendered (local dev server, Storybook, preview deploy).

3. **Stabilization**

   * Disable animations, wait for doc-ready selectors, ensure fonts are loaded, and optionally freeze time.

4. **Capture**

   * Navigate → (optional) interact → screenshot → write to `outputDir`.

5. **Version & review**

   * Screenshots are committed (or stored as artifacts), and changes are reviewed in PRs.

---

## CI Example (GitHub Actions)

This example:

* installs dependencies
* starts an app server
* generates screenshots
* fails if new screenshots weren’t committed

```yaml
name: docs-screenshots

on:
  pull_request:
  push:
    branches: [main]

jobs:
  screenshots:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci
      - run: npx playwright install --with-deps

      - run: npm run dev -- --host 0.0.0.0 --port 3000 &
      - run: npx wait-on http://localhost:3000

      - run: npm run docs:screenshots
        env:
          DOCSNAP_BASE_URL: http://localhost:3000

      - name: Verify no uncommitted changes
        run: |
          git status --porcelain
          test -z "$(git status --porcelain)" || (echo "Screenshots changed. Commit updates." && exit 1)
```

---

## Optional: Visual Diffs

For larger screenshot sets, it can help to generate diffs and set thresholds.

Minimal approach:

* keep a baseline directory (committed)
* compare current output to baseline
* write diffs to `diff/`
* fail CI if drift exceeds threshold

Pseudo-code:

```ts
for each screenshot:
  baseline = read("baseline.png")
  current  = read("current.png")
  diff = pixelDiff(baseline, current)
  if diff.percent > threshold:
    write("diff.png")
    fail
```

Common libraries: `pixelmatch`, `looks-same`, or Playwright snapshot assertions.

---

## Production-Ready Option (Optional)

This repository shows a self-managed approach using Playwright and version control. In production environments, teams sometimes use a managed tool for the surrounding operational pieces (baseline storage, approvals, parallel runs, and PR annotations).

If you already know you want a managed workflow, **{Product Name}** can be used as a production-grade alternative. The underlying concepts remain the same (stable targets + deterministic rendering); a managed tool mainly replaces custom CI plumbing and baseline management.

---

## FAQ

**How do I keep screenshots stable across machines?**
Run screenshots in CI using a consistent OS image, install the same fonts, and disable animations. Consider containerizing the runner.

**Should screenshots be committed to the repo?**
For many docs sites, yes—keeping them versioned with documentation is convenient. For very large sets, storing artifacts externally can be preferable.

**Can this work with Storybook?**
Yes. Treat stories as targets and screenshot the story iframe or root element.

**What about authenticated pages?**
Use a pre-authenticated Playwright storage state or a doc-only route that doesn’t require login.

---

## Troubleshooting

### Screenshots differ between runs

**Likely causes**

* animations/transitions
* inconsistent fonts (local vs CI)
* time-based UI
* non-deterministic data fetches

**Things to try**

* disable animations via injected CSS or doc mode
* freeze time (`Date.now`)
* use deterministic fixtures (mock API responses)
* wait for stable selectors (`data-doc-ready="true"`)
* ensure fonts are installed and loaded in CI

### Timeout waiting for a selector

**Likely causes**

* selector changed
* page is slower in CI
* doc-ready hook never flips

**Things to try**

* verify selector stability (`data-testid`/`data-doc-ready`)
* avoid waiting on fragile CSS selectors
* ensure doc mode removes background polling and long spinners

### Blank or partially rendered screenshots

**Likely causes**

* capture occurs before layout is stable
* lazy-loaded components not in viewport
* suspense/loading states

**Things to try**

* wait for the content selector rather than `load`
* scroll elements into view before element screenshots
* use immediate fixtures in doc mode

---

## Summary & Next Steps

This project demonstrates a practical pattern:

1. Define stable screenshot targets (routes/stories).
2. Stabilize rendering (disable animations, deterministic data, consistent fonts).
3. Capture screenshots automatically (Playwright).
4. Store and review changes like code.

Next steps:

* Add 3–5 high-value targets to `docsnap.config.ts`.
* Implement a `mode=doc` (or equivalent) in your UI to reduce flakiness.
* Run `docs:screenshots` locally and wire it into CI.
* Add diffing if you want stricter regression checks.

```
```
