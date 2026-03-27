---
layout: post
title:  "Cloudflare WAF Custom Rules로 백엔드 API 보안 강화하기"
slug: "cloudflare-waf-custom-rules-api-security"
date:   2026-03-27 00:00:00 +0900
categories: blog
---
<style>a, li, code { word-break: break-all; }</style>

<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-121955159-1"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-121955159-1');
</script>
<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
<!-- fureweb-github -->
<ins class="adsbygoogle"
     style="display:block"
     data-ad-client="ca-pub-6234418861743010"
     data-ad-slot="8427857156"
     data-ad-format="auto"></ins>
<script>
(adsbygoogle = window.adsbygoogle || []).push({});
</script>

<div class="fb-like" data-href="https://fureweb-com.github.io{{page.url}}" data-layout="button_count" data-action="like" data-size="small" data-show-faces="true" data-share="true"></div>

---

백엔드 API 서버를 운영하다 보면 자동화된 취약점 스캐너의 요청이 끊임없이 들어옵니다. `.env`, `docker-compose.yml`, `aws.env.json`, `.git/config` 같은 설정 파일을 탐색하는 요청들인데, 서비스와 무관한 경로임에도 서버가 매번 처리하고 404를 응답해야 합니다.

이 글에서는 **Cloudflare의 무료 WAF Custom Rules**를 활용하여 이런 악성 요청을 서버에 도달하기 전에 차단하는 방법을 다룹니다.

## 문제: 자동화된 취약점 스캐닝

서버 로그를 확인하면 아래와 같은 요청이 반복적으로 들어오는 것을 볼 수 있습니다.

```
GET /app/docker-compose.yml
GET /aws-codecommit/
GET /docker/overlay/config.json
GET /opt/mailcow-dockerized/mailcow.conf
GET /aws.env.json
GET /serverless.yml
GET /sam-template.yaml
GET /attacker/docker-compose.yml
GET /amplify/team-provider-info.json
GET /.git/config
GET /.env
```

이들은 대부분 VPS나 호스팅 서버에서 실행되는 자동화된 스캐너입니다. 설정 파일이 실수로 노출된 서버를 찾아 API 키, 데이터베이스 자격 증명 등을 탈취하려는 목적입니다.

서버에서 이 요청들을 하나하나 처리하면 불필요한 리소스가 소모됩니다. 더 좋은 방법은 **서버 앞단에서 차단**하는 것입니다.

## 전제 조건

- Cloudflare에 도메인이 등록되어 있어야 합니다.
- API 서버의 DNS 레코드가 **Proxied(주황색 구름)** 상태여야 합니다.

Cloudflare 대시보드에서 **DNS** 페이지를 열고 API 서버 레코드를 확인합니다.

| Type | Name | Proxy status |
|------|------|-------------|
| A | api | Proxied (주황색 구름) |

회색 구름(DNS only)이면 클릭하여 주황색으로 변경합니다. Proxied 상태에서만 Cloudflare의 보안 기능이 적용됩니다.

## 방법 1: 서버 측 경로 화이트리스트

Cloudflare 설정 전에, 서버 자체에서도 방어 계층을 추가하는 것이 좋습니다. API 서버에는 정해진 경로만 존재하므로, 허용된 경로 외에는 즉시 차단합니다.

Node.js(Fastify) 예시:

```js
const ALLOWED_PREFIXES = ['/keys', '/auth', '/billing', '/admin', '/users', '/health']

fastify.addHook('onRequest', async (request, reply) => {
  const url = request.url.split('?')[0]

  if (!ALLOWED_PREFIXES.some(p => url === p || url.startsWith(p + '/'))) {
    reply.code(404).header('connection', 'close').send()
    return
  }
})
```

이 훅은 라우팅 전에 실행되므로, Fastify가 라우트 매칭을 시도하지 않아 처리 비용이 최소화됩니다. `connection: close` 헤더로 연결도 즉시 종료합니다.

Express라면 미들웨어로 동일하게 구현할 수 있습니다.

```js
app.use((req, res, next) => {
  const url = req.path
  if (!ALLOWED_PREFIXES.some(p => url === p || url.startsWith(p + '/'))) {
    return res.status(404).set('connection', 'close').end()
  }
  next()
})
```

이것만으로도 서버 측 방어는 되지만, 요청이 서버까지 도달한다는 점은 변하지 않습니다.

## 방법 2: Cloudflare WAF Custom Rules

Cloudflare의 Custom Rules를 사용하면 요청이 **서버에 도달하기 전에** 차단할 수 있습니다. Free 플랜에서 **5개까지 무료**로 사용 가능합니다.

