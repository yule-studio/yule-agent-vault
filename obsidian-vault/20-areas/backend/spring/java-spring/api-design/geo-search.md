---
title: "위치 기반 검색 (당근 스타일) — PostGIS / Redis Geo"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - geo
  - postgis
---

# 위치 기반 검색 (당근 스타일) — PostGIS / Redis Geo

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | PostGIS / Redis Geo / 동네 인증 / 거리 정렬 |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가

```
POST /api/v1/me/location           # 내 동네 설정 (좌표 + 동네 검증)
GET  /api/v1/items/nearby          # 내 위치 반경 N km 의 아이템
GET  /api/v1/users/nearby          # 근처 사용자 (개인정보 + 정확도 낮춤)
GET  /api/v1/neighborhoods/{id}    # 동네 (행정동) 정보
```

### 1.1 비기능

- **WGS84 (EPSG:4326)** 좌표계 표준
- **반경 검색** 1~10km 일반
- **거리 정렬** + 페이지네이션
- **개인정보 보호** — 정확 좌표 노출 X, 동네 단위 (행정동)
- **PostGIS 지원** PostgreSQL extension
- **Redis Geo** 옵션 — 핫 데이터 빠른 검색

---

## 2. 도구 비교 — PostGIS vs Redis Geo

| | PostGIS | Redis Geo |
| --- | --- | --- |
| latency | ms | μs |
| 영속 | ✅ | RDB AOF 의존 |
| 쿼리 풍부 | 매우 | 단순 (radius / box) |
| 데이터 크기 | 대형 OK | RAM 제약 |
| 트랜잭션 | ACID | 단일 명령 atomic |
| 사용처 | 진실의 원천 | 실시간 검색 캐시 |

**권장**: PostGIS 가 진실 + Redis Geo 가 hot path (동네별 활성 아이템).

---

## 3. DB 스키마 (PostGIS)

### 3.1 extension + 컬럼

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

