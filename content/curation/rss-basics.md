---
title: "RSS: 웹의 구독 인프라"
date: 2026-04-14T23:56:16+09:00
categories: ["curation"]
tags: ["RSS", "기술", "웹", "파이프라인"]
draft: false
sources:
  - title: "RSS 2.0 Specification"
    url: "https://www.rssboard.org/rss-specification"
    publisher: "RSS Advisory Board"
  - title: "Atom Syndication Format (RFC 4287)"
    url: "https://datatracker.ietf.org/doc/html/rfc4287"
    publisher: "IETF"
  - title: "What is RSS?"
    url: "https://www.w3schools.com/xml/xml_rss.asp"
    publisher: "W3Schools"
---

매일 수십 개의 사이트를 직접 돌아다니며 새 글을 확인하는 것은 비효율적이다. RSS는 그 문제를 1990년대 말에 이미 풀었다. 지금도 대부분의 뉴스 사이트, 블로그, GitHub 저장소, 팟캐스트가 이 형식을 조용히 지원하고 있다.

---

## RSS가 뭔가

**RSS(Really Simple Syndication)**는 웹사이트가 새 콘텐츠를 표준화된 형식으로 배포하는 방법이다. 독자나 프로그램이 사이트를 직접 방문하지 않아도, 피드 URL 하나만 알고 있으면 새 글이 올라왔는지 확인할 수 있다.

비유하자면 이렇다. 관심 있는 가게 앞에 직접 매일 가보는 대신, 그 가게가 "신상 입고됐습니다"라고 문자를 보내주는 방식이다. RSS 피드가 그 문자를 자동으로 보내는 인프라다.

Netscape가 1999년 처음 공개했고, Dave Winer가 2002년 RSS 2.0으로 정리했다. 블로그 문화가 폭발하던 시기와 맞물려 빠르게 퍼졌다.

---

## 어디서 찾나

대부분의 사이트는 RSS 피드를 숨기지 않고 제공한다. 찾는 방법은 세 가지다.

**1. 관례적인 URL 시도**

```
https://example.com/feed
https://example.com/rss
https://example.com/feed.xml
https://example.com/atom.xml
```

**2. HTML 소스의 `<link>` 태그 확인**

브라우저에서 페이지 소스를 열면 `<head>` 안에 이런 줄이 있는 경우가 많다.

```html
<link rel="alternate" type="application/rss+xml" title="Feed" href="/feed.xml">
```

**3. 잘 알려진 피드 URL 예시**

| 서비스 | 피드 URL 패턴 |
|---|---|
| GitHub 저장소 릴리즈 | `https://github.com/<user>/<repo>/releases.atom` |
| GitHub 저장소 커밋 | `https://github.com/<user>/<repo>/commits.atom` |
| Hacker News | `https://news.ycombinator.com/rss` |
| YouTube 채널 | `https://www.youtube.com/feeds/videos.xml?channel_id=<ID>` |
| 대부분의 WordPress 블로그 | `https://example.com/feed` |

---

## 피드의 구조

RSS 피드는 XML 문서다. 크게 두 부분으로 나뉜다. **channel** (피드 전체 메타데이터)과 그 안의 여러 **item** (개별 글).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>sj-meliora</title>
    <link>https://sj-meliora.github.io</link>
    <description>Ad Meliora — 더 나은 것을 향하여</description>

    <item>
      <title>Ad Meliora, Again</title>
      <link>https://sj-meliora.github.io/note/ad-meliora-again/</link>
      <description>작년 이맘때쯤이었다...</description>
      <pubDate>Mon, 14 Apr 2026 23:34:39 +0900</pubDate>
      <guid>https://sj-meliora.github.io/note/ad-meliora-again/</guid>
    </item>

  </channel>
</rss>
```

각 `<item>`에서 파이프라인이 실제로 쓰는 필드:

| 필드 | 의미 | 비고 |
|---|---|---|
| `<title>` | 글 제목 | 항상 있음 |
| `<link>` | 원문 URL | 항상 있음 |
| `<description>` | 본문 요약 또는 전문 | 사이트마다 다름 |
| `<pubDate>` | 발행 일시 | 중복 방지의 기준 |
| `<guid>` | 고유 식별자 | URL과 같은 경우가 많음 |

---

## RSS 2.0 vs Atom

두 표준이 함께 쓰인다. 거의 같은 목적이지만 설계 철학이 다르다.

| 항목 | RSS 2.0 | Atom |
|---|---|---|
| 등장 | 2002 | 2005 (RFC 4287) |
| 형식 | XML | XML |
| 날짜 형식 | RFC 822 (`Mon, 14 Apr 2026`) | ISO 8601 (`2026-04-14T23:56:16+09:00`) |
| 국제화 | 제한적 | 강함 |
| 본문 필드 | `<description>` | `<content>` (더 명확) |
| 확장성 | 느슨함 | 엄격한 네임스페이스 |

실용적으로는 둘 다 지원하는 파서를 쓰면 무관하다. Python의 `feedparser` 라이브러리가 두 형식을 동일하게 처리한다.

---

## 큐레이션 파이프라인에서의 위치

이 블로그의 일일 파이프라인에서 RSS는 **수집 레이어**를 담당한다.

```
RSS 피드 구독 목록
    ↓
feedparser로 새 item 수집 (pubDate 기준 24h 이내)
    ↓
item별 title + link + description 추출
    ↓
Anthropic API로 가공 (운영자 톤으로 요약)
    ↓
content/curation/*.md 생성
```

피드 하나당 하루 수 개에서 수십 개의 item이 나온다. 파이프라인이 이걸 필터링·요약해서 읽을 만한 형태로 만드는 것이 이 블로그 curation의 핵심이다.

---

다음 주제: 수집 방법론 — RSS만으로 커버되지 않는 정보원은 어떻게 다루는가.
