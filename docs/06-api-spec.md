# 1차 MVP REST API 초기 계약

이 문서는 1차 MVP의 요청·성공·오류 계약을 구현 가능한 수준으로 제시하는 초기 기준선이다. 업무 규칙과 권한은 확정하고, URI·길이·페이지·편의 필드는 초기안으로 둔다. Java DTO, 예외 클래스, JPA 매핑, Repository 쿼리와 동시성 해법은 확정하지 않는다.

관련 기준:

- [결정 기록](./01-decisions.md)
- [요구사항](./02-requirements.md)
- [ERD](./04-erd.md)
- [인증·인가](./05-auth.md)

## 1. 공통 계약

### 1.1 URI와 성공 응답

- URI는 `/api` 아래 복수형 리소스명을 사용하는 초기안이다.
- 모집 마감·재개와 매치 취소는 의미가 분명한 명령 URI에 `POST`를 사용한다.
- 성공 응답은 공통 `data` 래퍼 없이 리소스 또는 Page 본문을 직접 반환하는 추천안이다.
- 조회는 `200 OK`, 생성은 `201 Created`, 본문 없는 명령은 `204 No Content`를 기본으로 한다.
- 매치 개설은 `Location: /api/matches/{id}`를 반환한다.
- 1차 MVP에는 비동기 처리가 없으므로 `202 Accepted`를 사용하지 않는다.

### 1.2 JSON, enum과 시간

- JSON 필드명은 lower camel case를 사용한다.
- enum 값은 영문 대문자 SNAKE_CASE 문자열을 사용한다.
- 날짜·시간은 ISO 8601 offset date-time 문자열을 주고받는 초기안이다.

```json
"2026-08-15T10:00:00+09:00"
```

DB 저장 타입과 기준 시간대는 열린 결정이다.

### 1.3 검증 분류

| 입력 문제 | 초기 응답 |
|---|---|
| 깨진 JSON 또는 읽을 수 없는 본문 | `400 INVALID_REQUEST` |
| 필수 필드 누락·null·빈 문자열·공백·형식·범위 오류 | `400 VALIDATION_FAILED` |
| 잘못된 매치 시간 범위 | `400 MATCH_INVALID_TIME_RANGE` |
| 존재하지 않는 리소스 | `404 *_NOT_FOUND` |
| 현재 상태와 명령의 충돌 | `409`와 도메인 오류 코드 |

`PATCH`에서는 미전송 필드를 변경하지 않는다. `instructions` 제거에는 명시적 `null`을 허용하는 초기안이며, 다른 필드의 `null`은 검증 오류다.

문자열 길이 초기안:

- `title`: 최대 100자
- `instructions`: 최대 1000자
- `nickname`: 최대 30자
- `maxParticipants`: 최소 1, 상한은 열린 결정

### 1.4 페이지 계약

모든 목록 응답은 다음 Page 형태를 사용하는 추천안이다.

```json
{
  "content": [],
  "page": 0,
  "size": 20,
  "totalElements": 0,
  "totalPages": 0,
  "first": true,
  "last": true
}
```

| 항목 | 초기안 |
|---|---|
| 시작 번호 | 0 |
| 기본 size | 20 |
| 최대 size | 100 |
| 매치 기본 정렬 | `startAt ASC, id ASC` |
| 형태 | 전체 건수를 포함하는 Page |

Page와 Slice, 최대 size와 정렬은 실제 조회 구현과 실행 계획을 확인한 뒤 조정할 수 있다.

## 2. 응답 모델

### 2.1 `MemberSummary`

| 필드 | JSON 타입 | null | 설명 |
|---|---|---:|---|
| `id` | number | 불가 | 회원 ID |
| `email` | string | 불가 | 본인 응답에서만 제공 |
| `nickname` | string | 불가 | 표시명 |
| `role` | string | 불가 | `USER`, `ADMIN` 후보 |

### 2.2 `VenueResponse`

| 필드 | JSON 타입 | null | 설명 |
|---|---|---:|---|
| `id` | number | 불가 | 구장 ID |
| `name` | string | 불가 | 구장명 |
| `regionCode` | string | 가능 | 일반화된 지역 식별값 초기안 |
| `address` | string | 불가 | 주소 |
| `description` | string | 가능 | 구장 안내 |

