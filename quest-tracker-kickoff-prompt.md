# Claude Code Kickoff Prompt — Family Quest Tracker

> Paste everything below the line into Claude Code. It is written as a complete spec for **Session 1 (a working MVP)**, with Sessions 2–3 sketched at the end so you can stop after the MVP and resume later.

---

## Role & goal

You are building a **single-file, offline, touch-friendly household reward tracker** called **Quest Tracker**. It is the digital companion to a printed/laminated "Family Quest Board" already in use. It runs on one shared device in the kitchen (phone or tablet, used mostly in portrait), with no internet, no backend, and no accounts.

The two users are kids: **Nathan (8)** and **Paxton (5)**. A parent owns the device. The whole point of the tool is to make cooperation feel like a game the kids are *winning*, and to take the day-to-day point-counting and allowance math off the parent's hands.

## Hard constraints

- **One file:** `quest-tracker.html`. Everything inline — HTML, CSS, vanilla JS. **No frameworks, no build step, no npm, no bundler.**
- External resources allowed: **Google Fonts only** (`Baloo 2` + `Nunito`), with system fallbacks so it still works offline after first load.
- **Persistence: `localStorage`**, one JSON blob under the key `questTracker.v1`. (Persists per-browser on that device.) **Because iOS Safari can evict localStorage** under storage pressure or after long inactivity, also build a **Backup & Restore** (export/import the JSON) into Settings in Session 1 — this protects the kids' real earned balances. Auto-keep a second copy under `questTracker.v1.bak` on every write.
- **Primary target is iPhone (iOS Safari), launched from the Home Screen as a standalone web app.** Must also work fully offline and degrade gracefully in normal Safari and on desktop.
- **Mobile-first, iPhone-first.** Big tap targets (min 44px), thumb-reachable primary actions, no hover-dependent UI.
- Graceful first-run: if no saved state, seed sensible defaults (below) so it's usable immediately.

## Design language (match the printed board exactly)

Reuse these tokens so the app and the laminated charts read as one system.

```
--ink:#1f3a4d   --quest:#2f7d5b   --gold:#f3a712   --coral:#ef6f5c
--sky:#3f86bf   --paper:#fbf7ef   --line:#d8d2c4
soft variants (~12% tints): quest #e4f1ea, gold #fdf0d6, coral #fce4df, sky #e2eef7
```

- Display/headers: **Baloo 2** (rounded, friendly). Body/UI: **Nunito**.
- Nathan's accent = `--quest` (green). Paxton's accent = `--coral`.
- Rounded cards (12–18px radius), chunky friendly buttons, generous spacing. Cheerful but not cluttered — it's used by a 5-year-old.
- **Reward the eyes:** earning a star should *feel good* — a quick star "pop" + the balance number ticking up. Hitting a goal triggers a short celebration (confetti or a level-up burst). Keep animations brief and skippable; respect `prefers-reduced-motion`.

## iPhone / iOS requirements (treat as first-class, not an afterthought)

**Home-Screen install (so it launches like a real app, fullscreen, no Safari chrome):**
- Add the apple meta tags in `<head>`: `apple-mobile-web-app-capable` = `yes` (and the newer `mobile-web-app-capable`), `apple-mobile-web-app-status-bar-style` = `black-translucent`, and a sensible `apple-mobile-web-app-title`.
- Viewport: `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">`.
- Provide an `apple-touch-icon` as an **inline PNG data-URI** (generate a simple quest-shield icon on a `--ink` background, ~180×180) so the file stays self-contained. If a separate `icon.png` + `manifest.json` is acceptable instead, that's fine — but default to inline to preserve single-file.
- Keep everything in-page (no `<a href>` full navigations) so it never breaks out of standalone mode. Detect `navigator.standalone === false` and show a small one-time "Add to Home Screen → tap Share, then Add to Home Screen" hint when running in regular Safari.

**Safe areas & layout:**
- Respect the notch / Dynamic Island / home indicator: pad the app shell with `env(safe-area-inset-top/bottom/left/right)`. The top balance bar and any bottom action bar must not sit under the notch or the home indicator.
- Use a fixed app shell with an internal scroll container; set `overscroll-behavior: none` and prevent body rubber-band scrolling so it feels app-like, not web-page-like.

