> **Source code is private.** This repo exists for portfolio documentation purposes only. There is no application code here — just a written case study of the project.

# Hardy Rifle — Digital Rifle Builder

A 3D web configurator that lets customers design a custom hunting rifle, see it rendered in real time, and submit the build straight to the manufacturer.

Built for **Hardy Rifle Engineering**, a hunting rifle manufacturer based in New Zealand.

---

## What it does

A customer lands on the site, picks the country they're shopping from, and confirms they're old enough to buy a firearm. They choose one of four rifle platforms — **The Hybrid**, **Project X**, **Project X Hunter**, or **Project X Carbon Hunter** — and drop into a full-screen 3D scene of that rifle.

From there, a step-by-step wizard walks them through every part of the build: paint finish (including a three-layer custom paint mixer), action type, hand, barrel, calibre, stock, mounting interfaces, muzzle brake, suppressor, and bipod. Every selection updates the 3D model immediately. Pricing updates on each click and converts between currencies based on the chosen region.

The 3D viewer isn't just a turntable — they can orbit and zoom, switch backgrounds, take a screenshot, toggle cinematic post-processing effects, and enter fullscreen.

When they're happy:

- They can **share** their build with a six-character code that loads it back exactly.
- They can **download a build sheet PDF** of the spec.
- They can **find an authorised dealer** on a map filtered to their region.
- They can **submit the build** to Hardy. Submission asks for contact details, fires a six-digit code to their email to verify the address, and then sends them a confirmation email plus a notification (with the build sheet attached) to the right Hardy mailbox depending on whether they're NZ-based or international.

There's also a private staff-only Kanban board for the Hardy team to track every submitted build through five stages (backlog → onboarding → building → awaiting collection → completed), with drag-and-drop, notes, due dates, and inline editing of the customer's submitted spec.

## Results / Impact

