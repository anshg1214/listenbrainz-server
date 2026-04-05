# GSoC Proposal Review Questions — User Onboarding

---

## Clarification Questions (Did they write it themselves?)

1. Your `markComplete` function in the placeholder just returns `Promise.resolve()` and does nothing. What will it actually do in the real implementation?

2. You wrote `shouldStart = Boolean(pageKey) && Boolean(userToken)` — this is always `true` for any logged-in user. How does this check whether the user has already seen the tour?

3. What is `EventData` in your `handleEvent` function? Where does it come from?

4. You imported `STATUS` in your callback check (`data.status === STATUS.FINISHED`). What package does `STATUS` come from and what other values does it have?

5. In your GET endpoint, you do an INSERT before reading the status. Why?

6. You used `onEvent` on the Joyride component — where did you find that prop in the documentation?

7. Your `options` prop has `buttons: ["back","close","primary"]` — what does that actually do?

---

## Technical Questions (Depth of Understanding)

### On the React Joyride API

8. Your component uses `onEvent={handleEvent}`. Can you check the React Joyride documentation and confirm the correct prop name for receiving tour callbacks? What does it actually accept?

9. In your `options` prop you pass `buttons: ["back","close","primary"]`. Open the Joyride docs and show me which option controls the Skip button visibility and which controls the Back button. How would you configure those?

10. React Joyride renders a beacon on each step target by default. Your proposal mentions a "pulsating beacon" on Explore pages. Is this Joyride's built-in beacon, or are you building a custom component? If it's custom, how does clicking it start the Joyride tour?

### On Cross-Page Navigation

11. Your per-page tours are scoped to individual pages. But the welcome wizard tour includes steps across multiple pages (dashboard, then stats). How does Joyride move from `/user/:username/` to `/user/:username/stats/` mid-tour? Show me the code that would handle this — Joyride doesn't navigate by itself.

12. When your tour navigates to a new page and that page fetches data asynchronously (like the Stats page, which loads from the Spark API), what happens if Joyride tries to find the target element before the component has mounted? How do you handle that?

### On the Custom Hook

13. In your placeholder hook, `shouldStart` is `Boolean(pageKey) && Boolean(userToken)`. This means it returns `true` for every logged-in user on every page, always. What is the actual condition you intend to check? Walk me through what the real hook does differently.

14. Your `markComplete` function in the placeholder is `return Promise.resolve()` — it does nothing. In the real implementation, what API call does it make, and what happens if that API call fails? Does the tour state stay as `not_started` in the DB? What does the user see?

15. How does your hook know which `page_key` to use for a given page? Who decides the key names, and where are they defined? What happens if two different developers use different keys for the same page?

### On the Backend Design

16. Your GET endpoint does an `INSERT ... ON CONFLICT DO NOTHING` before reading the status. Why does a read operation write to the database? What problems could this cause if the same endpoint is called multiple times rapidly (e.g., React StrictMode double-invokes effects in development)?

17. Your ENUM has two values: `not_started` and `completed`. What status does a user have while they are actively mid-tour — say they've completed 4 of 8 steps and closed the browser? Can they resume, or do they start over?

18. Your POST reset endpoint sets status back to `not_started`. If a user clicks "restart tour" in Settings, does the tour fire immediately on the settings page, or only when they next visit the relevant page? Walk me through the user flow.

### On Architecture

19. You have one `OnboardingTour` component. Each page will have its own instance of it with its own `pageKey` and `steps`. If a user is on the Stats page and the Stats tour is running, and they manually navigate away mid-tour — what happens? Does the tour break, does it try to find missing elements, or does it gracefully stop?

20. Where in the React component tree does your `<OnboardingTour>` get rendered? Is it inside the page component itself, or in a layout wrapper? What are the implications of each choice for the cross-page welcome wizard tour?

21. You call the API on every page load to check `should_start`. For a user who has completed all tours, this is 8+ API calls per session that all return `completed`. How would you optimize this?

---

## Timeline Questions

22. Count the page tours in your timeline: you have 8 tours across Weeks 6–10 plus the connect hub, welcome modal, backend, and documentation. At ~5 hours per tour that's 40 hours for tours alone. Please remove at least 3 tours from the core scope and list them as stretch goals. Which ones would you cut first and why?

23. The welcome modal (Week 3) links to the connect hub page, but the connect hub is not built until Weeks 4–5. The modal has an "Explore more options" link that leads to a page that doesn't exist yet. How do you handle this?

