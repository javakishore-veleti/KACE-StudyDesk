# KACE-StudyDesk

**Public Shopify Hydrogen storefront for selling micro-courses with drip-release lessons.**

Customer-facing storefront deployed on top of `<STUDYDESK_STORE>.myshopify.com`. Courses are sold as Shopify products; lessons are unlocked on a schedule (day 1, day 3, day 7, …) gated by `kace_reward_rule` MetaObjects evaluated by `KACECommerceEngine`.

---

## Placeholders used in this README

| Placeholder | Description | Example |
| --- | --- | --- |
| `<GITHUB_HANDLE>` | GitHub account the repos live under | `your-github-handle` |
| `<STUDYDESK_STORE>` | StudyDesk dev store handle | `studydesk-dev` |
| `<KACE_ENGINE_URL>` | URL of KACECommerceEngine in the current env | `http://localhost:8080` in dev |
| `<CUSTOMER_ACCOUNT_API_CLIENT_ID>` | Client ID for Shopify Customer Account API on this store | (from Dev Dashboard) |
| `<CUSTOMER_ACCOUNT_API_ISSUER_URL>` | OIDC issuer URL for this store's customer auth | `https://shopify.com/<store-id>` |

---

## What does KACE mean?

**KACE** is the umbrella brand for this 4-repo project. Treat it as a standalone acronym (like IKEA or NASA) — don't re-expand it in every sentence. Historically the letters came from **K**ishore **A**pps **C**ommerce **E**ngine; today it's the suite's brand name.

**The 4 repos:**