`regionCode`는 1차 MVP에서 닫힌 enum으로 확정하지 않는다. 문서 예시 값은 형식 예시일 뿐 경기도 전용 모델을 뜻하지 않는다.

### 2.3 `CourtResponse`

| 필드 | JSON 타입 | null | 설명 |
|---|---|---:|---|
| `id` | number | 불가 | 코트 ID |
| `venueId` | number | 불가 | 소속 구장 ID |
| `name` | string | 불가 | 코트명 |
| `description` | string | 가능 | 코트 안내 |

매치 응답 안의 코트 요약은 소속 구장 요약을 함께 포함할 수 있다.

### 2.4 `MatchSummaryResponse`

내 참가 매치 등 축약 목록에서 사용하는 초기 모델이다.

| 필드 | JSON 타입 | null | 설명 |
|---|---|---:|---|
| `id` | number | 불가 | 매치 ID |
| `title` | string | 불가 | 제목 |
| `status` | string | 불가 | Match 상태 |
| `recruitmentStatus` | string | 불가 | 모집 상태 |
| `startAt`, `endAt` | string | 불가 | 일정 |
| `remainingSlots` | number | 불가 | 남은 자리 |
| `displayStatus` | string | 불가 | 화면 표시 상태 |

### 2.5 `MatchResponse`

| 필드 | JSON 타입 | null | 저장/파생 | 설명 |
|---|---|---:|---|---|
| `id` | number | 불가 | 저장 | 매치 ID |
| `host` | object | 불가 | 조합 | 주최자 `id`, `nickname` |
| `court` | object | 불가 | 조합 | 코트와 구장 요약 |
| `title` | string | 불가 | 저장 | 제목 |
| `instructions` | string | 가능 | 저장 | 안내 사항 |
| `recommendedSkill` | string | 불가 | 저장 | 권장 실력 |
| `status` | string | 불가 | 저장 | `SCHEDULED`, `CANCELLED` |
| `recruitmentStatus` | string | 불가 | 저장 | `OPEN`, `CLOSED` |
| `startAt`, `endAt` | string | 불가 | 저장 | 일정 |
| `maxParticipants` | number | 불가 | 저장 | 최대 정원 |
| `confirmedParticipants` | number | 불가 | 저장/파생 미정 | 활성 참가자 수 |
| `remainingSlots` | number | 불가 | 파생 | 남은 자리 |
| `displayStatus` | string | 불가 | 파생 | 화면 표시 상태 |
| `joinable` | boolean | 불가 | 파생 | 매치 자체가 지금 참가 가능한 상태인지 |
| `hostParticipates` | boolean | 불가 | 파생 | 주최자가 `CONFIRMED`인지 |
| `currentMemberParticipating` | boolean | 불가 | 파생 | 현재 회원이 `CONFIRMED`인지 |
| `currentMemberHost` | boolean | 불가 | 파생 | 현재 회원이 주최자인지 |
| `canJoin` | boolean | 불가 | 파생 | 현재 인증 회원이 참가·재참가할 수 있는지 |
| `canCancelParticipation` | boolean | 불가 | 파생 | 현재 인증 회원이 참가를 취소할 수 있는지 |
| `canEdit`, `canCancelMatch` | boolean | 불가 | 파생 | 현재 인증 회원이 주최자 명령을 수행할 수 있는지 |

`joinable`은 회원 관계와 무관한 매치 상태다.

```text
status == SCHEDULED
AND recruitmentStatus == OPEN
AND now < startAt
AND remainingSlots > 0
```

`canJoin`은 `joinable`에 현재 회원의 인증·참가 상태를 추가로 반영한다. 비회원에게는 `joinable=true`일 수 있지만 `canJoin=false`다.

`displayStatus` 초기 우선순위:

1. `CANCELLED`: `status == CANCELLED`
2. `ENDED`: `now >= endAt`
3. `IN_PROGRESS`: `startAt <= now < endAt`
4. `CLOSED`: `recruitmentStatus == CLOSED`
5. `FULL`: `remainingSlots == 0`
6. `AVAILABLE`: 위 조건에 해당하지 않는 예정 매치

