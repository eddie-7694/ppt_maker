---
name: ppt-maker
description: Create a beautiful, self-contained HTML presentation (like PowerPoint/Keynote, but runs in the browser with zero dependencies). Use this skill whenever the user asks to make slides, a presentation, a deck, or wants to present information visually — even if they say "PPT", "Keynote", "slideshow", or just "make slides about X". The output is a single .html file with professional design, smooth animations, keyboard navigation (arrow keys + F for fullscreen), and slide numbers on every slide.
---

# ppt-maker

Generate a polished, single-file HTML/CSS/JS presentation from a topic or outline. No frameworks, no CDN links, no external dependencies — everything is inline.

## What you produce

A single `.html` file the user can open in any browser and present immediately. It looks and feels like a real slide deck: full-viewport 16:9 layout, smooth slide transitions, keyboard navigation, and a professional visual design.

## Core architecture (always use this structure)

```
body
├── .progress-bar-container  (fixed, top:0, full width, 3-4px)
├── .stage                   (fixed inset:0, flex center, dark bg)
│   └── .deck                (aspect-ratio:16/9, width:min(100vw,177.78vh), position:relative)
│       └── .slide × N       (position:absolute; inset:0 — stacked, only .active is visible)
├── .slide-counter           (fixed, bottom-right, shows "N / Total" — always visible)
├── .thumbnail-nav           (fixed, bottom-center, dot buttons)
└── <script>                 (Deck class with all logic inline)
```

**Why this layout:**
- Stage uses flex centering on the full viewport. Deck uses `min(100vw, 177.78vh)` so the 16:9 ratio is always preserved with letterbox/pillarbox on unusual screens. Slides use absolute stacking so CSS transitions just move them in/out without reflow.
- Slide counter is a body-level fixed element (not inside .deck) so CSS transforms on slides never affect it.

## Navigation requirements (always implement all of these)

- **Arrow keys** — `ArrowRight`/`ArrowDown`/`Space`/`PageDown` → next; `ArrowLeft`/`ArrowUp`/`PageUp` → prev
- **`F` key** — toggle fullscreen via `document.documentElement.requestFullscreen()` / `document.exitFullscreen()`
- **`Home`/`End`** — jump to first/last slide
- **Click/swipe** — pointer events on .stage (50px threshold, 500ms max)
- **Prev/Next buttons** — `‹` `›` fixed on left/right edges

Block key repeat (`if (e.repeat) return`). Prevent default on handled keys.

## Slide counter (required on every slide)

Place a fixed `.slide-counter` element as a direct child of `<body>` (never inside `.deck`):

```html
<div class="slide-counter" id="slideCounter" aria-live="polite">1 / N</div>
```

```css
.slide-counter {
  position: fixed;
  bottom: 20px; right: 24px;
  font-size: clamp(11px, 1.2vw, 14px);
  color: rgba(255,255,255,0.65);
  font-variant-numeric: tabular-nums;
  user-select: none;
  z-index: 150;
  font-family: inherit;
}
```

Update it in `_updateUI()` with `slideCounter.textContent = \`\${n} / \${this.total}\``.

## Slide transition (GPU-only, no layout thrash)

Only animate `transform` and `opacity` — never `width`, `height`, `top`, `left`.

```css
.slide {
  position: absolute; inset: 0;
  transition: transform 0.45s cubic-bezier(0.25,0.46,0.45,0.94), opacity 0.45s ease;
  transform: translateX(100%); opacity: 0; pointer-events: none;
}
.slide.active  { transform: translateX(0);    opacity: 1; pointer-events: auto; z-index: 1; }
.slide.prev    { transform: translateX(-100%); opacity: 0; }
.slide.next    { transform: translateX(100%);  opacity: 0; }
.slide.no-transition { transition: none !important; }
```

Lock navigation during transitions (`this.locked = true`). Listen for `transitionend` on the `transform` property, with a `setTimeout` fallback to unlock.

## Global visual system — apply to every slide

This is the design DNA. Every slide must use these foundations before choosing a layout.

### Google Fonts
```html
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;700&family=Inter:wght@400;700&family=JetBrains+Mono:wght@400&display=swap" rel="stylesheet">
```

### Design tokens
```css
:root {
  --cyan:  #00E5FF;  --green: #00C875;
  --green-dim: rgba(0,200,117,0.12); --green-border: rgba(0,200,117,0.3);
  --gold: #C6A55C;   --yellow: #FFD600;
  --white: #FFFFFF;  --muted: rgba(255,255,255,0.45);
  --card-bg: rgba(255,255,255,0.05); --card-border: rgba(255,255,255,0.1);
  --red: #FF5555; --red-dim: rgba(255,85,85,0.12); --red-border: rgba(255,85,85,0.3);
}
```

### Background — dark charcoal with vignette (no grid)
NOT pure #000. Use a deep charcoal with corner glow only — no grid lines:
```css
html, body { background: #0a0e14; }
.deck      { background: #0a0e14; }
.slide     { background: transparent; }

/* Corner vignette glow */
.deck::after {
  content: '';
  position: absolute; inset: 0; z-index: 0; pointer-events: none;
  background:
    radial-gradient(ellipse 60% 50% at 0% 0%,   rgba(0,200,117,0.07) 0%, transparent 60%),
    radial-gradient(ellipse 60% 50% at 100% 100%, rgba(0,229,255,0.06) 0%, transparent 60%);
}
/* All slide content must be z-index: 1+ to appear above background */
.slide > * { position: relative; z-index: 1; }
```

### Progress bar — rainbow gradient
```css
.progress-bar-fill {
  background: linear-gradient(90deg, #0070F3, #00E5FF, #00C875);
}
```

### Section pill badge — centered, replaces blinking label
Every slide gets a pill badge instead of the old `slide-label`:
```html
<div class="section-badge">섹션 2 · PLAN APPROVAL</div>
```
```css
.section-badge {
  display: inline-flex; align-items: center; padding: 0.3em 1em;
  border: 1px solid rgba(0,229,255,0.35); border-radius: 100px;
  font-family: 'JetBrains Mono', monospace;
  font-size: clamp(0.65rem, 1.1vw, 0.95rem);
  letter-spacing: 0.14em; color: rgba(255,255,255,0.65);
  background: rgba(0,229,255,0.05); margin-bottom: 0.9em;
}
```

