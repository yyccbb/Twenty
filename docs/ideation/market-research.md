# Twenty: Competitive Market Research

**Date:** 2026-10-25
**Scope:** iOS-first productivity app (web later). It combines a 20-minute-block calendar, scheduled planning prompts, planned-vs-actual tracking with a late-start penalty, AI goal breakdown, and a token/badge economy.

> **About the sources.** Pricing for this category changes often and varies by region. Many third-party comparison pages are written by competitors (Morgen, Saner.ai), and they often disagree with each other. Prices below are indicative, so check them on each vendor's own page before using them in a pitch.

---

## 1. Executive summary

- **No app found combines all five features.** No app found enforces a fixed block size that isn't a standard calendar step (such as 20 minutes). None found automatically moves a late start to the next block either.
- **The nearest competitors split into two camps.** Visual day planners (Structured, Tiimo, Sunsama) cover features 1, 2 and partly 4. Gamified focus and habit apps (Forest, Habitica, Finch, Todoist Karma) cover feature 5. Time trackers (Toggl, Timing, EARLY) cover the "actual" half of feature 3, but not in relation to a plan.
- **The market is proven and crowded.** Tiimo, a visual ADHD planner with an AI co-planner, won Apple's **2025 iPhone App of the Year** ([Tiimo](https://www.tiimoapp.com/resource-hub/tiimo-winner-2025-app-store-awards)). Structured has about 165k US App Store ratings at 4.8 stars ([Habitbox](https://habitbox.app/blog/structured-app-review)). AI goal breakdown alone is now a commodity: many 2026 App Store entrants offer it for $1.99/month or less.
- **The main differentiator is the "snap-to-next-block" late-start rule.** It is also the biggest UX risk. Reviews of Forest and Habitica show that punishment can drive users away as easily as it motivates them.

---

## 2. Competitor landscape by category

### 2.1 Time-blocking and day planners

