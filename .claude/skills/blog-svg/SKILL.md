---
name: blog-svg
description: "Use this skill when creating SVG diagrams or illustrations for the sj-meliora blog. Covers the exact color palette, CSS structure, dark/light mode handling, file placement, and markdown embedding rules. Trigger phrases: 'SVG 그려줘', 'SVG 만들어줘', 'diagram 만들어줘', 'illustration 그려줘', 'static/images에 넣을', or any request for a diagram to include in a blog post."
---

# sj-meliora Blog SVG Guide

블로그 포스트에 삽입할 SVG를 만들 때 따라야 하는 규칙 전체를 담는다.
Claude Code와 claude.ai 양쪽에서 참조할 수 있도록 자급자족하는 문서다.

---

## 1. 왜 이 규칙이 필요한가

블로그는 PaperMod 테마 기반 Hugo 사이트로, 라이트/다크 모드 토글을 지원한다.
SVG를 `<img>` 태그로 로드하면 부모 문서의 CSS가 SVG 내부에 적용되지 않는다.
그래서 SVG는 **자체 `<style>` 블록 안에** 두 모드의 색상을 모두 하드코딩해야 한다.

다크 모드 감지는 두 경로를 동시에 지원한다:
- `@media (prefers-color-scheme: dark)` — OS 설정 기반 (img 태그로 로드 시 작동)
- `.dark .class-name` — 부모 문서의 `.dark` 클래스 기반 (인라인 SVG 사용 시 작동)

---

## 2. 색상 팔레트

### 라이트/다크 공통 구조

색상은 **의미 단위(quadrant)** 로 묶어 정의한다.
각 quadrant는 `fill`(배경), `stroke`(테두리), `title`(굵은 텍스트), `sub`(보조 텍스트) 네 가지 역할을 가진다.

### 기본 4색 팔레트 (amber / teal / gray / blue)

```css
/* ── Light mode (default) ── */
.q-amber-fill   { fill:   #FAEEDA; }
.q-amber-stroke { stroke: #BA7517; }
.q-amber-title  { fill:   #854F0B; }
.q-amber-sub    { fill:   #BA7517; }

.q-teal-fill    { fill:   #E1F5EE; }
.q-teal-stroke  { stroke: #0F6E56; }
.q-teal-title   { fill:   #0F6E56; }
.q-teal-sub     { fill:   #1D9E75; }

.q-gray-fill    { fill:   #F1EFE8; }
.q-gray-stroke  { stroke: #888780; }
.q-gray-title   { fill:   #5F5E5A; }
.q-gray-sub     { fill:   #888780; }

.q-blue-fill    { fill:   #E6F1FB; }
.q-blue-stroke  { stroke: #185FA5; }
.q-blue-title   { fill:   #0C447C; }
.q-blue-sub     { fill:   #185FA5; }

/* dot / label (포인트 마커 + 레이블) */
.dot-amber, .label-amber { fill: #712B13; }
.dot-teal,  .label-teal  { fill: #085041; }
.dot-blue,  .label-blue  { fill: #0C447C; }

/* 축 / 캡션 */
.axis       { stroke: #5F5E5A; }
.axis-label, .caption { fill: #5F5E5A; }
```

```css
/* ── Dark mode via OS preference ── */
@media (prefers-color-scheme: dark) {
  .q-amber-fill   { fill:   #633806; }
  .q-amber-stroke { stroke: #FAC775; }
  .q-amber-title  { fill:   #FAEEDA; }
  .q-amber-sub    { fill:   #FAC775; }

  .q-teal-fill    { fill:   #085041; }
  .q-teal-stroke  { stroke: #9FE1CB; }
  .q-teal-title   { fill:   #E1F5EE; }
  .q-teal-sub     { fill:   #9FE1CB; }

  .q-gray-fill    { fill:   #2C2C2A; }
  .q-gray-stroke  { stroke: #B4B2A9; }
  .q-gray-title   { fill:   #D3D1C7; }
  .q-gray-sub     { fill:   #B4B2A9; }

  .q-blue-fill    { fill:   #0C447C; }
  .q-blue-stroke  { stroke: #B5D4F4; }
  .q-blue-title   { fill:   #E6F1FB; }
  .q-blue-sub     { fill:   #B5D4F4; }

  .dot-amber, .label-amber { fill: #FAC775; }
  .dot-teal,  .label-teal  { fill: #9FE1CB; }
  .dot-blue,  .label-blue  { fill: #B5D4F4; }

  .axis       { stroke: #B4B2A9; }
  .axis-label, .caption { fill: #B4B2A9; }
}
```

