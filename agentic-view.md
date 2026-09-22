# Knowledge Transfer: `agentic-view.tsx`

> **File**: [src/component/agent/agentic-view.tsx](src/component/agent/agentic-view.tsx) (~6,250 lines)
> **Audience**: this doc assumes you know basic React (components, `useState`, `useEffect`) and TypeScript, but have never seen this file or this feature before.

---

## 1. What is this file, in one paragraph?

This is the **one giant screen** that powers the "Assess & Modernize" AI-driven workflow in Concierto Modernize. A user picks a portfolio, connects a repo (or a mainframe app), and from there an **AI agent** (a chatbot on the backend) drives them step-by-step through: onboarding → code scan → assessment → recommendations → cost (TCO) calculation → downloading a report → picking a modernization pathway → submitting for approval → handing off to "Transformation Studio".

The AI doesn't just chat — it tells the frontend *what to render next* (a form, a card, a scan, an assessment). This file is the **dispatcher + state machine** that listens to the AI's instructions, calls the right backend APIs, updates screen state, and reports results back to the AI. Almost everything the user sees on this screen — the chat, the side checklist, the little embedded cards (forms, tables, progress bars) — is coordinated from this one component.

There are also two "special" flows baked into the same file:
- **Mainframe applications** have a parallel, mostly-separate set of steps/APIs from normal ("standard") applications.
- **TCO (Total Cost of Ownership)** and **Database onboarding** are each their own mini state-machines living inside this file.

If you remember one thing: **this file is a router between "what the AI said to do" and "what the UI shows / API gets called."**

---

## 2. Where does it fit on screen?