### 설정 방법

1. Cloudflare 대시보드 접속
2. **Security > Security rules** 이동
3. **Create rule > Custom rules** 선택

### 규칙: API 경로 외 모든 요청 차단

가장 효과적인 방식은 허용할 경로만 명시하고 나머지를 모두 차단하는 것입니다.

**Rule name:**
```
Allow only API paths
```

**Edit expression** 클릭 후 아래 내용을 붙여넣습니다:

```
(http.host eq "api.example.com" and not starts_with(http.request.uri.path, "/keys") and not starts_with(http.request.uri.path, "/auth") and not starts_with(http.request.uri.path, "/billing") and not starts_with(http.request.uri.path, "/admin") and not starts_with(http.request.uri.path, "/users") and not starts_with(http.request.uri.path, "/health"))
```

`api.example.com`은 실제 API 도메인으로 변경하고, 경로 목록은 서비스의 실제 API 경로에 맞게 수정합니다.

**Choose action:** `Block`

**Deploy** 클릭으로 즉시 적용됩니다.

### Expression 구조 설명

```
(
  http.host eq "api.example.com"          // API 도메인에만 적용
  and not starts_with(uri.path, "/keys")  // /keys/* 허용
  and not starts_with(uri.path, "/auth")  // /auth/* 허용
  ...                                     // 나머지 허용 경로
)
```

조건을 모두 만족하는 요청, 즉 **API 도메인이면서 허용된 경로가 아닌 요청**이 Block 대상이 됩니다.

### 규칙이 적용되면

스캐너의 요청은 Cloudflare 엣지에서 즉시 차단됩니다.

```
GET /docker-compose.yml     → Cloudflare에서 Block (서버 미도달)
GET /.env                   → Cloudflare에서 Block (서버 미도달)
GET /aws.env.json           → Cloudflare에서 Block (서버 미도달)
GET /keys/abc123            → 서버로 정상 전달
GET /auth/me                → 서버로 정상 전달
```

서버 로그에 스캐너 요청이 더 이상 나타나지 않고, 서버 리소스도 절약됩니다.

## 추가 규칙 예시

Custom Rules는 5개까지 사용할 수 있으므로, 필요에 따라 추가 규칙을 만들 수 있습니다.

### 특정 국가 차단

서비스 대상이 아닌 국가에서 오는 악성 요청을 차단합니다.

```
(http.host eq "api.example.com" and ip.geoip.country in {"RU" "CN"})
```

### 특정 User-Agent 차단

알려진 스캐너 도구의 User-Agent를 차단합니다.

```
(http.host eq "api.example.com" and (
  http.user_agent contains "sqlmap" or
  http.user_agent contains "nikto" or
  http.user_agent contains "dirbuster"
))
```

## 서버 + Cloudflare 이중 방어

서버 측 화이트리스트와 Cloudflare WAF를 함께 사용하면 이중 방어가 됩니다.

```
[스캐너 요청]
    ↓
[Cloudflare WAF] → Block (대부분 여기서 차단)
    ↓ (허용된 경로)
[서버 화이트리스트] → 2차 검증
    ↓ (통과)
[API 라우터] → 정상 처리
```

Cloudflare를 우회하여 서버 IP로 직접 접근하는 경우를 대비해 서버 측 방어도 유지하는 것이 좋습니다. 서버 방화벽(iptables, ufw 등)에서 Cloudflare IP 대역만 허용하면 직접 접근 자체를 차단할 수도 있습니다.

## Cloudflare Free 플랜 보안 기능 정리

| 기능 | Free | Pro ($20/월) |
|------|------|-------------|
| Custom Rules | 5개 | 20개 |
| Rate Limiting Rules | 1개 | 2개 |
| Managed Rules (자동 WAF) | X | O |
| Bot Management | X | O |

Free 플랜의 Custom Rules 5개만으로도 기본적인 API 보안은 충분히 구성할 수 있습니다.

## 마무리

백엔드 API 서버를 운영한다면 자동화된 스캐너 요청은 피할 수 없습니다. 서버에서 일일이 처리하기보다, Cloudflare 같은 CDN/WAF를 앞단에 두고 엣지에서 차단하는 것이 훨씬 효율적입니다.

Cloudflare Free 플랜만으로도 Custom Rules를 통해 API 경로 화이트리스트를 구성할 수 있고, 서버 측 화이트리스트와 함께 사용하면 이중 방어가 완성됩니다. 설정에 드는 시간은 몇 분이지만, 서버 리소스 절약과 보안 강화 효과는 상당합니다.