### Title with mixed-color keyword
The most distinctive pattern: key words get a cyan→green gradient, the rest stays white.
```html
<h2>계획 먼저, <span class="hl">코딩 나중</span></h2>
<h2><span class="hl">TeammateIdle</span> vs <span class="hl-b">TaskCompleted</span></h2>
```
```css
h2 { font-size: clamp(1.6rem, 3.8vw, 3.8rem); font-weight: 700; line-height: 1.15; letter-spacing: -0.02em; color: var(--white); margin-bottom: 0.35em; word-break: keep-all; }
/* cyan→green gradient highlight */
.hl   { background: linear-gradient(90deg, var(--cyan), var(--green)); -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text; }
/* alternate: pure cyan for second term in A vs B */
.hl-b { color: var(--cyan); }
/* gold/yellow for warning emphasis */
.hl-y { color: var(--yellow); }
```

### Subtitle with underline accent
```html
<p class="subtitle">에이전트 팀의 토큰 사용량은 활성 팀원 수에 비례해서 증가합니다.</p>
```
```css
.subtitle {
  font-size: clamp(0.85rem, 1.55vw, 1.35rem); color: rgba(255,255,255,0.65);
  line-height: 1.65; margin-bottom: 1.2em;
  border-bottom: 2px solid var(--yellow); /* accent underline — use sparingly */
  display: inline-block; padding-bottom: 0.2em;
}
/* Without underline: just omit the border-bottom */
```

### Callout / Note box — use sparingly, only when genuinely needed
**When to use:** Only when there's a critical warning, surprising fact, or punchline the slide content alone doesn't deliver. The title + cards are usually sufficient — do NOT add a callout just to fill space.

**Width and wrapping:** ~62–65% of slide width. Content wraps naturally to 2 lines — do NOT force single-line. Let the text breathe.

```html
<div class="callout">💡 <strong style="color:var(--green)">exit code</strong>는 스크립트가 끝날 때 보내는 신호 번호. 0 = OK, 2 = 차단 — Claude Code가 이 신호로 다음 동작을 결정합니다.</div>
<div class="callout warning">⚠️ 팀원 3명이면 단일 세션의 <strong style="color:var(--red)">3배 이상</strong> 토큰 소모 + 조율 오버헤드. Opus로 돌리면 5x 플랜도 빠르게 소진됩니다.</div>
```
```css
.callout {
  width: 64%; padding: 0.9em 1.5em; border-radius: 10px;
  background: rgba(255,255,255,0.04); border: 1px solid rgba(255,255,255,0.1);
  font-size: clamp(0.75rem, 1.25vw, 1.1rem); color: rgba(255,255,255,0.75);
  line-height: 1.65; text-align: left; margin-top: 4%; word-break: keep-all;
}
.callout.warning { border-color: rgba(255,85,85,0.3); background: rgba(255,85,85,0.05); }
```

### Citation (인용구 스타일) — callout 대안
Use instead of a callout when the bottom note is a quote, official statement, or attributable source. Left cyan bar + italic text + attribution line.

```html
<div class="citation">
  <div class="citation-text">"각 팀원은 완전히 독립된 Claude Code 세션이다."</div>
  <div class="citation-attr">— 공식 문서</div>
</div>
```
```css
.citation      { width: 56%; margin-top: 4%; padding: 0.8em 1.4em; border-left: 3px solid var(--cyan); text-align: center; }
.citation-text { font-size: clamp(0.82rem, 1.4vw, 1.22rem); color: rgba(255,255,255,0.85); line-height: 1.6; font-style: italic; }
.citation-attr { font-size: clamp(0.62rem, 1vw, 0.88rem); color: var(--green); margin-top: 0.5em; font-style: normal; }
```

### Slide number badge
```css
.slide-num { position: absolute; top: 4%; right: 4%; font-size: clamp(9px,0.78vw,11px); color: rgba(255,255,255,0.18); font-variant-numeric: tabular-nums; }
```

### Hint overlay
```
← → 키 또는 스와이프  |  F: 전체화면  |  S: 강사 멘트
```

## CRITICAL RULE — Script fidelity (read before choosing any layout)

스크립트에서 슬라이드를 만들 때 반드시 지켜야 할 규칙. **레이아웃 선택보다 우선순위가 높다.**

### 1. 헤드카피/서브카피는 원문 그대로 배치한다
| 스크립트 필드 | HTML 위치 | 규칙 |
|---|---|---|
| **헤드카피** | `<h2>` | 원문 텍스트를 그대로. 단어 순서·내용 변경 금지. |
| **서브카피** | `<p class="subtitle">` | 원문 그대로. 레이아웃 카드 제목으로 빼거나 h2로 올리지 않는다. |

**절대 금지:**
- 헤드카피를 누락하고 서브카피를 `h2`로 올리는 것
- 헤드카피/서브카피를 레이아웃 구성 요소(카드 제목, flow 단계명 등)로 분해·재배치하는 것
- 두 필드를 합쳐서 하나의 문장으로 재작성하는 것

### 2. 스크립트에 없는 내용은 절대 추가하지 않는다
- 스크립트에 2개 항목이 있으면 슬라이드에도 2개만 만든다.
- "흐름이 단순해 보인다", "빈 공간이 남는다" 같은 이유로 RESULT 카드, 추가 step, 보충 문구를 창작해서 넣지 않는다.
- 강사 멘트(instructor notes)의 내용은 notes panel 전용이다 — 슬라이드 본문에 옮겨 쓰지 않는다.

### 3. 레이아웃은 콘텐츠를 담는 그릇이지, 콘텐츠를 바꾸는 도구가 아니다
레이아웃을 선택한 뒤 "이 레이아웃에 맞게" 텍스트를 수정하는 순간 규칙 위반이다.
올바른 순서: **원문 텍스트 확정 → 텍스트에 맞는 레이아웃 선택 → 레이아웃 구현.**

---

## Slide layout selection — variety is essential

### 핵심 원칙: 시각적 단조로움 방지

**슬라이드를 만들기 전에 전체 덱을 조감한다.** 아래 체크리스트를 통과한 뒤 구현한다.

#### 레이아웃 다양성 체크리스트 (구현 전 필수)

1. **연속 중복 금지** — 같은 레이아웃을 2장 연속으로 쓰지 않는다.
2. **정렬 변화** — center-aligned 슬라이드가 전체의 70% 이하여야 한다. Split·Steps처럼 좌측 정렬 구조를 반드시 섞는다.
3. **수직/수평 교차** — 가로 나열(Horizontal Flow, Eq+Cards) 다음 슬라이드는 세로 나열(Steps) 또는 단일 임팩트(Big Statement, Stat Callout)로 전환한다.
4. **카드 그리드 연속 금지** — Eq+Cards·VS Card·2×2 Grid·Feature List는 연속으로 배치하지 않는다.
5. **callout 남용 금지** — 전체 슬라이드의 절반 이하에서만 사용한다.
6. **section-badge 생략 허용** — 임팩트 슬라이드(Big Statement, Stat Callout)에서는 생략하거나 위치를 바꾼다.