Rendered top to bottom (see JSX around [agentic-view.tsx:6090-6236](src/component/agent/agentic-view.tsx#L6090-L6236)):

```
┌─────────────────────────────────────────────┐
│               AgentFlowHeader                │  ← app/report summary, download buttons
├───────────────┬───────────────────────────────┤
│  LeftPanel    │           ChatPanel            │  ← chat messages + inline "panel cards"
│ (milestones/  │  (conversation + forms/tables  │     (e.g. select repository, scan
│  checklist)   │   embedded as chat cards)      │      progress, recommendation list...)
└───────────────┴───────────────────────────────┘
   ⬑ collapsible via chevron button
```

Full-screen overlays (questionnaire popups, portfolio-level TCO popup) can appear on top of the chat panel.

The outer exported component `AgenticView` ([agentic-view.tsx:6239-6247](src/component/agent/agentic-view.tsx#L6239-L6247)) is just a thin wrapper that can **fully reset** the screen by remounting the real component (`AgenticViewInner`) with a new React `key` — this is how "start over" works without a page reload.

---

## 3. Glossary — things you'll see everywhere

| Term | Meaning |
|---|---|
| **Milestone (M1–M6)** | The 6 big phases of the standard-application journey. Drives the left panel checklist. |
| **`form_key`** | A string the AI sends back telling the frontend which form/panel to show next (e.g. `'select-existing-portfolio'`). Matched in a big `switch` — see §6. |
| **Panel / panel card** | An embedded UI widget shown *inside* the chat conversation (a form, a table, a progress bar), not a separate page. |
| **"Frozen" panel/props** | A snapshot of a panel's props taken at the moment it's completed, so old chat history doesn't visually change when live state later changes. |
| **Standard vs Mainframe** | Two parallel flows. Controlled by `selectedApplicationType`. Almost every feature (onboarding, scan, assessment, recommendations) has two versions — always check which branch you're in. |
| **TCO** | Total Cost of Ownership — the cost-estimation sub-flow, with its own nested panel state machine. |
| **Silent chat** | A message sent to the AI *without* showing a chat bubble to the user — used to feed backend results back to the AI behind the scenes. |
| **Ref mirroring state** | A `useRef` that's kept in sync with a `useState` value, used to read the *latest* value inside callbacks/timers that would otherwise see stale data (see §7). |

---

## 4. The big picture: imports & dependencies

You don't need to memorize these, but knowing the *categories* helps you navigate:

- **Redux**: global loading spinner, questionnaire state, selected portfolio, app details (`react-redux`, `../../redux`).
- **RTK Query hooks** (the majority of imports): one hook per backend endpoint, grouped by domain — `services/administration`, `services/agent` (the AI chat itself), `services/applications`, `services/assessment`, `services/database`, `services/recommendation`, `services/studio`, `services/tco`.
- **Local "agent" folder modules** (same directory as this file): session persistence helpers, the panel registry/context, chat message types, sub-flow helpers for optional settings / recommendations / TCO, and the child components (`ChatPanel`, `LeftPanel`, `AgentFlowHeader`, `ReviewQuestionnairePanel`).

**Tip**: if you need to understand a specific sub-flow, it's usually easier to open its dedicated helper file (e.g. `tco/*`, `db-mapping-utils`, `panel-registry`) than to find it inline in this file.

---

## 5. State — what's actually being tracked

This component has **90+ pieces of `useState`**. Don't try to memorize them all — instead know the *groups*:

1. **Portfolio/search/filters** — search box, pagination, selected portfolio.
2. **Conversation core** — `conversationMessages` (the chat log), `threadId` (AI conversation id), `disableChat`, `pendingTool` (when the AI is waiting on a yes/no approval).
3. **Onboarding** — selected repository, application, database, application type (mainframe/standard).
4. **Scan flow** — scanning progress, activity log, per-component results.
5. **Assessment flow** — assessment id/status, report data, document generation flag.
6. **Milestone/progress booleans** — one flag per completed step (`isScanDone`, `isAssessmentDone`, etc.) — these drive the left panel checklist.
7. **Recommendations/pathway** — recommendation list, selected pathway, submit-for-approval details.
8. **TCO flow** — active TCO panel, selected cloud/tier, generated reports.
9. **Optional settings sub-flow** — workflow/user mapping, approvers, uploaded infra.

**Important quirk**: `conversationMessages` is *not* set with the raw `useState` setter directly — there's a custom wrapper around it ([agentic-view.tsx:287-298](src/component/agent/agentic-view.tsx#L287-L298)) that auto-tags every message with the current milestone and keeps a ref mirror in sync. **Always use that wrapper when adding messages**, or milestone tagging silently breaks.

---

## 6. The core dispatcher: `handleChatResponse`

This is **the single most important function in the file**: [agentic-view.tsx:2511-3178](src/component/agent/agentic-view.tsx#L2511-L3178).

Every time the AI responds (via the streaming chat API), this function looks at `res.type` and `res.form_key` and decides what to do — open a specific panel, call a specific API, or start a scan/assessment. It's a giant `switch` over ~35 `form_key` values.

**If you're asked to add a new AI-driven step**, you will very likely need to touch **three places** together:
1. A new `case` in `handleChatResponse`'s switch ([agentic-view.tsx:2511](src/component/agent/agentic-view.tsx#L2511)).
2. A matching `case` in `buildPanelPropsRef.current`'s switch ([agentic-view.tsx:4593-5244](src/component/agent/agentic-view.tsx#L4593-L5244)), which supplies the props for that panel.
3. Usually also an entry in the panel registry / `RightPanelType` union (in `./type` and `panel-registry`).

Forgetting one of these three is the most common way this kind of feature goes wrong.

---

## 7. Patterns you must understand before editing this file

### a) Refs that mirror state (stale-closure workaround)
You'll see pairs like `threadId` / `threadIdRef`, `selectedPortfolio` / `selectedPortfolioRef`. This exists because functions passed into `streamAgentChat`'s callbacks, or code running inside `setInterval` polling loops, are created once and can "see" old state values forever (a classic React closure trap). The fix used throughout: keep a `useRef` in sync with the state, and read `ref.current` instead of the state variable inside any async/interval/callback code.

**Rule of thumb**: if you're inside a polling loop or a stream callback and need the "current" value of something, look for a `xxxRef` before assuming the plain state variable is safe to read.

### b) Refs holding functions (forward references)
Some handlers need to call each other, but can't reference a function defined later in the same component body. The fix: store the latest function in a ref right after defining it (e.g. `handleChatResponseRef.current = handleChatResponse`), and call `xxxRef.current(...)` from the other place. You'll see this a lot — it's intentional, not a mistake.

### c) Frozen panel snapshots
Once a panel card is "completed" (e.g. a form was submitted), it stays visible in the chat history — but the live state it was built from keeps changing for the *next* step. So the code "freezes" a snapshot of the props at completion time (`buildFrozenPropsForActivePanel`, `closeChatPanel`) so old cards don't visually change later. This dual live-vs-frozen representation is probably the trickiest concept in the file — take time to understand it before touching panel-closing logic.

### d) Magic strings, not enums
`form_key` values, panel types, and even some structured data sent back to the AI (e.g. a chat message with a `\nconfig:{...}` suffix appended to plain text) are just string literals matched in switches — there's no central enum. When searching for "where does X panel get triggered," search for its string literal (e.g. `'tco-basic-estimate'`), not a type name.

### e) Standard vs. Mainframe branches
Nearly every flow has two versions. Before assuming a single code path, check whether the surrounding code branches on `selectedApplicationType === 'mainframe'`.

### f) Errors mostly fail silently
Most `.catch()` blocks just reset to an empty array/default and log to the console — there's rarely a dedicated error UI. When something "doesn't work," check the browser console/network tab, not the screen.

### g) A few things to *not* assume are wired up
`jobSpecificTaskMappings` and `baseTaskTypeToMessageAndView` ([agentic-view.tsx:1984-2009](src/component/agent/agentic-view.tsx#L1984-L2009)) are declared but never populated — looks like dead/legacy code. Don't build on top of them without checking with the team first.

---

## 8. Network calls & polling

The AI chat itself streams via `streamAgentChat` (from `services/agent/stream-chat`): push a placeholder "typing" message → stream text into it as it arrives → on completion, call `handleChatResponse` with the parsed result.

Several flows use `setInterval` to poll a REST endpoint every few seconds until a status flips to done/failed (all cleaned up on unmount to avoid leaked timers):
- Scan progress (~5s)
- Mainframe scan progress (~2s, uses a raw fetch instead of RTK Query)
- Assessment status (~5s)
- AI artifact/document generation (~5s)
- Per-database assessment progress (~5s, one interval per database id)

If you're debugging "why did the UI not update," check whether the relevant polling interval is still running and hasn't been accidentally cleared or duplicated.

---

## 9. Suggested first steps as a fresher

1. Run the app locally and walk through the flow yourself once (create a portfolio → connect a repo → scan → assess → recommend → TCO → download) so you have a mental model of the *user-visible* steps before reading code.
2. Read §6 (`handleChatResponse`) and §7 (patterns) first — they explain 80% of "why is this written this way."
3. Pick **one** flow (e.g. the scan flow: `handleLaunchScan` → `startScanPolling` → the `form_key` case that opens the scan panel) and trace it start to finish, rather than trying to read the file top-to-bottom.
4. When you need to add or change behavior, search for the relevant `form_key` string literal first — it'll lead you to all three places it needs to change (dispatcher, panel-props builder, registry).
5. Ask a senior dev before touching the ref-mirroring or frozen-props logic — it's easy to reintroduce stale-closure bugs that only show up intermittently.

---

*This document is a guided overview, not exhaustive documentation — line numbers refer to the file as of the time this doc was written and may drift as the file changes.*