24. Testing is scheduled only in Week 11. If something breaks during testing, you have one week left to fix it. How would you restructure this?

25. Week 8 says "touring all the options in the Explore page." The Explore page has 8 tools. Is this a tour of the Explore index only, or does it enter each sub-tool? List exactly which pages are covered in that week.

---

## Follow-up Responses (Candidate Replies)

---

### Q: What advantage does saving tour state in the database give you that localStorage does not?

**Verdict: Accepted.**

Cross-device persistence is a legitimate reason for DB storage — a user logging in from a
new browser or device shouldn't see tours they already completed. The point about localStorage
being clearable by the user causing tours to reappear is also valid. The re-onboarding feature
from Settings genuinely benefits from DB storage since the user may trigger it from any device.
No issues with this answer.

---

### Q: How does Joyride move from `/user/:username/` to `/user/:username/stats/` mid-tour?

**Verdict: Good design. Better than the original.**

The candidate dropped cross-page Joyride navigation entirely and replaced it with a CTA at the
end of the dashboard tour ("Want to see your stats?"). Clicking it fires a POST to mark the
dashboard tour complete, navigates to the stats page, and the stats tour starts automatically
because `should_start` is checked on mount via the React Query cache.

This avoids the fragile `navigate()` + polling approach and is more UX-friendly. The tours
read as sequential to the user without Joyride needing to cross page boundaries. Good call.

---

### Q: What happens if Joyride tries to find the target element before the page has mounted?

**Verdict: Solid. One thing to verify.**

The `readyTour` local state that gates the tour on both data settlement and DOM presence is the
right pattern. `EVENTS.TARGET_NOT_FOUND` handling to skip non-critical steps is correct Joyride
API usage.

**Flag:** The candidate mentions `targetWaitTimeout` as a Joyride config option for polling
until a non-critical element renders. Verify this prop exists in the version of React Joyride
they are using before writing it into the implementation. It is not a documented standard prop
in Joyride v2/v3 — they may need to implement this polling manually.

---

### Q: Why does the GET endpoint do an INSERT before reading the status?

**Verdict: Fixed correctly. The new design is better.**

The candidate removed the INSERT from the GET endpoint entirely. The new approach is a single
bulk GET on login that returns a JSON object mapping all `page_key → status` pairs. If a key
is missing from the response, the UI defaults to `not_started` locally. This also resolves the
optimization question (8+ calls per session) in one change.

---

### Q: A user navigates away mid-tour. What happens?

**Verdict: Intentional design. Should be documented clearly in the proposal.**

No `STATUS.FINISHED` or `STATUS.SKIPPED` fires on mid-navigation, so no POST is sent, and the
tour status remains `not_started`. On the next visit to that page the tour restarts from the
beginning. This is a deliberate UX decision — only explicit skip or finish triggers a POST.

This is reasonable and simple. The proposal should state this behavior explicitly so it is not
mistaken for a bug during review.

---

### Q: You call the API on every page load to check `should_start` — 8+ calls per session for users who have completed all tours. How do you optimize this?

**Verdict: Correct and clean.**

Single bulk GET on login fetches all tour statuses as one JSON object, cached in React Query
with a large `staleTime`. Individual page mounts check the React Query cache instead of making
API calls. When a tour completes, both the cache and the DB are updated via a POST. If the
cache is evicted, the DB copy ensures the next bulk fetch is still correct.

This directly resolves the concern. No issues.

---

### Summary

| Question | Answer quality | Action needed |
|----------|---------------|---------------|
| DB vs localStorage | Accepted — cross-device argument is valid | None |
| Cross-page tour navigation | Better design than original — CTA-based handoff | Update proposal to reflect new approach |
| Async loading race condition | Solid — `readyTour` gate is correct | Verify `targetWaitTimeout` is a real Joyride prop |
| GET endpoint INSERT | Fixed — bulk fetch on login is clean | None |
| Mid-tour navigation | Intentional design | Document this behavior explicitly in the proposal |
| API call optimization | Correct — React Query bulk cache | None |

---

## Updated Proposal Review

The candidate incorporated the follow-up feedback into a revised proposal. Assessment of what changed and what did not.

---

### What Improved

**Architecture decisions section added.** The candidate now explicitly documents the rationale behind key choices (DB vs localStorage, React Query caching, CTA-based multi-page handoff, mid-tour abandonment behavior). This section did not exist before. It reads as genuine understanding — the explanations match the code they wrote and the reasoning they gave in follow-up.