```css
/* ── Dark mode via .dark class (인라인 SVG 대응) ── */
.dark .q-amber-fill   { fill:   #633806; }
.dark .q-amber-stroke { stroke: #FAC775; }
.dark .q-amber-title  { fill:   #FAEEDA; }
.dark .q-amber-sub    { fill:   #FAC775; }

.dark .q-teal-fill    { fill:   #085041; }
.dark .q-teal-stroke  { stroke: #9FE1CB; }
.dark .q-teal-title   { fill:   #E1F5EE; }
.dark .q-teal-sub     { fill:   #9FE1CB; }

.dark .q-gray-fill    { fill:   #2C2C2A; }
.dark .q-gray-stroke  { stroke: #B4B2A9; }
.dark .q-gray-title   { fill:   #D3D1C7; }
.dark .q-gray-sub     { fill:   #B4B2A9; }

.dark .q-blue-fill    { fill:   #0C447C; }
.dark .q-blue-stroke  { stroke: #B5D4F4; }
.dark .q-blue-title   { fill:   #E6F1FB; }
.dark .q-blue-sub     { fill:   #B5D4F4; }

.dark .dot-amber, .dark .label-amber { fill: #FAC775; }
.dark .dot-teal,  .dark .label-teal  { fill: #9FE1CB; }
.dark .dot-blue,  .dark .label-blue  { fill: #B5D4F4; }

.dark .axis       { stroke: #B4B2A9; }
.dark .axis-label, .dark .caption { fill: #B4B2A9; }
```

### 테마 배경색 참조 (SVG 안에서 배경 rect를 그릴 때)

| 모드 | 배경(`--theme`) | 카드(`--entry`) |
|------|----------------|----------------|
| Light | `rgb(255, 255, 255)` | `rgb(255, 255, 255)` |
| Dark  | `rgb(29, 30, 32)`    | `rgb(46, 46, 51)`    |

SVG 자체는 보통 배경 없이(투명) 만든다. 배경 rect가 필요하다면 위 색상을 `<style>` 블록에 클래스로 추가해 사용한다.

---

## 3. SVG 기본 골격

모든 SVG는 아래 구조를 따른다.

```xml
<svg xmlns="http://www.w3.org/2000/svg"
     width="680" height="540"
     viewBox="0 0 680 540"
     font-family="system-ui, -apple-system, 'Segoe UI', sans-serif">
<defs>
  <!-- 화살표 마커 등 재사용 요소 -->
  <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5"
          markerWidth="6" markerHeight="6" orient="auto-start-reverse">
    <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke"
          stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
  </marker>
  <style><![CDATA[
    /* ── Light mode (default) ── */
    .q-amber-fill   { fill:   #FAEEDA; }
    .q-amber-stroke { stroke: #BA7517; }
    .q-amber-title  { fill:   #854F0B; }
    .q-amber-sub    { fill:   #BA7517; }

    .q-teal-fill    { fill:   #E1F5EE; }
    .q-teal-stroke  { stroke: #0F6E56; }
    .q-teal-title   { fill:   #0F6E56; }
    .q-teal-sub     { fill:   #1D9E75; }

    .q-gray-fill    { fill:   #F1EFE8; }
    .q-gray-stroke  { stroke: #888780; }
    .q-gray-title   { fill:   #5F5E5A; }
    .q-gray-sub     { fill:   #888780; }

    .q-blue-fill    { fill:   #E6F1FB; }
    .q-blue-stroke  { stroke: #185FA5; }
    .q-blue-title   { fill:   #0C447C; }
    .q-blue-sub     { fill:   #185FA5; }

    .dot-amber, .label-amber { fill: #712B13; }
    .dot-teal,  .label-teal  { fill: #085041; }
    .dot-blue,  .label-blue  { fill: #0C447C; }

    .axis        { stroke: #5F5E5A; }
    .axis-label, .caption { fill: #5F5E5A; }

    /* ── Dark mode via OS preference ── */
    @media (prefers-color-scheme: dark) {
      .q-amber-fill   { fill:   #633806; }
      .q-amber-stroke { stroke: #FAC775; }
      .q-amber-title  { fill:   #FAEEDA; }
      .q-amber-sub    { fill:   #FAC775; }

      .q-teal-fill    { fill:   #085041; }
      .q-teal-stroke  { stroke: #9FE1CB; }
      .q-teal-title   { fill:   #E1F5EE; }
      .q-teal-sub     { fill:   #9FE1CB; }

      .q-gray-fill    { fill:   #2C2C2A; }
      .q-gray-stroke  { stroke: #B4B2A9; }
      .q-gray-title   { fill:   #D3D1C7; }
      .q-gray-sub     { fill:   #B4B2A9; }

      .q-blue-fill    { fill:   #0C447C; }
      .q-blue-stroke  { stroke: #B5D4F4; }
      .q-blue-title   { fill:   #E6F1FB; }
      .q-blue-sub     { fill:   #B5D4F4; }

      .dot-amber, .label-amber { fill: #FAC775; }
      .dot-teal,  .label-teal  { fill: #9FE1CB; }
      .dot-blue,  .label-blue  { fill: #B5D4F4; }

      .axis        { stroke: #B4B2A9; }
      .axis-label, .caption { fill: #B4B2A9; }
    }

    /* ── Dark mode via .dark class (인라인 SVG 대응) ── */
    .dark .q-amber-fill   { fill:   #633806; }
    .dark .q-amber-stroke { stroke: #FAC775; }
    .dark .q-amber-title  { fill:   #FAEEDA; }
    .dark .q-amber-sub    { fill:   #FAC775; }

    .dark .q-teal-fill    { fill:   #085041; }
    .dark .q-teal-stroke  { stroke: #9FE1CB; }
    .dark .q-teal-title   { fill:   #E1F5EE; }
    .dark .q-teal-sub     { fill:   #9FE1CB; }

    .dark .q-gray-fill    { fill:   #2C2C2A; }
    .dark .q-gray-stroke  { stroke: #B4B2A9; }
    .dark .q-gray-title   { fill:   #D3D1C7; }
    .dark .q-gray-sub     { fill:   #B4B2A9; }

    .dark .q-blue-fill    { fill:   #0C447C; }
    .dark .q-blue-stroke  { stroke: #B5D4F4; }
    .dark .q-blue-title   { fill:   #E6F1FB; }
    .dark .q-blue-sub     { fill:   #B5D4F4; }

    .dark .dot-amber, .dark .label-amber { fill: #FAC775; }
    .dark .dot-teal,  .dark .label-teal  { fill: #9FE1CB; }
    .dark .dot-blue,  .dark .label-blue  { fill: #B5D4F4; }

    .dark .axis        { stroke: #B4B2A9; }
    .dark .axis-label, .dark .caption { fill: #B4B2A9; }
  ]]></style>
</defs>

<!-- SVG 내용 -->
</svg>
```