### 2.6 `ParticipationResponse`

| 필드 | JSON 타입 | null | 설명 |
|---|---|---:|---|
| `id` | number | 불가 | 참가 관계 ID |
| `matchId` | number | 불가 | 매치 ID |
| `memberId` | number | 불가 | 회원 ID |
| `status` | string | 불가 | 참가 상태 |
| `createdAt`, `updatedAt` | string | 불가 | 생성·변경 시각 |

### 2.7 `ParticipatingMatchItem`

내 참가 매치 목록의 항목 모델이다.

| 필드 | JSON 타입 | null | 설명 |
|---|---|---:|---|
| `participation` | object | 불가 | `ParticipationResponse` |
| `match` | object | 불가 | `MatchSummaryResponse` |

## 3. 공통 오류 응답

```json
{
  "timestamp": "2026-08-02T10:00:00Z",
  "status": 409,
  "code": "MATCH_FULL",
  "message": "남은 자리가 없습니다.",
  "path": "/api/matches/42/participations",
  "fieldErrors": []
}
```

검증 오류:

```json
{
  "timestamp": "2026-08-02T10:00:00Z",
  "status": 400,
  "code": "VALIDATION_FAILED",
  "message": "요청 값이 올바르지 않습니다.",
  "path": "/api/matches",
  "fieldErrors": [
    {
      "field": "title",
      "rejectedValue": "",
      "message": "제목은 비어 있을 수 없습니다."
    }
  ]
}
```

원칙:

- `code`는 클라이언트 분기용 안정적인 대문자 SNAKE_CASE 식별자다.
- `message`는 변경 가능한 한국어 설명이다.
- `fieldErrors`가 없으면 빈 배열을 사용한다.
- 비밀번호·세션·CSRF 값은 `rejectedValue`로 반환하지 않는다.
- 스택 트레이스, SQL, 테이블명과 내부 예외 클래스는 노출하지 않는다.
- 예상하지 못한 오류는 내부 정보를 감춘 `500 INTERNAL_SERVER_ERROR`로 반환한다.
- `requestId` 또는 `traceId`는 운영 단계의 열린 결정이다.

## 4. API 목록

| ID | Method | URI | 접근 | 성공 초기안 |
|---|---|---|---|---|
| `MEM-001` | POST | `/api/members` | 공개 | 201 + 회원 본문 |
| `MEM-002` | PATCH | `/api/members/me/profile` | 로그인 | 200 + 회원 본문 |
| `AUTH-001` | POST | `/api/auth/login` | 공개 | 200 + 현재 회원 |
| `AUTH-002` | POST | `/api/auth/logout` | 로그인 | 204 |
| `AUTH-003` | GET | `/api/auth/me` | 로그인 | 200 + 현재 회원 |
| `AUTH-004` | GET | `/api/auth/csrf` | 공개 후보 | 204 + `XSRF-TOKEN` 쿠키 |
| `VEN-001` | GET | `/api/venues` | 공개 | 200 + Page |
| `VEN-002` | GET | `/api/venues/{venueId}` | 공개 | 200 + 구장 |
| `VEN-003` | GET | `/api/venues/{venueId}/courts` | 공개 | 200 + Page |
| `MAT-001` | GET | `/api/matches` | 공개 | 200 + Page |
| `MAT-002` | GET | `/api/matches/{matchId}` | 공개 | 200 + 매치 |
| `MAT-003` | POST | `/api/matches` | 로그인 | 201 + 매치 |
| `MAT-004` | PATCH | `/api/matches/{matchId}` | 주최자 | 200 + 매치 |
| `MAT-005` | POST | `/api/matches/{matchId}/recruitment/close` | 주최자 | 204 |
| `MAT-006` | POST | `/api/matches/{matchId}/recruitment/reopen` | 주최자 | 204 |
| `MAT-007` | POST | `/api/matches/{matchId}/cancel` | 주최자 | 204 |
| `MAT-008` | GET | `/api/members/me/hosted-matches` | 로그인 | 200 + Page |
| `PAR-001` | POST | `/api/matches/{matchId}/participations` | 로그인 | 최초 201 / 재참가 200 |
| `PAR-002` | DELETE | `/api/matches/{matchId}/participations/me` | 로그인 | 204 |
| `PAR-003` | GET | `/api/members/me/participating-matches` | 로그인 | 200 + Page |
| `PAR-004` | GET | `/api/matches/{matchId}/participations` | 주최자 | 200 + Page |

