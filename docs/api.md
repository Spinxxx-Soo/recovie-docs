# API 명세

본 문서는 **영화 시나리오 분석 기록 플랫폼(Recovie)**의 API 엔드포인트와 스펙을 정의합니다.

- Base URL: `/api/v1`
- 인증: Bearer JWT (`Authorization: Bearer <accessToken>`)
- 공통 응답 규칙:
  - 성공: 2xx + JSON
  - 실패: 4xx/5xx + 에러 JSON

---

## 1. 공통 에러 응답 포맷

```json
{
  "timestamp": "2026-02-10T12:00:00+09:00",
  "path": "/api/v1/movies",
  "status": 400,
  "error": "Bad Request",
  "message": "query 파라미터 형식이 올바르지 않습니다.",
  "requestId": "req_2f3c..."
}
```

* `status`: HTTP 상태코드
* `message`: 사용자/개발자 확인용 메시지
* `requestId`: 로그 추적용(선택)

---

## 2. Auth

### 2.1 회원가입

* **URL**: `POST /auth/signup`
* **Auth**: 없음
* **Request**

```json
{
  "email": "user@recovie.com",
  "password": "P@ssw0rd!",
  "nickname": "밍"
}
```

* **Response (201)**

```json
{
  "userId": 12,
  "email": "user@recovie.com",
  "nickname": "밍",
  "createdAt": "2026-02-10T12:10:00+09:00"
}
```

* **Status Codes**: `201`, `400`, `409`

---

### 2.2 로그인

* **URL**: `POST /auth/login`
* **Auth**: 없음
* **Request**

```json
{
  "email": "user@recovie.com",
  "password": "P@ssw0rd!"
}
```

* **Response (200)**

```json
{
  "accessToken": "jwt_access_token",
  "refreshToken": "jwt_refresh_token",
  "expiresIn": 3600
}
```

* **Status Codes**: `200`, `401`, `400`

---

### 2.3 로그아웃(선택)

* **URL**: `POST /auth/logout`
* **Auth**: 필요
* **Request**: 없음
* **Response (204)**: Body 없음
* **Status Codes**: `204`, `401`

---

## 3. Movies (비로그인 조회 가능)

### 3.1 영화 목록 조회 (검색/페이징)

* **URL**: `GET /movies`

* **Auth**: 없음

* **Query Params**

  * `query` (string, optional): 제목 검색
  * `genre` (string, optional): 장르 필터(예: `Drama`)
  * `sort` (string, optional): `popular | recent | release_date | analysis_count`
  * `page` (int, optional, default=1)
  * `size` (int, optional, default=20, max=50)

* **Response (200)**

```json
{
  "page": 1,
  "size": 20,
  "total": 1842,
  "items": [
    {
      "movieId": 101,
      "externalMovieId": "tmdb:550",
      "title": "Fight Club",
      "releaseDate": "1999-10-15",
      "posterUrl": "https://...",
      "genres": ["Drama"],
      "analysisCount": 57
    }
  ]
}
```

* **Status Codes**: `200`, `400`, `500`

---

### 3.2 영화 상세 조회

* **URL**: `GET /movies/{movieId}`

* **Auth**: 없음

* **Path Params**

  * `movieId` (int, required)

* **Response (200)**

```json
{
  "movieId": 101,
  "externalMovieId": "tmdb:550",
  "title": "Fight Club",
  "originalTitle": "Fight Club",
  "overview": "....",
  "runtime": 139,
  "releaseDate": "1999-10-15",
  "posterUrl": "https://...",
  "backdropUrl": "https://...",
  "genres": ["Drama"],
  "credits": {
    "director": "David Fincher",
    "cast": ["Brad Pitt", "Edward Norton"]
  },
  "stats": {
    "analysisCount": 57,
    "topTags": ["theme", "character", "structure"]
  },
  "lastSyncedAt": "2026-02-10T02:00:00+09:00"
}
```

* **Status Codes**: `200`, `404`, `500`

---

### 3.3 (선택) 영화 외부 API 동기화 트리거(관리자)

* **URL**: `POST /admin/movies/sync`
* **Auth**: 필요(관리자 권한)
* **Request**

```json
{
  "source": "TMDB",
  "mode": "INITIAL",
  "pages": 5
}
```

* `mode`: `INITIAL | UPDATE`
* **Response (202)**

```json
{
  "jobId": "sync_20260210_001",
  "status": "QUEUED"
}
```

* **Status Codes**: `202`, `401`, `403`, `400`, `500`

---

## 4. Analyses (로그인 필요: 저장/수정/삭제)

### 4.1 분석 작성

* **URL**: `POST /movies/{movieId}/analyses`
* **Auth**: 필요
* **Request**

```json
{
  "title": "1막 전환점 분석",
  "content": "주인공의 결핍이 ...",
  "visibility": "PRIVATE",
  "tags": ["structure", "character"],
  "rating": 4
}
```

* `visibility`: `PUBLIC | PRIVATE | UNLISTED`
* **Response (201)**

```json
{
  "analysisId": 9001,
  "movieId": 101,
  "userId": 12,
  "title": "1막 전환점 분석",
  "visibility": "PRIVATE",
  "createdAt": "2026-02-10T12:20:00+09:00"
}
```

* **Status Codes**: `201`, `400`, `401`, `404`, `500`

---

### 4.2 내 분석 목록 조회 (페이징)

* **URL**: `GET /me/analyses`

* **Auth**: 필요

* **Query Params**

  * `movieId` (int, optional)
  * `visibility` (string, optional): `PUBLIC|PRIVATE|UNLISTED`
  * `page` (int, optional, default=1)
  * `size` (int, optional, default=20)

* **Response (200)**