**Touch feel (kids will mash and long-press everything):**
- Global: `* { -webkit-tap-highlight-color: transparent; -webkit-touch-callout: none; -webkit-user-select: none; user-select: none; }` and `touch-action: manipulation` on interactive elements to kill the 300ms delay and double-tap zoom.
- No accidental text selection or the iOS copy/define callout on buttons and labels.

**Inputs / the parent PIN:**
- Build the PIN entry as a **custom on-screen numeric keypad inside the app**, not a native text input — this avoids the iOS keyboard covering the UI and avoids focus-zoom. If any real `<input>` is used anywhere, its `font-size` must be **≥16px** or iOS will zoom the page on focus.

**Sound (for the Session-2 toggle, but wire the unlock now):**
- iOS blocks audio until a user gesture. Unlock/resume the `AudioContext` on the first tap; only play sounds from within tap handlers. Don't rely on `navigator.vibrate` for haptics — **iOS Safari doesn't support the Vibration API**, so feedback must be visual/audio.

**Testing target:** Safari on a recent iPhone, both in normal Safari and after Add-to-Home-Screen (standalone). It must look right on a notched device in portrait and survive force-quitting and relaunching the home-screen app (state intact).

## Data model

```js
{
  version: 1,
  settings: {
    starValueCents: 25,        // $ per star for cash-out (editable)
    payDay: "Sunday",
    parentPin: "0000",         // gate for shop redeem / cash-out / settings
    soundOn: true
  },
  kids: [
    {
      id: "nathan",
      name: "Nathan",
      level: 8,
      accent: "quest",
      mode: "saver",           // shows money + goal + savings emphasis
      stars: 0,                // current redeemable balance
      lifetimeStars: 0,        // never decreases (for level/progress)
      goal: { label: "", targetStars: 0 },
      dailyQuests: [
        { id, label: "Make my bed", emoji: "🛏️" },
        { id, label: "Clear my plate", emoji: "🍽️" },
        { id, label: "Pack my stuff for tomorrow", emoji: "🎒" },
        { id, label: "Brush teeth (AM & PM)", emoji: "🦷" }
      ],
      doneByDate: { "2025-06-14": ["questId1","questId2"] }  // checked daily quests, per calendar day
    },
    {
      id: "paxton", name: "Paxton", level: 5, accent: "coral",
      mode: "instant",         // emphasizes instant rewards + big visuals, hides money
      stars: 0, lifetimeStars: 0,
      goal: { label: "5 stars = a prize!", targetStars: 5 },
      dailyQuests: [
        { id, label: "Toys in the bin", emoji: "🧸" },
        { id, label: "Plate in the sink", emoji: "🍽️" },
        { id, label: "Get dressed", emoji: "👕" },
        { id, label: "Brush my teeth", emoji: "🦷" }
      ],
      doneByDate: {}
    }
  ],
  bonusQuests: [               // behavior stars — the real point of the app, shared list
    { id, label: "Listened the first time", emoji: "👂", stars: 1 },
    { id, label: "Helped without being asked", emoji: "🤝", stars: 1 },
    { id, label: "Kind to my brother", emoji: "💛", stars: 1 },
    { id, label: "Took an Extra Quest", emoji: "🌟", stars: 2 }
  ],
  rewards: [                   // the shop; scope controls which kid sees it
    { id, label: "Pick tonight's show", cost: 3, scope: "all" },
    { id, label: "Choose a treat", cost: 3, scope: "all" },
    { id, label: "15 min extra play", cost: 4, scope: "all" },
    { id, label: "Movie night, you pick", cost: 10, scope: "all" },
    { id, label: "Have a friend over", cost: 15, scope: "all" }
  ],
  log: [                       // every star/redeem/cashout event, newest first
    { ts, kidId, type: "earn"|"redeem"|"cashout"|"undo", label, delta }
  ]
}
```

## Screens & interactions (Session 1)

**1. Home / kid picker**
- Two large kid cards (Nathan green, Paxton coral), each showing name, a level badge, and current ⭐ balance.
- Tap a card → that kid's board. A small lock icon in the corner opens **Parent** (PIN-gated).