## 5. 회원·인증 API

### 5.1 회원가입 — `MEM-001`

`POST /api/members`

| 필드 | 타입 | 필수 | null | 핵심 제약 |
|---|---|---:|---:|---|
| `email` | string | 예 | 불가 | 이메일 형식, 중복 불가 |
| `password` | string | 예 | 불가 | 빈·공백 불가, 최종 길이·강도 열린 결정 |
| `nickname` | string | 예 | 불가 | 빈·공백 불가, 최대 30자 초기안 |

요청:

```json
{
  "email": "player@example.com",
  "password": "Mvp-pass-2026!",
  "nickname": "풋살러"
}
```

성공: `201 Created`

```json
{
  "id": 7,
  "email": "player@example.com",
  "nickname": "풋살러",
  "role": "USER"
}
```

대표 오류:

- `INVALID_REQUEST`
- `VALIDATION_FAILED`
- `MEMBER_EMAIL_DUPLICATED`
- 닉네임 유일 정책 선택 시 `MEMBER_NICKNAME_DUPLICATED`
- `CSRF_TOKEN_INVALID`

### 5.2 내 프로필 수정 — `MEM-002`

`PATCH /api/members/me/profile`

초기 범위에서는 닉네임만 수정한다.

```json
{
  "nickname": "새닉네임"
}
```

- 로그인 필요
- 적어도 한 필드를 전달해야 한다.
- `nickname`은 null·빈·공백을 허용하지 않는다.
- 성공: `200 OK` + 변경된 `MemberSummary`
- 오류: `AUTHENTICATION_REQUIRED`, `MEMBER_NOT_FOUND`, `VALIDATION_FAILED`, 조건부 `MEMBER_NICKNAME_DUPLICATED`, `CSRF_TOKEN_INVALID`

### 5.3 로그인 — `AUTH-001`

`POST /api/auth/login`

```json
{
  "email": "player@example.com",
  "password": "Mvp-pass-2026!"
}
```

성공: `200 OK`, `MemberSummary`와 세션 쿠키

로그인 실패는 이메일 존재 여부를 노출하지 않고 `INVALID_CREDENTIALS`를 반환한다.

### 5.4 로그아웃 — `AUTH-002`

`POST /api/auth/logout`

- 로그인과 CSRF 토큰 필요
- 요청 본문 없음
- 성공: `204 No Content`
- 오류: `AUTHENTICATION_REQUIRED`, `CSRF_TOKEN_INVALID`

### 5.5 현재 회원 조회 — `AUTH-003`

`GET /api/auth/me`

- 로그인 필요
- 요청 본문 없음
- 성공: `200 OK` + `MemberSummary`
- 오류: `AUTHENTICATION_REQUIRED`, `MEMBER_NOT_FOUND`

별도 프로필 GET API는 1차 MVP에서 만들지 않는다.

### 5.6 CSRF 초기화 후보 — `AUTH-004`

`GET /api/auth/csrf`

초기 추천:

- 공개 API
- 요청 본문 없음
- 성공: `204 No Content`
- 응답에서 `XSRF-TOKEN` 쿠키 발급
- 클라이언트는 쿠키 값을 읽어 상태 변경 요청 헤더로 전송
- 캐시 금지

쿠키·헤더 이름과 저장소 구현은 프론트엔드 통합 시 확정한다.

## 6. 구장·코트 API

### 6.1 구장 목록 — `VEN-001`

`GET /api/venues`

Query 후보:

- `regionCode`
- `name`
- `page`
- `size`

`regionCode`는 일반 문자열 초기안이며 닫힌 enum으로 확정하지 않는다.