-- 행정동 (동네)
CREATE TABLE neighborhoods (
    id          CHAR(26) PRIMARY KEY,
    code        VARCHAR(10) NOT NULL,        -- 행정동 코드 (예: 1168010100)
    name        VARCHAR(100) NOT NULL,       -- "삼성동"
    sido        VARCHAR(50) NOT NULL,        -- "서울특별시"
    sigungu     VARCHAR(50) NOT NULL,        -- "강남구"
    center      GEOGRAPHY(POINT, 4326) NOT NULL,
    boundary    GEOGRAPHY(POLYGON, 4326),     -- 경계 (옵션)
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_neighborhoods_code ON neighborhoods (code);
CREATE INDEX ix_neighborhoods_center ON neighborhoods USING GIST (center);

-- 사용자 위치
CREATE TABLE user_locations (
    user_id          CHAR(26) PRIMARY KEY REFERENCES users(id),
    neighborhood_id  CHAR(26) NOT NULL REFERENCES neighborhoods(id),
    coordinate       GEOGRAPHY(POINT, 4326) NOT NULL,
    verified_at      TIMESTAMPTZ NOT NULL,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_user_locations_coord ON user_locations USING GIST (coordinate);
CREATE INDEX ix_user_locations_neighborhood ON user_locations (neighborhood_id);

-- 아이템 (예: 당근 중고거래 글)
CREATE TABLE items (
    id              CHAR(26) PRIMARY KEY,
    seller_id       CHAR(26) NOT NULL REFERENCES users(id),
    title           VARCHAR(100) NOT NULL,
    price_krw       BIGINT NOT NULL,
    coordinate      GEOGRAPHY(POINT, 4326) NOT NULL,
    neighborhood_id CHAR(26) NOT NULL REFERENCES neighborhoods(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'SELLING',     -- SELLING / RESERVED / SOLD
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_items_coord ON items USING GIST (coordinate);
CREATE INDEX ix_items_neighborhood_status ON items (neighborhood_id, status, created_at DESC);
```

**GIST 인덱스** 가 핵심 — 공간 쿼리 (within / nearby) 가속.

### 3.2 좌표 입력

```sql
-- WKT (Well-Known Text)
INSERT INTO items (coordinate, ...) VALUES (ST_GeogFromText('POINT(127.0276 37.4979)'), ...);

-- 또는 좌표 함수
INSERT INTO items (coordinate, ...) VALUES (ST_MakePoint(127.0276, 37.4979)::geography, ...);
-- ⚠️ 순서: (longitude, latitude) — 위경도 반대!
```

> **함정 (좌표 순서)**: `POINT(lng lat)` — 경도 먼저. 일반적 표기 (lat, lng) 와 반대.

---

## 4. 반경 검색 — `ST_DWithin`

```sql
SELECT id, title, price_krw, ST_Distance(coordinate, :origin) AS distance_m
FROM items
WHERE status = 'SELLING'
  AND ST_DWithin(coordinate, :origin, 5000)       -- 5km 이내
ORDER BY coordinate <-> :origin                    -- KNN — GIST 인덱스 활용
LIMIT 20;
```

```java
// JPA Native query
@Query(value = """
    SELECT i.id AS id, i.title AS title, i.price_krw AS priceKrw,
           ST_Distance(i.coordinate,
             ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')')) AS distanceM
    FROM items i
    WHERE i.status = 'SELLING'
      AND ST_DWithin(i.coordinate,
            ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')'), :radiusM)
    ORDER BY i.coordinate <->
             ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')')
    LIMIT :limit
    """, nativeQuery = true)
List<ItemNearbyView> findNearby(@Param("lat") double lat, @Param("lng") double lng,
                                @Param("radiusM") int radiusM, @Param("limit") int limit);
```

**`ST_DWithin`** vs **`ST_Distance < X`**:
- `ST_DWithin` 이 인덱스 활용 → **반드시 ST_DWithin** 으로 필터링
- `ST_Distance` 는 SELECT 의 distance 계산 / ORDER BY 만

---

## 5. 동네 인증 (Onboarding)

당근의 핵심 — 사용자가 실제로 그 동네에 있어야 글 작성 가능.

```java
@Service
@RequiredArgsConstructor
public class VerifyNeighborhoodUseCase {

    private final UserLocationRepository userLocations;
    private final NeighborhoodRepository neighborhoods;
    private final Clock clock;

    @Transactional
    public NeighborhoodInfo handle(UserId userId, double lat, double lng) {
        // 1. 좌표가 어느 동네에 속하는지
        var neighborhood = neighborhoods.findContaining(lng, lat)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND, "해당 위치의 동네를 찾을 수 없습니다."));

        // 2. user_locations upsert
        userLocations.upsert(userId, neighborhood.id(), lng, lat, Instant.now(clock));

        return new NeighborhoodInfo(neighborhood.id().value(), neighborhood.name(),
            neighborhood.sido(), neighborhood.sigungu());
    }
}

// repository — boundary 가 있으면 ST_Contains
@Query(value = """
    SELECT * FROM neighborhoods
    WHERE ST_Contains(boundary::geometry,
            ST_GeomFromText('POINT(' || :lng || ' ' || :lat || ')', 4326))
    LIMIT 1
    """, nativeQuery = true)
Optional<NeighborhoodEntity> findContaining(double lng, double lat);
```

GPS 좌표 → 행정동 매핑은:
- 공공데이터포털 의 **행정구역 경계 데이터** (PostGIS 로 import)
- 또는 **Kakao / Naver 지도 API** 의 reverse geocoding (외부 호출)

당근은 GPS + 거리 확인 (예: 동네 인증 시 사용자가 동 중심에서 1km 내) + 정기 재인증 (2개월).

---

## 6. Redis Geo (옵션 — 핫 데이터)

```java
@Service
@RequiredArgsConstructor
public class RedisGeoIndex {

    private final RedisTemplate<String, String> redis;

    public void addItem(ItemId id, double lng, double lat) {
        redis.opsForGeo().add("items:geo", new Point(lng, lat), id.value());
    }

    public List<NearbyItem> nearby(double lng, double lat, double radiusKm, int limit) {
        var circle = new Circle(new Point(lng, lat),
            new Distance(radiusKm, Metrics.KILOMETERS));
        var args = RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
            .includeDistance().includeCoordinates().sortAscending().limit(limit);
        var results = redis.opsForGeo().radius("items:geo", circle, args);
        if (results == null) return List.of();
        return results.getContent().stream()
            .map(r -> new NearbyItem(
                r.getContent().getName(),
                r.getDistance().getValue(),
                r.getContent().getPoint().getX(), r.getContent().getPoint().getY()))
            .toList();
    }

    public void remove(ItemId id) { redis.opsForGeo().remove("items:geo", id.value()); }
}
```

→ DB 의 진실 + Redis hot index 양쪽 동기화. 도메인 이벤트 listener:

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void on(ItemCreated event) { redisGeo.addItem(event.itemId(), event.lng(), event.lat()); }

@TransactionalEventListener(phase = AFTER_COMMIT)
public void on(ItemDeleted event) { redisGeo.remove(event.itemId()); }
```

---

## 7. Controller

```java
@RestController
@RequestMapping("/api/v1")
@PreAuthorize("isAuthenticated()")
@RequiredArgsConstructor
public class LocationController {

    private final VerifyNeighborhoodUseCase verifyNeighborhood;
    private final ItemQueryService items;

    @PostMapping("/me/location")
    public ResponseEntity<CommonResponse<NeighborhoodInfo>> setLocation(
        @Valid @RequestBody SetLocationRequest req, Authentication auth
    ) {
        var info = verifyNeighborhood.handle(new UserId(auth.getName()), req.lat(), req.lng());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, info, "동네 인증 완료"));
    }

    @GetMapping("/items/nearby")
    public ResponseEntity<CommonResponse<List<ItemNearbyView>>> nearby(
        @RequestParam @DecimalMin("33") @DecimalMax("43") double lat,
        @RequestParam @DecimalMin("124") @DecimalMax("132") double lng,
        @RequestParam(defaultValue = "5000") @Min(100) @Max(20000) int radiusM,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) int limit
    ) {
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            items.findNearby(lat, lng, radiusM, limit), "조회 성공"));
    }
}

public record SetLocationRequest(
    @DecimalMin("33") @DecimalMax("43") double lat,
    @DecimalMin("124") @DecimalMax("132") double lng
) {}
```

> 한국 좌표 범위 (lat 33~43, lng 124~132) 로 입력 검증. 다른 좌표 = 잘못된 요청.

---

## 8. 개인정보 / privacy

- 사용자 좌표 정확히 노출 X — **동네 단위만**
- 아이템 검색 시 **거리 100m 단위 반올림** 또는 동네 이름만 표시
- 좌표 자체는 DB 에 저장 (검색용) 이지만 응답에는 동네 + "약 500m" 처럼

```java
public record ItemNearbyView(String id, String title, long priceKrw, double distanceM) {
    public String distanceLabel() {
        if (distanceM < 100) return "100m 이내";
        if (distanceM < 1000) return Math.round(distanceM / 100) * 100 + "m";
        return String.format("%.1fkm", distanceM / 1000);
    }
}
```

---

## 9. 함정 모음

### 함정 1 — 좌표 순서
PostGIS `POINT(lng lat)` — 경도 먼저. 코드 컨벤션에 `(lat, lng)` 사용했다면 변환 메서드 통일.

### 함정 2 — `geometry` vs `geography`
`geometry` = 평면 좌표계 (km 단위 미적용), `geography` = 구면 (지구 표면 거리). **`geography` 권장**.

### 함정 3 — GIST 인덱스 없음
`ST_DWithin` 이 full scan. **반드시 GIST**.

### 함정 4 — `ST_Distance < X` 로 필터
인덱스 무용. **`ST_DWithin`**.

### 함정 5 — 너무 큰 radius
20km+ = 결과 수만. 운영 한계 (10km / 100건).

### 함정 6 — 정확 좌표 노출
스토킹 위험. **동네 + 거리 라벨**.

### 함정 7 — 매번 reverse geocoding 외부 호출
비용 / latency. **행정동 경계 PostGIS import + 캐시**.

### 함정 8 — Redis Geo 와 DB drift
동기화 누락 시 검색 결과 ≠ 실제. 정기 reconcile job.

### 함정 9 — 좌표 입력 검증 없음
악의적 사용자가 음수 좌표 등. lat/lng 범위 검증.

### 함정 10 — 좌표 정밀도
6자리 (1m 정밀도) 면 충분. 더 많은 정밀도 = 무의미한 차이.

---

## 10. 운영 체크리스트

- [ ] PostGIS extension 설치
- [ ] `geography(POINT, 4326)` 사용
- [ ] GIST 인덱스
- [ ] `ST_DWithin` 으로 필터 (`ST_Distance` 아님)
- [ ] 좌표 범위 입력 검증
- [ ] 응답엔 동네 + 거리 라벨 (좌표 X)
- [ ] 동네 인증 정기 재확인 (2개월)
- [ ] Redis Geo (hot) ↔ PostGIS (진실) 동기화 job

---

## 11. 관련

- [[product-search]] — 검색과 결합 (위치 + 카테고리)
- [[cache-redis]] — Redis Geo hot index
- [[../common/security-config]]
- [[api-design|↑ api-design hub]]