#### 구조 유형 — 한 덱에서 반드시 섞어 쓴다

| 구조 유형 | 해당 레이아웃 | 권장 비중 |
|---|---|---|
| **임팩트형** (텍스트·숫자 하나가 전부) | Big Statement, Stat Callout, Quote Block | 15–30% |
| **분할형** (좌우·상하 분리) | Split, VS Card, Before/After, Photo Split | 20–35% |
| **나열형** (카드·아이템 여러 개) | Eq+Cards, 2×2 Grid, Feature List, Icon Grid | 20–40% |
| **순서형** (단계·프로세스) | Numbered Steps, Flow Diagram, Horizontal Flow | 15–30% |
| **데이터형** (표·차트) | Comparison Table, Horizontal Bar Chart | 0–20% |

#### 나쁜 패턴 vs 좋은 패턴

```
❌ 슬라이드 2: badge + h2 + subtitle + 카드 3개 + callout  (중앙 정렬)
   슬라이드 3: badge + h2 + subtitle + 카드 3개 + callout  (중앙 정렬)
   슬라이드 4: badge + h2 + subtitle + 카드 2개 + callout  (중앙 정렬)
   슬라이드 5: badge + h2 + subtitle + 카드 2개 + callout  (중앙 정렬)
→ 레이아웃 이름이 달라도 구조·정렬·밀도가 같으면 단조롭다
```

```
✅ 슬라이드 2: Split          — 좌: 대형 키워드 / 우: 설명 리스트  (좌우 분할)
   슬라이드 3: Stat Callout   — 숫자 하나 압도적으로              (임팩트 중앙)
   슬라이드 4: Before/After   — 패널이 화면 대부분 차지            (풀 블리드)
   슬라이드 5: Numbered Steps — 세로 정렬, 좌측 녹색 바 강조       (세로 좌측)
→ 구조·정렬·밀도·색 무게가 매 슬라이드마다 다르다
```

### Layout choice guide

| Content type | Best layout |
|---|---|
| Core thesis, shocking fact, dramatic reveal | **Big Statement** |
| Single data point that must land hard | **Stat Callout** |
| Famous/expert quote | **Quote Block** |
| A = B definition, concept equation | **Equation + Cards** |
| Key term left + explanation right | **Split** |
| A vs B comparison with detail | **VS Card** |
| Wrong vs right, old vs new | **Before/After** |
| 4 equal items summary/review | **2×2 Grid** |
| Sequential process 4–6 steps in a row | **Horizontal Flow** |
| Complex item with checklist/tags inside | **Info Panel** |
| Row×Column matrix comparison (✓/✗) | **Comparison Table** |
| Relative scale bar chart | **Horizontal Bar Chart** |
| Sequential process (A→B→C) with icons | **Flow Diagram** |
| Step-by-step action plan (ordered) | **Numbered Steps** |
| 3–5 parallel features with icons | **Feature List** |
| 3–4 equal concepts without sequence | **Icon Grid** |
| Expert/person intro with photo | **Photo Split** |

Think like a director: vary the pacing. A Big Statement slide after a dense list gives the audience breathing room and emphasis. Don't default to Feature List or Steps for everything.

---

### 1. Equation + Cards — "A = B" definition + supporting cards
Use when: introducing a core concept with an equation-style title, then showing its components below. The most common layout in the reference slides.

```html
<div class="section-badge">섹션 4 · HOOKS</div>
<h2>Hooks = <span class="hl">이벤트 기반 자동화</span></h2>
<p class="subtitle">"어떤 일이 일어나면, 이 스크립트를 자동으로 실행해라"</p>
<div class="eq-cards">
  <div class="eq-card">
    <div class="eq-card-icon">📡</div>
    <div class="eq-card-title">이벤트 발생</div>
    <div class="eq-card-desc">팀원이 작업 완료<br>또는 idle 상태</div>
  </div>
  <div class="eq-arrow">→</div>
  <div class="eq-card active">…</div>
  <div class="eq-arrow">→</div>
  <div class="eq-card">…</div>
</div>
<div class="callout">💡 핵심 설명</div>
```
```css
.eq-cards     { display: flex; align-items: center; gap: 1.5%; width: 82%; margin-top: 1.5%; }
.eq-card      { flex: 1; display: flex; flex-direction: column; align-items: center; gap: 0.5em;
                padding: 1.2em 1em; background: var(--card-bg); border: 1px solid var(--card-border);
                border-radius: 12px; text-align: center; }
.eq-card.active { border-color: var(--green-border); background: var(--green-dim); }
.eq-card-icon { font-size: clamp(1.4rem, 3vw, 2.8rem); line-height: 1; }
.eq-card-title { font-size: clamp(0.75rem, 1.5vw, 1.3rem); font-weight: 700; color: var(--white); }
.eq-card-desc  { font-size: clamp(0.7rem, 1.1vw, 0.95rem); color: var(--muted); line-height: 1.5; }
.eq-arrow     { color: var(--green); font-size: clamp(1rem, 2vw, 1.8rem); flex-shrink: 0; }
```

---

### 2. VS Card — A vs B side-by-side detail comparison
Use when: two concepts need detailed parallel explanation (not just before/after). Each card has a title, description, behaviors, and a code/example block.