성공: `200 OK` + `VenueResponse` Page

```json
{
  "content": [
    {
      "id": 3,
      "name": "풋살파크",
      "regionCode": "GYEONGGI_SUWON",
      "address": "경기도 수원시",
      "description": null
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 1,
  "totalPages": 1,
  "first": true,
  "last": true
}
```

### 6.2 구장 상세 — `VEN-002`

`GET /api/venues/{venueId}`

- 성공: `200 OK` + `VenueResponse`
- 오류: `VENUE_NOT_FOUND`, `VALIDATION_FAILED`

### 6.3 구장별 코트 목록 — `VEN-003`

`GET /api/venues/{venueId}/courts`

- Query: `page`, `size`
- 성공: `200 OK` + `CourtResponse` Page
- 코트가 없는 기존 구장은 빈 Page
- 오류: `VENUE_NOT_FOUND`, `VALIDATION_FAILED`

## 7. 매치 API

### 7.1 매치 목록 — `MAT-001`

`GET /api/matches`

| Query | 타입 | 필수 | 의미 |
|---|---|---:|---|
| `venueId`, `courtId` | number | 아니오 | 장소 필터 |
| `regionCode` | string | 아니오 | 일반 지역 값 |
| `recommendedSkill` | string | 아니오 | 권장 실력 |
| `recruitmentStatus` | string | 아니오 | `OPEN`, `CLOSED` |
| `hasRemainingSlots` | boolean | 아니오 | 남은 자리 여부 |
| `startFrom`, `startTo` | string | 아니오 | 시작 시각 범위 |
| `page`, `size` | number | 아니오 | 페이지 |

기본 조회 조건:

```text
status == SCHEDULED
AND recruitmentStatus == OPEN
AND now < startAt
AND remainingSlots > 0
```

마감 또는 정원 마감 매치는 명시적 필터로 조회한다. 취소·시작·종료 매치는 일반 탐색에서 제외한다.

성공: `200 OK` + `MatchResponse` Page

비회원 공개 응답 예시:

```json
{
  "content": [
    {
      "id": 42,
      "host": {"id": 7, "nickname": "풋살러"},
      "court": {
        "id": 11,
        "name": "A 코트",
        "venue": {"id": 3, "name": "풋살파크", "regionCode": "GYEONGGI_SUWON"}
      },
      "title": "토요일 오전 풋살",
      "instructions": null,
      "recommendedSkill": "BEGINNER",
      "status": "SCHEDULED",
      "recruitmentStatus": "OPEN",
      "startAt": "2026-08-15T10:00:00+09:00",
      "endAt": "2026-08-15T12:00:00+09:00",
      "maxParticipants": 12,
      "confirmedParticipants": 5,
      "remainingSlots": 7,
      "displayStatus": "AVAILABLE",
      "joinable": true,
      "hostParticipates": true,
      "currentMemberParticipating": false,
      "currentMemberHost": false,
      "canJoin": false,
      "canCancelParticipation": false,
      "canEdit": false,
      "canCancelMatch": false
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 1,
  "totalPages": 1,
  "first": true,
  "last": true
}
```

### 7.2 매치 상세 — `MAT-002`

`GET /api/matches/{matchId}`

- 공개 API
- 취소·진행·종료 매치도 ID로 직접 조회 가능
- 성공: `200 OK` + `MatchResponse`
- 오류: `MATCH_NOT_FOUND`, `VALIDATION_FAILED`

### 7.3 매치 개설 — `MAT-003`

`POST /api/matches`

| 필드 | 타입 | 필수 | null | 핵심 제약 |
|---|---|---:|---:|---|
| `courtId` | number | 예 | 불가 | 존재하는 Court |
| `title` | string | 예 | 불가 | 공백 불가, 최대 100자 초기안 |
| `instructions` | string | 아니오 | 가능 | 전달 시 공백 불가, 최대 1000자 초기안 |
| `recommendedSkill` | string | 예 | 불가 | 지원 코드 |
| `startAt` | string | 예 | 불가 | offset date-time, 미래 시각 초기안 |
| `endAt` | string | 예 | 불가 | `startAt`보다 뒤 |
| `maxParticipants` | number | 예 | 불가 | 최소 1, 상한 열린 결정 |
| `hostParticipates` | boolean | 예 | 불가 | true이면 정원에 포함 |