- **Live at:** [builder.hardyrifle.com](https://builder.hardyrifle.com)
- **Eliminated manual sales follow-up steps** — customers self-serve their own configuration, the verification + email pipeline replaces back-and-forth quoting, and the staff manage board removes the need for a separate spreadsheet/CRM workflow for tracking incoming builds.

## Tech Stack

**Framework & runtime**
- **Next.js 16 (App Router)** — full-stack framework. Pages, server-rendered layouts, and API routes all live in the same project.
- **React 19** — UI layer.
- **TypeScript** — used for the model upload script and config; most application code is JS/JSX with `allowJs` enabled.

**Styling & UI**
- **Tailwind CSS 4** — utility-first styling for almost every component.
- **Framer Motion** — page transitions, modal animations, wizard step transitions, and the configurator guide carousel.
- **GSAP** — used inside the 3D viewer for camera and scene tweens.
- **Lucide React** + **Grommet Icons** — icon libraries (Lucide is the primary one; Grommet shows up in a couple of places in the 3D viewer).
- **Flag Icons** — country flags on the region selector.

**3D rendering**
- **Three.js** — underlying WebGL renderer.
- **@react-three/fiber** — React renderer for Three.js.
- **@react-three/drei** — helpers (`OrbitControls`, `Environment`, `Html`, etc.).
- **@react-three/postprocessing** + **postprocessing** + **n8ao** — N8AO ambient occlusion, SMAA, SSAA, and Bloom effects in a configurable post-processing chain.
- Three's `GLTFLoader` is wrapped in a custom dedup/cache layer so the same GLB is never fetched twice.

**Database, storage, auth**
- **Supabase (PostgreSQL)** — primary data store. Tables include `customers`, `rifle_builds`, `email_verifications`, `shared_builds`, `build_submissions`, `order_submissions`. Row-level security policies are defined in SQL migrations under `supabase/migrations/`.
- **Supabase Storage** — three buckets: `rifle-glb-models` (public, holds all the GLB rifle models), `rifle_build_preview` (public, screenshot uploads of finished builds), and `build-sheets` (private, signed-off PDF copies of submitted builds).
- **@supabase/supabase-js** — used in two flavours: an anon-key client in the browser for shared-build links, and a service-role client server-side for everything that needs to bypass RLS.

**Email**
- **@getbrevo/brevo** — Brevo transactional email is the primary provider. Sends the verification code, the customer build confirmation, and the staff notification (with PDF attachment).
- **Resend** — used by an older order-confirmation route that still exists in the codebase. Brevo is the actively used path.
- The HTML email templates are hand-written strings; **react-email** is in `package.json` but isn't imported in the source I read.

**PDF generation**
- **jsPDF** + **html2canvas** — client-side PDF generation. The build sheet React component is mounted off-screen at a fixed 960px width, rasterised, and embedded as a single JPEG in the PDF.
- **puppeteer-core** + **@sparticuz/chromium** — server-side fallback for PDF generation, designed to work inside Vercel's serverless functions. The client path is preferred; the server path is the safety net.

**Maps**
- **Leaflet** + **react-leaflet** — dealer locator map. Tiles served from Stadia Maps' "Alidade Smooth Dark" theme. The map is loaded with `next/dynamic({ ssr: false })` to avoid `window is not defined` on the server.

**Drag-and-drop (staff board)**
- **@hello-pangea/dnd** — powers the five-column Kanban on `/manage`.

**Build-share & utilities**
- **lz-string** — compresses serialised build state into URL-safe strings for share links.

**Analytics**
- **react-ga4** — Google Analytics 4 page-view tracking, gated on a real measurement ID.
- **@vercel/analytics** — Vercel's built-in analytics, mounted in the root layout.

**Wix integration (asset CDN)**
- **@wix/sdk** + **@wix/stores** are listed as dependencies (Hardy's main commerce site is built on Wix), but at the source level the integration is read-only: a `WixProductImage` component plus a `lib/optimizeWixImage.js` helper that rewrites `static.wixstatic.com` URLs to apply CDN resize/compress/WebP transforms. Recommended add-on product images come from Wix's CDN.

**Deployment**
- **Vercel** — hinted at by the use of `@sparticuz/chromium`, the 30-second `maxDuration` on PDF/email routes, and `@vercel/analytics`.

## Architecture

The whole thing is a single Next.js application. The frontend, the API routes, and a small CLI script for uploading 3D models all live in the same repo, with path aliases (`@components`, `@lib`, `@data`, `@utils`, etc.) keeping imports tidy.

**Frontend.** The user-facing flow is a sequence of pages under `app/`: a landing/region-gate page (`app/page.jsx`), the 3D configurator (`app/rifles/[url]/App.jsx` — by far the largest file, ~100KB), a review page, a dealer locator, a summary/checkout page, and a build sheet preview page. Each of these is a client component because the 3D scene, the wizard state, and the map all need browser APIs. In-progress build state is held in `sessionStorage` rather than a database — there are no user accounts in the consumer flow, so a customer's choices live in their tab until they submit.

**3D scene.** `RifleMainViewer.jsx` is the heart of the configurator. It composes a rifle from a "skeleton" GLB (which encodes attachment points and varies by action length and hand) plus a "body" GLB (which varies by stock/riser and hand), then mounts barrels, muzzle brakes, suppressors, bipods, and accessories at those attachment points. Custom paint is applied as a three-layer shader/material override based on a palette in `data/customPaintPalette.js`. GLBs are streamed from Supabase Storage, and a small custom loader (`lib/rifle/gltfCachedLoad.js`) deduplicates concurrent loads of the same URL.

**Region awareness.** `data/utils/regionBuilderRules.js` is the single source of truth for what's allowed where: which upgrade keys are excluded per region (e.g., suppressors are filtered out for non-NZ markets), what default Leaflet view to show, which dealer region codes correspond to each builder region, and how to display weights (kg vs lbs for the US). The wizard's step list is generated by intersecting the rifle's allowed upgrades, the region's exclusions, and a few rifle-specific shortcuts (e.g., the PX Hunter has a single fixed barrel, so its barrel step is skipped).

**Submission flow.** `POST /api/submit-build` is a two-action endpoint:
1. `action: 'submit_build'` — looks up the customer by email. If unverified, generates a 6-digit code, stores it in `email_verifications` with a 15-minute expiry, and sends it via Brevo.
2. `action: 'verify_code'` — checks the code, upserts the customer record, enforces a per-customer daily build limit (3 per UTC day), generates an order reference (`RB-YYYY-XXXX`), inserts the build into `rifle_builds`, uploads the PDF to the private `build-sheets` bucket, and fires two emails: a customer confirmation and a region-routed staff notification (NZ → `info@hardyrifle.co.nz`, everything else → `int@hardyrifle.co.nz`).

**Build sharing.** A user can save and share a build at any point. The selections object is JSON-stringified, compressed with `lz-string`, and either embedded directly in a URL or persisted to Supabase's `shared_builds` table under a 6-character random code. The same anon-key Supabase client is used for both reading and writing shared builds, with permissive RLS policies on that table only.

**Staff manage board.** `/manage` is gated by an env flag (`NEXT_PUBLIC_ENABLE_STAFF_UI=true`) on both the page and the matching API routes. It loads every row in `rifle_builds`, joins the `customers` table, and renders five drag-and-drop columns. Reordering or stage changes hit `PATCH /api/manage/builds/[id]` with a `columnOrders` payload, which updates `stage` and `position` for every affected row in parallel. The same endpoint also handles inline edits to the build spec, customer info, dealer, and notes.

**Third-party services.** Brevo (email), Resend (legacy email), Stadia Maps (Leaflet tiles), Google Analytics, Vercel Analytics, and Wix (image CDN for accessory products). Everything else is self-hosted on Vercel + Supabase.

## Key Technical Decisions

**1. Client-first PDF generation, server fallback.**
The build sheet is rendered as a normal React component (`PDFBuildSheet.jsx`), then rasterised and embedded in a PDF via `html2canvas` + `jsPDF` directly in the browser. There's also a server route using `puppeteer-core` + `@sparticuz/chromium`, but it's the fallback. The reason is twofold: serverless Chromium has a slow cold start and is genuinely flaky on Vercel for non-trivial DOMs, and rendering on the user's device guarantees the fonts and layout match exactly what they saw on screen. The trade-off is bundle size — `html2canvas` and `jsPDF` aren't small, and the off-screen rendering trick (mount at fixed 960px width, fixed left:-9999px) is fragile if anyone changes the build sheet's CSS. To keep both paths in sync, the html2canvas options and the PDF export tunables (scale, JPEG quality, capture width) all live in a single shared module (`data/buildSheetShared.js`).

**2. Email verification instead of accounts.**
There's no signup/login for customers. Instead, the first time a customer submits a build, the API issues a 6-digit code to their email, stores it with a 15-minute expiry, and only persists the build after the code is verified. The verification table is also flagged `used: true` once consumed, and any open verifications for that email are invalidated when a new one is issued. This was likely chosen because the conversion target is "submit a quote request", not "manage an account" — most customers never come back twice, and a sign-up form would have been pure friction. The trade-off is that a verified-customer table with no password isn't really an auth system, so anything sensitive (like the staff manage board) has to live behind a separate gate, which it does (env flag + service-role-only API routes).

**3. Two database tables for two generations of the submission flow.**
Both `build_submissions` (older, written via the Resend route, indexed by order reference) and `rifle_builds` (newer, written via the Brevo + verification route, joined to a `customers` table) exist. The Brevo flow is clearly the active one — it has the verification table, the daily rate-limit logic, and the staff manage board built on top of it. The older table is still wired up for the legacy `send-order-confirmation` route. This is a classic "we built v2 alongside v1 and didn't delete v1" situation. It's pragmatic: keeping both means in-flight orders from the old flow are never lost, but it does mean a future migration is owed.

**4. A custom GLTF loader cache instead of just `useLoader`.**
React Three Fiber's `useLoader` already caches by URL, but the configurator imperatively swaps body and accessory GLBs on every selection change, often before R3F's render loop has had a chance to register the load. `lib/rifle/gltfCachedLoad.js` adds two `Map`s — `settled` (resolved URLs) and `inflight` (in-progress promises) — so a second request for the same model returns the same promise instead of starting a new download. Skeleton + body are also loaded as a single `useLoader([url1, url2])` call to avoid a hook waterfall. The trade-off is a tiny in-memory leak — settled GLBs are never evicted — but rifle GLB inventories are bounded (a few dozen URLs per session) so it doesn't matter in practice.

**5. Region rules decoupled from the catalog.**
The product catalog (`data/server-side/data.json`) is shared across all regions and rarely changes. The compliance rules — which upgrades are legal where, what units to display, which dealer pool to show — live entirely in `data/utils/regionBuilderRules.js`. So when, say, suppressor regulations change for a market, only the rules file is touched. The catalog stays stable. The trade-off is that the rules file ends up being the "scary" file: if it's wrong, an option that shouldn't be sellable in a region becomes available, or vice versa.

## Hardest Problems

**The configurator wizard itself.** The wizard isn't a fixed sequence of steps — it's generated per render from the intersection of (a) the selected rifle's allowed upgrades, (b) the region's excluded keys, (c) special cases for rifles that have a single fixed option for a given step (e.g., PX Hunter's barrel, the hunter stock with only one purchasable variant). On top of that, fields can map to different upgrade keys depending on the rifle (`calibre` resolves to `barrelConfig_hybrid` for the Hybrid, `barrelConfig_pxh` for the Hunter, etc.) so jumping back to "edit calibre" from the review page has to know which step that actually is for this build. This is solved by `buildWizardSteps()` and `getStepIndexForField()` in the rules file, plus a couple of helpers (`isFixedSingleBarrelUpgrade`, `isStockHunterSingleChoiceUpgrade`) that act as feature flags for "skip this step entirely".

**Making the PDF identical across three rendering paths.** The same build sheet gets rendered in at least three places: as an on-page preview, as a downloaded PDF on the customer's device, and as an attachment on the staff email. All three are the same React component (`PDFBuildSheet.jsx`), but the capture conditions differ wildly — narrow mobile viewports can lay an off-screen ref out as 0×0 to html2canvas, fonts may not be loaded yet at capture time, images may not have decoded yet. The fix is the off-screen mount strategy in `lib/buildSheetPdfClient.js`: a fresh React root is mounted off-screen at a fixed width, fonts are awaited via `document.fonts.ready` *and* an explicit `document.fonts.load('700 2rem Serpentine')`, all images are awaited, and two `requestAnimationFrame`s are used to make sure layout has flushed before `html2canvas` reads the DOM.

**The submission API as a single transactional unit.** `POST /api/submit-build` does a lot in one request: customer lookup, conditional verification email, code validation, customer upsert, daily-limit count, build insert, PDF upload to private storage, customer email, region-routed staff email, mark verification used. There's no actual database transaction — Supabase via PostgREST doesn't expose one — so each step has its own error handling, with explicit error codes (`DB_CUSTOMERS_INSERT`, `BREVO_VERIFICATION`, `RATE_LIMIT_DAILY_BUILDS`, etc.) and a `requestId` threaded through every log line so a failed submission can be reconstructed from logs. Staff email failures are deliberately non-fatal: the build is already saved and the customer already notified, so staff getting a missing email is a follow-up problem, not an order-loss problem.

**Loading 3D models fast enough.** A rifle build can pull half a dozen GLBs in quick succession as the user clicks through paint, action, barrel, stock. Without deduping, switching back to a previous selection re-downloads the same model. The combination of (a) hosting all GLBs in a Supabase public bucket via `lib/storage/getModelUrl.js`, (b) the dedup cache in `gltfCachedLoad.js`, and (c) loading skeleton + body in a single parallel `useLoader` call means that swapping a single option usually only fetches the one new GLB.

**Drag-and-drop with optimistic state and a backend round-trip.** The staff board reorders cards optimistically in local state, then sends the entire new column ordering to the API. If the API call fails, the board calls `onRefresh()` to reload from the server and undo the local change. The server-side handler updates every affected row's `stage` and `position` in parallel via `Promise.all`. It's not a real transaction but for a small staff team it's fine.

## What I learned

The biggest lesson on this project was how much of the "hard" work in a configurator like this isn't the 3D rendering — it's the surrounding infrastructure. The Three.js scene was tractable because R3F is a well-trodden path. What was actually hard was making the build state survive being shared, downloaded, emailed, persisted, edited by staff, and re-loaded back into the configurator without drift. Every one of those exit points needs the *same* model of "what is a build", and getting that single shape right (the `selections` object, plus a couple of derived fields) was probably the most impactful single decision.

I also got a much better appreciation for the boring pieces — feature flags, rate limits, error codes, request IDs in log lines — once the project grew beyond a single happy path. The submit-build route in particular taught me the value of being explicit about what's fatal and what's recoverable: a Brevo staff email failing should not roll back a saved build, and structuring the code to make that obvious is more useful than any clever abstraction.

On the front-end side, the dual on-screen / off-screen render of the same build sheet for PDF capture was a fight. It works now, but the lesson was: if you're going to rasterise a React tree, build the rasterisation pipeline at the same time as the component, not after. Trying to bolt html2canvas onto an already-styled component is how you end up debugging mobile layout edge cases at 11pm.

And finally — a project like this isn't really finished. The codebase carries two generations of the submission flow side-by-side, an unused dependency or two, a couple of legacy imports. Shipping a working v2 next to a working v1 is, in my experience, the realistic shape of moving a real product forward without breaking what's already serving customers.
