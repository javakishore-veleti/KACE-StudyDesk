# KACE-StudyDesk

**Public Hydrogen storefront for selling micro-courses with drip-release lessons on `studydesk-dev.myshopify.com`.**

---

## What does KACE mean?

**KACE** is the umbrella brand for this 4-repo project. It is a standalone acronym
(treat it like IKEA or NASA — don't re-expand it in every sentence). Historically
the letters came from **K**ishore **A**pps **C**ommerce **E**ngine, but today KACE
is the name of a 4-service suite built on Shopify by
[**Kishore Veleti**](https://github.com/javakishore-veleti) — Shopify Partner org
*Kishore Applications* (Partner ID `4868609`, org id `214691442`).

**The 4 repos in the KACE suite:**

| Repo | Role |
| --- | --- |
| [KACECommerceEngine](https://github.com/javakishore-veleti/KACECommerceEngine) | TypeScript + Fastify middleware — rule engine facade, rewards, Shopify BFF, hand-rolled SessionStorage. **The brain.** |
| [KACE-PromptKart](https://github.com/javakishore-veleti/KACE-PromptKart) | Hydrogen public storefront selling AI prompt packs. |
| [**KACE-StudyDesk**](https://github.com/javakishore-veleti/KACE-StudyDesk) *(this repo)* | Hydrogen public storefront selling micro-courses. |
| [KACE-Extended-Rules-Sidecar](https://github.com/javakishore-veleti/KACE-Extended-Rules-Sidecar) | Spring Boot + Drools JVM sidecar. |

---

## What is `KACE-StudyDesk` specifically?

A **public Shopify Hydrogen storefront** for selling short **micro-courses** (cloud,
architecture, AI) with **drip-release lesson delivery**. Dev URL:
`studydesk-dev.myshopify.com`.

### Core UX patterns

#### 1. Guest-vs-logged-in gating (same as KACE-PromptKart)

- **Guests** see course title, hero image, syllabus teaser, intro video thumbnail — no
  price, no Enroll button.
- **Logged-in customers** (Shopify Customer Account API) see the full price (with any
  rule-driven discount), full syllabus, video previews, Enroll button, drip progress.

#### 2. Drip-release lesson access

After a customer purchases a course, lessons unlock on a schedule (day 1, day 3, day 7,
etc.). The access check is expressed as a MetaObject rule:

```jsonc
// "studydesk-lesson-access-check"
conditions: all([
  customer.hasPurchased == true,
  lesson.unlockAt <= now()
])
actions: [ GRANT_ACCESS ]
```

If the rule fires, the storefront shows the lesson player. If it doesn't (because the
lesson is still locked), the storefront shows a countdown instead.

### Responsibilities of this repo

- React/Remix/Hydrogen storefront UI.
- Shopify Storefront API for public catalog.
- Shopify Customer Account API for student login.
- Lesson playback UI (player, progress tracking).
- Thin HTTP client to `KACECommerceEngine` for every business decision (gating,
  pricing, drip access, rewards).

### What this repo is NOT

- Not a Shopify merchant-installed admin app.
- Does not call Shopify Admin GraphQL directly.
- Does not own the drip-schedule data — that lives in Postgres inside
  `KACECommerceEngine`, populated on the `orders/paid` webhook.

### Status

⏳ **Pre-scaffold.** Repo has a placeholder README, MIT license, and `.gitignore`.
Actual Hydrogen code ships in a later phase.

---

## Planned tech stack

| Layer | Choice |
| --- | --- |
| Language | TypeScript (strict) |
| Framework | Hydrogen (Remix-based) |
| Shopify libs | `@shopify/hydrogen`, `@shopify/hydrogen-react`, `@shopify/customer-account-api-client` |
| UI | React + Tailwind CSS |
| Video player | Shopify-hosted video (v0) → Mux / Cloudflare Stream later |
| Deployment (future) | Shopify Oxygen or Cloudflare Workers |
| Local dev | `shopify hydrogen dev` on port `:3001` |

---

## Planned directory layout

```
app/
├── routes/
│   ├── courses.$handle.tsx
│   ├── courses.$handle.lessons.$lesson.tsx   # gated by drip rule
│   └── account/                              # student dashboard, drip progress
├── components/
│   ├── CourseCard, CourseDetail, SyllabusTeaser
│   ├── LessonPlayer, DripCountdown
│   ├── EnrollCta, DiscountPill, BadgeShelf
└── lib/                                       # server-side helpers
    ├── api/loaders/ (course, lesson, account)
    ├── service/ + impl/workflows/             # RenderCourseDetailWorkflow, RenderLessonWorkflow
    ├── tasks/                                 # reusable (FetchCustomerToken, CallKaceEvaluateRules, CheckDripAccess)
    ├── core/                                  # framework-free (RenderModel, KaceAction, domain types)
    ├── dao/ + impl/                           # storefront + KACE HTTP
    ├── dtos/ constants/ utils/ config/
```

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Related planning docs (not in this repo)

Full design lives in the parent planning folder (`Shopify_Middleware/`):

- `README_KACE-StudyDesk.md` — full architecture of this storefront
- `README_KACECommerceEngine.md` — the middleware this storefront calls
- `Development_Plan.xlsx` — phase-by-phase roadmap
- `diagrams/` — architecture + mindmap images