### 주요 규칙

- `width`/`height`는 픽셀 단위로 명시. 기본 권장 크기는 **680 × 540** (또는 내용에 맞게 조정)
- `font-family`는 반드시 `system-ui, -apple-system, 'Segoe UI', sans-serif` 사용 — 블로그 본문 폰트와 일치
- `<style>` 블록은 반드시 `<![CDATA[ ... ]]>` 래퍼로 감싼다
- 색상은 element에 `fill`/`stroke` 속성으로 직접 쓰지 않고 **클래스**로 적용
- `stroke="context-stroke"` 패턴은 마커(화살촉)에서만 사용

---

## 4. 다이어그램 유형별 팁

### 2축 행렬 (Quadrant chart)

- 4개 사각형에 각각 amber / teal / gray / blue를 배치
- `rx="4"` 로 모서리 살짝 둥글게
- 축 화살표는 `marker-end="url(#arrow)"` 사용
- 텍스트 위계: title(14px, weight 500) → sub(12px)

### 흐름도 / 레이어 다이어그램

- 박스는 `rect` + `rx="4"` + quadrant 색상
- 연결선은 `line` 또는 `path` + `marker-end="url(#arrow)"`
- 레이어 구분 시 gray → teal → blue 순서(덜 강조 → 더 강조)

### 비교 다이어그램

- 좌/우 또는 상/하 배치로 두 옵션 비교
- 한쪽은 amber, 반대쪽은 teal 또는 blue

---

## 5. 파일 배치 및 마크다운 삽입

### 파일 저장 위치

```
static/images/<파일명>.svg
```

파일명 규칙: 글의 슬러그와 연결되는 소문자 kebab-case
예: `lsp-agent-stack.svg`, `path-cost-diagram.svg`

### 마크다운 삽입 구문

```markdown
![대체 텍스트](/images/<파일명>.svg)
```

Hugo는 `unsafe = true` 옵션으로 설정되어 있으므로 필요 시 raw `<img>` 태그도 가능하나,
기본은 마크다운 이미지 구문을 사용한다.

### 가운데 정렬 (PaperMod 지원 구문)

```markdown
![대체 텍스트](/images/<파일명>.svg#center)
```