성공:

- `201 Created`
- `Location: /api/matches/{id}`
- `MatchResponse`
- 주최자 참가 선택 시 Match와 참가 관계를 같은 트랜잭션에서 생성

대표 오류:

- `AUTHENTICATION_REQUIRED`
- `COURT_NOT_FOUND`
- `MATCH_INVALID_TIME_RANGE`
- `VALIDATION_FAILED`
- `CSRF_TOKEN_INVALID`

### 7.4 매치 수정 — `MAT-004`

`PATCH /api/matches/{matchId}`

수정 후보 필드:

- `title`
- `instructions`
- `recommendedSkill`
- `courtId`
- `startAt`
- `endAt`
- `maxParticipants`

확정 규칙:

- 주최자만 가능
- 취소·시작한 매치는 수정 불가
- 변경 후 시간 범위가 유효해야 함
- 정원을 활성 참가자 수보다 작게 줄일 수 없음
- 정원 증가로 모집 상태를 자동 재개하지 않음

권장 실력, 코트와 시간은 활성 일반 참가자가 없을 때만 수정 가능하다는 초기 정책을 적용한다.

성공: `200 OK` + 변경된 `MatchResponse`

### 7.5 모집 마감 — `MAT-005`

`POST /api/matches/{matchId}/recruitment/close`

- 주최자, `SCHEDULED`, 시작 전
- 요청 본문 없음
- `OPEN → CLOSED`
- 성공: `204 No Content`
- 대표 오류: `MATCH_ACCESS_DENIED`, `MATCH_ALREADY_STARTED`, `MATCH_ALREADY_CANCELLED`, `MATCH_RECRUITMENT_CLOSED`

### 7.6 모집 재개 — `MAT-006`

`POST /api/matches/{matchId}/recruitment/reopen`

- 주최자, `SCHEDULED`, 시작 전, 남은 자리 있음
- 요청 본문 없음
- `CLOSED → OPEN`
- 성공: `204 No Content`
- 이미 OPEN이거나 남은 자리가 없으면 `MATCH_REOPEN_NOT_ALLOWED` 초기안

### 7.7 매치 취소 — `MAT-007`

`POST /api/matches/{matchId}/cancel`

확정 규칙:

- 주최자만 가능
- `SCHEDULED` 상태여야 함
- **`현재 시각 < startAt`일 때만 가능**
- Match를 `CANCELLED`로 바꾸고 모든 `CONFIRMED` 참가를 `CANCELLED_BY_MATCH`로 전이
- 두 변경은 하나의 유스케이스 트랜잭션에서 함께 성공하거나 롤백

성공: `204 No Content`

대표 오류:

- `MATCH_ACCESS_DENIED`
- `MATCH_ALREADY_STARTED`
- `MATCH_ALREADY_CANCELLED`

### 7.8 내가 주최한 매치 — `MAT-008`

`GET /api/members/me/hosted-matches`

Query 후보:

- `status`
- `recruitmentStatus`
- `startFrom`, `startTo`
- `page`, `size`

성공: `200 OK` + `MatchResponse` Page

## 8. 참가 API

### 8.1 참가·재참가 — `PAR-001`

`POST /api/matches/{matchId}/participations`

요청 본문 없음.

검증:

- Match 존재
- `SCHEDULED`
- `OPEN`
- 시작 전
- 남은 자리 있음
- 관계 없음 또는 `CANCELLED_BY_MEMBER`

성공:

- 최초 관계 생성: `201 Created`
- `CANCELLED_BY_MEMBER` 재참가: `200 OK`
- 본문: `ParticipationResponse`

대표 오류:

- `MATCH_NOT_FOUND`
- `MATCH_ALREADY_STARTED`
- `MATCH_ALREADY_CANCELLED`
- `MATCH_RECRUITMENT_CLOSED`
- `MATCH_FULL`
- `PARTICIPATION_ALREADY_CONFIRMED`
- `PARTICIPATION_REJOIN_NOT_ALLOWED`