**GET endpoint fixed correctly.** The original GET did an `INSERT ... ON CONFLICT DO NOTHING` before reading. The new design is a single bulk `SELECT` on login returning a `{page_key: status}` map. Missing keys default to `not_started` on the frontend. This is the right fix.

**`readyTour` prop added.** The `OnboardingTourProps` now includes `readyTour: boolean`, and the tour start condition is `runTour = shouldStart && readyTour`. This gates the Joyride instance on both the hook check and the page's own data-readiness signal. Correct pattern.

**Tests added.** Each week's deliverable now lists a test milestone. This is a structural improvement even if the test descriptions are high-level.

**Multi-page tour replaced with CTA.** The welcome wizard no longer attempts to use Joyride across page boundaries. Instead the dashboard tour ends with a "View your stats →" button that navigates manually. The Stats tour auto-starts because React Query detects `not_started` on mount. Clean design.

**Mid-tour abandonment documented.** The proposal now explicitly states: only `STATUS.FINISHED` or `STATUS.SKIPPED` triggers a POST. Browser close or manual navigation does not save state. Tour restarts from step 1 on the next visit. This is correctly labeled as intentional behavior, not a bug.

---

### Persistent Issues

These were flagged before. They are still present in the updated proposal.

**1. Wrong Joyride prop: `onEvent` → `callback`**

```tsx
<Joyride
  steps={steps}
  run={runTour}
  onEvent={handleEvent}   ← WRONG
  ...
/>
```

The correct prop is `callback`, not `onEvent`. This is a fundamental API error. The `CallBackProps` type the candidate imports is the right type — it is just wired to the wrong prop name. A candidate who ran this code would see it silently do nothing when tour steps fire.

**2. POST route URL has a missing `<`**

```python
@user_settings_api_bp.post("/onboarding/string:page_key>/complete")
```

Should be:

```python
@user_settings_api_bp.post("/onboarding/<string:page_key>/complete")
```

The opening `<` is missing. Flask will not parse `page_key` as a variable — it will treat the entire string literally and every POST will 404.

**3. `%S` uppercase in SQL parameter**

```python
cursor.execute("""
    INSERT INTO user_onboarding_status ...
    VALUES (%S, %S, %S, %S)
""", (user_id, page_key, "completed", datetime.utcnow()))
```

`%S` is not a valid psycopg2 placeholder. It must be `%s` (lowercase). This will raise a `psycopg2.ProgrammingError` at runtime on every POST.

**4. `py` typo in INSERT values**

```python
(user_id, py, status, updated_at)
```

`py` is an undefined variable. Should be `page_key`. This would raise a `NameError` immediately.

**5. `options.buttons` is not a valid Joyride prop**

```tsx
options={{ buttons: ["back", "close", "primary"] }}
```

Joyride's `floaterProps` and `styles` are valid nested config objects, but `buttons` is not a recognized key in Joyride's `options` prop. This silently does nothing. If the goal is to control button visibility, the candidate needs to look at the `locale` prop (button text) and `showSkipButton` / `showBackButton` boolean props.

**6. `targetWaitTimeout` is not a standard Joyride prop**

The candidate mentions `targetWaitTimeout` as a Joyride config option for polling until a DOM element is present. This prop does not exist in Joyride v2 or v3. If the candidate intends to use it, they need to implement element polling manually (a `setInterval` loop checking `document.querySelector`) rather than relying on a built-in that does not exist.

**7. Connect hub route is still in `/settings/`**

The connect hub is placed at `/settings/connect-hub` in the proposal. The welcome modal in Week 3 links to it. But Week 3 comes before Weeks 4–5 when the connect hub is built. More importantly, a connect hub should be a public-facing page (like `/connect/` or part of the onboarding route group), not a settings subpage. Settings pages require login and have a different visual context.

---

### Summary

| Issue | Status |
|-------|--------|
| `onEvent` should be `callback` | **Still present** |
| POST route missing `<` | **Still present** |
| `%S` uppercase placeholder | **Still present** |
| `py` undefined variable | **Still present** |
| `options.buttons` invalid | **Still present** |
| `targetWaitTimeout` unverified | **Still present** |
| Connect hub in wrong location | **Still present** |
| GET endpoint INSERT | Fixed |
| Cross-page tour navigation | Fixed (CTA approach) |
| API call optimization | Fixed (React Query bulk cache) |
| Mid-tour abandonment behavior | Now documented |
| `readyTour` gate | Added |

