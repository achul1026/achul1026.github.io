---
layout: post
title: "JWT 검증에서 alg를 토큰에 맡기면 생기는 일"
summary: "RFC 7519와 RFC 8725를 읽고, 헤더의 alg를 그대로 믿는 검증기와 알고리즘을 고정한 검증기를 직접 만들어 위조 토큰 4종을 넣어 봤다."
categories: [정리]
topic: 보안
tags: [JWT, RFC8725, 인증, Node.js]
date: 2026-10-19 09:00:00 +0900
---

JWT를 쓰면서 라이브러리의 `verify`가 알아서 다 해 준다고 생각해 왔다. 그런데 RFC 8725(JWT Best Current Practices)를 읽어 보니 위협 목록의 첫 줄이 서명 검증이 허술한 경우다. 이 글은 RFC 7519와 8725를 읽고 정리한 뒤, 작은 검증기를 두 가지로 만들어 돌려 본 기록이다. 경험담이 아니라 문서를 읽고 실습한 글이다.

## 문서가 말하는 것

RFC 8725 2.1절은 두 가지 공격을 든다. 하나는 헤더의 `alg`를 `none`으로 바꾸는 것이다. 서명이 없는 토큰을 그대로 받아 주는 라이브러리가 있었다고 한다. 다른 하나는 `RS256`을 `HS256`으로 바꾸고, 서버가 가진 RSA 공개키를 HMAC 비밀키처럼 쓰게 만드는 것이다. 공개키는 말 그대로 공개되어 있으니 공격자도 같은 키로 서명할 수 있다.

대응은 3.1절에 있다. 라이브러리는 호출하는 쪽이 허용 알고리즘 집합을 지정할 수 있어야 하고, 그 밖의 알고리즘은 쓰면 안 된다. 키 하나는 알고리즘 하나에만 쓴다. 3.2절은 `none`을 TLS 같은 다른 수단으로 보호되는 경우에만 쓰라고 한다.

RFC 7519는 `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`를 등록 클레임으로 정의하지만 전부 선택 사항이다. 어떤 걸 필수로 볼지는 애플리케이션이 정한다. `exp`를 처리하려면 현재 시각이 `exp`보다 앞서야 하고, 시계 오차는 몇 분 정도 허용해도 된다고 한다. `aud`가 있는데 내가 거기 없으면 거부해야 한다(7519 4.1.3). RFC 8725 3.9절은 한 발급자가 여러 수신자용 토큰을 발급하면 `aud`를 반드시 넣고 검증하라고 한다.

## 실습: 검증기 두 개

Node.js 25.9.0의 `node:crypto`만 썼다. 라이브러리 없이 검증 로직을 직접 쓰면 어디서 믿음이 들어가는지 보인다.

나쁜 쪽은 토큰 헤더의 `alg`를 읽어서 그 알고리즘으로 검증한다.

```js
function verifyNaive(token, key) {
  const [h, p, s] = token.split('.');
  const header = JSON.parse(Buffer.from(h, 'base64url'));
  const input = `${h}.${p}`;
  if (header.alg === 'none') return JSON.parse(Buffer.from(p, 'base64url'));
  if (header.alg === 'HS256') {
    const exp = createHmac('sha256', key).update(input).digest('base64url');
    if (exp !== s) throw new Error('bad signature');
  } else if (header.alg === 'RS256') {
    if (!createVerify('RSA-SHA256').update(input).verify(key, s, 'base64url')) throw new Error('bad signature');
  }
  return JSON.parse(Buffer.from(p, 'base64url'));
}
```

나은 쪽은 허용 알고리즘을 호출하는 쪽이 고정하고, `iss`, `aud`, `exp`를 확인한다.

```js
function verifyStrict(token, publicKey, { alg, iss, aud }) {
  const parts = token.split('.');
  if (parts.length !== 3) throw new Error('malformed');
  const [h, p, s] = parts;
  const header = JSON.parse(Buffer.from(h, 'base64url'));
  if (header.alg !== alg) throw new Error(`alg not allowed: ${header.alg}`);
  if (!createVerify('RSA-SHA256').update(`${h}.${p}`).verify(publicKey, s, 'base64url')) throw new Error('bad signature');
  const c = JSON.parse(Buffer.from(p, 'base64url'));
  if (c.iss !== iss) throw new Error('bad iss');
  if (![].concat(c.aud ?? []).includes(aud)) throw new Error('bad aud');
  if (typeof c.exp !== 'number' || Date.now() / 1000 >= c.exp) throw new Error('expired or no exp');
  return c;
}
```

RSA 2048 키 쌍을 만들고 토큰 5개를 넣었다. 위조 토큰은 `sub`를 `admin`으로 바꿨다.

| 토큰 | 나쁜 검증기 | 나은 검증기 |
|---|---|---|
| 정상 RS256 | 통과 (sub=u1) | 통과 (sub=u1) |
| `alg=none`, 서명 없음 | 통과 (sub=admin) | 거부 (alg not allowed: none) |
| `HS256`, 공개키 PEM으로 서명 | 통과 (sub=admin) | 거부 (alg not allowed: HS256) |
| `aud`가 다른 서비스 | 통과 | 거부 (bad aud) |
| `exp` 지남 | 통과 | 거부 (expired) |

나쁜 검증기는 위조 토큰 2개와 `aud`가 다른 토큰, 만료된 토큰을 전부 받아 줬다. 2.1절에서 읽은 공격이 문서에만 있는 이야기가 아니라 이 정도 코드로 재현된다는 점이 인상적이었다. 다만 이건 내가 일부러 허술하게 쓴 검증기다. 실제 라이브러리가 어떻게 동작하는지는 이 실습으로 알 수 없고, 어떤 라이브러리의 어느 버전이 취약했는지도 이 글에서는 다루지 않는다.

## 남은 것

- 키 하나를 알고리즘 하나에만 쓰라는 3.1절의 요구는 위 코드에서 `alg`를 고정하는 것으로 갈음했다. 키 객체에 알고리즘을 묶는 방식은 해 보지 않았다.
- 3.11절의 `typ` 헤더 구분, 3.12절의 서로 다른 JWT 종류 간 검증 규칙 분리는 구현하지 않았다.
- 시계 오차 허용(leeway)은 넣지 않았다. 몇 분이 적당한지는 문서에 정해져 있지 않다.
- 실제 서비스에서 어떤 라이브러리를 쓰고 있는지에 따라 `verify`에 넘겨야 할 옵션이 다르다. 쓰는 라이브러리 문서에서 허용 알고리즘 옵션이 있는지 먼저 확인해야겠다.

## 출처

- IETF, [RFC 8725: JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html), 2020년 2월, BCP 225 (2.1, 3.1, 3.2, 3.9, 3.11, 3.12절). 2026년 10월 11일 확인.
- IETF, [RFC 7519: JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519.html) (4.1절 등록 클레임, 6절 Unsecured JWT, 7.2절 검증). 2026년 10월 11일 확인.
- 실습 환경: Node.js 25.9.0, macOS, `node:crypto`만 사용.