`UNIQUE(match_id, member_id)`는 관계 중복 방어선 후보다. 마지막 자리 경합의 구체 해결 방식은 동시성 테스트 후 결정한다.

### 8.2 참가 취소 — `PAR-002`

`DELETE /api/matches/{matchId}/participations/me`

- 로그인 필요
- 요청 본문 없음
- 자신의 참가가 `CONFIRMED`이고 시작 전이어야 함
- `CONFIRMED → CANCELLED_BY_MEMBER`
- 성공: `204 No Content`

대표 오류:

- `MATCH_NOT_FOUND`
- `PARTICIPATION_NOT_FOUND`
- `PARTICIPATION_CANCEL_NOT_ALLOWED`

반복 취소를 멱등한 204로 볼지 `409 PARTICIPATION_NOT_ACTIVE`로 볼지는 열린 결정이다. 어느 선택에서도 상태와 남은 자리가 다시 변해서는 안 된다.

### 8.3 내가 참가한 매치 — `PAR-003`

`GET /api/members/me/participating-matches`

Query 후보:

- `participationStatus`
- `matchStatus`
- `startFrom`, `startTo`
- `page`, `size`

성공: `200 OK` + `ParticipatingMatchItem` Page

```json
{
  "content": [
    {
      "participation": {
        "id": 81,
        "matchId": 42,
        "memberId": 9,
        "status": "CONFIRMED",
        "createdAt": "2026-08-02T10:00:00Z",
        "updatedAt": "2026-08-02T10:00:00Z"
      },
      "match": {
        "id": 42,
        "title": "토요일 오전 풋살",
        "status": "SCHEDULED",
        "recruitmentStatus": "OPEN",
        "startAt": "2026-08-15T10:00:00+09:00",
        "endAt": "2026-08-15T12:00:00+09:00",
        "remainingSlots": 7,
        "displayStatus": "AVAILABLE"
      }
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 1,
  "totalPages": 1,
  "first": true,
  "last": true
}
```

### 8.4 주최자의 참가자 목록 — `PAR-004`

`GET /api/matches/{matchId}/participations`

- 주최자만 가능
- Query: `status`, `page`, `size`
- 기본 상태: `CONFIRMED`
- 성공: `200 OK` + 회원 요약을 포함한 참가 Page
- 참가자 응답에는 이메일, 비밀번호 해시와 세션 정보를 포함하지 않음
- 오류: `MATCH_NOT_FOUND`, `MATCH_ACCESS_DENIED`, `VALIDATION_FAILED`

## 9. HTTP 상태 코드 정책

| HTTP | 사용 조건 |
|---:|---|
| `200` | 일반 조회, 본문이 필요한 수정·로그인·재참가 |
| `201` | 회원·매치·최초 참가 관계 생성 |
| `204` | 로그아웃, CSRF 초기화 후보, 모집 마감·재개, 매치 취소, 참가 취소 |
| `400` | JSON 문법·필드·query 검증, 잘못된 시간 범위 |
| `401` | 로그인 실패 또는 인증되지 않은 보호 요청 |
| `403` | 소유권 부족 또는 CSRF 실패 |
| `404` | 리소스 없음 |
| `409` | 현재 상태와 명령 충돌 |
| `500` | 예상하지 못한 서버 오류 |

1차 MVP에서는 400과 409로 요청 값 오류와 상태 충돌을 구분하고 422를 사용하지 않는 추천안이다.

## 10. 오류 코드 카탈로그