| Repo | Role |
| --- | --- |
| [KACECommerceEngine](https://github.com/<GITHUB_HANDLE>/KACECommerceEngine) | TS + Fastify middleware. Rule facade, rewards, Shopify BFF, hand-rolled SessionStorage. **The brain.** |
| [KACE-PromptKart](https://github.com/<GITHUB_HANDLE>/KACE-PromptKart) | Hydrogen public storefront — AI prompt packs. |
| [**KACE-StudyDesk**](https://github.com/<GITHUB_HANDLE>/KACE-StudyDesk) *(this repo)* | Hydrogen public storefront — micro-courses with drip release. |
| [KACE-Extended-Rules-Sidecar](https://github.com/<GITHUB_HANDLE>/KACE-Extended-Rules-Sidecar) | Java + Spring Boot + Drools JVM sidecar. |

---

## Architecture

### Where this storefront sits

```
Customer browser
        │
        ▼
┌──────────────────────────────────────────────┐
│  KACE-StudyDesk  (Hydrogen + Remix, :3001)   │
│                                              │
│  • Shopify Storefront API                    │
│    (public catalog — courses, bundles)       │
│  • Shopify Customer Account API              │
│    (student login via OAuth/OIDC)            │
│                                              │
│  On course view / lesson view / cart update: │
│  POST <KACE_ENGINE_URL>/api/v1/rules/evaluate│
│  POST <KACE_ENGINE_URL>/api/v1/rewards/apply │
│  GET  <KACE_ENGINE_URL>/api/v1/drip/schedule │
└────────────────┬─────────────────────────────┘
                 │
                 ▼
     ┌──────────────────────────┐
     │  KACECommerceEngine      │
     │  (Fastify TS middleware) │
     └──────────┬───────────────┘
                │ `orders/paid` webhook → EarnRewardsWorkflow creates drip-schedule rows
                ▼
          Postgres (tenant=STUDYDESK, drip_schedule table)
```

Core properties:

- **Same thin-rendering pattern as KACE-PromptKart.** Business decisions (price gating, discounts, lesson access) come from KACE's rule engine; the storefront only renders the resulting `RenderModel`.
- **Drip-schedule data lives in KACE's Postgres**, not here. This storefront reads it via KACE APIs.
- **Never calls Admin GraphQL directly.**

### Core UX patterns

#### 1. Guest-vs-logged-in gating (same as KACE-PromptKart)

- **Guests** see course title, hero image, syllabus teaser, intro video thumbnail — **no** price, no Enroll button.
- **Logged-in customers** see full price (with any rule-driven discount), full syllabus, video previews, Enroll button, drip progress.

#### 2. Drip-release lesson access

After purchase, lessons unlock on a schedule (day 1, day 3, day 7, …). The access check is expressed as two `kace_reward_rule` MetaObjects authored in the Shopify admin:

```jsonc
// Lesson is unlocked → grant access
{
  "id":       "studydesk-lesson-access-check",
  "trigger":  "lesson.access-check",
  "priority": 100,
  "conditions": { "all": [
    { "fact": "customer.isAuthenticated", "operator": "==", "value": true },
    { "fact": "customer.hasPurchased",    "operator": "==", "value": true },
    { "fact": "lesson.unlockAt",          "operator": "<=", "value": "now()" }
  ]},
  "actions": [ { "type": "GRANT_ACCESS", "params": { "resource": "lesson" } } ]
}

// Lesson still locked → render countdown CTA
{
  "id":       "studydesk-lesson-locked",
  "trigger":  "lesson.access-check",
  "priority": 90,
  "conditions": { "all": [
    { "fact": "customer.hasPurchased", "operator": "==", "value": true },
    { "fact": "lesson.unlockAt",       "operator": ">",  "value": "now()" }
  ]},
  "actions": [
    { "type": "SHOW_CTA", "params": { "text": "Unlocks {lesson.unlockAt | relative}", "disabled": true } }
  ]
}
```

No access logic in the React component. `<LessonPlayer>` renders only if `renderModel.grantAccess === true`; otherwise `<DripCountdown>` renders.

> For how KACE's own `SessionStorage` gets invoked when this storefront authenticates a student and when drip-schedule rows are written on `orders/paid`, see **[README_How_SessionStorage_Invoked.md](https://github.com/<GITHUB_HANDLE>/KACECommerceEngine/blob/main/README_How_SessionStorage_Invoked.md)** in KACECommerceEngine.

### Drip-schedule creation on purchase

- Webhook `orders/paid` on `<STUDYDESK_STORE>` → KACECommerceEngine runs `EarnRewardsWorkflow`.
- A workflow-specific task (`CreateDripScheduleTask`) reads the course's lesson list from a Shopify metafield (`course.custom.lessons`) and creates one drip row per lesson with `unlockAt = orderPaidAt + lesson.offsetDays`.
- Drip schedule lives in KACE's Postgres, tenant-scoped.

### Rendering pipeline for a lesson view

```
/courses/$handle/lessons/$lesson.tsx
   loader(args)
      │
      ▼
   RenderLessonWorkflow
      │  reusable tasks (src/tasks/):
      │    FetchCustomerTokenTask
      │    FetchLessonMetadataTask
      │    CallKaceDripScheduleTask          ← loads the drip-schedule row
      │    CallKaceEvaluateRulesTask         ← trigger=lesson.access-check
      │    ApplyActionsToRenderModelTask
      │    BuildLoaderResponseTask
      │  workflow-specific tasks:
      │    CheckDripAccessTask               ← sanity-check before rendering
      ▼
   { lesson, renderModel } → <LessonPlayer/> OR <DripCountdown/>
```

---

## Planned directory layout

```
app/
├── root.tsx  entry.server.tsx  entry.client.tsx  tailwind.css
│
├── routes/
│   ├── _index.tsx                             home: featured courses
│   ├── courses._index.tsx                     course listing
│   ├── courses.$handle.tsx                    course detail
│   ├── courses.$handle.lessons.$lesson.tsx    lesson playback (gated by drip rule)
│   ├── cart.tsx
│   └── account/
│       ├── login.tsx  callback.tsx  logout.tsx
│       ├── _index.tsx                          "My Courses" + drip progress + badges
│       └── courses.$handle.tsx                 per-course dashboard (unlocked/locked lessons)
│
├── components/
│   ├── CourseCard, CourseDetail, SyllabusTeaser
│   ├── LessonPlayer, DripCountdown
│   ├── EnrollCta, DiscountPill, BadgeShelf
│
└── lib/                                        server-side helpers (mirrors backend layout)
    ├── api/loaders/                            course.loader, lesson.loader, account.loader
    ├── service/ + impl/workflows/
    │   ├── RenderCourseDetailWorkflow/…
    │   ├── RenderLessonWorkflow/tasks/CheckDripAccessTask.ts
    │   └── ApplyCartRewardsWorkflow/…
    ├── tasks/                                  reusable
    │   ├── FetchCustomerTokenTask.ts
    │   ├── CallKaceEvaluateRulesTask.ts
    │   ├── CallKaceApplyRewardsTask.ts
    │   ├── CallKaceDripScheduleTask.ts
    │   ├── ApplyActionsToRenderModelTask.ts
    │   └── BuildLoaderResponseTask.ts
    ├── core/
    │   ├── workflow/
    │   ├── domain/                             RenderModel, KaceAction, Course, Lesson, Customer
    │   └── action-appliers/                    HideField, ShowCta, ApplyDiscount, GrantAccess, GrantBadge, Registry
    ├── dao/ + impl/                            storefront + KACE HTTP
    ├── dtos/ constants/ utils/ config/
```

Same `api/service/core/dao/dtos/constants/utils/config + tasks/workflow` layout as `KACECommerceEngine` and `KACE-PromptKart`.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| Language | TypeScript (strict) |
| Framework | Hydrogen (Remix-based) |
| Shopify libs | `@shopify/hydrogen`, `@shopify/hydrogen-react`, `@shopify/customer-account-api-client` |
| UI | React + Tailwind CSS |
| Video player | Shopify-hosted video (v0) → Mux / Cloudflare Stream later |
| Deployment (future) | Shopify Oxygen or Cloudflare Workers |
| Local dev | `shopify hydrogen dev --port 3001` |
| Tests (future) | Vitest + Playwright (guest view, login, locked lesson, unlocked lesson) |

---

## Status

⏳ **Pre-scaffold.** Repo currently holds this README, an MIT license, and a `.gitignore`. Hydrogen scaffolding + KACE integration + drip-release plumbing ship in a later phase.

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Related planning docs

Full design docs for this storefront (and the full KACE suite) live in the parent planning folder — architecture, per-service READMEs, diagrams, development-plan spreadsheet.
