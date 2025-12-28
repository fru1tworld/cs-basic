# CDN (Content Delivery Network)

## 목차
1. [개요](#개요)
2. [CDN 동작 원리](#cdn-동작-원리)
3. [엣지 캐싱](#엣지-캐싱)
4. [캐시 무효화](#캐시-무효화)
5. [주요 CDN 제공업체](#주요-cdn-제공업체)
---

## 개요

CDN(Content Delivery Network)은 전 세계에 분산된 서버 네트워크를 통해 콘텐츠를 사용자에게 빠르게 전달하는 시스템입니다. 사용자와 지리적으로 가까운 서버(엣지 서버)에서 콘텐츠를 제공하여 지연 시간을 최소화합니다.

### 핵심 이점
- **지연 시간 감소**: 사용자와 가까운 서버에서 응답
- **대역폭 비용 절감**: 오리진 서버 부하 분산
- **가용성 향상**: 분산된 인프라로 장애 대응
- **DDoS 보호**: 분산된 트래픽 처리 능력
- **HTTPS 지원**: 엣지에서 TLS 종료

### CDN이 필요한 시나리오
- 전 세계 사용자를 대상으로 하는 서비스
- 대용량 정적 파일 (이미지, 동영상, JS/CSS) 제공
- 트래픽 급증 대응 (이벤트, 마케팅 캠페인)
- 스트리밍 서비스
- API 응답 캐싱

---

## CDN 동작 원리

### 기본 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CDN Architecture                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                         ┌──────────────┐                                │
│                         │    Origin    │                                │
│                         │    Server    │                                │
│                         └──────┬───────┘                                │
│                                │                                        │
│              ┌─────────────────┼─────────────────┐                      │
│              │                 │                 │                      │
│              ▼                 ▼                 ▼                      │
│    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│    │   Regional   │  │   Regional   │  │   Regional   │                │
│    │ Edge Cache   │  │ Edge Cache   │  │ Edge Cache   │                │
│    │   (US-East)  │  │   (EU-West)  │  │   (AP-Tokyo) │                │
│    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
│           │                 │                 │                        │
│     ┌─────┴─────┐     ┌─────┴─────┐     ┌─────┴─────┐                  │
│     ▼           ▼     ▼           ▼     ▼           ▼                  │
│ ┌───────┐  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐          │
│ │ Edge  │  │ Edge  │ │ Edge  │ │ Edge  │ │ Edge  │ │ Edge  │          │
│ │ PoP   │  │ PoP   │ │ PoP   │ │ PoP   │ │ PoP   │ │ PoP   │          │
│ │ NYC   │  │  LA   │ │London │ │Paris  │ │Tokyo  │ │ Seoul │          │
│ └───┬───┘  └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘          │
│     │          │         │         │         │         │              │
│     ▼          ▼         ▼         ▼         ▼         ▼              │
│   Users      Users     Users     Users     Users     Users            │
│                                                                        │
└─────────────────────────────────────────────────────────────────────────┘

PoP: Point of Presence (접속 지점)
```

### 요청 흐름

```
사용자 요청 흐름
────────────────────────────────────────────────────────

1. DNS 조회
   ┌────────┐   DNS Query    ┌────────────┐
   │ Client │ ──────────────▶│ DNS Server │
   └────────┘                └────────────┘
                                   │
                                   │ Anycast IP or
                                   │ Geolocation-based IP
                                   ▼
                             ┌────────────┐
                             │ Nearest    │
                             │ Edge Server│
                             └────────────┘

2. 엣지 서버 처리
   ┌────────┐   Request    ┌────────────┐
   │ Client │ ────────────▶│ Edge Server│
   └────────┘              └────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
              Cache HIT                 Cache MISS
                    │                         │
                    ▼                         ▼
            ┌──────────────┐         ┌──────────────┐
            │ Return from  │         │ Fetch from   │
            │ Local Cache  │         │ Origin       │
            └──────────────┘         └──────────────┘
                                           │
                                           ▼
                                    ┌──────────────┐
                                    │ Cache & Return│
                                    └──────────────┘
```

### DNS 기반 라우팅

```python
# DNS 응답 예시 (GeoDNS)
# 사용자 위치에 따라 다른 IP 반환

# 한국 사용자
$ dig cdn.example.com
cdn.example.com.  300  IN  A  203.0.113.10  # 서울 PoP

# 미국 사용자
$ dig cdn.example.com
cdn.example.com.  300  IN  A  198.51.100.20  # 뉴욕 PoP

# Anycast 방식
# 같은 IP지만 네트워크 경로에 따라 가장 가까운 서버로 라우팅
$ dig cdn.example.com
cdn.example.com.  300  IN  A  192.0.2.1  # Anycast IP
```

### 캐시 계층 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    Tiered Caching Architecture               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1: Edge PoPs (L1 Cache)                              │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐           │
│  │PoP1 │ │PoP2 │ │PoP3 │ │PoP4 │ │PoP5 │ │PoP6 │  ...      │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘           │
│     │       │       │       │       │       │               │
│     └───────┴───────┼───────┴───────┴───────┘               │
│                     │                                        │
│  Layer 2: Regional Cache (L2 Cache)                         │
│                     │                                        │
│            ┌────────┴────────┐                              │
│            │                 │                              │
│         ┌──▼──┐          ┌───▼──┐                           │
│         │US-E │          │APAC  │                           │
│         │Cache│          │Cache │                           │
│         └──┬──┘          └──┬───┘                           │
│            │                │                               │
│            └────────┬───────┘                               │
│                     │                                        │
│  Origin Shield (Optional L3)                                │
│                     │                                        │
│              ┌──────▼──────┐                                │
│              │   Origin    │                                │
│              │   Shield    │                                │
│              └──────┬──────┘                                │
│                     │                                        │
│              ┌──────▼──────┐                                │
│              │   Origin    │                                │
│              │   Server    │                                │
│              └─────────────┘                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘

장점:
- L1 미스 시 L2에서 조회 (오리진 부하 감소)
- Origin Shield로 오리진 보호
- 캐시 히트율 향상
```

---

## 엣지 캐싱

### 캐시 키 구성

```
캐시 키 = URL + Query String + Headers + Cookies (설정에 따라)

예시:
────────────────────────────────────────────────────────

기본 캐시 키:
GET /images/logo.png
Cache Key: example.com/images/logo.png

Query String 포함:
GET /api/products?category=electronics&page=1
Cache Key: example.com/api/products?category=electronics&page=1

Query String 순서 정규화:
GET /api/products?page=1&category=electronics
Cache Key: example.com/api/products?category=electronics&page=1
(정렬된 Query String)

헤더 포함 (Vary):
GET /api/data
Accept-Language: ko-KR
Cache Key: example.com/api/data:lang=ko-KR

디바이스별 캐싱:
GET /page
User-Agent: (Mobile)
Cache Key: example.com/page:device=mobile
```

### Cache-Control 헤더

```python
# 캐시 제어 헤더 예시

# 1. 정적 자산 (이미지, JS, CSS) - 장기 캐싱
Cache-Control: public, max-age=31536000, immutable
# public: CDN에서 캐싱 가능
# max-age: 1년
# immutable: 변경되지 않음 (재검증 불필요)

# 2. HTML 페이지 - 짧은 캐싱 + 재검증
Cache-Control: public, max-age=300, must-revalidate
ETag: "abc123"
# 5분 캐싱, 이후 재검증 필수

# 3. API 응답 - 조건부 캐싱
Cache-Control: public, max-age=60, stale-while-revalidate=30
# 60초 캐싱, 만료 후 30초간 오래된 응답 제공하며 백그라운드 갱신

# 4. 개인화된 콘텐츠 - CDN 캐싱 금지
Cache-Control: private, no-store
# CDN에서 캐싱 안 함, 브라우저도 저장 안 함

# 5. 민감한 데이터
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache  # HTTP/1.0 호환
```

### 응답 헤더 설정 (서버 측)

```python
# Python/Flask 예제
from flask import Flask, send_file, make_response
from functools import wraps

app = Flask(__name__)

def cache_control(max_age=3600, public=True, immutable=False):
    """캐시 제어 데코레이터"""
    def decorator(f):
        @wraps(f)
        def wrapped(*args, **kwargs):
            response = make_response(f(*args, **kwargs))

            directives = []
            if public:
                directives.append('public')
            else:
                directives.append('private')

            directives.append(f'max-age={max_age}')

            if immutable:
                directives.append('immutable')

            response.headers['Cache-Control'] = ', '.join(directives)
            return response
        return wrapped
    return decorator


@app.route('/static/<path:filename>')
@cache_control(max_age=31536000, immutable=True)
def static_files(filename):
    """정적 파일 - 1년 캐싱"""
    return send_file(f'static/{filename}')


@app.route('/api/products')
@cache_control(max_age=300, public=True)
def get_products():
    """API 응답 - 5분 캐싱"""
    products = fetch_products()
    response = make_response(jsonify(products))
    response.headers['Vary'] = 'Accept-Language'  # 언어별 캐싱
    return response


@app.route('/api/user/profile')
@cache_control(max_age=0, public=False)
def get_profile():
    """개인화된 콘텐츠 - 캐싱 안 함"""
    return jsonify(get_current_user_profile())
```

```javascript
// Node.js/Express 예제
const express = require('express');
const app = express();

// 정적 파일 캐싱
app.use('/static', express.static('public', {
    maxAge: '1y',
    immutable: true,
    etag: true
}));

// API 캐싱 미들웨어
const cacheMiddleware = (maxAge = 300) => {
    return (req, res, next) => {
        res.set({
            'Cache-Control': `public, max-age=${maxAge}`,
            'Surrogate-Control': `max-age=${maxAge}`,  // CDN 전용
            'Vary': 'Accept-Encoding'
        });
        next();
    };
};

app.get('/api/products', cacheMiddleware(300), async (req, res) => {
    const products = await getProducts();
    res.json(products);
});

// Stale-While-Revalidate 패턴
app.get('/api/feed', (req, res) => {
    res.set({
        'Cache-Control': 'public, max-age=60, stale-while-revalidate=300',
        'CDN-Cache-Control': 'max-age=60'  // Cloudflare 전용
    });
    res.json(getFeed());
});
```

### Vary 헤더와 캐시 분리

```
Vary 헤더: 같은 URL이지만 다른 버전을 캐싱해야 할 때

예시:
GET /page HTTP/1.1
Accept-Encoding: gzip

응답:
HTTP/1.1 200 OK
Content-Encoding: gzip
Vary: Accept-Encoding
Cache-Control: public, max-age=3600

결과:
- Accept-Encoding: gzip → 압축된 버전 캐싱
- Accept-Encoding: identity → 비압축 버전 캐싱
```

```python
# Vary 헤더 활용 예시

# 1. 압축 방식에 따른 캐싱
response.headers['Vary'] = 'Accept-Encoding'

# 2. 언어에 따른 캐싱
response.headers['Vary'] = 'Accept-Language'

# 3. 디바이스에 따른 캐싱 (커스텀 헤더)
response.headers['Vary'] = 'X-Device-Type'

# 4. 여러 조건 조합
response.headers['Vary'] = 'Accept-Encoding, Accept-Language'

# 주의: Vary: * 또는 너무 많은 조합은 캐시 효율 저하
```

---

## 캐시 무효화

### Purge 방식

```bash
# URL 단위 무효화

# Cloudflare API
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
     -H "Authorization: Bearer {api_token}" \
     -H "Content-Type: application/json" \
     --data '{"files":["https://example.com/image.png"]}'

# AWS CloudFront
aws cloudfront create-invalidation \
    --distribution-id EDFDVBD6EXAMPLE \
    --paths "/images/image.png" "/css/*"

# Akamai CCU API
curl -X POST "https://api.ccu.akamai.com/ccu/v3/invalidate/url/production" \
     -H "Content-Type: application/json" \
     --data '{
       "objects": [
         "https://example.com/image.png"
       ]
     }'
```

```python
# Python으로 CDN 무효화 자동화

import boto3
import requests
from datetime import datetime

class CDNPurger:
    """CDN 캐시 무효화 관리"""

    def __init__(self, provider: str, config: dict):
        self.provider = provider
        self.config = config

    def purge_urls(self, urls: list):
        """URL 목록 무효화"""
        if self.provider == 'cloudflare':
            return self._purge_cloudflare(urls)
        elif self.provider == 'cloudfront':
            return self._purge_cloudfront(urls)
        else:
            raise ValueError(f"Unknown provider: {self.provider}")

    def _purge_cloudflare(self, urls: list):
        """Cloudflare 캐시 무효화"""
        response = requests.post(
            f"https://api.cloudflare.com/client/v4/zones/{self.config['zone_id']}/purge_cache",
            headers={
                'Authorization': f"Bearer {self.config['api_token']}",
                'Content-Type': 'application/json'
            },
            json={'files': urls}
        )
        return response.json()

    def _purge_cloudfront(self, paths: list):
        """CloudFront 캐시 무효화"""
        client = boto3.client('cloudfront')

        response = client.create_invalidation(
            DistributionId=self.config['distribution_id'],
            InvalidationBatch={
                'Paths': {
                    'Quantity': len(paths),
                    'Items': paths
                },
                'CallerReference': f"purge-{datetime.now().isoformat()}"
            }
        )
        return response

    def purge_all(self):
        """전체 캐시 무효화"""
        if self.provider == 'cloudflare':
            return self._purge_cloudflare_all()
        elif self.provider == 'cloudfront':
            return self._purge_cloudfront(['/*'])

    def _purge_cloudflare_all(self):
        """Cloudflare 전체 캐시 무효화"""
        response = requests.post(
            f"https://api.cloudflare.com/client/v4/zones/{self.config['zone_id']}/purge_cache",
            headers={
                'Authorization': f"Bearer {self.config['api_token']}",
                'Content-Type': 'application/json'
            },
            json={'purge_everything': True}
        )
        return response.json()


# 배포 파이프라인에서 사용
def deploy_and_purge():
    """배포 후 CDN 무효화"""

    # 1. 새 버전 배포
    deploy_new_version()

    # 2. CDN 캐시 무효화
    purger = CDNPurger('cloudfront', {
        'distribution_id': 'EDFDVBD6EXAMPLE'
    })

    # 변경된 파일만 무효화
    changed_files = get_changed_files()
    purger.purge_urls(changed_files)

    # 또는 와일드카드 사용
    purger.purge_urls([
        '/static/js/*',
        '/static/css/*'
    ])
```

### 버전 기반 캐시 무효화

```html
<!-- 파일명에 버전/해시 포함 -->

<!-- 기존 방식 (비권장) -->
<link rel="stylesheet" href="/css/style.css">
<script src="/js/app.js"></script>

<!-- 버전 기반 (권장) -->
<link rel="stylesheet" href="/css/style.v2.3.1.css">
<script src="/js/app.a1b2c3d4.js"></script>

<!-- Query String 방식 (일부 CDN에서 무시될 수 있음) -->
<link rel="stylesheet" href="/css/style.css?v=2.3.1">
```

```python
# 빌드 시 파일 해시 추가

import hashlib
import os
import shutil
import json

def hash_file(filepath):
    """파일 해시 계산"""
    with open(filepath, 'rb') as f:
        return hashlib.md5(f.read()).hexdigest()[:8]

def build_with_hashing(src_dir, dest_dir):
    """해시된 파일명으로 빌드"""
    manifest = {}

    for root, dirs, files in os.walk(src_dir):
        for file in files:
            if file.endswith(('.js', '.css', '.png', '.jpg')):
                src_path = os.path.join(root, file)
                file_hash = hash_file(src_path)

                # 해시 추가된 파일명
                name, ext = os.path.splitext(file)
                hashed_name = f"{name}.{file_hash}{ext}"

                # 복사
                rel_path = os.path.relpath(src_path, src_dir)
                dest_path = os.path.join(dest_dir, os.path.dirname(rel_path), hashed_name)
                os.makedirs(os.path.dirname(dest_path), exist_ok=True)
                shutil.copy2(src_path, dest_path)

                # 매니페스트에 기록
                manifest[rel_path] = os.path.join(os.path.dirname(rel_path), hashed_name)

    # 매니페스트 저장
    with open(os.path.join(dest_dir, 'manifest.json'), 'w') as f:
        json.dump(manifest, f, indent=2)

    return manifest


# 템플릿에서 사용
def asset_url(path):
    """해시된 자산 URL 반환"""
    manifest = load_manifest()
    return f"/static/{manifest.get(path, path)}"

# Jinja2 템플릿
# <link rel="stylesheet" href="{{ asset_url('css/style.css') }}">
# 결과: <link rel="stylesheet" href="/static/css/style.a1b2c3d4.css">
```

### Surrogate Keys / Cache Tags

```python
# Surrogate Key (Fastly) / Cache Tag (Cloudflare) 활용
# 관련된 캐시를 그룹으로 무효화

class CacheTagManager:
    """캐시 태그 기반 무효화"""

    def __init__(self, cdn_client):
        self.cdn = cdn_client

    def tag_response(self, response, tags: list):
        """응답에 캐시 태그 추가"""
        # Cloudflare
        response.headers['Cache-Tag'] = ', '.join(tags)
        # Fastly
        response.headers['Surrogate-Key'] = ' '.join(tags)
        return response

    def purge_by_tag(self, tag: str):
        """태그로 캐시 무효화"""
        return self.cdn.purge_tag(tag)


# 사용 예제: 상품 페이지
@app.route('/products/<int:product_id>')
def product_page(product_id):
    product = get_product(product_id)

    response = make_response(render_template('product.html', product=product))

    # 캐시 태그 설정
    tags = [
        f"product-{product_id}",
        f"category-{product['category_id']}",
        f"brand-{product['brand_id']}",
        "products"
    ]

    response.headers['Cache-Tag'] = ', '.join(tags)
    response.headers['Cache-Control'] = 'public, max-age=3600'

    return response


# 상품 업데이트 시
def update_product(product_id, data):
    # DB 업데이트
    save_product(product_id, data)

    # 관련 캐시 무효화
    cache_tags = CacheTagManager(cdn_client)
    cache_tags.purge_by_tag(f"product-{product_id}")


# 카테고리 전체 업데이트 시
def update_category(category_id):
    # 해당 카테고리의 모든 상품 캐시 무효화
    cache_tags.purge_by_tag(f"category-{category_id}")
```

### 캐시 무효화 전략 비교

| 전략 | 장점 | 단점 | 사용 사례 |
|------|------|------|-----------|
| TTL 기반 | 구현 간단 | 즉시 갱신 불가 | 변경 빈도 낮은 콘텐츠 |
| URL Purge | 정확한 무효화 | API 호출 비용 | 특정 파일 업데이트 |
| 버전/해시 | 무효화 불필요 | 빌드 프로세스 필요 | 정적 자산 |
| Cache Tags | 그룹 무효화 | CDN 지원 필요 | 연관 콘텐츠 무효화 |
| 전체 Purge | 확실한 갱신 | 캐시 히트율 저하 | 긴급 상황, 대규모 변경 |

---

## 주요 CDN 제공업체

### Cloudflare

```
특징:
- 전 세계 330+ PoP
- 무료 플랜 제공
- 통합 보안 (WAF, DDoS, Zero Trust)
- Workers (엣지 컴퓨팅)
- Argo Smart Routing (성능 최적화)

장점:
- 설정 간편
- 무료로 시작 가능
- 강력한 보안 기능
- Workers로 엣지 로직 실행

단점:
- Enterprise 기능 제한
- 중국 지역 제한적
```

```python
# Cloudflare Workers 예제 (JavaScript)
"""
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)

  // A/B 테스트
  const variant = Math.random() < 0.5 ? 'A' : 'B'

  // 캐시 키에 변형 포함
  const cacheKey = new Request(url.toString() + '?variant=' + variant, request)
  const cache = caches.default

  let response = await cache.match(cacheKey)
  if (!response) {
    // 오리진에서 가져와서 캐싱
    response = await fetch(request)
    response = new Response(response.body, response)
    response.headers.set('X-Variant', variant)

    // 엣지에서 캐싱
    event.waitUntil(cache.put(cacheKey, response.clone()))
  }

  return response
}
"""

# Cloudflare API 사용
import CloudFlare

def setup_cloudflare_caching(zone_id):
    cf = CloudFlare.CloudFlare()

    # Page Rule 설정
    page_rule = {
        'targets': [
            {'target': 'url', 'constraint': {'operator': 'matches', 'value': '*example.com/static/*'}}
        ],
        'actions': [
            {'id': 'cache_level', 'value': 'cache_everything'},
            {'id': 'edge_cache_ttl', 'value': 86400},
            {'id': 'browser_cache_ttl', 'value': 31536000}
        ]
    }

    cf.zones.pagerules.post(zone_id, data=page_rule)
```

### AWS CloudFront

```
특징:
- AWS 서비스와 긴밀한 통합
- 600+ PoP (Edge + Regional Edge Cache)
- Lambda@Edge, CloudFront Functions
- Origin Shield
- S3, ALB, API Gateway 등 AWS 오리진 최적화

장점:
- AWS 생태계 통합
- 세밀한 캐시 정책 제어
- 상세한 로그/모니터링
- 비용 효율적 (AWS 트래픽 할인)

단점:
- 설정 복잡
- 무료 티어 제한적
- AWS 외부 연동 비용
```

```python
import boto3

def create_cloudfront_distribution():
    """CloudFront 배포 생성"""
    client = boto3.client('cloudfront')

    distribution_config = {
        'CallerReference': 'my-distribution-2024',
        'Origins': {
            'Quantity': 1,
            'Items': [
                {
                    'Id': 'myS3Origin',
                    'DomainName': 'my-bucket.s3.amazonaws.com',
                    'S3OriginConfig': {
                        'OriginAccessIdentity': ''
                    }
                }
            ]
        },
        'DefaultCacheBehavior': {
            'TargetOriginId': 'myS3Origin',
            'ViewerProtocolPolicy': 'redirect-to-https',
            'CachePolicyId': '658327ea-f89d-4fab-a63d-7e88639e58f6',  # CachingOptimized
            'Compress': True,
            'AllowedMethods': {
                'Quantity': 2,
                'Items': ['GET', 'HEAD']
            }
        },
        'Enabled': True,
        'Comment': 'My CDN Distribution',
        'PriceClass': 'PriceClass_100',  # 미국, 유럽만
    }

    response = client.create_distribution(
        DistributionConfig=distribution_config
    )
    return response


def create_cache_policy():
    """커스텀 캐시 정책 생성"""
    client = boto3.client('cloudfront')

    cache_policy_config = {
        'Name': 'my-cache-policy',
        'MinTTL': 1,
        'MaxTTL': 31536000,
        'DefaultTTL': 86400,
        'ParametersInCacheKeyAndForwardedToOrigin': {
            'EnableAcceptEncodingGzip': True,
            'EnableAcceptEncodingBrotli': True,
            'HeadersConfig': {
                'HeaderBehavior': 'whitelist',
                'Headers': {
                    'Quantity': 1,
                    'Items': ['Accept-Language']
                }
            },
            'CookiesConfig': {
                'CookieBehavior': 'none'
            },
            'QueryStringsConfig': {
                'QueryStringBehavior': 'whitelist',
                'QueryStrings': {
                    'Quantity': 2,
                    'Items': ['page', 'size']
                }
            }
        }
    }

    response = client.create_cache_policy(
        CachePolicyConfig=cache_policy_config
    )
    return response


# CloudFront Functions (경량 엣지 로직)
cloudfront_function_code = """
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // 기본 문서 처리
    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    }

    // URL 정규화
    request.uri = request.uri.toLowerCase();

    return request;
}
"""

# Lambda@Edge (복잡한 로직)
lambda_edge_code = """
exports.handler = async (event) => {
    const request = event.Records[0].cf.request;

    // 지역 기반 리다이렉트
    const country = request.headers['cloudfront-viewer-country'][0].value;

    if (country === 'CN') {
        return {
            status: '302',
            headers: {
                location: [{ value: 'https://china.example.com' + request.uri }]
            }
        };
    }

    return request;
};
"""
```

### Akamai

```
특징:
- 세계 최대 CDN (365,000+ 서버, 135+ 국가)
- 엔터프라이즈 중심
- 고급 보안 기능
- 미디어 스트리밍 특화
- EdgeWorkers (엣지 컴퓨팅)

장점:
- 가장 넓은 커버리지
- 최고 수준의 성능
- 전담 지원
- 고급 보안 (Kona WAF)

단점:
- 높은 비용
- 복잡한 설정
- 계약 기반
```

```python
# Akamai Property Manager API 예제
import requests
from akamai.edgegrid import EdgeGridAuth

class AkamaiCDN:
    def __init__(self, client_token, client_secret, access_token, host):
        self.auth = EdgeGridAuth(
            client_token=client_token,
            client_secret=client_secret,
            access_token=access_token
        )
        self.host = host

    def purge_urls(self, urls):
        """URL 무효화"""
        response = requests.post(
            f'https://{self.host}/ccu/v3/invalidate/url/production',
            auth=self.auth,
            json={'objects': urls}
        )
        return response.json()

    def purge_cache_tags(self, tags):
        """캐시 태그로 무효화"""
        response = requests.post(
            f'https://{self.host}/ccu/v3/invalidate/tag/production',
            auth=self.auth,
            json={'objects': tags}
        )
        return response.json()

    def get_purge_status(self, purge_id):
        """무효화 상태 확인"""
        response = requests.get(
            f'https://{self.host}/ccu/v3/purge/{purge_id}',
            auth=self.auth
        )
        return response.json()


# EdgeWorkers 코드 예제 (JavaScript)
"""
import { httpRequest } from 'http-request';
import { createResponse } from 'create-response';

export async function responseProvider(request) {
    // 오리진 요청 전 로직
    const response = await httpRequest(request.url);

    // 응답 수정
    const body = await response.text();
    const modifiedBody = body.replace('OLD_TEXT', 'NEW_TEXT');

    return createResponse(
        response.status,
        response.headers,
        modifiedBody
    );
}
"""
```

### CDN 제공업체 비교

| 특성 | Cloudflare | CloudFront | Akamai |
|------|------------|------------|--------|
| **PoP 수** | 330+ | 600+ | 365,000+ 서버 |
| **무료 플랜** | O | 제한적 | X |
| **가격** | 저렴 | 중간 | 높음 |
| **설정 난이도** | 쉬움 | 중간 | 어려움 |
| **엣지 컴퓨팅** | Workers | Lambda@Edge | EdgeWorkers |
| **AWS 통합** | 가능 | 최적화 | 가능 |
| **중국 커버리지** | 제한적 | 가능 | 우수 |
| **보안 기능** | 포함 | 별도 | 포함 |
| **지원** | 커뮤니티/유료 | AWS 지원 | 전담 팀 |
| **적합한 대상** | 스타트업~중견 | AWS 사용자 | 대기업 |

---

## 참고 자료

- [Cloudflare Docs](https://developers.cloudflare.com/cache/)
- [AWS CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [Akamai Developer Documentation](https://developer.akamai.com/)
- [MDN - HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Web.dev - HTTP Cache](https://web.dev/articles/http-cache)
- [Cloudflare - Benchmarking Edge Network Performance](https://blog.cloudflare.com/benchmarking-edge-network-performance/)
- [CDN Planet](https://www.cdnplanet.com/)