| 코드 | HTTP | 의미 | 결정 상태 |
|---|---:|---|---|
| `INVALID_REQUEST` | 400 | 읽을 수 없는 요청 본문 | 초기안 |
| `VALIDATION_FAILED` | 400 | 필드·path·query 검증 실패 | 초기안 |
| `INTERNAL_SERVER_ERROR` | 500 | 예상하지 못한 서버 오류 | 초기안 |
| `AUTHENTICATION_REQUIRED` | 401 | 로그인 필요 | 초기안 |
| `INVALID_CREDENTIALS` | 401 | 로그인 자격 증명 실패 | 의미 확정 |
| `CSRF_TOKEN_INVALID` | 403 | CSRF 토큰 없음·불일치 | 초기안 |
| `MEMBER_NOT_FOUND` | 404 | 현재 회원 데이터 없음 | 초기안 |
| `MEMBER_EMAIL_DUPLICATED` | 409 | 이메일 충돌 | 초기안 |
| `MEMBER_NICKNAME_DUPLICATED` | 409 | 닉네임 충돌 | 조건부 열린 결정 |
| `VENUE_NOT_FOUND` | 404 | 구장 없음 | 초기안 |
| `COURT_NOT_FOUND` | 404 | 코트 없음 | 초기안 |
| `MATCH_NOT_FOUND` | 404 | 매치 없음 | 초기안 |
| `MATCH_ACCESS_DENIED` | 403 | 매치 소유권 부족 | 초기안 |
| `MATCH_ALREADY_STARTED` | 409 | 시작 후 금지 명령 | 의미 확정 |
| `MATCH_ALREADY_CANCELLED` | 409 | 취소된 매치 | 초기안 |
| `MATCH_RECRUITMENT_CLOSED` | 409 | 모집 마감 | 초기안 |
| `MATCH_FULL` | 409 | 남은 자리 없음 | 의미 확정 |
| `MATCH_INVALID_TIME_RANGE` | 400 | 종료 시각이 시작 시각보다 늦지 않음 | 초기안 |
| `MATCH_CAPACITY_BELOW_CONFIRMED_COUNT` | 409 | 활성 참가자보다 작은 정원 | 의미 확정 |
| `MATCH_REOPEN_NOT_ALLOWED` | 409 | 현재 상태에서 모집 재개 불가 | 초기안 |
| `PARTICIPATION_ALREADY_CONFIRMED` | 409 | 이미 참가 중 | 초기안 |
| `PARTICIPATION_NOT_FOUND` | 404 | 참가 관계 없음 | 초기안 |
| `PARTICIPATION_NOT_ACTIVE` | 409 | 활성 참가 아님 | 반복 취소 정책의 조건부 후보 |
| `PARTICIPATION_CANCEL_NOT_ALLOWED` | 409 | 현재 상태에서 참가 취소 불가 | 초기안 |
| `PARTICIPATION_REJOIN_NOT_ALLOWED` | 409 | 재참가 금지 상태 | 초기안 |

필드별 오류 코드를 만들지 않고 `VALIDATION_FAILED.fieldErrors`를 사용한다. Java 예외 클래스와 오류 코드를 일대일로 고정하지 않는다.

## 11. 열린 결정

| 항목 | 선택지 | 현재 추천안 | 결정 시점 |
|---|---|---|---|
| 참가 수 처리 | 매번 집계 / Match 저장 / 혼합 | 없음 | 참가 구현 전 |
| 마지막 자리 경합 | 비관·낙관 락 / 조건부 갱신 등 | 없음 | 동시성 테스트 후 |
| 반복 참가 취소 | 멱등 204 / 409 | 없음 | 참가 취소 구현 전 |
| 닉네임 중복 | 허용 / 유일 | 없음 | 회원 구현 전 |
| 시간·타임존 | UTC / 지역 시각 보존 | 없음 | Match 영속화 전 |
| 페이지 방식 | Page/Slice, 0/1 기반 | 0 기반 Page | 목록 구현 전 |
| 성공 래퍼 | 직접 반환 / 공통 래퍼 | 직접 반환 | 첫 API 구현 전 |
| CSRF 쿠키·헤더 | Spring 기본 / 커스텀 | 쿠키 전달 우선 검토 | 인증 구현 전 |
| 현재 사용자 편의 필드 | 서버 계산 / 일부 제거 | `joinable`과 사용자별 `can*` 분리 | 목록·상세 구현 전 |
| 지역 표현 | 자유 문자열 / 코드 체계 / 별도 컬럼 | 닫힌 enum은 보류 | 구장 시드 작성 전 |

열린 결정이 확정되면 결정 기록, 요구사항, ERD와 테스트에 함께 반영한다.