```html
<div class="section-badge">섹션 4 · 훅 비교</div>
<h2><span class="hl">TeammateIdle</span> vs <span class="hl-b">TaskCompleted</span></h2>
<div class="vs-grid">
  <div class="vs-card">
    <div class="vs-card-title" style="color:var(--green)">TeammateIdle</div>
    <div class="vs-card-desc">팀원이 할 일을 끝내고 쉬고 있을 때 발동</div>
    <div class="vs-card-body">exit 2 → 피드백을 전달해서<br>팀원이 계속 작업하도록</div>
    <div class="vs-card-quote">"쉬고 있는 직원에게 추가 업무를 주는 것"</div>
    <div class="vs-code-block"><span class="code-comment"># 활용 예시</span><br>"작업 끝나면 요약 보고서를 작성해라"</div>
  </div>
  <div class="vs-card">…</div>
</div>
```
```css
.vs-grid      { display: flex; gap: 2%; width: 82%; margin-top: 1.5%; }
.vs-card      { flex: 1; display: flex; flex-direction: column; gap: 0.6em;
                padding: 1.4em 1.6em; background: var(--card-bg);
                border: 1px solid var(--card-border); border-radius: 14px; text-align: center; }
.vs-card-title { font-size: clamp(0.85rem, 1.7vw, 1.5rem); font-weight: 700; }
.vs-card-desc  { font-size: clamp(0.55rem, 1vw, 0.88rem); color: var(--muted); line-height: 1.5; }
.vs-card-body  { font-size: clamp(0.7rem, 1.35vw, 1.2rem); font-weight: 700; color: var(--white); line-height: 1.5; }
.vs-card-quote { font-size: clamp(0.48rem, 0.85vw, 0.75rem); color: var(--muted); font-style: italic; }
.vs-code-block { background: rgba(0,0,0,0.4); border: 1px solid rgba(255,255,255,0.1); border-radius: 8px;
                 padding: 0.6em 1em; font-family: 'JetBrains Mono', monospace;
                 font-size: clamp(0.48rem, 0.85vw, 0.75rem); color: var(--white); text-align: left; }
.code-comment  { color: var(--muted); }
.code-hl       { color: var(--cyan); }  /* highlight keywords in code */
.code-red      { color: var(--red); }
```

---

### 3. 2×2 Grid — 4 numbered items in a 2-column grid
Use when: summarizing exactly 4 key points (e.g., "오늘 배운 4가지"). The numbered label is the accent color, not a circle. Bottom callout reinforces the takeaway.

```html
<div class="section-badge">핵심 정리</div>
<h2>오늘 배운 <span class="hl">4가지</span></h2>
<div class="grid22">
  <div class="grid22-card">
    <div class="grid22-num">01</div>
    <div class="grid22-title">컨텍스트 계승</div>
    <div class="grid22-desc">팀원은 새 세션으로 생성 — CLAUDE.md, MCP, 스킬 전부 자동.</div>
  </div>
  <!-- 02, 03, 04 … -->
</div>
<div class="callout">🚀 이 4가지만 적용해도 에이전트 팀을 훨씬 효율적으로 쓸 수 있습니다.</div>
```
```css
.grid22       { display: grid; grid-template-columns: 1fr 1fr; gap: 1.2%; width: 78%; margin-top: 1.5%; }
.grid22-card  { display: flex; flex-direction: column; gap: 0.4em; padding: 1.2em 1.4em;
                background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 12px; }
.grid22-num   { font-size: clamp(1rem, 2vw, 1.8rem); font-weight: 700; color: var(--cyan); line-height: 1; }
.grid22-title { font-size: clamp(0.75rem, 1.4vw, 1.25rem); font-weight: 700; color: var(--white); }
.grid22-desc  { font-size: clamp(0.5rem, 0.92vw, 0.8rem); color: var(--muted); line-height: 1.55; }
```

---

### 4. Horizontal Flow — numbered steps in a single row with arrows
Use when: showing a linear process of 4–6 steps horizontally. One step can be "active" (highlighted). Code block or callout below explains a key step.

```html
<div class="section-badge">섹션 2 · PLAN APPROVAL</div>
<h2>계획 먼저, <span class="hl">코딩 나중</span></h2>
<p class="subtitle">팀원이 바로 코딩하면 방향이 틀어질 때 토큰 전부 낭비. 한 줄만 추가하세요.</p>
<div class="hflow">
  <div class="hflow-step">
    <div class="hflow-num">01</div>
    <div class="hflow-title">팀 생성</div>
    <div class="hflow-sub">plan approval</div>
  </div>
  <div class="hflow-arrow">→</div>
  <div class="hflow-step active"> <!-- active = highlighted step -->
    <div class="hflow-num">04</div>
    <div class="hflow-title">승인 / 거절</div>
    <div class="hflow-sub">리드가 판단</div>
  </div>
  <!-- more steps -->
</div>
<div class="code-block-full">
  <div class="code-comment"># 프롬프트에 이 한 줄만 추가</div>
  각 팀원이 변경하기 전에 <span class="code-hl">plan approval</span>을 받도록 해줘.
</div>
```
```css
.hflow        { display: flex; align-items: center; gap: 0.8%; width: 88%; margin-top: 1.5%; }
.hflow-step   { flex: 1; display: flex; flex-direction: column; align-items: center; gap: 0.3em;
                padding: 0.9em 0.6em; background: var(--card-bg);
                border: 1px solid var(--card-border); border-radius: 10px; text-align: center; }
.hflow-step.active { border-color: var(--green-border); background: var(--green-dim); }
.hflow-num    { font-size: clamp(0.72rem, 1.2vw, 1.05rem); font-weight: 700; color: var(--cyan); letter-spacing: 0.06em; }
.hflow-title  { font-size: clamp(0.92rem, 1.7vw, 1.5rem); font-weight: 700; color: var(--white); }
.hflow-sub    { font-size: clamp(0.68rem, 1.1vw, 0.95rem); color: var(--muted); }
.hflow-arrow  { color: var(--green); font-size: clamp(1rem, 1.8vw, 1.6rem); flex-shrink: 0; }
/* Code block full-width */
.code-block-full { width: 60%; margin-top: 1.5%; padding: 0.8em 1.4em;
                   background: rgba(0,0,0,0.45); border: 1px solid rgba(255,255,255,0.1);
                   border-radius: 10px; font-family: 'JetBrains Mono', monospace;
                   font-size: clamp(0.55rem, 1vw, 0.88rem); color: var(--white); line-height: 1.7; }
```

---

### 5. Info Panel — large card with checklist / tag chips
Use when: showing what a concept "loads" or "includes" — a rich detail card with check/X items and tag chips.