The architecture improvements are real and reflect genuine understanding. The persistent code bugs — particularly the wrong Joyride prop name, the Flask route syntax error, and the psycopg2 placeholder — are the kind of errors that would be caught immediately by running the code. Their continued presence across revisions is a concern.

---

## External Review Notes

Additional issues raised on the updated proposal, with assessment.

---

### Schema & Data Model

**Status enum is too binary.**

The `not_started` / `completed` two-value enum loses information the proposal actually needs. A user who abandons halfway and returns on a new device restarts from zero with no record of prior progress. At minimum, a `dismissed` value is needed to distinguish an intentional skip from a full completion — these are different user behaviors and treating them identically loses signal about which tours are working. Columns like `steps_seen` (int array), `last_step` (int), and `dismissed_at` (timestamptz) would support resume and analytics, but that is scope expansion beyond 90 hours. The `dismissed` enum value alone is a low-cost, high-value fix.

**No version column.**

Re-onboarding from settings currently requires manually flipping status to `not_started`. More importantly, when ListenBrainz ships a new major feature, there is no mechanism to force-show an updated tour only to users who completed the old version. A single `version` integer column on the table, compared against a `CURRENT_TOUR_VERSION` constant in the frontend, would solve this permanently. This is a valid long-term concern but is premature for a first implementation — it should be noted as a future extension rather than built now.

**Beacon dismissal is untracked.**

If the pulsating beacon on Explore sub-pages is still in scope, its dismissed state is stored nowhere. Falling back to localStorage for beacon state directly contradicts the rationale for using DB storage. Beacon state per page needs its own `page_key` row or a dedicated boolean column. **Note:** the updated proposal dropped cross-page Joyride in favor of the CTA handoff — if beacons are gone entirely, this concern does not apply. Confirm with the candidate whether beacons are still in scope.

**Root cause of all DB schema issues:** These complications stem from choosing DB storage for tour state. A localStorage-only design sidesteps the enum, version, and beacon tracking concerns entirely, and is sufficient for a 90h Easy scope project.

---

### API & Backend

**Flask route URL typo (already flagged — still unfixed).**

```python
@user_settings_api_bp.post("/onboarding/string:page_key>/complete")
```

Missing the opening `<`. Flask will not parse `page_key` as a variable. Every POST returns 404. Must be:

```python
@user_settings_api_bp.post("/onboarding/<string:page_key>/complete")
```

---

### Frontend & Joyride Integration

**`onEvent` is not the correct Joyride v3 API (already flagged — still unfixed).**

```tsx
<Joyride onEvent={handleEvent} ... />
```

The v3 API uses `callback`, not `onEvent`. This will silently do nothing — `markComplete` never fires and the entire persistence layer breaks.

**`markComplete` fires on `FINISHED` and `SKIPPED` indiscriminately.**

Both terminal statuses write the same DB row with no distinction. A user who completes all steps is engaged; a user who skips on step 2 may be confused or the tour is poorly targeted. Treating them identically loses signal. At minimum, store the terminal status (`completed` vs `dismissed`) separately — which is also addressed by adding a `dismissed` value to the enum above.

---

### Multi-Page Tour Handoff

**Cache warm assumption is a race condition.**

The CTA-based handoff from the dashboard tour to the stats page relies on React Query already having fetched `not_started` for the stats `page_key`. This works with a warm cache but fails on a new session or after cache eviction: there is a race between `GET /onboarding/statuses` resolving and the stats page mounting. The tour either does not fire or fires before the DOM is ready.

Fix: pass a `?resume_tour=welcome` query param when the CTA navigates. The stats page reads this param synchronously on mount — independently of cache state — and starts the tour immediately. This is a URL-level signal that survives cache eviction, new sessions, and slow connections.

---

### Assessment

| Issue | Severity | Status |
|-------|----------|--------|
| `onEvent` → `callback` | Blocker | Still unfixed |
| Flask route missing `<` | Blocker | Still unfixed |
| `dismissed` enum value | Should fix | Not in proposal |
| `markComplete` FINISHED/SKIPPED distinction | Should fix | Not in proposal |
| Cache race on tour handoff | Should fix | Not in proposal |
| `steps_seen` / resume support | Scope expansion | Out of scope for 90h |
| Version column | Future extension | Out of scope for now |
| Beacon dismissal tracking | Depends on scope | Verify if beacons still exist |
