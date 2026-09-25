---
name: mockup-app
description: Create visual mobile mockup screens for the Znakomstva dating app. Use when asked to mockup, preview, design, sketch, or visualize any app screen before writing React Native code. Generates interactive HTML phone-frame previews as Artifacts — zero dependencies, no external services, no API keys.
---

# Mockup App — Znakomstva Dating App

Create interactive phone-frame mockups as HTML Artifacts for the Znakomstva React Native + Expo dating app.

## When to Use

- User says "mockup", "preview", "visualize", "покажи экран", "нарисуй экран", "design this screen"
- Planning a new screen before writing any React Native / Expo code
- Reviewing screen layout and UX flow
- Comparing layout options visually

## What This Skill Does NOT Do

- Does NOT write React Native code
- Does NOT create Expo components
- Does NOT modify any production files
- Does NOT call external APIs
- Does NOT send project data anywhere

---

## Design System Tokens

Use these consistently across every screen mockup.

### Colors
```
--c-bg:          #0F0F1A   /* page background */
--c-surface:     #1A1A2E   /* card background */
--c-surface-2:   #252540   /* elevated card */
--c-primary:     #E8395D   /* brand pink / CTA */
--c-primary-dk:  #C02E4C   /* pressed state */
--c-secondary:   #FF6B6B   /* coral accent */
--c-gold:        #FFD93D   /* super-like / premium */
--c-text:        #FFFFFF
--c-text-muted:  #9090A8
--c-border:      #2A2A45
--c-online:      #4CAF50
--c-danger:      #F44336
```

### Typography (Inter only)
```
--t-xs:   11px / 400
--t-sm:   13px / 400
--t-base: 15px / 400
--t-lg:   17px / 600
--t-xl:   20px / 700
--t-2xl:  24px / 800
--t-3xl:  30px / 800
```

### Spacing scale
```
4 / 8 / 12 / 16 / 20 / 24 / 32 / 48px
```

---

## Phone Frame Spec

- Device: iPhone 15 Pro equivalent — 390 × 844 px logical
- Top safe area / notch: 54 px
- Bottom safe area: 34 px
- Tab bar height: 64 px
- Scale on desktop view: 0.65×
- Always dark theme — background body: #060611

---

## Workflow

### Step 1 — Understand the Screen

Identify:
- Screen name and route (e.g. `DiscoverScreen`, `ChatListScreen`)
- User's goal on this screen
- Key UI elements (cards, buttons, inputs, lists, modals)
- Navigation type: tab root / stack push / modal / bottom sheet

### Step 2 — Load `artifact-design` skill

Before writing any HTML, call `Skill("artifact-design")` to follow page design rules.

### Step 3 — Write the HTML Mockup

Every mockup MUST use this outer shell:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>[ScreenName] — Znakomstva</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
  <style>
    /* CSS tokens, phone frame, status bar, tab bar, content */
  </style>
</head>
<body>
  <div class="scene">
    <div class="device">
      <div class="notch"></div>
      <div class="status-bar">
        <span class="time">9:41</span>
        <div class="status-icons">
          <svg><!-- signal --></svg>
          <svg><!-- wifi --></svg>
          <svg><!-- battery --></svg>
        </div>
      </div>

      <div class="screen">
        <!-- SCREEN CONTENT HERE -->
      </div>

      <div class="tab-bar">
        <!-- 4-5 tab items with icon + label -->
      </div>
    </div>
    <p class="screen-label">[ScreenName]</p>
  </div>
</body>
</html>
```

### Step 4 — Publish as Artifact

```
Artifact(action="publish",
  file_path="...",
  icon="phone",
  title="[ScreenName] — Znakomstva Mockup",
  description="[one sentence about the screen]"
)
```

---

## Screen Catalog — Templates

### 1. Discover (Swipe)
- Full-screen profile card, 3:4 photo ratio, gradient overlay (bottom 40%)
- Name, age, city overlay on photo
- 3 action buttons row: ✕ dislike (white), ★ super-like (gold), ♥ like (pink)
- Top bar: logo + filter icon + notifications bell
- Active tab: Discover (flame icon)

### 2. Match Modal (Overlay)
- Dark semi-transparent overlay over previous screen
- "Это совпадение! 🎉" heading
- Two circular avatars overlapping (animate with CSS pulse)
- "Написать сообщение" — primary CTA button
- "Продолжить свайпать" — ghost link

### 3. Messages (Chat List)
- List of conversations: avatar (60×60, rounded) + name + last message preview + time
- Online dot (green, 10px) on active users
- Unread count badge (pink pill) if unread
- Search bar at top
- Active tab: Messages (chat bubble icon)

### 4. Chat (1-on-1)
- Header: back arrow + avatar + name + online status
- Message bubbles: sent = right, `--c-primary` bg; received = left, `--c-surface-2` bg
- Input bar pinned above tab bar: text field + send button
- Timestamps between message groups

### 5. My Profile
- Header gradient or cover photo (200px tall)
- Avatar (96px) overlapping header at bottom-left, with camera edit icon
- Name, age, city
- Row: Likes count | Matches count | Views count
- Photo grid (2 columns)
- Bio text section
- Active tab: Profile (person icon)

### 6. Settings / Edit Profile
- Stack screen (no tab bar, or tab bar visible)
- Form rows: name, age, bio, location, preferences
- Save button (sticky at bottom)

### 7. Onboarding / Welcome
- No tab bar
- Full-screen gradient or photo background
- App logo + tagline
- "Войти через Google" + "Войти через Apple" buttons
- "Зарегистрироваться" link

---

## Multiple Screens

**Option A — Separate Artifacts:** Publish each screen separately (preferred for review).

**Option B — Tabbed Artifact:** Single HTML with `<nav>` buttons at top switching between screens via JS — use when user asks for a "flow" or "walkthrough".

---

## Rules

1. Phone frame is ALWAYS visible — never publish a bare HTML page without it.
2. Always dark theme (the app is dark-only at MVP stage).
3. No Tailwind CDN, Bootstrap, or other CSS frameworks — inline `<style>` only.
4. External scripts: cdnjs.cloudflare.com or cdn.jsdelivr.net/npm/ only.
5. External fonts: Google Fonts only (Inter).
6. No placeholder `<img src="https://...external...">` — use CSS gradients for photos.
7. Keep mockup static by default; simple CSS hover/active states are OK.
8. Never write React Native or Expo code from this skill.

---

## Wrap Up

After each published mockup, tell the user:

1. **Screen:** Name and what it represents
2. **Link:** Artifact URL
3. **RN screen:** Which React Native screen file this will become (`src/screens/...`)
4. **Notes:** Any UX suggestions or questions to resolve before coding