```html
<div class="section-badge">섹션 1 · 컨텍스트 계승</div>
<h2>팀원 = <span class="hl">새 Claude Code 세션</span></h2>
<div class="info-panel">
  <div class="info-panel-header">
    <span class="info-panel-icon">🤖</span>
    <span class="info-panel-heading">팀원이 로드하는 것</span>
  </div>
  <div class="info-chips">
    <span class="chip ok">✓ 시스템 프롬프트</span>
    <span class="chip ok">✓ CLAUDE.md</span>
    <span class="chip ok">✓ git status</span>
    <span class="chip ok">✓ MCP 도구</span>
    <span class="chip ok">✓ 스킬 전부 자동</span>
    <span class="chip no">✗ 대화 기록</span>
  </div>
  <div class="info-panel-note">✓ 스킬은 전부 자동으로 로드 — 새 Claude Code 세션과 동일</div>
</div>
<div class="callout" style="border-left: 3px solid var(--cyan); padding-left:1.2em; background:transparent; border-radius:0; width:65%;">
  "각 팀원은 완전히 독립된 Claude Code 세션이다."<br>
  <span style="color:var(--muted); font-size:0.85em">— 공식 문서</span>
</div>
```
```css
.info-panel   { width: 72%; background: var(--card-bg); border: 1px solid var(--card-border);
                border-radius: 14px; padding: 1.4em 1.8em; display: flex; flex-direction: column;
                gap: 0.9em; text-align: left; margin-top: 1.2%; }
.info-panel-header { display: flex; align-items: center; gap: 0.6em; }
.info-panel-icon   { font-size: clamp(1rem, 1.8vw, 1.6rem); }
.info-panel-heading { font-size: clamp(0.75rem, 1.4vw, 1.2rem); font-weight: 700; color: var(--white); }
.info-chips   { display: flex; flex-wrap: wrap; gap: 0.5em; }
.chip         { padding: 0.25em 0.8em; border-radius: 6px; font-size: clamp(0.45rem, 0.82vw, 0.72rem);
                font-family: 'JetBrains Mono', monospace; border: 1px solid; }
.chip.ok      { color: var(--cyan); border-color: rgba(0,229,255,0.3); background: rgba(0,229,255,0.06); }
.chip.no      { color: var(--red);  border-color: var(--red-border);    background: var(--red-dim); }
.info-panel-note { font-size: clamp(0.48rem, 0.85vw, 0.75rem); color: var(--muted);
                   border-top: 1px solid rgba(255,255,255,0.08); padding-top: 0.6em; }
```

---

### 6. Comparison Table — matrix with ✓/✗/△ cells
Use when: comparing 2–3 subjects across 3–5 attributes in a grid.

```html
<h2>핵심 차이 = <span class="hl">스킬 상속 방식</span></h2>
<table class="comp-table">
  <thead>
    <tr><th></th><th>CLAUDE.md</th><th>MCP 도구</th><th>스킬</th><th>대화 기록</th></tr>
  </thead>
  <tbody>
    <tr>
      <td class="comp-row-label">팀원</td>
      <td><span class="mark ok">✓ 자동</span></td>
      <td><span class="mark ok">✓ 자동</span></td>
      <td><span class="mark ok">✓ 전부 자동</span></td>
      <td><span class="mark no">✗ 프롬프트만</span></td>
    </tr>
    <tr>
      <td class="comp-row-label">서브에이전트</td>
      <td><span class="mark ok">✓ 자동</span></td>
      <td><span class="mark ok">✓ 자동</span></td>
      <td><span class="mark warn">△ 선택 지정</span></td>
      <td><span class="mark no">✗ 프롬프트만</span></td>
    </tr>
  </tbody>
</table>
```
```css
.comp-table   { width: 78%; border-collapse: collapse; margin-top: 1.5%; font-size: clamp(0.52rem, 0.95vw, 0.84rem); }
.comp-table th { padding: 0.6em 1em; color: var(--muted); font-weight: 400; border-bottom: 1px solid rgba(255,255,255,0.1); text-align: center; }
.comp-table td { padding: 0.65em 1em; border-bottom: 1px solid rgba(255,255,255,0.06); text-align: center; }
.comp-row-label { text-align: left; color: var(--white); font-weight: 700; font-size: clamp(0.6rem, 1.1vw, 0.95rem); }
.mark      { font-size: clamp(0.52rem, 0.95vw, 0.84rem); font-weight: 700; }
.mark.ok   { color: var(--cyan); }
.mark.no   { color: var(--red); }
.mark.warn { color: var(--yellow); }
```

---

### 7. Horizontal Bar Chart — relative scale bars
Use when: showing relative magnitude across 2–4 items. Each bar has a label, filled bar, and value. Color encodes severity/ranking.

```html
<h2>토큰 비용, <span class="hl">얼마나 차이날까?</span></h2>
<p class="subtitle">에이전트 팀의 토큰 사용량은 활성 팀원 수에 비례해서 증가합니다.</p>
<div class="hbar-chart">
  <div class="hbar-row">
    <div class="hbar-label">단일 세션</div>
    <div class="hbar-track"><div class="hbar-fill" style="width:20%; background:#0070F3;">1x</div></div>
  </div>
  <div class="hbar-row">
    <div class="hbar-label">서브에이전트</div>
    <div class="hbar-track"><div class="hbar-fill" style="width:46%; background:var(--gold);">~2x</div></div>
  </div>
  <div class="hbar-row">
    <div class="hbar-label">에이전트 팀</div>
    <div class="hbar-track"><div class="hbar-fill" style="width:100%; background:var(--red);">3x+</div></div>
  </div>
</div>
<div class="callout warning">⚠️ 핵심 경고 메시지</div>
```
```css
.hbar-chart  { width: 72%; display: flex; flex-direction: column; gap: 0.9em; margin-top: 1.5%; }
.hbar-row    { display: flex; align-items: center; gap: 1.2em; }
.hbar-label  { font-size: clamp(0.6rem, 1.1vw, 0.95rem); font-weight: 700; color: var(--white); min-width: 22%; text-align: right; }
.hbar-track  { flex: 1; height: clamp(28px, 4vw, 48px); background: rgba(255,255,255,0.06); border-radius: 8px; overflow: hidden; }
.hbar-fill   { height: 100%; border-radius: 8px; display: flex; align-items: center; justify-content: flex-end; padding-right: 0.8em; font-weight: 700; font-size: clamp(0.6rem, 1.1vw, 0.95rem); color: #fff; }
```

---

### 8. Big Statement — single powerful sentence
Use when: key thesis, shocking statistic, emotional turning point, section break. No component — just large text.

```html
<div class="slide-label">섹션 레이블</div>
<h2 class="big-statement">완벽한 준비란<br>존재하지 않는다</h2>
<p class="statement-sub">지금 당장 시작하는 것이 유일한 전략이다</p>
```
```css
.big-statement { font-size: clamp(2.2rem, 5.5vw, 5rem); font-weight: 700; line-height: 1.15; letter-spacing: -0.02em; color: var(--white); margin-bottom: 0.4em; word-break: keep-all; }
.statement-sub { font-size: clamp(0.85rem, 1.8vw, 1.6rem); color: var(--green); letter-spacing: 0.02em; }
```

---

### 2. Stat Callout — one number, maximum impact
Use when: a single data point is the whole point of the slide.