**2. Kid board** (the main screen)
- Big current **⭐ balance** at top. Below it, for `saver` kids, the cash value (`stars × starValueCents`); hidden for `instant` kids.
- **Today's Team Quests:** today's daily quests as big tappable rows. Tapping checks/unchecks (toggles membership in `doneByDate[today]`). **These earn NO stars** — they're "what our team does." Show a subtle "X of N done today" progress.
- **Bonus XP:** one big button per `bonusQuests` item. Tapping awards its stars → star-pop animation, balance ticks up, logged. **Tappable repeatedly** (a kid can listen well twice).
- **Goal tracker:** a fill bar from 0 → `goal.targetStars`, filling with current stars. Completing it fires the celebration and (for Paxton) prompts "Pick your prize!".
- **Reward Shop** button.

**3. Reward Shop**
- Cards for each reward in scope for this kid, with ⭐ cost. "Redeem" is disabled if balance is too low.
- Redeeming is **PIN-gated** (so kids can't drain rewards solo), subtracts stars, logs a `redeem`, and shows a confirmation ("Enjoy your movie night! 🎉").

**4. Cash out** (saver kids only, e.g. Nathan)
- Shows balance × rate = dollars available. "Cash out" (PIN-gated) zeroes some/all stars into a logged `cashout` and shows the amount owed for pay day.

**Core behavioral rules to encode (important — these make it work):**
- **Stars are only ever added by earning.** There is no "remove a star" punishment button anywhere. Stars decrease only via the kid's own positive choice to redeem/cash out. (Undo exists for *mistakes*, not discipline.)
- **Fresh start daily:** daily-quest checkmarks are scoped to the calendar date and reset visually each new day. No stre***-shaming or carried-over guilt.
- **Undo:** every earn/redeem/cashout shows a brief "Undo" toast (~5s) that cleanly reverses the action and balances. Kids mis-tap constantly.
- **Per-kid mode:** `instant` (Paxton) hides money/cash-out and leans on big visuals + the small goal; `saver` (Nathan) surfaces money, goal, and savings.

**Parent area (PIN-gated):** edit kid names/levels/accent, edit each kid's daily quests, edit bonus quests and the reward shop (label + cost + scope), set `starValueCents`, `payDay`, `parentPin`, `soundOn`, **Backup & Restore (export/import the full JSON — copy-to-clipboard and paste-to-restore both work without file pickers, which is important on iOS)**, plus **Reset today** and **Reset all data** (with confirm).

## Acceptance criteria (Session 1 is "done" when)

1. Opening `quest-tracker.html` with no prior data shows both kids seeded and usable.
2. Tapping a bonus button awards stars, animates, updates the balance, and survives a page reload.
3. Daily quests check/uncheck per day and reset on a new calendar date.
4. Shop redeem and Nathan's cash-out work, are PIN-gated, and log correctly.
5. The goal bar fills and celebrates at target.
6. Undo reverses the last action correctly.
7. Parent settings edit kids/quests/rewards/PIN and persist; Backup export/import round-trips the full state.
8. Looks coherent with the printed board (fonts, palette) and is fully usable one-handed on a phone in portrait.
9. **On iPhone:** installs to the Home Screen and launches fullscreen (no Safari bars); content clears the notch and home indicator via safe-area insets; taps have no gray flash, no 300ms delay, and no accidental text-selection/callout; the PIN keypad is the in-app one; state survives force-quitting and relaunching the installed app.

## Build instructions

Build the **complete file in one pass**, then give me a short summary of what's in it and anything you'd want to tighten in Session 2. Keep the JS organized into clear sections (state/persistence, render, actions, parent admin) even though it's one file. Don't ask me clarifying questions unless something is genuinely ambiguous — make a sensible call and note it.

---

## Future sessions (don't build yet)

**Session 2 — history & polish:** an activity log / "this week" view per kid, a printable weekly summary (so you can show them their week), streak counters (encouraging, never punishing), sound effects + a richer level-up celebration, and a quick "Extra Quest jar" draw button (+2 ⭐).

**Session 3 — durability & sync:** JSON export/import for backup, an optional weekly auto-snapshot, a simple progress chart over time, and a "pay day" screen that totals each kid's cash-out since the last pay day so you settle up in one tap.