| App | Platforms | Pricing (indicative) | Notes |
|---|---|---|---|
| **Structured** | iOS, iPadOS, Mac, Android (late 2023), Web (Aug 2024) | Free; Pro ~$6.99/mo, $29.99/yr, $99.99 lifetime (US, Sep 2026). Prices change often ([Habitbox](https://habitbox.app/blog/structured-app-review), [AppPricingLab](https://apppricinglab.com/iap/apple/1499198946)) | Visual timeline, AI planning assistant and "Replan" in Pro. iOS gets features first and the web app is the most limited ([Saner review](https://blog.saner.ai/structured-review/), [Structured blog](https://structured.app/blog/structuredfive)) |
| **Tiimo** | iOS, iPadOS, watchOS, Android, web | Free (mobile, no AI); Pro ~$7.99/mo or $79.99/yr; Family $119.99/yr ([Lifestack](https://lifestack.ai/blog/tiimo-pricing)) | ADHD-first visual schedule, AI co-planner that splits text into tasks, subtasks and time estimates, focus timer ([App Store](https://apps.apple.com/app/tiimo/id1480220328)) |
| **Sunsama** | Web, Mac/Win (Electron), iOS, Android | ~$20/mo, no free tier ([Morgen](https://www.morgen.so/blog-posts/sunsama-pricing)) | Guided daily planning ritual, workload warnings, evening shutdown ([Sunsama](https://www.sunsama.com/features/daily-planning-and-shutdown)) |
| **Akiflow** | Web, desktop, mobile | ~$19/mo ([Morgen](https://www.morgen.so/pt/blog-posts/sunsama-vs-akiflow)) | Universal inbox plus drag-to-calendar time blocking, aimed at power users |
| **Motion** | Web, desktop, iOS, Android | From $29/mo (annual) or $49/mo (monthly), no free plan, AI credits ([Morgen](https://morgen.so/blog-posts/motion-pricing)) | AI auto-scheduling that keeps re-optimizing. Complaints cover price, trial billing, feature bloat and a weak mobile app ([Morgen review](https://www.morgen.so/blog-posts/motion-review), [Capterra](https://www.capterra.ie/reviews/214264/motion)) |
| **Reclaim.ai** | Web plus Google/Outlook calendars | Free; Starter ~$10; Business ~$15–18 per user per month ([SchedulingKit](https://schedulingkit.com/pricing-guides/reclaim-ai-pricing)) | Auto-scheduled habits and focus time on your existing calendar |
| **Amie / Fantastical / Google Calendar** | Various | Amie freemium ([Toolradar](https://toolradar.com/compare/fantastical-vs-amie)); Fantastical ~$4.75–6.99/mo ([Morgen](https://morgen.so/blog-posts/fantastical-pricing)); Google free | General-purpose calendars. Snap granularity is cosmetic (5/15/30 min) and is not a rule the app enforces |
| **Hour Blocks, TimeBloc, ClearDay** | iOS | Free or low-cost | Fixed hourly or 15-minute grids. Hour Blocks allows one event per hour slot, the closest to a "timetable" model ([App Store](https://apps.apple.com/app/id1456275153), [ClearDay](https://apps.apple.com/us/app/id6744560565)) |

### 2.2 Planned-vs-actual time tracking

| App | Platforms | Pricing | Notes |
|---|---|---|---|
| **Toggl Track** | Web, desktop, iOS, Android | Generous free tier; paid from ~$9/user/mo ([ClickUp](https://clickup.com/blog/clockify-vs-toggl/)) | Calendar integration, timers can start from calendar events, project estimate vs tracked charts, Pomodoro |
| **Clockify** | Web, desktop, mobile | Free; cheaper paid tiers | Stopwatch or manual entries, dashboard of tracked vs remaining time |
| **Timing** | macOS (plus iOS companion) | From ~$9/mo ([Timing](https://timingapp.com/blog/mac-time-tracking-apps/)) | Automatic app and website tracking, data stored locally, AI summaries |
| **EARLY (formerly Timeular)** | Desktop, mobile, plus an $69 physical cube | ~$7.50–15.80/user/mo, no free plan ([fahimai](https://www.fahimai.com/?p=61834)) | Hardware-assisted tracking |
| **Session** | macOS, iOS | Freemium subscription | Pomodoro plus intention logging and per-session analytics |

**Gap:** these tools measure time but don't judge it against a plan. They do not snap or penalize lateness against a planned block.

### 2.3 Timeboxing and Pomodoro

- **Forest:** iOS and Android. Moved from a one-time $3.99 purchase to Forest Plus (~$59.99/yr) in Dec 2025 ([Headway](https://makeheadway.com/blog/forest-app-review/)). A virtual tree dies if you leave the app, and coins fund planting real trees (2M+ planted) ([App Store](https://apps.apple.com/us/app/forest-focus-for-productivity/id866450515)).
- **Focus To-Do, Flow, Be Focused:** Pomodoro timers with task lists, sold freemium or by one-time unlock. None of them have a calendar-block model.
- **Llama Life:** Every task on the list gets a countdown timer. ~$6/mo or $48/yr ([TrendingAITools](https://www.trendingaitools.com/ai-tools/llama-life/)).

### 2.4 Gamified productivity

| App | Platforms | Pricing | Mechanics | Notable feedback |
|---|---|---|---|---|
| **Habitica** | Web, iOS, Android (open source) | Free; optional subscription and gems | RPG: XP, gold, HP loss for missed dailies, party damage | Party damage feels punishing when teammates go inactive, novelty fades by month two, heavy onboarding ([Unstar](https://unstar.app/blog/habitica-streaks-productive-way-of-life-habit-bull-habit-tracker-apps-ranked-2026)). An LMU study found the reward/punishment system had counterproductive effects ([LMU](https://epub.ub.uni-muenchen.de/77668)) |
| **Forest** | iOS, Android | See above | Grow or kill a tree, coins | "Dead tree guilt" is a recurring theme. Some find it motivating, others shaming ([GearBrain](https://www.gearbrain.com/forest-productivity-smartphone-app-2622588862.html), [Bustle](https://www.bustle.com/wellness/forest-focus-screen-time-app-review)) |
| **Finch** | iOS, Android | Free; Plus ~$39.99–69.99/yr ([AppPricingLab](https://apppricinglab.com/iap/apple/1528595748)) | Self-care pet, energy, rainbow stones for cosmetics | Praised for a no-punishment tone. Complaints cover bugs (32% of negative iOS reviews), pricing differences between platforms, and the pet feeling like a chore ([Appbot](https://appbot.co/blog/finch-app-reviews-emotional-attachment-user-retention-product-loyalty/), [Headway](https://makeheadway.com/blog/finch-app-review/)) |
| **Focus Friend (Hank Green)** | iOS, Android | Free plus in-app purchases | Bean knits only while you focus. Knitted items are currency for room decor | Went viral in 2025, gentle tone ([NBC](https://www.nbcnews.com/tech/tech-news/hank-green-focus-friend-app-rcna225967)) |
| **LifeUp** | Android (plus ulives on iOS) | One-time purchase | XP and coins spent in a **user-defined** shop | Popular with power users, supported through Patreon ([Patreon](https://www.patreon.com/lifeupapp/about)) |
| **Todoist Karma** | All platforms | Included | Points for on-time completion, streaks, levels up to "Enlightened" (50k). **Overdue tasks lose points** after 4–5 days. Vacation mode keeps streaks. Can be switched off ([Todoist](https://todoist.com/help/articles/introduction-to-karma-OgWkWy)) | Closest precedent for "fewer points if late", and it is lenient and optional |

### 2.5 AI goal breakdown

- **Goblin Tools Magic ToDo:** Free on the web, ~$1 on mobile, with a "spiciness" slider from 1 to 5 chillies for how fine the breakdown is ([Saner](https://www.saner.ai/blogs/goblin-tools-review)). Beloved by ADHD users, but it has no calendar.
- **Motion AI, Notion AI, ClickUp AI, Taskade:** Project and workspace-level breakdown, aimed at teams and too heavy for personal use.
- **Saner.ai:** ADHD "personal assistant" combining notes, tasks and calendar. Free tier with 30 AI requests a month, Starter ~$8, Standard ~$16/mo ([Yespress](https://yespress.io/products/saner-ai.md), [eesel](https://www.eesel.ai/blog/saner-ai-pricing)).
- **2026 App Store entrants:** *Goals – AI Action Planner* (Mar 2026, $1.99/mo), *SparkYear* (auto-rebuilds the schedule after a missed day, has a "Rescue Mode"), *StepJoy* and *Stepwise* ([App Store](https://apps.apple.com/us/app/-/id6757492168), [SparkYear](https://apps.apple.com/us/app/-/id6755699796)). Goal-to-steps is now a commodity, so the value is in what happens **after** the breakdown (scheduling and accountability).

### 2.6 ADHD and anti-procrastination

Tiimo, Structured and Llama Life lead this space. **Inflow** is CBT-based ADHD coaching, priced $22.49–199.99 across tiers ([AppPricingLab](https://apppricinglab.com/iap/apple/1528183849)). **Brili** offers routine timers for kids and adults; no current data was found on it. The tone across this category is strongly **non-judgmental**, which matters for feature 3.

---

## 3. Feature matrix

Key: ✅ = has it, ◐ = partial or similar, ❌ = absent

| App | 1. Fixed-block calendar | 2. Proactive planning prompts | 3. Actual-time logging with late-start snap | 4. AI goal → tasks onto calendar | 5. Tokens → badges |
|---|---|---|---|---|---|
| **Twenty (planned)** | ✅ 20 min enforced | ✅ configurable | ✅ snaps and penalizes | ✅ | ✅ |
| Structured | ❌ (free durations) | ◐ reminders, inbox | ◐ check-off only | ◐ AI assistant | ❌ |
| Tiimo | ❌ | ◐ | ◐ focus timer | ✅ AI co-planner | ❌ |
| Sunsama | ❌ | ✅ daily ritual at a set time | ◐ planned vs actual time on tasks | ❌ | ❌ |
| Motion | ❌ | ❌ (auto-plans for you) | ❌ | ✅ | ❌ |
| Reclaim | ❌ | ❌ | ❌ | ◐ habit auto-scheduling | ❌ |
| Akiflow | ❌ | ◐ planning ritual | ❌ | ◐ | ❌ |
| Hour Blocks / ClearDay | ◐ hourly / 15-min grid | ❌ | ❌ | ❌ | ❌ |
| Toggl / Clockify / Timing | ❌ | ◐ tracking reminders | ◐ actual only, no snap | ❌ | ❌ |
| Forest / Focus Friend | ❌ | ❌ | ◐ focus sessions | ❌ | ✅ coins / cosmetics |
| Habitica | ❌ | ◐ daily reset | ❌ | ❌ | ✅ gold, gear, HP penalty |
| Finch | ❌ | ✅ daily check-ins | ❌ | ❌ | ✅ |
| Todoist Karma | ❌ | ❌ | ❌ | ◐ AI assist | ◐ points/levels, late penalty |
| Goblin Tools | ❌ | ❌ | ❌ | ✅ (no calendar) | ❌ |
| Saner.ai | ❌ | ◐ | ❌ | ✅ | ❌ |

---

## 4. Tech stacks (where public)

| App | Client | Backend / sync | Source |
|---|---|---|---|
| Tiimo | Native **SwiftUI** (iOS/iPadOS/watchOS, migrated from Xamarin). Earlier web app in Angular | Earlier stack was .NET Web API on Azure, current state unconfirmed | [Apple Developer](https://developer.apple.com/articles/tiimo), [The Hub job](https://thehub.io/jobs/674db3b07b6b0dbbfd816ad5) |
| Structured | Native Apple apps, separate Android and web | Two sync paths: **iCloud** (Apple only) and **Structured Cloud** (cross-platform, needed for web/Android). iCloud sync reliability is a frequent complaint | [Help center](https://help.structured.app/en/articles/330754), [Toolguide](https://toolguide.io/en/tool/structured/) |
| Sunsama | **Electron** desktop, React web | Reportedly Meteor, Node.js, MongoDB on AWS (self-reported on StackShare, unverified) | [DesktopInsights](https://desktopinsights.com/apps/sunsama), [StackShare](https://stackshare.io/mds4mstuke/sunsama) |
| Habitica | Vue.js web, native iOS/Android | Node.js/Express, MongoDB replica set, open source | [Railway](https://railway.com/deploy/habitica), [GitTrend](https://gittrend.io/repo/HabitRPG/habitica) |
| Motion, Akiflow, Finch, Forest | Not publicly documented | — | — |

---

## 5. Differentiation and gaps

### What looks new
1. **A fixed, non-standard block size the app enforces (20 min).** Existing grids are hourly (Hour Blocks) or simply snap to a step (Google, Fantastical, ClearDay). None treats the block as a hard unit.
2. **A late-start rule that shrinks the block.** No app found moves a missed start to the next block boundary. Todoist Karma's overdue-point loss and Habitica's HP damage are the closest precedents, and both work at the scale of days, not minutes.
3. **A plan → actual → reward loop in one product.** Today a user would need Structured or Sunsama, plus Toggl, plus Habitica or Forest.
4. **Gamification tied to timeliness, not just completion.** Most token systems reward finishing or focusing. Paying out less when late is uncommon.

### Gaps competitors leave open
- Planners show the plan but rarely record what actually happened. Trackers record what happened but ignore the plan.
- AI breakdown tools (Goblin Tools and the 2026 entrants) stop at the list and don't place tasks on the calendar.
- Many users are tired of subscription creep: Structured's lifetime price tripled, Forest moved to a subscription, Motion keeps raising prices. That leaves room for a fair, transparent pricing model.

### Risks
- **The late-start penalty may feel punishing.** ADHD users are a core audience for this category, and its leaders (Tiimo, Finch) deliberately avoid shame. Forest's "dead tree guilt" and Habitica's party damage both drive users away. Mitigations:
  - frame it as "your block starts at :40" rather than "you lost 20 minutes"
  - give an occasional "grace token" or freeze, like Todoist's vacation mode and Finch's no-punishment tone
  - make the penalty optional or adjustable
  - show a recovery path, as SparkYear's Rescue Mode does
- **Churn in gamified apps.** Novelty fades within about 1–2 months (Habitica), and engagement with gamification features does not mean retention ([Mind the Product](https://www.mindtheproduct.com/why-gamification-kills-the-products-its-supposed-to-save/)). Badges need a long-term "endgame". Consider user-defined rewards like LifeUp's custom shop, and seasonal collections.
- **20-minute rigidity.** It will clash with real meetings (30 or 60 minutes) imported from external calendars. Decide how imported events are displayed: rounded outward, or shown as "non-Twenty" events.
- **Commoditized AI.** Goal breakdown alone will not differentiate Twenty, so its value must come from scheduling and follow-through.
- **Incumbents could copy it.** Structured or Tiimo could add planned-vs-actual tracking. The defensible part is the opinionated method plus the token economy.

---

## 6. Lessons for design and architecture

### Planning prompts and notifications
- **Sunsama** lets users set a "when do you plan your day" time under Settings → Rituals and prompts them then. If yesterday's shutdown was skipped, planning starts by reviewing those leftover tasks. The evening shutdown is a three-step flow: review, reflect, wrap up ([Sunsama help](https://help.sunsama.com/docs/daily-planning), [Shutdown](https://help.sunsama.com/docs/daily-shutdown)). For Twenty this means:
  - make the nightly prompt a short guided flow (review today's actuals → carry over → plan tomorrow's blocks), not a bare push notification
  - schedule it as a local notification, so it works offline
- **Start-time prompts** should be local notifications set at each block boundary, plus one more at +3 minutes (end of the grace period). On iOS, use **Live Activities and widgets** to show the running block and a one-tap "Start now" or "Done" action, so recording takes less effort than ignoring it.
- **Toggl** starting timers from calendar events shows that recording actuals should be pre-filled from the plan. The user should only confirm or adjust.

### Syncing iOS and web
- **Structured** runs two sync backends: iCloud (Apple-only) and its own cloud for Android and web. Sync bugs and lost purchases are among its most common support topics ([Help: troubleshooting](https://help.structured.app/en/categories/99010-troubleshooting)). Since web is on Twenty's roadmap, start with **one account-based backend** from day one rather than iCloud/CloudKit. Some options:
  - Supabase/Postgres with row-level security, or Firebase/Firestore
  - an offline-first local store on iOS (SwiftData/GRDB) with a sync queue
- **The token ledger must be computed on the server.** Tokens are per-account and tradable. Treat awards as append-only ledger entries calculated on the server from recorded start and finish times, which prevents device-clock tampering. Keep all 20-minute snap logic in one shared, tested module, used by both the server and clients.
- **Store times as UTC instants** with the user's time zone. Snap to 20-minute marks in local time, which matters for time zones offset by 30 or 45 minutes.
- **Native SwiftUI is the stronger choice for an iOS-first app.** Tiimo's migration (accessibility and animation gains) and the importance of widgets and Live Activities support it. If web is a near-term priority, a shared TypeScript domain or API layer plus a React web app is the common pattern (Sunsama, Habitica).

### AI integration
- Do the breakdown **server-side** so keys stay private, costs can be metered, and prompts can be tuned without app updates. Motion's credit-based AI pricing shows that AI cost has to be planned into the business model ([Morgen](https://morgen.so/blog-posts/motion-pricing)).
- Follow Goblin Tools: offer a **granularity control** (its "spiciness" slider), and output tasks sized in **multiples of 20 minutes** so they drop straight onto the grid.
- Follow Tiimo and Structured: AI output should be **editable suggestions** placed in an inbox or tray, not changes made directly to the calendar. Motion's fully automatic rescheduling is praised for power, but some users find it opaque.
- Gate AI behind the paid tier while keeping core planning free, as Structured and Tiimo do. This keeps free-tier AI costs in check.

---

## 7. Positioning takeaway

Twenty can credibly pitch itself as **"Plan in 20-minute periods, show up on time, earn your badges."** The nearest products (Structured, Tiimo, Sunsama) cover planning but not accountability. Gamified apps cover motivation but not the calendar. The strategic bet is whether the late-start rule feels like a helpful nudge or a punishment. Test that with early users, give them grace mechanics, and default to a forgiving tone.