```html
<div class="stat-block">
  <div class="stat-number">$78억</div>
  <div class="stat-unit">달러</div>
  <div class="stat-desc">사브리 수비 클라이언트 총 매출</div>
  <div class="stat-context">VSL 시스템 하나로 만들어낸 숫자</div>
</div>
```
> **stat-context 작성 규칙:** 한 줄에 딱 맞게 표시될 수 있도록 **15자 이내**로 짧게 작성한다. 긴 문장은 `stat-desc`로 올리고 `stat-context`는 간결한 보조 레이블로만 사용한다.
```css
.stat-block   { display: flex; flex-direction: column; align-items: center; gap: 0.3em; }
.stat-number  { font-size: clamp(3.5rem, 9vw, 8rem); font-weight: 700; color: var(--green); line-height: 1; letter-spacing: -0.03em; }
.stat-unit    { font-size: clamp(1rem, 2.2vw, 2rem); color: var(--muted); margin-top: -0.2em; }
.stat-desc    { font-size: clamp(0.9rem, 1.8vw, 1.6rem); font-weight: 700; color: var(--white); margin-top: 0.5em; }
.stat-context { font-size: clamp(0.78rem, 1.3vw, 1.12rem); color: var(--muted); margin-top: 0.2em; word-break: keep-all; }
```

---

### 3. Quote Block — expert quote with attribution
Use when: a famous person's words reinforce the point, or a testimonial.

```html
<div class="quote-block">
  <div class="quote-mark">"</div>
  <blockquote class="quote-text">완벽함을 기다리지 마라.<br>지금 당장 시작하라.</blockquote>
  <div class="quote-author">— 알렉스 호르모지</div>
  <div class="quote-role">$100M Offers 저자</div>
</div>
```
```css
.quote-block  { max-width: 72%; display: flex; flex-direction: column; align-items: center; gap: 0.4em; }
.quote-mark   { font-size: clamp(3rem, 7vw, 6rem); color: var(--green); line-height: 0.8; opacity: 0.6; font-family: Georgia, serif; }
.quote-text   { font-size: clamp(1.1rem, 2.4vw, 2.2rem); font-weight: 700; color: var(--white); text-align: center; line-height: 1.5; word-break: keep-all; }
.quote-author { font-size: clamp(0.75rem, 1.4vw, 1.2rem); color: var(--green); font-weight: 700; margin-top: 0.6em; }
.quote-role   { font-size: clamp(0.55rem, 1vw, 0.9rem); color: var(--muted); }
```

---

### 4. Flow Diagram — sequential process A→B→C
Use when: showing a repeating loop or linear pipeline.

```css
.flow-box       { border: 2px solid var(--green); border-radius: 12px; padding: 1.1em 1.8em; background: var(--green-dim); min-width: 18%; text-align: center; }
.flow-box-title { font-size: clamp(0.9rem, 1.9vw, 1.7rem); font-weight: 700; color: var(--white); line-height: 1.3; }
.flow-box-sub   { font-size: clamp(0.62rem, 1.2vw, 1.05rem); color: var(--green); margin-top: 0.3em; }
.flow-arrow     { color: var(--green); font-size: clamp(1.4rem, 3vw, 2.8rem); flex-shrink: 0; }
.flow-desc      { font-size: clamp(0.72rem, 1.3vw, 1.2rem); color: var(--muted); margin-top: 2.5%; }
```

---

### 5. Before / After — contrast comparison
Use when: showing wrong vs right, old vs new, with vs without.

```css
.ba-container { display: flex; width: 72%; min-height: 40%; border-radius: 12px; overflow: hidden; border: 1px solid rgba(255,255,255,0.08); }
.ba-panel     { flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 2.2em 2em; gap: 0.55em; }
.ba-before    { background: rgba(255,255,255,0.03); }
.ba-after     { background: var(--green-dim); }
.ba-after.danger { background: var(--red-dim); }
.ba-icon      { width: clamp(40px,5.5vw,60px); height: clamp(40px,5.5vw,60px); border-radius: 50%; font-size: clamp(1rem,1.8vw,1.6rem); font-weight: 700; }
.ba-label     { font-size: clamp(0.85rem, 1.5vw, 1.3rem); font-weight: 700; }
.ba-desc      { font-size: clamp(0.72rem, 1.2vw, 1.05rem); color: var(--muted); text-align: center; line-height: 1.6; }
```

---

### 6. Numbered Steps — ordered action plan (card style)
Use when: step-by-step instructions, ordered process. ALWAYS center `.steps-wrapper` on slide.

```css
.steps-wrapper { width: fit-content; max-width: 68%; min-width: 55%; display: flex; flex-direction: column; gap: 0.5em; margin-top: 1%; }
.step-row  { display: flex; align-items: center; gap: 1.2em; padding: 0.65em 1.3em; background: var(--card-bg); border: 1px solid var(--card-border); border-left: 3px solid var(--green); border-radius: 12px; text-align: left; }
.step-num  { width: clamp(38px,5vw,52px); height: clamp(38px,5vw,52px); border-radius: 50%; background: var(--green); color: #000; font-weight: 700; font-size: clamp(0.9rem,1.8vw,1.5rem); display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.step-title { font-size: clamp(0.85rem, 1.7vw, 1.5rem); font-weight: 700; color: var(--white); }
.step-desc  { font-size: clamp(0.55rem, 0.95vw, 0.85rem); color: var(--muted); margin-top: 0.15em; }
```
HTML: `<div class="step-num">1</div><div class="step-body">…</div>` — no `step-left-col`, no connector rows.

---

### 7. Progress Bars — metrics comparison
Use when: showing relative percentages, performance comparison, before/after numbers.

```css
.progress-rows  { width: 66%; display: flex; flex-direction: column; gap: 1.5em; margin-top: 1.5%; }
.progress-label { font-size: clamp(0.85rem,1.5vw,1.4rem); font-weight: 700; min-width: 22%; text-align: right; white-space: nowrap; }
.progress-track { flex: 1; height: clamp(14px,1.8vw,22px); background: rgba(255,255,255,0.08); border-radius: 100px; overflow: hidden; }
.progress-pct   { font-size: clamp(0.9rem,1.7vw,1.5rem); font-weight: 700; min-width: 3.8em; }
/* Animate: data-target-width on .progress-fill-bar, trigger via requestAnimationFrame on step reveal */
```

---

### 8. Feature List — parallel items with icons
Use when: 3–5 features, benefits, or concepts with brief explanations.