```json
{
  "page": 1,
  "size": 20,
  "total": 120,
  "items": [
    {
      "analysisId": 9001,
      "movieId": 101,
      "movieTitle": "Fight Club",
      "title": "1막 전환점 분석",
      "visibility": "PRIVATE",
      "tags": ["structure", "character"],
      "updatedAt": "2026-02-10T12:22:00+09:00"
    }
  ]
}
```

* **Status Codes**: `200`, `401`, `400`, `500`

---

### 4.3 분석 상세 조회 (본인 또는 공개만)

* **URL**: `GET /analyses/{analysisId}`

* **Auth**: 선택(비로그인은 PUBLIC만 가능)

* **Path Params**

  * `analysisId` (int)

* **Response (200)**

```json
{
  "analysisId": 9001,
  "movieId": 101,
  "movieTitle": "Fight Club",
  "userId": 12,
  "nickname": "밍",
  "title": "1막 전환점 분석",
  "content": "....",
  "visibility": "PRIVATE",
  "tags": ["structure", "character"],
  "rating": 4,
  "createdAt": "2026-02-10T12:20:00+09:00",
  "updatedAt": "2026-02-10T12:22:00+09:00"
}
```

* **Status Codes**:

  * `200`
  * `401` (비로그인/토큰없음인데 PRIVATE 접근)
  * `403` (로그인했지만 소유자 아님 + PRIVATE)
  * `404`
  * `500`

---

### 4.4 분석 수정

* **URL**: `PUT /analyses/{analysisId}`
* **Auth**: 필요
* **Request**

```json
{
  "title": "수정된 제목",
  "content": "수정된 내용...",
  "visibility": "UNLISTED",
  "tags": ["theme", "structure"],
  "rating": 5
}
```

* **Response (200)**

```json
{
  "analysisId": 9001,
  "updatedAt": "2026-02-10T12:30:00+09:00"
}
```

* **Status Codes**: `200`, `400`, `401`, `403`, `404`, `500`

---

### 4.5 분석 삭제

* **URL**: `DELETE /analyses/{analysisId}`
* **Auth**: 필요
* **Response (204)**: Body 없음
* **Status Codes**: `204`, `401`, `403`, `404`, `500`

---

## 5. Stats / Discovery (비로그인 조회 가능)

### 5.1 Top 분석 영화 (플랫폼 전체)

* **URL**: `GET /stats/top-analyzed-movies`

* **Auth**: 없음

* **Query Params**

  * `limit` (int, optional, default=10, max=50)

* **Response (200)**

```json
{
  "items": [
    {
      "movieId": 101,
      "title": "Fight Club",
      "analysisCount": 57
    }
  ]
}
```

* **Status Codes**: `200`, `400`, `500`

---

### 5.2 카테고리별 영화 모음

* **URL**: `GET /collections`

* **Auth**: 없음

* **Query Params**

  * `type` (string, required): `genre | tag | theme`
  * `key` (string, required): 예) `Drama`, `structure`
  * `page` (int, optional, default=1)
  * `size` (int, optional, default=20)

* **Response (200)**

```json
{
  "type": "genre",
  "key": "Drama",
  "page": 1,
  "size": 20,
  "total": 320,
  "items": [
    {
      "movieId": 101,
      "title": "Fight Club",
      "releaseDate": "1999-10-15",
      "posterUrl": "https://...",
      "analysisCount": 57
    }
  ]
}
```

* **Status Codes**: `200`, `400`, `500`

---

## 6. Recommendation (오늘의 추천 / 개인화)

### 6.1 오늘의 분석 영화 추천 (비로그인/로그인 모두)

* **URL**: `GET /recommendations/today`
* **Auth**: 선택
* **Response (200)**

```json
{
  "date": "2026-02-10",
  "strategy": "hybrid_trending_personalized",
  "items": [
    {
      "movieId": 501,
      "title": "Interstellar",
      "reason": "당신이 자주 분석한 테마(시간/구조)와 유사",
      "score": 0.87
    }
  ]
}
```

* **Status Codes**: `200`, `500`

---

### 6.2 개인화 추천 (로그인 필요)

* **URL**: `GET /recommendations/me`

* **Auth**: 필요

* **Query Params**

  * `limit` (int, optional, default=20, max=50)

* **Response (200)**

```json
{
  "strategy": "embedding_hybrid",
  "items": [
    {
      "movieId": 777,
      "title": "The Dark Knight",
      "reason": "분석한 키워드(갈등/윤리)와 유사",
      "score": 0.81
    }
  ]
}
```

* **Status Codes**: `200`, `401`, `400`, `500`

---

## 7. (추가 제안) 공개 분석 피드(커뮤니티 기능, 선택)

> 추후 확장 시 “PUBLIC 분석 공유/검색”이 플랫폼 매력 포인트가 됩니다.

### 7.1 공개 분석 목록

* **URL**: `GET /analyses/public`
* **Auth**: 없음
* **Query Params**

  * `query` (string, optional): 제목/내용 검색
  * `movieId` (int, optional)
  * `tag` (string, optional)
  * `page`, `size`
* **Status Codes**: `200`, `400`, `500`

---

## 8. 상태 코드 정리

* `200 OK`: 조회/수정 성공
* `201 Created`: 생성 성공
* `202 Accepted`: 비동기 작업 큐 등록 성공(동기화 트리거)
* `204 No Content`: 삭제/로그아웃 성공
* `400 Bad Request`: 파라미터/바디 검증 실패
* `401 Unauthorized`: 인증 필요/토큰 만료
* `403 Forbidden`: 권한 없음(소유자 아님/관리자 아님)
* `404 Not Found`: 리소스 없음
* `409 Conflict`: 중복(회원가입 이메일 등)
* `500 Internal Server Error`: 서버 오류
