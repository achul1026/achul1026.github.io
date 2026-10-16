---
layout: post
title: "no-cache는 캐시하지 말라는 뜻이 아니다"
summary: "RFC 9111을 읽고 no-cache, no-store, max-age, ETag 재검증이 각각 무엇을 요구하는지 정리한 뒤, 작은 Node.js 서버로 200과 304의 응답 크기를 쟀다."
categories: [정리]
topic: 백엔드
tags: [HTTP, 캐시, RFC9111, ETag, Node.js]
date: 2026-10-26 09:00:00 +0900
---

`Cache-Control: no-cache`를 캐시를 끄는 설정으로 알고 있었다. 이름만 보면 그렇게 읽힌다. RFC 9111(HTTP Caching)을 읽어 보니 의미가 달랐다. 이 글은 경험담이 아니라 RFC를 읽고 정리한 뒤 작은 서버로 동작을 확인한 기록이다.

## 문서가 말하는 것

RFC 9111 4.2절에 따르면 저장된 응답은 신선 수명(freshness lifetime)이 현재 나이(age)보다 길면 신선하다. 수명은 `s-maxage`(공유 캐시만), `max-age`, `Expires - Date` 순서로 정해진다(4.2.1). 이런 명시적 값이 없고 `Last-Modified`가 있으면 캐시가 경험칙으로 수명을 정할 수 있는데, 문서는 마지막 수정 이후 경과 시간의 10% 정도를 예로 든다(4.2.2).

신선하지 않은 응답은 바로 버리지 않는다. 캐시는 저장해 둔 `ETag`를 `If-None-Match`에 실어 원 서버에 조건부 요청을 보내고(4.3.1), 304 Not Modified가 오면 저장본을 갱신해 다시 쓴다(4.3.3, 4.3.4).

디렉티브는 5.2.2절에 있다. 내가 정리한 것은 이렇다.

| 디렉티브 | 요구하는 것 |
|---|---|
| `no-cache` | 저장은 해도 된다. 쓰기 전에 원 서버에 재검증해야 한다 (5.2.2.4) |
| `no-store` | 요청과 응답의 어떤 부분도 저장하지 않는다 (5.2.2.5) |
| `private` | 공유 캐시는 저장하지 않는다. 브라우저 같은 개인 캐시는 가능 (5.2.2.7) |
| `public` | 원래 저장이 금지되는 응답도 저장해도 된다고 명시 (5.2.2.9) |
| `must-revalidate` | 신선하지 않게 된 뒤에는 검증 없이 재사용하지 않는다 (5.2.2.2) |

그러니 "매번 최신인지 확인하되 안 바뀌었으면 본문은 다시 받지 않겠다"가 `no-cache` + `ETag` 조합이고, "이 응답은 어디에도 남기지 마라"가 `no-store`다. 로그인 후 개인정보 응답처럼 남으면 안 되는 것에는 `no-store`를 쓰는 게 문서의 의도에 맞다고 이해했다.

## 실습: 200과 304

Node.js 25.9.0의 `node:http`와 `node:crypto`만 썼다. 2,000개 항목짜리 JSON(약 59KB)을 세 경로로 내보낸다.

```js
const body = JSON.stringify({ items: Array.from({ length: 2000 }, (_, i) => ({ id: i, name: 'item-' + i })) });
const etag = '"' + createHash('sha256').update(body).digest('hex').slice(0, 16) + '"';
http.createServer((req, res) => {
  const h = { 'Content-Type': 'application/json' };
  if (req.url === '/etag') {
    h.ETag = etag; h['Cache-Control'] = 'no-cache';
    if (req.headers['if-none-match'] === etag) { res.writeHead(304, h); return res.end(); }
  } else if (req.url === '/maxage') h['Cache-Control'] = 'max-age=60';
  else if (req.url === '/nostore') h['Cache-Control'] = 'no-store';
  res.writeHead(200, h); res.end(body);
}).listen(3999);
```

curl로 요청하고 상태 코드와 받은 본문 크기(`size_download`)를 봤다. curl은 캐시를 두지 않으므로 캐시 동작이 아니라 서버가 보내는 응답을 확인한 것이다.

| 요청 | 상태 | 받은 본문 |
|---|---|---|
| `/etag` 첫 요청 | 200 | 59,791 byte |
| `/etag` + 맞는 `If-None-Match` | 304 | 0 byte |
| `/etag` + 틀린 `If-None-Match` | 200 | 59,791 byte |
| `/maxage`, `/nostore` | 200 | 59,791 byte (헤더만 다르다) |

재검증이 성공하면 본문 59KB가 0 byte가 된다. 대신 왕복은 여전히 한 번 일어난다. `max-age=60`은 60초 동안 왕복 자체를 없애지만 그 사이 서버 쪽 변경은 보이지 않는다. 이 트레이드오프가 두 방식을 가르는 기준이라고 정리했다. 위 수치는 로컬 호스트에서 한 번씩 요청한 값이고 지연 시간은 재지 않았다.

## 남은 것

- 실제 브라우저나 CDN이 이 헤더를 어떻게 처리하는지는 확인하지 않았다. 문서가 말하는 것과 구현 사이에 차이가 있는지 모른다.
- `Vary`, `stale-while-revalidate`, `immutable`은 읽지 않았다. 문서 끝부분 약 7,000자도 읽지 못했다.
- 해시를 매 요청 계산하지 않고 시작 시 한 번 계산했다. 동적 응답에서 ETag 계산 비용이 얼마나 드는지는 재지 않았다.

## 출처

- IETF, [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html), 2022년 6월 (4.2, 4.2.1, 4.2.2, 4.3, 5.2.2절). 2026년 10월 17일 확인.
- 실습 환경: Node.js 25.9.0, macOS, curl, 로컬 호스트.