```css
.feature-list  { width: fit-content; max-width: 68%; display: flex; flex-direction: column; gap: 0.6em; align-self: center; }
.feature-item  { display: flex; align-items: center; gap: 1em; padding: 0.7em 1.2em; background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 10px; border-left: 3px solid var(--green); }
.feature-icon  { width: clamp(36px,4.5vw,52px); height: clamp(36px,4.5vw,52px); border-radius: 10px; background: var(--green-dim); border: 1px solid var(--green-border); font-size: clamp(1rem,1.6vw,1.4rem); display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.feature-title { font-size: clamp(0.8rem, 1.4vw, 1.2rem); font-weight: 700; color: var(--white); }
.feature-sub   { font-size: clamp(0.55rem, 0.95vw, 0.85rem); color: var(--muted); margin-top: 0.15em; }
```

---

### 9. Icon Grid — parallel categories, no hierarchy
Use when: 3–4 equal concepts that don't have a sequence. Arrange in a row or 2×2 grid.

```html
<div class="icon-grid">
  <div class="icon-cell">
    <div class="icon-emoji">📢</div>
    <div class="icon-label">광고</div>
    <div class="icon-sub">어글리 VSL로 런칭</div>
  </div>
  <!-- repeat -->
</div>
```
```css
.icon-grid  { display: flex; gap: 2.5%; width: 80%; margin-top: 2%; justify-content: center; flex-wrap: wrap; }
.icon-cell  { flex: 1; min-width: 18%; max-width: 22%; display: flex; flex-direction: column; align-items: center; gap: 0.5em; padding: 1.4em 1em; background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 14px; }
.icon-emoji { font-size: clamp(1.6rem, 3.5vw, 3.2rem); line-height: 1; }
.icon-label { font-size: clamp(0.75rem, 1.5vw, 1.3rem); font-weight: 700; color: var(--white); }
.icon-sub   { font-size: clamp(0.5rem, 0.9vw, 0.8rem); color: var(--muted); text-align: center; line-height: 1.4; }
```

---

### 10. Split — label/concept left, detail right
Use when: introducing a term + its explanation, or a concept + examples side by side.

```html
<div class="split-layout">
  <div class="split-left">
    <div class="split-tag">핵심 개념</div>
    <div class="split-title">피드백<br>루프</div>
  </div>
  <div class="split-divider"></div>
  <div class="split-right">
    <p class="split-body">시장에 던지고 → 반응 수집 → 개선 → 재런칭을 반복하는 수익화 사이클.</p>
    <ul class="split-list">
      <li>완벽주의 제거</li>
      <li>데이터 기반 의사결정</li>
    </ul>
  </div>
</div>
```
```css
.split-layout { display: flex; align-items: center; width: 84%; gap: 4%; margin-top: 2%; }
.split-left   { flex: 0 0 32%; display: flex; flex-direction: column; gap: 0.4em; }
.split-tag    { font-family: 'JetBrains Mono', monospace; font-size: clamp(0.5rem,0.9vw,0.8rem); color: var(--green); letter-spacing: 0.18em; text-transform: uppercase; }
.split-title  { font-size: clamp(1.8rem, 4vw, 3.8rem); font-weight: 700; color: var(--white); line-height: 1.1; }
.split-divider { width: 1px; align-self: stretch; background: rgba(255,255,255,0.12); flex-shrink: 0; }
.split-right  { flex: 1; display: flex; flex-direction: column; gap: 0.8em; text-align: left; }
.split-body   { font-size: clamp(0.72rem, 1.4vw, 1.25rem); color: rgba(255,255,255,0.8); line-height: 1.7; }
.split-list   { list-style: none; display: flex; flex-direction: column; gap: 0.4em; }
.split-list li { font-size: clamp(0.65rem, 1.2vw, 1.05rem); color: var(--muted); padding-left: 1.2em; position: relative; }
.split-list li::before { content: '▸'; color: var(--green); position: absolute; left: 0; }
```

---

### 11. Photo Split — 좌측 인물사진 + 우측 인용/소개
Use when: 전문가·사례 인물을 사진과 함께 소개할 때. 사용자가 이미지 파일을 제공한 경우에 사용.

- 사진은 슬라이드 **좌측 50%** 전체를 채움 (object-fit: cover, object-position: top center)
- 사진 우측 끝에 그라디언트 페이드 처리 (검정으로 자연스럽게 이어짐)
- 텍스트 영역은 **우측 52%**, 좌우 패딩 포함, 세로 중앙 정렬
- 슬라이드 자체에 `padding: 0` 필수 (`.slide-portrait { padding: 0 !important; }`)

```html
<section class="slide slide-portrait">
  <!-- 좌측: 사진 -->
  <div class="portrait-photo-half">
    <img src="photo.jpg" alt="인물명" />
  </div>
  <!-- 우측: 텍스트 -->
  <div class="portrait-text-half">
    <div class="portrait-overline">섹션 레이블</div>
    <div class="portrait-quote-mark">"</div>
    <p class="portrait-quote">핵심 인용문이<br>여기 들어갑니다.</p>
    <p class="portrait-sub">부연 설명 한두 줄.</p>
    <div class="portrait-name">인물 이름</div>
    <div class="portrait-title">직책 · 주요 실적</div>
  </div>
</section>
```

```css
.slide-portrait { padding: 0 !important; }

/* 좌측 사진 */
.portrait-photo-half {
  position: absolute; top: 0; left: 0; bottom: 0; width: 50%;
}
.portrait-photo-half img {
  width: 100%; height: 100%; object-fit: cover; object-position: top center; display: block;
}
.portrait-photo-half::before {
  content: ''; position: absolute; inset: 0; z-index: 1;
  background: linear-gradient(to left, #000 0%, transparent 25%),
              linear-gradient(to top, rgba(0,0,0,0.4) 0%, transparent 40%);
}

/* 우측 텍스트 */
.portrait-text-half {
  position: absolute; top: 0; right: 0; bottom: 0; width: 52%;
  display: flex; flex-direction: column;
  justify-content: center; align-items: flex-start;
  padding: 7% 8% 7% 6%; text-align: left;
}
.portrait-overline {
  font-family: 'JetBrains Mono', monospace; font-size: clamp(0.48rem, 0.88vw, 0.78rem);
  letter-spacing: 0.22em; color: var(--green); margin-bottom: 0.7em;
  display: flex; align-items: center; gap: 0.5em;
}
.portrait-overline::before { content: '>'; animation: blink-cursor 1s step-end infinite; }
.portrait-quote-mark {
  font-size: clamp(2.5rem, 6vw, 5.5rem); color: var(--green);
  opacity: 0.35; line-height: 0.75; font-family: Georgia, serif; margin-bottom: 0.15em;
}
.portrait-quote {
  font-size: clamp(1.1rem, 2.3vw, 2.1rem); font-weight: 700;
  color: var(--white); line-height: 1.5; word-break: keep-all; margin-bottom: 1em;
}
.portrait-sub {
  font-size: clamp(0.6rem, 1.15vw, 1rem); color: var(--muted); line-height: 1.65; margin-bottom: 1.5em;
}
.portrait-name {
  font-size: clamp(0.85rem, 1.6vw, 1.4rem); font-weight: 700; color: var(--white); margin-top: 1.2em;
}
.portrait-title {
  font-size: clamp(0.5rem, 0.9vw, 0.8rem); color: var(--green);
  letter-spacing: 0.08em; margin-top: 0.25em;
}
```

