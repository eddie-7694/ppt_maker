# 순수 바닐라 웹 슬라이드 프레젠테이션 리서치

> 외부 라이브러리 없이 HTML + CSS + JavaScript만으로 구현하는 최적의 방법 연구
> 4개 주제를 병렬 리서치하여 통합 정리

---

## 목차

1. [슬라이드 레이아웃 & DOM 구조](#1-슬라이드-레이아웃--dom-구조)
2. [키보드 & 마우스 네비게이션](#2-키보드--마우스-네비게이션)
3. [슬라이드 전환 애니메이션](#3-슬라이드-전환-애니메이션)
4. [슬라이드 내 콘텐츠 애니메이션 (빌드 효과)](#4-슬라이드-내-콘텐츠-애니메이션-빌드-효과)
5. [통합 추천 구조](#5-통합-추천-구조)
6. [주의사항 종합](#6-주의사항-종합)

---

## 1. 슬라이드 레이아웃 & DOM 구조

### 1-1. Full-Viewport CSS 레이아웃 방법 비교

| 방식 | 장점 | 단점 | 슬라이드 적합성 |
|------|------|------|----------------|
| **Flexbox** | 중앙 정렬이 직관적 (`justify-content` + `align-items`) | 슬라이드 겹침 구조에 부적합 | 무대(Stage) 레이아웃에 적합 |
| **CSS Grid** | `place-items: center` 한 줄로 완전한 중앙 정렬 | 슬라이드 스택 전환 시 추가 작업 필요 | 무대 레이아웃에 적합 |
| **Absolute Positioning** | 같은 위치에 겹쳐 쌓을 수 있어 전환 애니메이션 최적 | 중앙 정렬 코드가 상대적으로 장황 | 개별 슬라이드 스택에 최적 |

**권장: Flexbox(무대) + Absolute Positioning(슬라이드 스택) 혼합**

```css
/* 무대: Flexbox로 중앙 정렬 */
.presentation-stage {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100vw;
  height: 100dvh;       /* iOS Safari 대응: dvh 사용 */
  background: #1a1a1a;  /* letterbox/pillarbox 배경 */
  overflow: hidden;
}

/* 덱: 16:9 비율 고정 */
.slide-deck {
  aspect-ratio: 16 / 9;
  width: min(100vw, 177.78vh);  /* 가로/세로 중 제한적인 쪽에 맞춤 */
  position: relative;
  overflow: hidden;
}

/* 슬라이드: absolute 스택 */
.slide {
  position: absolute;
  inset: 0;  /* top/right/bottom/left: 0 단축 */
  width: 100%;
  height: 100%;
}
```

### 1-2. 16:9 비율 고정 방법 비교

| 방법 | 코드 | 브라우저 지원 | 권장 여부 |
|------|------|--------------|-----------|
| `aspect-ratio: 16/9` | 직관적, 명시적 | Chrome 88+, FF 89+, Safari 15+ | 신규 프로젝트 권장 |
| `padding-top: 56.25%` | 너비 기준 % 계산 특성 이용 | IE 포함 전체 | 레거시 지원 필요 시만 |

**Letterbox/Pillarbox 동작 원리:**
```
width: min(100vw, 177.78vh) 설정 시:
- 가로로 긴 화면 → 100vw 제한 → 위아래 검은 여백 (letterbox)
- 세로로 긴 화면 → 177.78vh 제한 → 좌우 검은 여백 (pillarbox)
- 정확히 16:9 → 양쪽 모두 꽉 참
```

### 1-3. 반응형 폰트 (clamp 활용)

```css
.slide__title    { font-size: clamp(1.5rem, 4vw, 3.5rem); }
.slide__subtitle { font-size: clamp(1rem, 2vw, 1.8rem); }
.slide__body     { font-size: clamp(0.875rem, 1.5vw, 1.4rem); }
```
미디어쿼리 없이 슬라이드 크기에 비례해 자동 스케일링. 최솟값/최댓값으로 가독성 보장.

### 1-4. 시맨틱 HTML & 접근성

```html
<div class="presentation-stage">
  <div class="slide-deck"
       role="region"
       aria-label="슬라이드 프레젠테이션"
       aria-roledescription="presentation">

    <section class="slide active"
             role="group"
             aria-roledescription="slide"
             aria-label="1 / 5"
             tabindex="0">
      <!-- 슬라이드 내용 -->
      <aside class="speaker-notes">화자 노트</aside>
    </section>

    <section class="slide"
             aria-hidden="true"   <!-- 비활성 슬라이드 숨김 -->
             tabindex="-1">
      <!-- ... -->
    </section>
  </div>

  <!-- 슬라이드 번호: aria-live로 자동 읽기 -->
  <div class="slide-counter" aria-live="polite">1 / 5</div>
</div>
```

**접근성 핵심:**
- `aria-roledescription="presentation"` — 스크린리더가 "프레젠테이션"으로 읽음
- `aria-hidden="true"` — 비활성 슬라이드를 스크린리더에서 완전 제외
- `aria-live="polite"` — 슬라이드 번호 변경 시 자동 읽어줌

### 핵심 패턴 요약

1. **무대-덱-슬라이드 3단계 구조**: Stage(Flexbox 중앙 정렬) → Deck(16:9 비율) → Slide(absolute 스택)
2. **`display:none` 금지**: `opacity:0 + visibility:hidden` 조합만 CSS transition이 작동함
3. **`clamp(min, Nvw, max)`**: 미디어쿼리 없이 모든 화면 크기 대응, 최솟값/최댓값으로 가독성 경계 보장

---

## 2. 키보드 & 마우스 네비게이션

### 2-1. 키보드 이벤트 처리

**`keydown` vs `keyup` 선택**: 슬라이드에는 `keydown`이 적합
- 즉각 반응, `preventDefault()`로 브라우저 기본 동작(Space 스크롤) 차단 가능
- `e.repeat: true` 체크로 키 홀드 반복 차단

```javascript
document.addEventListener('keydown', (e) => {
  // 입력 요소 포커스 시 무시
  if (['INPUT','TEXTAREA','SELECT'].includes(e.target.tagName)) return;
  if (e.target.isContentEditable) return;
  if (e.repeat) return; // 키 홀드 반복 차단

  let handled = false;
  switch (e.key) {
    case 'ArrowRight': case 'ArrowDown': case ' ': case 'PageDown':
      goNext(); handled = true; break;
    case 'ArrowLeft': case 'ArrowUp': case 'PageUp':
      goPrev(); handled = true; break;
    case 'Home': goTo(0); handled = true; break;
    case 'End':  goTo(total - 1); handled = true; break;
  }
  if (handled) e.preventDefault(); // Space 스크롤 등 기본 동작 차단
});
```

### 2-2. 터치/스와이프 — Pointer Events API

마우스·터치·펜을 단일 코드로 통합 처리:

```javascript
const el = document.getElementById('stage');
let startX, startY, startT, active = false;

el.addEventListener('pointerdown', e => {
  if (!e.isPrimary) return; // 멀티터치 첫 번째 포인터만 처리
  active = true;
  startX = e.clientX; startY = e.clientY; startT = Date.now();
  el.setPointerCapture(e.pointerId); // 범위 이탈 후에도 이벤트 수신
});

el.addEventListener('pointermove', e => {
  if (!active || !e.isPrimary) return;
  const dx = Math.abs(e.clientX - startX);
  const dy = Math.abs(e.clientY - startY);
  if (dx > dy && dx > 10) e.preventDefault(); // 수평 스와이프 시 스크롤 방지
}, { passive: false }); // passive:false 필수 (preventDefault 사용)

el.addEventListener('pointerup', e => {
  if (!active || !e.isPrimary) return;
  active = false;
  const dx = e.clientX - startX;
  const dy = e.clientY - startY;
  if (Date.now() - startT > 500) return;       // 시간 초과
  if (Math.abs(dx) < 50) return;               // 거리 미달
  if (Math.abs(dy) > Math.abs(dx)) return;     // 수직 제스처
  dx < 0 ? goNext() : goPrev();
});
```

### 2-3. URL Hash 동기화

`location.hash` 방식이 정적 파일 배포에 적합 (`history.pushState` 병용):

```javascript
// 슬라이드 이동 시 URL 업데이트
function updateURL(index) {
  history.pushState({ idx: index }, '', `#slide-${index + 1}`);
}

// 브라우저 뒤로가기/앞으로가기 처리
window.addEventListener('popstate', (e) => {
  if (e.state?.idx !== undefined) {
    renderSlide(e.state.idx);
  } else {
    restoreFromHash(); // state 없을 때 hash 파싱으로 fallback
  }
});

// 페이지 최초 로드 시 복원
function restoreFromHash() {
  const m = location.hash.match(/^#slide-(\d+)$/);
  const idx = m ? Math.max(0, Math.min(+m[1] - 1, total - 1)) : 0;
  renderSlide(idx);
}
```

### 2-4. 진행 표시바 & 슬라이드 번호

```css
.progress-bar-fill {
  height: 100%;
  width: 0%;
  background: #4f9eff;
  transition: width 0.35s ease;
}

.slide-counter {
  font-variant-numeric: tabular-nums; /* 숫자 너비 고정, 레이아웃 흔들림 방지 */
}
```

```javascript
function updateUI(index, total) {
  // 진행바: 마지막 슬라이드에서 정확히 100%가 되려면 (total-1) 사용
  const pct = total > 1 ? (index / (total - 1)) * 100 : 100;
  document.querySelector('.progress-bar-fill').style.width = `${pct}%`;

  // 슬라이드 번호
  document.querySelector('.slide-counter').textContent = `${index + 1} / ${total}`;

  // 썸네일 닷
  document.querySelectorAll('.thumbnail-dot')
    .forEach((d, i) => d.classList.toggle('active', i === index));
}
```

### 핵심 패턴 요약

1. **`keydown` + `e.repeat` 차단 + 입력 요소 필터**: 즉각 반응성과 충돌 방지 동시 해결
2. **Pointer Events API**: `pointerdown→pointermove(passive:false)→pointerup` + `setPointerCapture`로 마우스·터치 단일 처리
3. **`history.pushState` + `popstate`**: 슬라이드 이동 시 `#slide-N` 기록, popstate에서 복원해 뒤로가기까지 완전 동기화

---

## 3. 슬라이드 전환 애니메이션

### 3-1. 방법 비교

| 방법 | 장점 | 단점 | 슬라이드 전환 적합성 |
|------|------|------|---------------------|
| **CSS Transition** | 클래스 토글만으로 완성, 코드 최소 | 방향별 동적 값 계산 어려움 | ✅ 권장 (단순 구현) |
| **CSS Animation** | 반복, keyframes 세밀 제어 | 클래스 붙이면 자동 실행, 타이밍 제어 어려움 | 슬라이드 전환에 과함 |
| **Web Animations API** | `.finished` Promise, 방향별 동적 값 | 코드량 증가 | ✅ 권장 (방향별 계산 필요 시) |

**핵심**: 방법보다 **어떤 CSS 속성을 애니메이션하느냐**가 성능을 결정한다.

### 3-2. GPU 가속 원리

```
브라우저 렌더링 파이프라인:
JavaScript → Style → Layout → Paint → Composite

- width/height/top/left 변경  →  Layout(Reflow) 재발생  →  가장 비쌈
- background-color/color 변경 →  Paint(Repaint) 재발생  →  비쌈
- transform/opacity 변경      →  Composite만 발생        →  GPU 처리, 가장 빠름
```

```css
/* 나쁜 예: Layout 유발 */
.slide { transition: left 0.5s; }

/* 좋은 예: GPU Composite만 */
.slide { transition: transform 0.5s; }
```

### 3-3. 대표 전환 효과 CSS

```css
/* 공통 기반 */
.slide {
  position: absolute;
  inset: 0;
  transition: transform 0.45s cubic-bezier(0.25, 0.46, 0.45, 0.94),
              opacity 0.45s ease;
  pointer-events: none;
}
.slide.active { pointer-events: auto; z-index: 1; }
```

**Slide (수평 이동):**
```css
.slide        { transform: translateX(100%); opacity: 0; }
.slide.active { transform: translateX(0%);   opacity: 1; }
.slide.prev   { transform: translateX(-100%); opacity: 0; }
.slide.next   { transform: translateX(100%);  opacity: 0; }
```

**Fade (투명도):**
```css
.slide        { opacity: 0; }
.slide.active { opacity: 1; }
.slide.prev, .slide.next { opacity: 0; }
```

**Zoom (확대/축소):**
```css
.slide        { opacity: 0; transform: scale(0.85); }
.slide.active { opacity: 1; transform: scale(1); }
.slide.prev   { opacity: 0; transform: scale(1.1); }
.slide.next   { opacity: 0; transform: scale(0.85); }
```

**Flip (회전):**
```css
.slideshow    { perspective: 1200px; }
.slide        { transform: rotateY(90deg);  backface-visibility: hidden; }
.slide.active { transform: rotateY(0deg); }
.slide.prev   { transform: rotateY(-90deg); }
.slide.next   { transform: rotateY(90deg); }
```

### 3-4. 이벤트 잠금 시스템

```javascript
function goTo(nextIdx, dir) {
  if (this.locked) return;
  this.locked = true;

  const from = slides[currentIdx];
  const to   = slides[nextIdx];

  // will-change: 전환 직전만 적용 (상시 적용 시 GPU 메모리 낭비)
  from.style.willChange = 'transform, opacity';
  to.style.willChange   = 'transform, opacity';

  // 진입 슬라이드를 transition 없이 시작 위치로 이동
  to.classList.add('no-transition');
  to.classList.add(dir === 'next' ? 'next' : 'prev');
  void to.offsetWidth; // 강제 reflow (이 없으면 transition이 건너뜀)
  to.classList.remove('no-transition');

  // 두 슬라이드 동시 이동
  from.classList.remove('active');
  from.classList.add(dir === 'next' ? 'prev' : 'next');
  to.classList.remove('prev', 'next');
  to.classList.add('active');

  // 3중 잠금 해제
  const unlock = () => {
    clearTimeout(fallback);
    from.style.willChange = 'auto';
    to.style.willChange   = 'auto';
    this.locked = false;
  };

  // 1. transitionend (propertyName 필터 필수)
  const onEnd = (e) => {
    if (e.propertyName !== 'transform') return;
    to.removeEventListener('transitioncancel', onCancel);
    unlock();
  };
  // 2. transitioncancel (전환 취소 시 발생)
  const onCancel = () => {
    to.removeEventListener('transitionend', onEnd);
    unlock();
  };
  to.addEventListener('transitionend', onEnd, { once: true });
  to.addEventListener('transitioncancel', onCancel, { once: true });
  // 3. setTimeout fallback (duration=0이거나 이벤트 미발생 시)
  const fallback = setTimeout(unlock, 500);
}
```

### 핵심 패턴 요약

1. **`transform`과 `opacity`만 애니메이션**: Layout·Paint 없이 GPU 컴포지터만 사용
2. **`will-change`는 전환 직전/직후만**: 상시 적용은 GPU 메모리 낭비 및 역효과
3. **잠금 해제 3중 보험**: `transitionend`(propertyName 필터) + `transitioncancel` + `setTimeout` fallback

---

## 4. 슬라이드 내 콘텐츠 애니메이션 (빌드 효과)

### 4-1. data-step 기반 순차 등장

```html
<section class="slide">
  <h2>제목</h2>
  <ul>
    <li data-step="1">첫 번째 항목</li>
    <li data-step="2">두 번째 항목</li>
    <li data-step="3">세 번째 항목</li>
  </ul>
  <p data-step="4">결론</p>
</section>
```

```css
/* 기본 숨김 (visibility로 스크린리더도 차단) */
[data-step] {
  opacity: 0;
  transform: translateY(14px);
  transition: opacity 0.35s ease, transform 0.35s ease;
  visibility: hidden;
}

[data-step].visible {
  opacity: 1;
  transform: translateY(0);
  visibility: visible;
}
```

```javascript
// 빌드와 슬라이드 전환 통합 로직
next() {
  const maxStep = getMaxStep(slides[currentIdx]);

  if (currentStep < maxStep) {
    // 빌드가 남아있으면 빌드 먼저 소진
    currentStep++;
    applyStep(slides[currentIdx], currentStep);
  } else {
    // 모든 빌드 소진 후 다음 슬라이드로
    if (currentIdx < slides.length - 1) goTo(currentIdx + 1, 'next');
  }
}

function applyStep(slide, step) {
  slide.querySelectorAll('[data-step]').forEach(el => {
    const visible = +el.dataset.step <= step;
    el.classList.toggle('visible', visible);
    // 프로그레스 바 애니메이션: visible 시 data-target-width로 세팅, 숨길 때 0% 리셋
    el.querySelectorAll('.progress-fill-bar[data-target-width]').forEach(bar => {
      if (visible) {
        requestAnimationFrame(() => { bar.style.width = bar.dataset.targetWidth; });
      } else {
        bar.style.width = '0%';
      }
    });
  });
}

function getMaxStep(slide) {
  const steps = slide.querySelectorAll('[data-step]');
  return steps.length ? Math.max(...[...steps].map(e => +e.dataset.step)) : 0;
}
```

**슬라이드 이탈/복귀 시 상태 관리:**
```javascript
function enterSlide(idx, fromBack = false) {
  currentIdx = idx;
  currentStep = fromBack ? getMaxStep(slides[idx]) : 0; // 뒤로 올 때는 완료 상태로

  // 빌드 상태 초기화
  slides[idx].querySelectorAll('[data-step]').forEach(el => el.classList.remove('visible'));
  applyStep(slides[idx], currentStep);
}
```

### 4-2. CSS animation-delay 자동 방식 (참고)

```html
<ul class="auto-build">
  <li style="--delay: 0">항목 A</li>
  <li style="--delay: 1">항목 B</li>
</ul>
```

```css
.slide.active .auto-build li {
  animation: fadeUp 0.4s ease forwards;
  animation-delay: calc(var(--delay) * 0.15s + 0.3s);
}

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* 슬라이드 비활성 시 즉시 리셋 */
.slide:not(.active) .auto-build li {
  opacity: 0;
  animation: none; /* 없으면 재진입 시 애니메이션 재생 안 됨 */
}
```

> **주의**: animation-delay 방식은 개별 클릭 제어 불가. 화자가 타이밍을 직접 제어해야 하는 발표용에는 `data-step` JS 방식이 적합.

### 4-3. 코드 블록 줄별 하이라이트

```html
<pre class="code-block">
  <code>
    <span class="line" data-highlight="1">const x = 10;</span>
    <span class="line" data-highlight="2">const y = 20;</span>
    <span class="line" data-highlight="3">console.log(x + y);</span>
  </code>
</pre>
```

```css
.line {
  display: block;
  padding: 2px 8px;
  border-left: 3px solid transparent;
  transition: background 0.3s, opacity 0.3s;
  opacity: 0.35; /* 비강조 줄: 흐리게 */
}

.line.highlighted {
  opacity: 1;
  background: rgba(167, 139, 250, 0.2);
  border-left-color: #a78bfa;
}

.line.past { opacity: 0.65; } /* 이미 지나간 줄 */
```

```javascript
function highlightLine(slide, step) {
  slide.querySelectorAll('.line[data-highlight]').forEach(line => {
    const h = +line.dataset.highlight;
    line.classList.toggle('highlighted', h === step);
    line.classList.toggle('past', h < step);
  });
}
```

### 4-4. 화자 노트 시스템

**인라인 토글 방식 (단순):**
```css
.speaker-notes { display: none; }

body.presenter-mode .speaker-notes {
  display: block;
  position: fixed;
  bottom: 0; left: 0; right: 0;
  background: rgba(0,0,0,0.9);
  color: #fff;
  padding: 14px 24px;
  font-size: 14px;
  border-top: 2px solid #a78bfa;
  z-index: 300;
}
```

```javascript
// S 키로 토글
document.addEventListener('keydown', e => {
  if (e.key === 's' || e.key === 'S') {
    document.body.classList.toggle('presenter-mode');
  }
});
```

**팝업 창 방식 (postMessage 동기화):**
```javascript
function openNotesWindow() {
  const win = window.open('', 'notes', 'width=600,height=400');
  win.document.write(/* HTML */ `
    <script>
      window.addEventListener('message', e => {
        if (e.origin !== window.opener?.location.origin) return; // origin 검증 필수
        document.getElementById('notes').innerHTML = e.data.notes;
      });
    <\/script>
    <div id="notes"></div>
  `);
  win.document.close();
  return win;
}

// 슬라이드 변경 시 동기화
function syncNotes(notesWindow) {
  if (!notesWindow || notesWindow.closed) return;
  const notes = currentSlide.querySelector('.speaker-notes')?.innerHTML || '';
  notesWindow.postMessage({ notes }, window.location.origin);
}
```

### 핵심 패턴 요약

1. **빌드 = 스텝 카운터 비교**: `data-step` 숫자와 `currentStep` 비교로 `.visible` 토글, 빌드 소진 후 슬라이드 전환
2. **상태 초기화 전략**: 이탈 시 `classList.remove('visible')` 즉시 리셋, 이전 슬라이드 복귀 시 `fromBack=true`로 maxStep까지 채워서 진입
3. **화자 노트 동기화**: `window.open` 팝업 + `postMessage({notes})` + 수신 시 반드시 `origin` 검증

---

## 5. 통합 추천 구조

### HTML 아키텍처

```
<body>
  ├── .progress-bar-container        // 상단 진행바
  ├── .stage#stage                   // 무대: Flexbox 중앙 정렬
  │   └── .deck#deck                 // 덱: 16:9 고정, absolute 기준점
  │       ├── section.slide.active   // 슬라이드 1 (활성)
  │       │   ├── .slide__inner      // 내부 레이아웃
  │       │   │   ├── [data-step]    // 빌드 효과 요소
  │       │   │   └── .code-block    // 코드 블록
  │       │   └── .speaker-notes    // 화자 노트 (숨김)
  │       ├── section.slide.next     // 슬라이드 2 (대기)
  │       └── ...
  ├── button.nav-btn.prev-btn        // 이전 버튼
  ├── button.nav-btn.next-btn        // 다음 버튼
  ├── .slide-counter                 // 슬라이드 번호
  └── nav.thumbnail-nav              // 썸네일 닷 네비게이터
```

> **주의**: 버튼, 카운터, 닷 네비게이터는 반드시 `.stage` 밖 `<body>` 직계 자식에 배치.
> `.stage`에 `transform`이 없어야 자식 `position:fixed`가 뷰포트 기준으로 동작.

### JavaScript 클래스 구조

```javascript
class Deck {
  constructor()           // 초기화: 슬라이드 수집, 이벤트 바인딩
  _maxStep(slide)         // 슬라이드의 최대 빌드 스텝 수
  _applyStep(slide, step) // 스텝에 따라 .visible 토글 + 코드 하이라이트
  _enterSlide(idx)        // 슬라이드 진입: 빌드 초기화, UI 갱신, URL 업데이트
  next()                  // 빌드 우선 → 소진 시 다음 슬라이드
  prev()                  // 빌드 역행 → 스텝=0 시 이전 슬라이드
  _goTo(nextIdx, dir)     // 슬라이드 전환 애니메이션 + 3중 잠금 해제
  _updateUI()             // 진행바, 번호, 썸네일 닷 갱신
  _buildThumbnails()      // 썸네일 닷 DOM 생성
  _initKeyboard()         // 키보드 이벤트 바인딩
  _initPointer()          // Pointer Events 바인딩
  _initButtons()          // 버튼 + popstate 바인딩
  _restoreFromHash()      // URL hash에서 초기 슬라이드 복원
}
```

---

## 6. 주의사항 종합

### 레이아웃

| 이슈 | 문제 | 해결책 |
|------|------|--------|
| `100vh` iOS Safari | 주소창 포함 계산되어 스크롤 발생 | `100dvh` 사용 |
| `transform` 조상 요소 | 자식 `position:fixed`가 뷰포트 대신 해당 요소 기준으로 동작 | UI 컨트롤을 `transform` 없는 요소 하위에 배치 |
| `display:none` 사용 | CSS `transition` 불가 | `opacity:0 + visibility:hidden` 조합 사용 |
| `aspect-ratio` 구형 Safari | Safari 14 이하 미지원 | `@supports not (aspect-ratio: 16/9)` 블록에 `padding-top:56.25%` 폴백 |

### 네비게이션

| 이슈 | 문제 | 해결책 |
|------|------|--------|
| `passive:true` 기본값 | `pointermove`에서 `preventDefault()` 호출 시 에러 | `{ passive: false }` 명시 |
| Space 키 기본 동작 | `preventDefault()` 누락 시 페이지 스크롤 동시 발생 | `handled=true` 경로에서 반드시 호출 |
| 진행바 계산 | `index / totalSlides` 사용 시 마지막 슬라이드 100% 미도달 | `index / (totalSlides - 1)` 사용 |
| `popstate` state 없음 | `pushState` 전 페이지에서 뒤로가기 시 `event.state`가 null | `location.hash` 파싱으로 fallback |

### 전환 애니메이션

| 이슈 | 문제 | 해결책 |
|------|------|--------|
| `transitionend` 다중 발생 | 속성 수만큼 이벤트 발생 | `e.propertyName === 'transform'`으로 필터링 |
| 강제 reflow 누락 | `classList.add` 직후 transition이 건너뜀 | `void el.offsetWidth` 삽입 |
| `transitionend` 미발생 | duration=0이거나 transition 취소 시 미발생 | `transitioncancel` + `setTimeout` fallback 병행 |
| `will-change` 남용 | 모든 슬라이드에 상시 적용 시 GPU 메모리 낭비 | 전환 직전/직후에만 토글 |

### 빌드 효과

| 이슈 | 문제 | 해결책 |
|------|------|--------|
| `animation:none` 누락 | 슬라이드 재진입 시 animation-delay 방식 재생 안 됨 | `animation:none` 적용 후 `offsetWidth` 강제 재트리거 |
| `opacity:0`만 사용 | 스크린리더가 숨긴 요소를 읽음 | `visibility:hidden` 병행 필수 |
| 빌드 전 슬라이드 전환 | 스페이스가 빌드 건너뛰고 전환 | `if (currentStep < maxStep)` 분기 우선 처리 |
| `postMessage` origin 미검증 | 다른 origin 메시지 수신 취약 | `if (e.origin !== window.opener?.location.origin) return` 필수 |
| `window.open` 팝업 차단 | 이벤트 핸들러 외부 호출 시 차단 | 키/클릭 이벤트 핸들러 내부에서만 호출 |

---

## 7. 슬라이드 비주얼 컴포넌트 패턴

> 실제 레퍼런스 이미지 분석 기반의 슬라이드별 비주얼 컴포넌트 설계 가이드.
> 슬라이드 내용을 요청받으면, 맥락에 맞는 컴포넌트를 선택해 아래 패턴으로 구현한다.

### 디자인 시스템 기본 원칙

```css
:root {
  --green:        #00C875;        /* 주요 액센트 — CTA, 강조, 활성 상태 */
  --green-dim:    rgba(0,200,117,0.12);  /* 카드/박스 배경 */
  --green-border: rgba(0,200,117,0.28); /* 카드/박스 테두리 */
  --gold:         #C6A55C;        /* 보조 액센트 — 타이틀 강조 */
  --muted:        rgba(255,255,255,0.38); /* 설명 텍스트 */
  --card-bg:      rgba(255,255,255,0.05);
  --card-border:  rgba(255,255,255,0.09);
}
```

**레이아웃 고정 규칙:**
- 슬라이드 기본 padding: `4% 8% 5%` (상단 4%, 좌우 8%, 하단 5%)
  - **핵심:** padding `%`는 항상 **가로 폭** 기준. 16:9에서 `4%` 가로 ≈ 높이의 7.1% → 타이틀이 최상단에 위치
  - **이유:** `justify-content: flex-start` + 상단 패딩 고정으로 **모든 슬라이드 타이틀이 동일 높이**에 위치
  - **금지:** padding-top 10% 이상 — width 기준 % 특성상 실제 높이의 18%+ 가 되어 타이틀이 과도하게 아래로 내려감
  - **금지:** `justify-content: center` — 슬라이드마다 콘텐츠 양이 달라 타이틀 위치가 들쭉날쭉해짐
- `justify-content: flex-start` — 콘텐츠를 상단부터 채움 (center 금지)
- 캡션 바 사용 금지 — 슬라이드 내 콘텐츠로 충분히 설명
- 콘텐츠 폭: 카드 우측 공란이 생기지 않도록 콘텐츠 길이에 맞춰 설정
  - 넘버드 스텝(steps-wrapper): **65%**
  - 프로그레스 바(progress-rows): **82%**
  - 피처 리스트(feature-list): **`fit-content` + `max-width: 68%`** + `white-space: nowrap` — 텍스트 길이에 맞게 자동 축소
  - Before/After 비교(ba-container): **72%** — 패널이 과하게 크지 않도록
- 본문 설명 카피(step-desc, feature-sub, ba-desc): `clamp(0.55rem, 0.95vw, 0.85rem)` — 타이틀 대비 충분히 작게

**레퍼런스 스크린샷 해석 규칙:**
- 사용자가 스크린샷을 첨부할 때 화면 **맨 하단의 자막/캡션 텍스트는 무시**한다
- 이는 캡처 도구(macOS 시스템 자막 등)가 자동 추가한 것이며 슬라이드 콘텐츠가 아님
- 슬라이드 내용은 화면 중앙 영역만 참고할 것

**폰트 스택:** `Noto Sans KR` (본문) · `Inter` (영문/라벨) · `JetBrains Mono` (코드)

---

### 7-1. 플로우 다이어그램 (Flow Diagram)

**언제 사용:** 사람 → 시스템 → 결과물의 흐름, 프로세스 단계, API 호출 흐름, 사용자 여정

**구조:** `[아이콘 노드]` → `[화살표]` → `[정보 박스]` → `[화살표]` → `[아이콘 노드]`

```html
<div class="flow-diagram">
  <div class="flow-node">
    <div class="flow-icon-wrap"><!-- SVG 또는 텍스트 아이콘 --></div>
    <div class="flow-node-label">사용자</div>
  </div>
  <div class="flow-arrow">→</div>
  <div class="flow-box">
    <div class="flow-box-title">AI 처리 엔진</div>
    <div class="flow-box-sub">데이터 분석 · 최적화</div>
  </div>
  <div class="flow-arrow">→</div>
  <div class="flow-node">
    <div class="flow-icon-wrap"><!-- AI 아이콘 --></div>
    <div class="flow-node-label">AI 에이전트</div>
  </div>
</div>
```

```css
.flow-diagram {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 2%;
  width: 80%;
  margin-top: 2%;
}
.flow-node {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.4em;
}
.flow-icon-wrap {
  width: clamp(44px, 7vw, 68px);
  height: clamp(44px, 7vw, 68px);
  border-radius: 50%;
  border: 1.5px solid rgba(255,255,255,0.2);
  background: rgba(255,255,255,0.04);
  display: flex;
  align-items: center;
  justify-content: center;
}
.flow-node-label {
  font-size: clamp(0.5rem, 1vw, 0.85rem);
  color: var(--muted);
  font-weight: 700;
}
.flow-arrow {
  color: var(--green);
  font-size: clamp(1rem, 2.5vw, 2rem);
  margin-bottom: 1.6em; /* 노드 라벨 공간 보정 */
  flex-shrink: 0;
}
.flow-box {
  border: 1.5px solid var(--green);
  border-radius: 10px;
  padding: 0.9em 1.4em;
  background: var(--green-dim);
  text-align: center;
  min-width: 16%;
}
.flow-box-title {
  font-size: clamp(0.65rem, 1.3vw, 1.1rem);
  font-weight: 700;
  color: var(--white);
}
.flow-box-sub {
  font-size: clamp(0.48rem, 0.85vw, 0.72rem);
  color: var(--green);
  margin-top: 0.2em;
}
```

---

### 7-2. Before / After 비교 (Split Comparison)

**언제 사용:** 솔루션 도입 전후, 기존 방식 vs 신규 방식, 문제 vs 해결 대비

**구조:** 좌(회색·흐림·✕) | 수직 구분선 | 우(초록·선명·✓)

```html
<div class="ba-container">
  <div class="ba-panel ba-before">
    <div class="ba-tag">BEFORE</div>
    <div class="ba-icon">✕</div>
    <div class="ba-label">기존 방식</div>
    <div class="ba-desc">수동 처리 · 높은 오류율<br>반복 작업으로 리소스 낭비</div>
  </div>
  <div class="ba-divider"></div>
  <div class="ba-panel ba-after">
    <div class="ba-tag">AFTER</div>
    <div class="ba-icon">✓</div>
    <div class="ba-label">AI 자동화</div>
    <div class="ba-desc">실시간 처리 · 오류 최소화<br>리소스 80% 절감</div>
  </div>
</div>
```

```css
.ba-container {
  display: flex;
  width: 80%;
  margin-top: 2%;
  border-radius: 12px;
  overflow: hidden;
  border: 1px solid rgba(255,255,255,0.08);
  min-height: 38%;
}
.ba-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 1.5em;
  gap: 0.45em;
}
.ba-before { background: rgba(255,255,255,0.03); }
.ba-after  { background: var(--green-dim); }
.ba-divider { width: 1px; background: rgba(255,255,255,0.12); flex-shrink: 0; }
.ba-tag {
  font-size: clamp(0.45rem, 0.8vw, 0.68rem);
  letter-spacing: 0.14em;
  text-transform: uppercase;
  font-weight: 700;
}
.ba-before .ba-tag { color: rgba(255,255,255,0.22); }
.ba-after  .ba-tag { color: var(--green); }
.ba-icon {
  width: clamp(30px, 4.5vw, 48px);
  height: clamp(30px, 4.5vw, 48px);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: clamp(0.85rem, 1.7vw, 1.4rem);
  font-weight: 700;
}
.ba-before .ba-icon { background: rgba(255,255,255,0.07); color: rgba(255,255,255,0.35); border: 1px solid rgba(255,255,255,0.12); }
.ba-after  .ba-icon { background: var(--green); color: #000; }
.ba-label { font-size: clamp(0.65rem, 1.2vw, 1rem); font-weight: 700; }
.ba-before .ba-label { color: rgba(255,255,255,0.4); }
.ba-after  .ba-label { color: var(--green); }
.ba-desc { font-size: clamp(0.48rem, 0.85vw, 0.72rem); color: var(--muted); text-align: center; line-height: 1.6; }
```

---

### 7-3. 넘버드 스텝 + CTA (Numbered Steps)

**언제 사용:** 도입 단계, 사용 방법, 순서가 중요한 프로세스, 단계별 가이드

**구조:** `[①] → 제목 + 설명` 순차 등장, 마지막에 CTA 버튼

```html
<div class="steps-wrapper">
  <!-- step-row와 step-connector-row를 교대로 배치 -->
  <div class="step-row" data-step="1">
    <div class="step-left-col"><div class="step-num">1</div></div>
    <div class="step-body">
      <div class="step-title">요구사항 정의</div>
      <div class="step-desc">비즈니스 목표와 KPI 수립</div>
    </div>
  </div>
  <div class="step-connector-row" data-step="2">
    <div class="step-connector-line"></div>
  </div>
  <div class="step-row" data-step="2"><!-- ... --></div>
  <!-- 반복 -->
  <button class="cta-button" data-step="5">도입 상담 시작하기 →</button>
</div>
```

```css
.steps-wrapper {
  width: 60%;
  display: flex;
  flex-direction: column;
  margin-top: 1.5%;
  align-items: flex-start;
}
.step-row {
  display: flex;
  gap: 0.85em;
  align-items: flex-start;
}
.step-left-col { display: flex; flex-direction: column; align-items: center; flex-shrink: 0; }
.step-num {
  width: clamp(20px, 2.8vw, 30px);
  height: clamp(20px, 2.8vw, 30px);
  border-radius: 50%;
  background: var(--green);
  color: #000;
  font-weight: 700;
  font-size: clamp(0.58rem, 1.05vw, 0.88rem);
  display: flex;
  align-items: center;
  justify-content: center;
}
.step-connector-row { display: flex; gap: 0.85em; }
.step-connector-line {
  width: clamp(20px, 2.8vw, 30px);
  display: flex;
  justify-content: center;
  flex-shrink: 0;
}
.step-connector-line::after {
  content: '';
  width: 2px;
  height: clamp(10px, 1.6vw, 18px);
  background: rgba(0,200,117,0.3);
  display: block;
}
.step-title { font-size: clamp(0.6rem, 1.15vw, 0.95rem); font-weight: 700; color: var(--white); line-height: 1.3; }
.step-desc  { font-size: clamp(0.46rem, 0.82vw, 0.7rem); color: var(--muted); margin-top: 0.15em; line-height: 1.5; }
.cta-button {
  margin-top: 2%;
  padding: 0.5em 1.6em;
  background: transparent;
  border: 1.5px solid var(--green);
  color: var(--green);
  border-radius: 100px;
  font-family: 'Noto Sans KR', 'Inter', sans-serif;
  font-size: clamp(0.58rem, 1.1vw, 0.9rem);
  font-weight: 700;
  cursor: pointer;
  letter-spacing: 0.04em;
}
```

---

### 7-4. 프로그레스 바 비교 (Progress Bars)

**언제 사용:** KPI 달성률, 기능별 성능 비교, 경쟁사 비교, 만족도 지표

**구조:** `라벨명 | ████████████░░░ | 85%` 행(row) 반복

**애니메이션 규칙:** `style="width:X%"` 대신 반드시 `data-target-width="X%"` 사용.
- `.progress-fill-bar`의 초기 width는 CSS에서 `0%`로 고정
- step이 visible 상태가 되면 JS `_applyStep`이 `bar.style.width = bar.dataset.targetWidth` 로 세팅
- CSS `transition: width 0.9s ...`가 자동으로 좌→우 채움 애니메이션 처리
- 슬라이드 재진입 시 `_enterSlide`에서 모든 바를 `width: 0%`로 리셋

```html
<div class="progress-rows">
  <div class="progress-row" data-step="1">
    <div class="progress-label">처리 속도</div>
    <div class="progress-track">
      <div class="progress-fill-bar green" data-target-width="92%"></div>
    </div>
    <div class="progress-pct" style="color:var(--green)">92%</div>
  </div>
  <div class="progress-row" data-step="2">
    <div class="progress-label">오류율 감소</div>
    <div class="progress-track">
      <div class="progress-fill-bar gold" data-target-width="78%"></div>
    </div>
    <div class="progress-pct" style="color:var(--gold)">78%</div>
  </div>
</div>
```

```css
.progress-rows {
  width: 68%;
  margin-top: 1.5%;
  display: flex;
  flex-direction: column;
  gap: 0.65em;
}
.progress-row {
  display: flex;
  align-items: center;
  gap: 1em;
}
.progress-label {
  font-size: clamp(0.52rem, 1vw, 0.85rem);
  font-weight: 700;
  color: var(--white);
  min-width: 22%;
  text-align: right;
  white-space: nowrap;
}
.progress-track {
  flex: 1;
  height: clamp(7px, 1.1vw, 12px);
  background: rgba(255,255,255,0.08);
  border-radius: 100px;
  overflow: hidden;
}
.progress-fill-bar {
  height: 100%;
  border-radius: 100px;
  width: 0%; /* 항상 0에서 시작 — JS가 data-target-width로 애니메이션 */
  transition: width 0.9s cubic-bezier(0.25, 0.46, 0.45, 0.94);
}
.progress-fill-bar.green { background: var(--green); }
.progress-fill-bar.gold  { background: var(--gold); }
.progress-fill-bar.dim   { background: rgba(255,255,255,0.2); }
.progress-pct {
  font-size: clamp(0.52rem, 1vw, 0.85rem);
  font-weight: 700;
  min-width: 3.2em;
  text-align: left;
}
```

> **팁:** `data-step`으로 각 행을 순차 등장시킬 수 있음. `width: 0%`에서 시작하고 `.visible`이 되면 실제 값으로 전환하면 바 애니메이션 효과.

---

### 7-5. 아이콘 피처 리스트 (Icon Feature List)

**언제 사용:** 제품/서비스 기능 소개, 핵심 역량 나열, USP(차별점) 강조

**구조:** 왼쪽 그린 보더 카드 `[아이콘] [제목 + 설명]` 반복

```html
<div class="feature-list">
  <div class="feature-item" data-step="1">
    <div class="feature-icon">⚡</div>
    <div class="feature-content">
      <div class="feature-title">실시간 데이터 처리</div>
      <div class="feature-sub">초당 10만 건 이상의 이벤트를 지연 없이 처리</div>
    </div>
  </div>
  <div class="feature-item" data-step="2"><!-- ... --></div>
</div>
```

```css
.feature-list {
  width: 68%;
  margin-top: 1.5%;
  display: flex;
  flex-direction: column;
  gap: 0.5em;
  text-align: left;
}
.feature-item {
  display: flex;
  align-items: center;
  gap: 1em;
  padding: 0.6em 1em;
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 10px;
  border-left: 3px solid var(--green);
}
.feature-icon {
  width: clamp(26px, 4vw, 42px);
  height: clamp(26px, 4vw, 42px);
  border-radius: 8px;
  background: var(--green-dim);
  border: 1px solid var(--green-border);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: clamp(0.75rem, 1.4vw, 1.15rem);
  flex-shrink: 0;
}
.feature-title { font-size: clamp(0.6rem, 1.1vw, 0.9rem); font-weight: 700; color: var(--white); line-height: 1.3; }
.feature-sub   { font-size: clamp(0.46rem, 0.82vw, 0.7rem); color: var(--muted); margin-top: 0.15em; line-height: 1.45; }
```

---

### 7-6. 투-컬럼 아이콘 비교 (Two-Column Icon Comparison)

**언제 사용:** 기능 유무 비교, 경쟁사 비교, 체크리스트 vs 비교, 지원/미지원 구분

**구조:** 그레이 원(✕ 아이콘) vs 그린 원(✓ 아이콘) — 라벨 아래에

```html
<div class="icon-compare-grid">
  <div class="icon-compare-col">
    <div class="ic-header ic-bad">기존 솔루션</div>
    <div class="ic-row">
      <div class="ic-cell ic-x">✕</div>
      <div class="ic-label">실시간 처리</div>
    </div>
    <div class="ic-row">
      <div class="ic-cell ic-x">✕</div>
      <div class="ic-label">자동 학습</div>
    </div>
  </div>
  <div class="icon-compare-col">
    <div class="ic-header ic-good">당사 솔루션</div>
    <div class="ic-row">
      <div class="ic-cell ic-check">✓</div>
      <div class="ic-label">실시간 처리</div>
    </div>
    <div class="ic-row">
      <div class="ic-cell ic-check">✓</div>
      <div class="ic-label">자동 학습</div>
    </div>
  </div>
</div>
```

```css
.icon-compare-grid {
  display: flex;
  gap: 4%;
  width: 72%;
  margin-top: 2%;
}
.icon-compare-col { flex: 1; display: flex; flex-direction: column; gap: 0.5em; }
.ic-header {
  font-size: clamp(0.55rem, 1vw, 0.85rem);
  font-weight: 700;
  padding-bottom: 0.4em;
  border-bottom: 1px solid rgba(255,255,255,0.1);
  margin-bottom: 0.3em;
}
.ic-good { color: var(--green); }
.ic-bad  { color: rgba(255,255,255,0.35); }
.ic-row {
  display: flex;
  align-items: center;
  gap: 0.6em;
}
.ic-cell {
  width: clamp(18px, 2.6vw, 26px);
  height: clamp(18px, 2.6vw, 26px);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: clamp(0.5rem, 0.9vw, 0.76rem);
  font-weight: 700;
  flex-shrink: 0;
}
.ic-check { background: var(--green); color: #000; }
.ic-x     { background: rgba(255,255,255,0.1); color: rgba(255,255,255,0.35); }
.ic-label { font-size: clamp(0.52rem, 0.95vw, 0.8rem); color: var(--muted); }
```

---

### 7-7. 슬라이드 유형 선택 가이드

| 슬라이드 목적 | 추천 컴포넌트 |
|---|---|
| 시스템/서비스 흐름 설명 | 플로우 다이어그램 |
| 도입 전후 효과 설명 | Before/After 비교 |
| 구현/도입 단계 안내 | 넘버드 스텝 + CTA |
| 성과 지표 / KPI 발표 | 프로그레스 바 비교 |
| 제품 기능 / USP 소개 | 아이콘 피처 리스트 |
| 경쟁사 비교 / 기능 유무 | 투-컬럼 아이콘 비교 |
| 개념 · 용어 설명 | 타이틀 + 부제 + 설명 텍스트 |
| 코드 · 기술 설명 | 코드 블록 + 줄 하이라이트 |

---

## 참고 파일

| 파일 | 설명 |
|------|------|
| `research-sample.html` | 모든 패턴이 적용된 최소 동작 가능한 단일 HTML 파일 (5슬라이드 — 플로우, Before/After, 스텝, 프로그레스 바, 피처 리스트 데모) |
| `RESEARCH.md` | 4개 주제의 리서치 결과 + 비주얼 컴포넌트 패턴을 통합 정리한 문서 (현재 파일) |