**이미지 임베드 방법:**
- 사용자가 파일 경로 제공 시 → base64로 인코딩해 `src="data:image/jpeg;base64,..."` 로 임베드 (파일 자급자족)
- 또는 상대경로 `src="photo.jpg"` 사용 (HTML과 같은 폴더에 이미지 있을 때)

## Build animations — when and how to apply

This is a lecture-style deck. Build effects control the pacing of information delivery — the audience should only see what the instructor is currently talking about.

### CSS
```css
[data-step] { opacity:0; transform:translateY(14px); transition:opacity .38s ease, transform .38s ease; visibility:hidden; }
[data-step].visible { opacity:1; transform:translateY(0); visibility:visible; }
```

### Rules — ALWAYS apply `data-step` to these:

| Layout | How to build |
|---|---|
| Numbered Steps (3+ items) | Each `.step-row` gets its own step |
| Feature List (3+ items) | Each `.feature-item` gets its own step |
| Horizontal Flow (4+ steps) | Each `.hflow-step` + its arrow gets its own step |
| VS Card | Left card first (`data-step="1"`), right card second (`data-step="2"`) |
| 2×2 Grid | Each card in reading order (01→02→03→04) |
| Equation + Cards | Cards build one by one after the title appears |
| Flow Diagram (3+ boxes) | Each box + arrow builds left to right |
| Comparison Table | Each row builds one at a time |
| Horizontal Bar Chart | Each bar builds one at a time |
| Callout box | Always last step on the slide |
| Progress Bars | Each bar row builds one at a time |

### Rules — do NOT apply `data-step` to these:
- Title slide
- Big Statement slide (full impact, appears all at once)
- Stat Callout (single number, appears all at once)
- Quote Block (appears all at once)
- Photo Split (appears all at once)
- Any slide with only 1–2 elements

### Judgment call — content drives the decision:
If a slide has a list of 2 items, skip the build. If a concept is meant to "land all at once" (e.g., a shocking comparison table), show it all at once. The build effect is for pacing explanation, not decoration — if the instructor would naturally reveal it step by step while talking, use `data-step`.

---

## Slide count — 스크립트 충실 원칙

스크립트에 있는 슬라이드 수와 순서를 그대로 따른다. 임의로 슬라이드를 추가·삭제·순서 변경하지 않는다. 슬라이드 1장 = 스크립트 1개 섹션.

## Fullscreen implementation

```js
case 'f': case 'F':
  if (!document.fullscreenElement) {
    document.documentElement.requestFullscreen().catch(() => {});
  } else {
    document.exitFullscreen().catch(() => {});
  }
  handled = true; break;
```

## Self-contained requirement

- No external stylesheet or script links (except optionally a single Google Fonts `<link>` for a nice font like Inter or system-ui)
- All CSS in `<style>` in `<head>`
- All JS in `<script>` at end of `<body>`
- The file must work when opened directly from the filesystem (no server needed)

## Content guidelines

When given a topic or script, generate slides that tell a coherent story with visual variety:

1. **Title slide** — Big Statement or just h2 + subtitle. No component.
2. **Content slides** — Read each slide's message and pick the layout that fits:
   - Shocking fact or core principle → Big Statement
   - Single critical number → Stat Callout
   - Expert/famous quote → Quote Block
   - A→B→C loop or pipeline → Flow Diagram
   - Wrong vs right mindset → Before / After
   - Step-by-step actions → Numbered Steps
   - Percentage data → Progress Bars
   - Parallel features/benefits → Feature List or Icon Grid
   - Term + deep explanation → Split
3. **Closing slide** — Big Statement + CTA button, or Numbered Steps summary.

**Variety rule:** In any deck of 8+ slides, no single layout should appear more than 2 times. Aim for 5+ different layouts. The reference design (이미지 분석 기준) uses: Equation+Cards, VS Card, 2×2 Grid, Horizontal Flow, Info Panel, Comparison Table, Horizontal Bar Chart — mix these freely.

**Title pattern:** Almost every slide title follows "흰색 텍스트 + `<span class='hl'>강조어</span>`" pattern. Never write a plain all-white title for content slides.

**Callout box:** Use `.callout` only when there is a genuinely important insight, warning, or punchline that the main slide content does not already make obvious. Do NOT add a callout just to fill space — the main copy in the title and cards is usually sufficient on its own. When a callout IS used, give it at least 4% top margin so it breathes. On slides where the build effect already delivers a final callout (data-step), the always-visible "static" callout is almost never needed — avoid both on the same slide.

**Code blocks:** When content involves commands, prompts, or technical snippets, always render them in `.code-block-full` with monospace font and colored syntax (`.code-hl` for cyan, `.code-red` for red).

**고유명사 표기 통일:** "알렉스 호르모지", "호르모지", "Hormozi" 등 모든 표기는 반드시 **알렉스 홀모지**로 통일한다. 예외 없음.

**헤드카피·서브카피 원문 준수:** 슬라이드의 `h2`(헤드카피)와 `p.subtitle`(서브카피)는 스크립트의 **헤드카피·서브카피 텍스트를 그대로** 사용한다. 임의로 바꾸거나 재해석·요약하지 않는다. 스크립트에 헤드카피/서브카피가 명시된 경우 한 글자도 변경하지 않는다.

**section-badge 사용 원칙:** `section-badge`는 슬라이드의 맥락을 나타내는 **짧은 레이블**(예: "섹션 2 · 개념 정의", "핵심 정리")로만 사용한다. 헤드카피나 서브카피 내용을 section-badge에 넣지 않는다. section-badge가 불필요하다면 생략해도 된다.

Keep text concise — slides are visual aids, not documents. Each bullet/step fits in one line. Use `data-step` builds on individual component items so the audience can follow along.

## Output

Save the file as `<topic-slug>.html` (e.g. `react-hooks.html`). Confirm the filename and tell the user they can open it in any browser to present.
