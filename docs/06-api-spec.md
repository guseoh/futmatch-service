# 1차 MVP REST API 초기 계약

이 문서는 1차 MVP의 요청·성공·오류 계약을 구현 가능한 수준으로 제시하는 **초기 설계 기준선**이다. 업무 규칙과 권한은 **확정**, URI·길이·페이지·편의 필드는 **초기안** 또는 **추천안**, 근거가 부족한 항목은 **열린 결정**이다. Java DTO·예외 클래스, JPA 매핑, Repository 쿼리와 동시성 해법은 확정하지 않는다.

관련 기준은 [결정 기록](./01-decisions.md), [요구사항](./02-requirements.md), [ERD](./04-erd.md), [인증·인가](./05-auth.md)를 함께 따른다.

## 1. 공통 계약

### 1.1 URI와 응답 원칙

- **초기 추천:** URI는 `/api` 아래 복수형 리소스명을 사용한다.
- **초기 추천:** 모집 마감·재개와 매치 취소는 의미가 분명한 명령 URI에 `POST`를 사용한다. `PATCH`로 상태 필드를 직접 받는 대안보다 허용 전이를 서버가 통제하기 쉽다. 명령이 늘어나면 상태 리소스 모델을 재검토한다.
- **초기 추천:** 성공 응답은 공통 `data` 래퍼 없이 리소스 또는 Page 본문을 직접 반환한다. 오류만 공통 형식을 사용한다.
- 조회는 `200 OK`, 생성은 `201 Created`, 본문 없는 명령은 `204 No Content`를 기본 후보로 사용한다.
- 비동기 처리는 없으므로 `202 Accepted`를 사용하지 않는다.
- 매치 개설은 조회 가능한 URI를 `Location`에 반환한다. 회원가입과 참가 생성은 현재 대응하는 공개 GET 리소스가 없으므로 `Location`을 사용하지 않는 **초기안**이다.

### 1.2 JSON, enum과 시간

- JSON 필드명은 lower camel case, enum 값은 영문 대문자 SNAKE_CASE 문자열을 사용한다.
- 매치 상태는 `SCHEDULED`, `CANCELLED`, 모집 상태는 `OPEN`, `CLOSED`, 참가 상태는 `CONFIRMED`, `CANCELLED_BY_MEMBER`, `CANCELLED_BY_MATCH`다.
- 권장 실력 **초기안**은 `ANY`, `INTRODUCTORY`, `BEGINNER`, `INTERMEDIATE_OR_ABOVE`다.
- **초기안:** 날짜·시간은 ISO 8601 offset date-time 문자열(예: `2026-08-15T10:00:00+09:00`)로 주고받는다. DB 저장 타입과 기준 시간대는 열린 결정이다.
- 응답 ID는 JSON number **초기안**이다. Java 타입과 장기적인 큰 정수 직렬화 정책은 구현 시 확인한다.

### 1.3 필드 누락·null·문자열 검증

| 입력 | 계약 |
|---|---|
| 필수 필드 누락 | `400 VALIDATION_FAILED` |
| 필수 필드의 명시적 `null` | `400 VALIDATION_FAILED` |
| 제목·닉네임·이메일의 빈 문자열 또는 공백 문자열 | `400 VALIDATION_FAILED` |
| 형식 오류 또는 지원하지 않는 enum | `400 VALIDATION_FAILED`; JSON 자체가 깨졌으면 `INVALID_REQUEST` |
| 범위 오류 | 필드 범위면 `400 VALIDATION_FAILED`, 시간 순서면 `400 MATCH_INVALID_TIME_RANGE` |
| Path 또는 명령 body의 존재하지 않는 참조 ID | 리소스별 `404 *_NOT_FOUND`; 목록 필터 ID는 검증 가능한 형식이면 빈 결과 초기안 |
| 현재 상태와 충돌하는 업무 요청 | `409`와 도메인별 오류 코드 |

매치 수정 `PATCH`에서는 미전송 필드를 변경하지 않는다. 명시적 `null`은 `instructions`를 지울 때만 허용하는 **초기안**이며, 다른 필드의 `null`은 검증 오류다. `instructions`가 전달됐을 때 빈 문자열·공백 문자열은 거부하고, 제거하려면 `null`을 사용한다.

문자열 길이 **초기안**은 `title` 최대 100자, `instructions` 최대 1000자, `nickname` 최대 30자다. 이메일·비밀번호의 최종 길이·강도, 최대 참가 정원의 상한은 열린 결정이다. `maxParticipants` 최솟값 1도 **초기안**이다.

### 1.4 페이지 계약

모든 컬렉션 조회는 다음 Page 구조를 사용하는 **초기 추천안**이다.

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

| 항목 | 초기 계약 | 상태 |
|---|---|---|
| `page` | 0부터 시작, 기본 0 | 추천안; 시작 번호는 열린 결정 |
| `size` | 기본 20, 최소 1, 최대 100 | 초기안; 최댓값은 열린 결정 |
| 형태 | 전체 건수를 포함하는 Page | 추천안; Page와 Slice는 열린 결정 |
| 매치 기본 정렬 | `startAt ASC, id ASC` | 초기안; 최종 정렬은 열린 결정 |
| 기타 목록 정렬 | 문서의 endpoint별 초기값 | 초기안 |

잘못된 페이지·size·정렬·필터 값은 `400 VALIDATION_FAILED`다. 클라이언트가 임의의 DB 컬럼명을 정렬 키로 전달하는 계약은 제공하지 않는다.

### 1.5 주요 응답 모델

#### `MemberSummary`

| 필드 | JSON 타입 | null 가능 | 저장/파생 | 설명 |
|---|---|---:|---|---|
| `id` | number | 아니오 | 저장 | 회원 ID |
| `email` | string | 아니오 | 저장 | 본인 응답에서만 제공 |
| `nickname` | string | 아니오 | 저장 | 표시명 |
| `role` | string | 아니오 | 저장 | `USER` 또는 `ADMIN` 초기 후보 |

#### `VenueSummary`와 `CourtSummary`

| 필드 | JSON 타입 | null 가능 | 저장/파생 | 설명 |
|---|---|---:|---|---|
| `id` | number | 아니오 | 저장 | 구장 또는 코트 ID |
| `name` | string | 아니오 | 저장 | 표시명 |
| `regionCode` | string | 예 | 저장 | 구장에만 존재하는 일반화된 지역 값 후보 |
| `address` | string | 아니오 | 저장 | 구장에만 존재 |
| `description` | string | 예 | 저장 | 안내 정보 |
| `venueId` | number | 아니오 | 저장 | 코트에만 존재 |

#### `MatchResponse`

| 필드 | JSON 타입 | null 가능 | 저장/파생 | 설명 |
|---|---|---:|---|---|
| `id` | number | 아니오 | 저장 | 매치 ID |
| `host` | object | 아니오 | 조합 | 주최자 `id`, `nickname` |
| `court` | object | 아니오 | 조합 | 코트와 소속 구장 요약 |
| `title` | string | 아니오 | 저장 | 제목 |
| `instructions` | string | 예 | 저장 | 안내 사항 |
| `recommendedSkill` | string | 아니오 | 저장 | 권장 실력 코드 |
| `status` | string | 아니오 | 저장 | `SCHEDULED` 또는 `CANCELLED` |
| `recruitmentStatus` | string | 아니오 | 저장 | `OPEN` 또는 `CLOSED` |
| `startAt`, `endAt` | string | 아니오 | 저장 | ISO 8601 offset date-time 초기안 |
| `maxParticipants` | number | 아니오 | 저장 | 주최자 참가를 포함한 최대 정원 |
| `confirmedParticipants` | number | 아니오 | 파생 또는 저장 미정 | `CONFIRMED` 참가자 수 |
| `remainingSlots` | number | 아니오 | 파생 | 정원에서 활성 참가 수를 뺀 값 |
| `displayStatus` | string | 아니오 | 파생 | 아래 표시 상태 초기안 |
| `hostParticipates` | boolean | 아니오 | 파생 | 주최자가 현재 `CONFIRMED`이면 true |
| `currentMemberParticipating` | boolean | 아니오 | 파생 | 현재 회원이 `CONFIRMED`이면 true |
| `currentMemberHost` | boolean | 아니오 | 파생 | 현재 회원이 주최자이면 true |
| `canJoin` | boolean | 아니오 | 파생 | 현재 회원이 지금 참가·재참가 가능하면 true |
| `canCancelParticipation` | boolean | 아니오 | 파생 | 현재 회원이 지금 참가 취소 가능하면 true |
| `canEdit`, `canCancelMatch` | boolean | 아니오 | 파생 | 현재 회원이 해당 명령을 수행할 수 있으면 true |

`displayStatus`는 저장 enum이 아닌 화면 편의 값 **초기안**이다. 우선순위는 `CANCELLED`, `ENDED`, `IN_PROGRESS`, `CLOSED`, `FULL`, `AVAILABLE` 순으로 계산한다. 비인증 공개 응답에서도 사용자 관계 필드를 생략하지 않고 `currentMember*`, `can*`를 `false`로 반환한다. 이는 응답 모양을 일정하게 하기 위한 초기안이며, 로그인하지 않은 사용자가 매치 자체를 볼 수 없다는 뜻은 아니다.

#### `ParticipationResponse`

| 필드 | JSON 타입 | null 가능 | 저장/파생 | 설명 |
|---|---|---:|---|---|
| `id` | number | 아니오 | 저장 | 참가 관계 ID 초기안 |
| `matchId`, `memberId` | number | 아니오 | 저장 | 관계 식별자 |
| `status` | string | 아니오 | 저장 | 참가 상태 |
| `createdAt`, `updatedAt` | string | 아니오 | 저장 | 생성·마지막 변경 시각 |

### 1.6 공통 오류 응답 계약

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

필드 검증 오류 예시는 다음과 같다.

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

| 필드 | 타입 | 규칙 |
|---|---|---|
| `timestamp` | string | 오류 응답 생성 시각, ISO 8601 |
| `status` | number | HTTP 상태 코드 숫자 |
| `code` | string | 클라이언트 분기용 안정적인 대문자 SNAKE_CASE 식별자 |
| `message` | string | 사람이 읽는 한국어 설명; `code`보다 변경 가능성이 큼 |
| `path` | string | 요청 경로 |
| `fieldErrors` | array | 없으면 빈 배열 |

스택 트레이스, SQL, 테이블명과 내부 예외 클래스는 노출하지 않는다. 비밀번호·세션·CSRF 등 인증정보의 `rejectedValue`는 반환하지 않는다. 예상하지 못한 오류는 내부 정보를 감춘 `500 INTERNAL_SERVER_ERROR`로 반환한다. `requestId`/`traceId`는 운영·관측 단계의 열린 결정이다.

## 2. API 목록

| ID | Method | URI | 접근 | 성공 초기안 |
|---|---|---|---|---|
| `MEM-001` | POST | `/api/members` | 공개 | 201 + 회원 본문 |
| `MEM-002` | GET | `/api/members/me/profile` | 로그인 | 200 + 프로필 |
| `MEM-003` | PATCH | `/api/members/me/profile` | 로그인 | 200 + 프로필 |
| `AUTH-001` | POST | `/api/auth/login` | 공개 | 200 + 현재 회원 |
| `AUTH-002` | POST | `/api/auth/logout` | 로그인 | 204 |
| `AUTH-003` | GET | `/api/auth/me` | 로그인 | 200 + 현재 회원 |
| `AUTH-004` | GET | `/api/auth/csrf` | 공개 후보 | 200 + 토큰; **열린 결정** |
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
| `PAR-001` | POST | `/api/matches/{matchId}/participations` | 로그인 | 최초 201 / 재참가 200 + 참가 본문 |
| `PAR-002` | DELETE | `/api/matches/{matchId}/participations/me` | 로그인 | 204 |
| `PAR-003` | GET | `/api/members/me/participating-matches` | 로그인 | 200 + Page |
| `PAR-004` | GET | `/api/matches/{matchId}/participations` | 주최자 | 200 + Page |

## 3. 회원·인증 API

### 3.1 회원가입 — `MEM-001`

| 항목 | 계약 |
|---|---|
| 목적 | 비회원이 로그인 가능한 회원 계정을 만든다. |
| Method / URI / 접근 | `POST /api/members` / 공개. 쿠키 세션의 CSRF 적용 범위는 [인증 문서](./05-auth.md)와 함께 결정한다. |
| Path / Query | 없음 |
| 검증 | 필수값·형식·빈/공백·길이, 이메일 중복. 닉네임 중복은 정책을 선택한 경우만 검사한다. |
| 성공 | `201 Created`, `MemberSummary` 본문. 자동 로그인하지 않고 `Location`을 사용하지 않는 초기안이다. |
| 상태 변화 | 비밀번호 원문이 아닌 적응형 해시와 최소 프로필을 가진 회원 한 명 생성 |
| 대표 오류 | `INVALID_REQUEST`, `VALIDATION_FAILED`, `MEMBER_EMAIL_DUPLICATED`, 조건부 `MEMBER_NICKNAME_DUPLICATED`, `CSRF_TOKEN_INVALID` |

| 필드 | JSON 타입 | 필수 | null | 핵심 제약 | 예시 |
|---|---|---:|---:|---|---|
| `email` | string | 예 | 불가 | 이메일 형식; 정규화·최종 길이는 열린 결정 | `player@example.com` |
| `password` | string | 예 | 불가 | 빈/공백 불가; 최종 강도·길이는 열린 결정 | `Mvp-pass-2026!` |
| `nickname` | string | 예 | 불가 | 공백 문자열 불가, 최대 30자 초기안 | `풋살러` |

요청 JSON:

```json
{"email":"player@example.com","password":"Mvp-pass-2026!","nickname":"풋살러"}
```

성공 응답 JSON:

```json
{"id":7,"email":"player@example.com","nickname":"풋살러","role":"USER"}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":409,"code":"MEMBER_EMAIL_DUPLICATED","message":"이미 사용 중인 이메일입니다.","path":"/api/members","fieldErrors":[]}
```

추적: `MEM-001` → 입력 검증 → 이메일 식별자 유일 규칙 → `members.email` UNIQUE 후보 → 201 → 중복·검증 오류.

### 3.2 내 프로필 조회 — `MEM-002`

| 항목 | 계약 |
|---|---|
| 목적 | 로그인 회원이 자신의 최소 프로필을 조회한다. |
| Method / URI / 접근 | `GET /api/members/me/profile` / 로그인 필요 |
| Path / Query / Request body | 없음 / 없음 / 없음 |
| 성공 | `200 OK`, `MemberSummary`에 `updatedAt`을 더한 본문 |
| 상태 변화 | 없음 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MEMBER_NOT_FOUND` |

요청 JSON: 본문 없음.

성공 응답 JSON:

```json
{"id":7,"email":"player@example.com","nickname":"풋살러","role":"USER","updatedAt":"2026-08-02T10:00:00Z"}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":401,"code":"AUTHENTICATION_REQUIRED","message":"로그인이 필요합니다.","path":"/api/members/me/profile","fieldErrors":[]}
```

추적: `MEM-002` → 세션 인증 → 현재 회원 조회 → 200 또는 인증·회원 없음 오류.

### 3.3 내 프로필 수정 — `MEM-003`

| 항목 | 계약 |
|---|---|
| 목적 | 로그인 회원이 허용된 최소 프로필 필드를 수정한다. |
| Method / URI / 접근 | `PATCH /api/members/me/profile` / 로그인 필요, 본인 리소스 |
| Path / Query | 없음 |
| 검증 | 적어도 한 필드 전달, `nickname` null·빈·공백 불가, 최대 30자 초기안 |
| 성공 | `200 OK`, 변경된 프로필 본문 |
| 상태 변화 | 현재 회원의 닉네임 변경 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MEMBER_NOT_FOUND`, `INVALID_REQUEST`, `VALIDATION_FAILED`, 닉네임 유일 정책 선택 시 `MEMBER_NICKNAME_DUPLICATED`, `CSRF_TOKEN_INVALID` |

| 필드 | JSON 타입 | 필수 | null | 핵심 제약 | 예시 |
|---|---|---:|---:|---|---|
| `nickname` | string | 조건부 | 불가 | 현재 유일한 수정 필드; 공백 불가, 최대 30자 초기안 | `새닉네임` |

요청 JSON:

```json
{"nickname":"새닉네임"}
```

성공 응답 JSON:

```json
{"id":7,"email":"player@example.com","nickname":"새닉네임","role":"USER","updatedAt":"2026-08-02T11:00:00Z"}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":400,"code":"VALIDATION_FAILED","message":"요청 값이 올바르지 않습니다.","path":"/api/members/me/profile","fieldErrors":[{"field":"nickname","rejectedValue":" ","message":"닉네임은 공백일 수 없습니다."}]}
```

추적: `MEM-003` → PATCH 필드 검증 → 닉네임 정책 → `members.nickname` 제약 후보 → 200 또는 검증·충돌 오류.

### 3.4 로그인 — `AUTH-001`

| 항목 | 계약 |
|---|---|
| 목적 | 자격 증명을 확인하고 인증 세션을 수립한다. |
| Method / URI / 접근 | `POST /api/auth/login` / 공개 |
| Path / Query | 없음 |
| 검증 | email·password 필수, null·빈·공백 불가. 실패 원인은 통합한다. |
| 성공 | `200 OK`, `MemberSummary` 본문과 세션 쿠키 |
| 상태 변화 | 인증 세션 수립; 회원 업무 데이터는 변경하지 않음 |
| 대표 오류 | `INVALID_REQUEST`, `VALIDATION_FAILED`, `INVALID_CREDENTIALS`, `CSRF_TOKEN_INVALID` |

| 필드 | JSON 타입 | 필수 | null | 핵심 제약 | 예시 |
|---|---|---:|---:|---|---|
| `email` | string | 예 | 불가 | 이메일 형식 | `player@example.com` |
| `password` | string | 예 | 불가 | 인증정보; 오류 `rejectedValue`에서 제외 | `Mvp-pass-2026!` |

요청 JSON:

```json
{"email":"player@example.com","password":"Mvp-pass-2026!"}
```

성공 응답 JSON:

```json
{"id":7,"email":"player@example.com","nickname":"풋살러","role":"USER"}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":401,"code":"INVALID_CREDENTIALS","message":"이메일 또는 비밀번호가 올바르지 않습니다.","path":"/api/auth/login","fieldErrors":[]}
```

추적: `AUTH-001` → 자격 증명 검증 → 비밀번호 해시 비교 → 세션 수립 → 200 또는 통합 로그인 오류.

### 3.5 로그아웃 — `AUTH-002`

| 항목 | 계약 |
|---|---|
| 목적 | 현재 인증 상태를 종료한다. |
| Method / URI / 접근 | `POST /api/auth/logout` / 로그인 필요 |
| Path / Query / Request body | 없음 / 없음 / 없음 |
| 성공 | `204 No Content`, 본문 없음 |
| 상태 변화 | 현재 세션 무효화와 세션 쿠키 만료 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `CSRF_TOKEN_INVALID` |

요청 JSON: 본문 없음. 성공 응답 JSON: 본문 없음.

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":403,"code":"CSRF_TOKEN_INVALID","message":"CSRF 토큰이 유효하지 않습니다.","path":"/api/auth/logout","fieldErrors":[]}
```

추적: `AUTH-002` → 세션·CSRF 확인 → 세션 무효화 → 204 또는 인증 오류.

### 3.6 현재 로그인 회원 조회 — `AUTH-003`

| 항목 | 계약 |
|---|---|
| 목적 | 클라이언트가 현재 인증 주체의 최소 정보를 확인한다. |
| Method / URI / 접근 | `GET /api/auth/me` / 로그인 필요 |
| Path / Query / Request body | 없음 / 없음 / 없음 |
| 성공 | `200 OK`, `MemberSummary` 본문 |
| 상태 변화 | 없음 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MEMBER_NOT_FOUND` |

요청 JSON: 본문 없음.

성공 응답 JSON:

```json
{"id":7,"email":"player@example.com","nickname":"풋살러","role":"USER"}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":401,"code":"AUTHENTICATION_REQUIRED","message":"로그인이 필요합니다.","path":"/api/auth/me","fieldErrors":[]}
```

추적: `AUTH-003` → 세션 인증 → 현재 회원 조회 → 200 또는 인증·회원 없음 오류.

### 3.7 CSRF 토큰 초기화 후보 — `AUTH-004`

이 API의 필요 여부와 최종 형태는 **열린 결정**이다. 쿠키 기반 세션 클라이언트가 별도 토큰 초기화 endpoint를 필요로 할 때의 초기 후보만 제시한다.

| 항목 | 계약 후보 |
|---|---|
| Method / URI / 접근 | `GET /api/auth/csrf` / 공개 후보 |
| Path / Query / Request body | 없음 / 없음 / 없음 |
| 성공 | `200 OK`, 헤더명과 토큰 값 본문 후보; 캐시 금지 |
| 상태 변화 | 업무 데이터 변화 없음; 보안 구현에 따라 세션 생성 가능 |
| 대표 오류 | `INTERNAL_SERVER_ERROR` |

요청 JSON: 본문 없음.

성공 응답 JSON 후보:

```json
{"headerName":"X-CSRF-TOKEN","token":"token-value"}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":500,"code":"INTERNAL_SERVER_ERROR","message":"요청을 처리하지 못했습니다.","path":"/api/auth/csrf","fieldErrors":[]}
```

추적: `AUTH-004` → 실제 클라이언트·CSRF 방식 확인 → 필요할 때만 계약 확정. 응답 필드명과 토큰 저장소를 이 문서로 미리 고정하지 않는다.

## 4. 구장·코트 API

### 4.1 구장 목록 — `VEN-001`

| 항목 | 계약 |
|---|---|
| 목적 | 준비된 구장을 검색·조회한다. |
| Method / URI / 접근 | `GET /api/venues` / 공개 |
| Query | `regionCode`, `name`, `page`, `size`; 필터는 모두 선택, 기본 정렬 `name ASC, id ASC` 초기안 |
| Request body | 없음 |
| 성공 | `200 OK`, `VenueSummary` Page |
| 상태 변화 | 없음 |
| 대표 오류 | `VALIDATION_FAILED` |

요청 JSON: 본문 없음. 예: `GET /api/venues?regionCode=GYEONGGI&page=0&size=20`.

성공 응답 JSON:

```json
{"content":[{"id":3,"name":"풋살파크","regionCode":"GYEONGGI_SUWON","address":"경기도 수원시","description":null}],"page":0,"size":20,"totalElements":1,"totalPages":1,"first":true,"last":true}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":400,"code":"VALIDATION_FAILED","message":"요청 값이 올바르지 않습니다.","path":"/api/venues","fieldErrors":[{"field":"size","rejectedValue":101,"message":"size는 100 이하여야 합니다."}]}
```

추적: `VEN-001` → 목록 조건 검증 → 운영 시드 구장 조회 → Page 200.

### 4.2 구장 상세 — `VEN-002`

| 항목 | 계약 |
|---|---|
| 목적 | 한 구장의 표시 정보를 조회한다. |
| Method / URI / 접근 | `GET /api/venues/{venueId}` / 공개 |
| Path | `venueId`: 양의 정수 ID |
| Query / Request body | 없음 / 없음 |
| 성공 | `200 OK`, `VenueSummary` 본문 |
| 상태 변화 | 없음 |
| 대표 오류 | `VENUE_NOT_FOUND`, `VALIDATION_FAILED` |

요청 JSON: 본문 없음.

성공 응답 JSON:

```json
{"id":3,"name":"풋살파크","regionCode":"GYEONGGI_SUWON","address":"경기도 수원시","description":null}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":404,"code":"VENUE_NOT_FOUND","message":"구장을 찾을 수 없습니다.","path":"/api/venues/999","fieldErrors":[]}
```

추적: `VEN-002` → `venueId` 검증·존재 확인 → 200 또는 404.

### 4.3 구장별 코트 목록 — `VEN-003`

| 항목 | 계약 |
|---|---|
| 목적 | 지정 구장에 속한 정보용 코트 목록을 조회한다. |
| Method / URI / 접근 | `GET /api/venues/{venueId}/courts` / 공개 |
| Path | `venueId`: 존재하는 구장 ID |
| Query | `page`, `size`; 기본 정렬 `name ASC, id ASC` 초기안 |
| Request body | 없음 |
| 성공 | `200 OK`, `CourtSummary` Page |
| 상태 변화 | 없음 |
| 대표 오류 | `VENUE_NOT_FOUND`, `VALIDATION_FAILED` |

요청 JSON: 본문 없음. 예: `GET /api/venues/3/courts?page=0&size=20`.

성공 응답 JSON:

```json
{"content":[{"id":11,"venueId":3,"name":"A 코트","description":"실외 인조잔디"}],"page":0,"size":20,"totalElements":1,"totalPages":1,"first":true,"last":true}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":404,"code":"VENUE_NOT_FOUND","message":"구장을 찾을 수 없습니다.","path":"/api/venues/999/courts","fieldErrors":[]}
```

코트가 없는 기존 구장은 빈 Page로 성공한다. 이 API는 실제 대관 가능 여부를 반환하지 않으며 `COURT_NOT_AVAILABLE` 오류도 사용하지 않는다.

추적: `VEN-003` → 구장 존재 확인 → `courts.venue_id` 관계 조회 → Page 200 또는 404.

## 5. 매치 API

### 5.1 매치 목록과 필터 — `MAT-001`

| 항목 | 계약 |
|---|---|
| 목적 | 누구나 참가 가능한 매치를 기본 탐색하고 마감·정원 마감 매치를 명시적으로 필터링한다. |
| Method / URI / 접근 | `GET /api/matches` / 공개 |
| Query | 아래 표와 `page`, `size`; Request body 없음 |
| 성공 | `200 OK`, `MatchResponse` Page |
| 상태 변화 | 없음 |
| 대표 오류 | `VALIDATION_FAILED` |

| Query | 타입 | 필수 | 핵심 제약과 기본값 |
|---|---|---:|---|
| `venueId`, `courtId` | number | 아니오 | 양의 정수, 존재하지 않으면 빈 목록 **초기안** |
| `regionCode` | string | 아니오 | 경기도로 고정하지 않은 지역 값 |
| `recommendedSkill` | string | 아니오 | 지원 enum |
| `recruitmentStatus` | string | 아니오 | `OPEN` 또는 `CLOSED` |
| `hasRemainingSlots` | boolean | 아니오 | true/false |
| `startFrom`, `startTo` | string | 아니오 | ISO 8601 offset date-time, 둘 다 있으면 `startFrom < startTo` |
| `page`, `size` | number | 아니오 | 공통 페이지 계약 |

가용성 필터 둘 다 생략하면 `OPEN`이면서 남은 자리가 있는 매치를 기본으로 한다. 하나라도 명시하면 명시한 조건만 추가한다. 따라서 정원 마감은 `hasRemainingSlots=false`, 모집 마감은 `recruitmentStatus=CLOSED`로 조회할 수 있다. 정원 도달 시 모집 상태를 자동으로 `CLOSED`로 저장할지는 열린 결정이므로 두 필터를 불필요하게 결합하지 않는다. 일반 탐색은 항상 `SCHEDULED`이고 아직 시작하지 않은 매치만 대상으로 하며 취소·시작·종료 매치는 제외한다. 기본 정렬은 `startAt ASC, id ASC` 초기안이다.

요청 JSON: 본문 없음. 예: `GET /api/matches?regionCode=GYEONGGI_SUWON&recommendedSkill=BEGINNER&page=0&size=20`.

성공 응답 JSON:

```json
{"content":[{"id":42,"host":{"id":7,"nickname":"풋살러"},"court":{"id":11,"name":"A 코트","venue":{"id":3,"name":"풋살파크","regionCode":"GYEONGGI_SUWON"}},"title":"토요일 오전 풋살","instructions":null,"recommendedSkill":"BEGINNER","status":"SCHEDULED","recruitmentStatus":"OPEN","startAt":"2026-08-15T10:00:00+09:00","endAt":"2026-08-15T12:00:00+09:00","maxParticipants":12,"confirmedParticipants":5,"remainingSlots":7,"displayStatus":"AVAILABLE","hostParticipates":true,"currentMemberParticipating":false,"currentMemberHost":false,"canJoin":false,"canCancelParticipation":false,"canEdit":false,"canCancelMatch":false}],"page":0,"size":20,"totalElements":1,"totalPages":1,"first":true,"last":true}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":400,"code":"VALIDATION_FAILED","message":"요청 값이 올바르지 않습니다.","path":"/api/matches","fieldErrors":[{"field":"recommendedSkill","rejectedValue":"EXPERT","message":"지원하지 않는 권장 실력입니다."}]}
```

추적: `MAT-001` → 필터 검증 → `matches.court_id`와 코트·구장 관계 및 활성 참가 수 조회 → Page 200.

### 5.2 매치 상세 — `MAT-002`

| 항목 | 계약 |
|---|---|
| 목적 | 누구나 매치의 시간·장소·주최자·모집·정원 정보를 조회한다. |
| Method / URI / 접근 | `GET /api/matches/{matchId}` / 공개 |
| Path | `matchId`: 양의 정수, 존재하는 매치 ID |
| Query / Request body | 없음 / 없음 |
| 성공 | `200 OK`, `MatchResponse`. 취소·시작·종료 매치도 ID로 직접 조회 가능하다. |
| 상태 변화 | 없음 |
| 대표 오류 | `MATCH_NOT_FOUND`, `VALIDATION_FAILED` |

요청 JSON: 본문 없음.

성공 응답 JSON:

```json
{"id":42,"host":{"id":7,"nickname":"풋살러"},"court":{"id":11,"name":"A 코트","venue":{"id":3,"name":"풋살파크","regionCode":"GYEONGGI_SUWON"}},"title":"토요일 오전 풋살","instructions":"10분 전에 모여 주세요.","recommendedSkill":"BEGINNER","status":"SCHEDULED","recruitmentStatus":"OPEN","startAt":"2026-08-15T10:00:00+09:00","endAt":"2026-08-15T12:00:00+09:00","maxParticipants":12,"confirmedParticipants":5,"remainingSlots":7,"displayStatus":"AVAILABLE","hostParticipates":true,"currentMemberParticipating":true,"currentMemberHost":false,"canJoin":false,"canCancelParticipation":true,"canEdit":false,"canCancelMatch":false}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":404,"code":"MATCH_NOT_FOUND","message":"매치를 찾을 수 없습니다.","path":"/api/matches/999","fieldErrors":[]}
```

추적: `MAT-002` → `matchId` 존재 확인 → Court를 통한 Venue 조합과 활성 참가 수 파생 → 200 또는 404.

### 5.3 매치 개설 — `MAT-003`

| 항목 | 계약 |
|---|---|
| 목적 | 로그인 회원이 코트를 선택해 매치를 개설하고 주최자가 된다. |
| Method / URI / 접근 | `POST /api/matches` / 로그인 필요 |
| Path / Query | 없음 |
| 검증 | 아래 필드, Court 존재, `endAt > startAt`; `startAt`이 미래인지는 초기안 |
| 성공 | `201 Created`, `MatchResponse`, `Location: /api/matches/{id}` |
| 상태 변화 | `SCHEDULED`·`OPEN` Match 생성. `hostParticipates=true`면 주최자의 `CONFIRMED` 참가도 같은 트랜잭션에서 생성 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `INVALID_REQUEST`, `VALIDATION_FAILED`, `COURT_NOT_FOUND`, `MATCH_INVALID_TIME_RANGE`, `CSRF_TOKEN_INVALID` |

| 필드 | JSON 타입 | 필수 | null | 핵심 제약 | 예시 |
|---|---|---:|---:|---|---|
| `courtId` | number | 예 | 불가 | 양의 정수, Court 존재 | `11` |
| `title` | string | 예 | 불가 | 빈·공백 불가, 최대 100자 초기안 | `토요일 오전 풋살` |
| `instructions` | string | 아니오 | 가능 | 제공 시 공백 불가, 최대 1000자 초기안 | `10분 전 집합` |
| `recommendedSkill` | string | 예 | 불가 | 지원 코드, 참가 제한으로 사용하지 않음 | `BEGINNER` |
| `startAt` | string | 예 | 불가 | 유효한 offset date-time, 미래 시각 초기안 | `2026-08-15T10:00:00+09:00` |
| `endAt` | string | 예 | 불가 | `startAt`보다 뒤 | `2026-08-15T12:00:00+09:00` |
| `maxParticipants` | number | 예 | 불가 | 정수, 최소 1 초기안; 최종 상한 열린 결정 | `12` |
| `hostParticipates` | boolean | 예 | 불가 | true면 일반 참가자처럼 정원 포함 | `true` |

요청 JSON:

```json
{"courtId":11,"title":"토요일 오전 풋살","instructions":"10분 전 집합","recommendedSkill":"BEGINNER","startAt":"2026-08-15T10:00:00+09:00","endAt":"2026-08-15T12:00:00+09:00","maxParticipants":12,"hostParticipates":true}
```

성공 응답 JSON:

```json
{"id":42,"host":{"id":7,"nickname":"풋살러"},"court":{"id":11,"name":"A 코트","venue":{"id":3,"name":"풋살파크","regionCode":"GYEONGGI_SUWON"}},"title":"토요일 오전 풋살","instructions":"10분 전 집합","recommendedSkill":"BEGINNER","status":"SCHEDULED","recruitmentStatus":"OPEN","startAt":"2026-08-15T10:00:00+09:00","endAt":"2026-08-15T12:00:00+09:00","maxParticipants":12,"confirmedParticipants":1,"remainingSlots":11,"displayStatus":"AVAILABLE","hostParticipates":true,"currentMemberParticipating":true,"currentMemberHost":true,"canJoin":false,"canCancelParticipation":true,"canEdit":true,"canCancelMatch":true}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":400,"code":"MATCH_INVALID_TIME_RANGE","message":"종료 시각은 시작 시각보다 뒤여야 합니다.","path":"/api/matches","fieldErrors":[]}
```

추적: `MAT-003` → 요청·Court·시간 검증 → `matches.host_member_id`, `court_id` FK 후보 → Match와 선택적 주최자 참가 원자 생성 → 201.

### 5.4 매치 수정 — `MAT-004`

| 항목 | 계약 |
|---|---|
| 목적 | 주최자가 시작 전 허용된 매치 정보를 부분 수정한다. |
| Method / URI / 접근 | `PATCH /api/matches/{matchId}` / 로그인 + 해당 주최자 |
| Path | `matchId`: 존재하는 매치 ID |
| Query | 없음 |
| 검증 | 한 필드 이상, 수정 후 전체 시간 범위, 정원과 활성 참가 수, 필드별 수정 정책 |
| 성공 | `200 OK`, 변경된 `MatchResponse` |
| 상태 변화 | 허용 필드만 변경. 정원 증가로 `CLOSED`를 자동 `OPEN`으로 바꾸지 않음 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MATCH_NOT_FOUND`, `MATCH_ACCESS_DENIED`, `MATCH_ALREADY_STARTED`, `MATCH_ALREADY_CANCELLED`, `COURT_NOT_FOUND`, `MATCH_INVALID_TIME_RANGE`, `MATCH_CAPACITY_BELOW_CONFIRMED_COUNT`, `INVALID_REQUEST`, `VALIDATION_FAILED`, `CSRF_TOKEN_INVALID` |

| 필드 | JSON 타입 | 필수 | null | 핵심 제약 |
|---|---|---:|---:|---|
| `title` | string | 아니오 | 불가 | 시작 전 수정, 공백 불가, 최대 100자 초기안 |
| `instructions` | string | 아니오 | 가능 | null은 제거, 문자열이면 공백 불가·최대 1000자 초기안 |
| `recommendedSkill` | string | 아니오 | 불가 | 활성 일반 참가자가 없을 때만 수정 초기안 |
| `courtId` | number | 아니오 | 불가 | Court 존재, 활성 일반 참가자가 없을 때만 수정 초기안 |
| `startAt`, `endAt` | string | 아니오 | 불가 | 활성 일반 참가자가 없을 때만 수정 초기안, 최종 범위 유효 |
| `maxParticipants` | number | 아니오 | 불가 | 최소 1 초기안, 전체 `CONFIRMED` 수 이상 |

요청 JSON:

```json
{"title":"토요일 오전 친선 풋살","instructions":null,"maxParticipants":14}
```

성공 응답 JSON:

```json
{"id":42,"host":{"id":7,"nickname":"풋살러"},"court":{"id":11,"name":"A 코트","venue":{"id":3,"name":"풋살파크","regionCode":"GYEONGGI_SUWON"}},"title":"토요일 오전 친선 풋살","instructions":null,"recommendedSkill":"BEGINNER","status":"SCHEDULED","recruitmentStatus":"OPEN","startAt":"2026-08-15T10:00:00+09:00","endAt":"2026-08-15T12:00:00+09:00","maxParticipants":14,"confirmedParticipants":5,"remainingSlots":9,"displayStatus":"AVAILABLE","hostParticipates":true,"currentMemberParticipating":true,"currentMemberHost":true,"canJoin":false,"canCancelParticipation":true,"canEdit":true,"canCancelMatch":true}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":409,"code":"MATCH_CAPACITY_BELOW_CONFIRMED_COUNT","message":"현재 참가 인원보다 정원을 줄일 수 없습니다.","path":"/api/matches/42","fieldErrors":[]}
```

추적: `MAT-004` → 인증·소유권·상태·필드 정책 검증 → 활성 참가 수와 정원 불변식 → 원자 수정 → 200 또는 403/404/409.

### 5.5 모집 마감 — `MAT-005`

| 항목 | 계약 |
|---|---|
| 목적 | 주최자가 시작 전 신규 참가 모집을 닫는다. |
| Method / URI / 접근 | `POST /api/matches/{matchId}/recruitment/close` / 로그인 + 해당 주최자 |
| Path / Query / Request body | `matchId` / 없음 / 없음 |
| 성공 | `204 No Content`, 본문 없음 |
| 상태 변화 | `OPEN → CLOSED` |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MATCH_NOT_FOUND`, `MATCH_ACCESS_DENIED`, `MATCH_ALREADY_STARTED`, `MATCH_ALREADY_CANCELLED`, `MATCH_RECRUITMENT_CLOSED`, `CSRF_TOKEN_INVALID` |

요청 JSON: 본문 없음. 성공 응답 JSON: 본문 없음.

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":409,"code":"MATCH_RECRUITMENT_CLOSED","message":"이미 모집이 마감되었습니다.","path":"/api/matches/42/recruitment/close","fieldErrors":[]}
```

추적: `MAT-005` → 인증·소유권·매치 상태·시간 확인 → 모집 상태 전이 → 204 또는 상태 충돌.

### 5.6 모집 재개 — `MAT-006`

| 항목 | 계약 |
|---|---|
| 목적 | 주최자가 남은 자리가 있는 시작 전 매치의 모집을 다시 연다. |
| Method / URI / 접근 | `POST /api/matches/{matchId}/recruitment/reopen` / 로그인 + 해당 주최자 |
| Path / Query / Request body | `matchId` / 없음 / 없음 |
| 성공 | `204 No Content`, 본문 없음 |
| 상태 변화 | 조건 충족 시 `CLOSED → OPEN` |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MATCH_NOT_FOUND`, `MATCH_ACCESS_DENIED`, `MATCH_ALREADY_STARTED`, `MATCH_ALREADY_CANCELLED`, `MATCH_REOPEN_NOT_ALLOWED`, `CSRF_TOKEN_INVALID` |

요청 JSON: 본문 없음. 성공 응답 JSON: 본문 없음.

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":409,"code":"MATCH_REOPEN_NOT_ALLOWED","message":"남은 자리가 없어 모집을 재개할 수 없습니다.","path":"/api/matches/42/recruitment/reopen","fieldErrors":[]}
```

이미 `OPEN`이거나 남은 자리가 없으면 같은 코드로 상태 충돌을 알리는 초기안이다. 세부 message는 조건별로 달라질 수 있으나 code 의미는 “현재 상태에서 재개 불가”로 유지한다.

추적: `MAT-006` → 인증·소유권·상태·시간·남은 자리 확인 → 모집 상태 전이 → 204 또는 상태 충돌.

### 5.7 매치 취소 — `MAT-007`

| 항목 | 계약 |
|---|---|
| 목적 | 주최자가 매치를 취소하고 활성 참가를 함께 취소한다. |
| Method / URI / 접근 | `POST /api/matches/{matchId}/cancel` / 로그인 + 해당 주최자 |
| Path / Query / Request body | `matchId` / 없음 / 없음 |
| 성공 | `204 No Content`, 본문 없음 |
| 상태 변화 | `SCHEDULED → CANCELLED`, 모든 `CONFIRMED → CANCELLED_BY_MATCH`; 하나의 유스케이스 트랜잭션 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MATCH_NOT_FOUND`, `MATCH_ACCESS_DENIED`, `MATCH_ALREADY_STARTED`, `MATCH_ALREADY_CANCELLED`, `CSRF_TOKEN_INVALID` |

요청 JSON: 본문 없음. 성공 응답 JSON: 본문 없음.

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":409,"code":"MATCH_ALREADY_CANCELLED","message":"이미 취소된 매치입니다.","path":"/api/matches/42/cancel","fieldErrors":[]}
```

**초기안:** 시작한 매치의 취소는 허용하지 않는다. 취소 시 모집 상태 컬럼까지 `CLOSED`로 바꿀지는 열린 결정이며, 어떤 선택에서도 취소 매치에는 참가할 수 없다.

추적: `MAT-007` → 인증·소유권·상태·시간 확인 → Match와 모든 활성 참가 원자 전이 → 204 또는 403/404/409.

### 5.8 내가 주최한 매치 — `MAT-008`

| 항목 | 계약 |
|---|---|
| 목적 | 로그인 회원이 자신이 주최한 매치를 상태와 관계없이 확인한다. |
| Method / URI / 접근 | `GET /api/members/me/hosted-matches` / 로그인 필요 |
| Query | `status`, `recruitmentStatus`, `startFrom`, `startTo`, `page`, `size`; 모두 선택 |
| Request body | 없음 |
| 성공 | `200 OK`, `MatchResponse` Page; 기본 정렬 `startAt DESC, id DESC` 초기안 |
| 상태 변화 | 없음 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `VALIDATION_FAILED` |

요청 JSON: 본문 없음. 예: `GET /api/members/me/hosted-matches?status=SCHEDULED&page=0&size=20`.

성공 응답 JSON:

```json
{"content":[{"id":42,"host":{"id":7,"nickname":"풋살러"},"court":{"id":11,"name":"A 코트","venue":{"id":3,"name":"풋살파크","regionCode":"GYEONGGI_SUWON"}},"title":"토요일 오전 풋살","instructions":null,"recommendedSkill":"BEGINNER","status":"SCHEDULED","recruitmentStatus":"OPEN","startAt":"2026-08-15T10:00:00+09:00","endAt":"2026-08-15T12:00:00+09:00","maxParticipants":12,"confirmedParticipants":5,"remainingSlots":7,"displayStatus":"AVAILABLE","hostParticipates":true,"currentMemberParticipating":true,"currentMemberHost":true,"canJoin":false,"canCancelParticipation":true,"canEdit":true,"canCancelMatch":true}],"page":0,"size":20,"totalElements":1,"totalPages":1,"first":true,"last":true}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":401,"code":"AUTHENTICATION_REQUIRED","message":"로그인이 필요합니다.","path":"/api/members/me/hosted-matches","fieldErrors":[]}
```

추적: `MAT-008` → 세션 회원 확인 → `matches.host_member_id` 관계 조회 → Page 200.

## 6. 참가 API

### 6.1 매치 참가·재참가 — `PAR-001`

| 항목 | 계약 |
|---|---|
| 목적 | 로그인 회원이 처음 참가하거나 회원 취소 상태에서 재참가한다. |
| Method / URI / 접근 | `POST /api/matches/{matchId}/participations` / 로그인 필요 |
| Path | `matchId`: 존재하는 매치 ID |
| Query / Request body | 없음 / 없음 |
| 검증 | `SCHEDULED`, `OPEN`, `현재 < startAt`, 남은 자리, 기존 참가 상태 |
| 성공 | 관계 없음이면 `201 Created`, `CANCELLED_BY_MEMBER` 재참가면 `200 OK`; 모두 `ParticipationResponse` 본문, `Location` 없음 초기안 |
| 상태 변화 | 없음 → `CONFIRMED` 생성 또는 `CANCELLED_BY_MEMBER → CONFIRMED`; 활성 참가 수 +1 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MATCH_NOT_FOUND`, `MATCH_ALREADY_STARTED`, `MATCH_ALREADY_CANCELLED`, `MATCH_RECRUITMENT_CLOSED`, `MATCH_FULL`, `PARTICIPATION_ALREADY_CONFIRMED`, `PARTICIPATION_REJOIN_NOT_ALLOWED`, `CSRF_TOKEN_INVALID` |

요청 JSON: 본문 없음.

성공 응답 JSON:

```json
{"id":81,"matchId":42,"memberId":9,"status":"CONFIRMED","createdAt":"2026-08-02T10:00:00Z","updatedAt":"2026-08-02T10:00:00Z"}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":409,"code":"MATCH_FULL","message":"남은 자리가 없습니다.","path":"/api/matches/42/participations","fieldErrors":[]}
```

한 회원·매치의 관계는 `UNIQUE(match_id, member_id)` 후보로 방어한다. 단순 선조회만으로 마지막 자리 경합이 해결된다고 가정하지 않으며 구체 락·갱신 방식은 동시성 테스트 시 결정한다. 실패 요청은 참가 상태와 정원에 영향을 주지 않아야 한다.

기존 관계가 `CANCELLED_BY_MATCH`이면 Match가 취소됐다는 일반 충돌보다 관계의 영구 재참가 금지를 구체적으로 나타내는 `PARTICIPATION_REJOIN_NOT_ALLOWED`를 우선 반환하는 **초기안**이다. 참가 관계가 없거나 다른 상태에서 취소된 Match에 요청하면 `MATCH_ALREADY_CANCELLED`를 사용한다.

추적: `PAR-001` → `matchId` 존재 → 상태·모집·시간·기존 관계·정원 검증 → 관계 UNIQUE와 정원 불변식 → 201/200 → 상태별 409.

### 6.2 내 참가 취소 — `PAR-002`

| 항목 | 계약 |
|---|---|
| 목적 | 회원이 시작 전 자신의 활성 참가를 취소한다. |
| Method / URI / 접근 | `DELETE /api/matches/{matchId}/participations/me` / 로그인 필요, 본인 참가 관계 |
| Path | `matchId`: 존재하는 매치 ID |
| Query / Request body | 없음 / 없음 |
| 검증 | 현재 참가가 `CONFIRMED`, `현재 < startAt` |
| 성공 | 첫 정상 취소는 `204 No Content`, 본문 없음 **추천안** |
| 상태 변화 | `CONFIRMED → CANCELLED_BY_MEMBER`; 활성 참가 수 -1. 주최자 관계는 유지 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MATCH_NOT_FOUND`, `PARTICIPATION_NOT_FOUND`, `PARTICIPATION_CANCEL_NOT_ALLOWED`, `CSRF_TOKEN_INVALID`; 반복 취소의 오류 여부는 열린 결정 |

요청 JSON: 본문 없음. 성공 응답 JSON: 본문 없음.

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":409,"code":"PARTICIPATION_CANCEL_NOT_ALLOWED","message":"매치 시작 후에는 참가를 취소할 수 없습니다.","path":"/api/matches/42/participations/me","fieldErrors":[]}
```

**열린 결정:** 이미 `CANCELLED_BY_MEMBER`인 반복 요청을 멱등한 204로 볼지 `409 PARTICIPATION_NOT_ACTIVE`로 볼지 결정하지 않는다. 어느 선택에서도 상태가 다시 바뀌거나 자리가 두 번 복구되어서는 안 된다. `CANCELLED_BY_MATCH`는 회원의 반복 취소가 아니라 매치 취소로 인해 영구 비활성화된 관계이므로 `409 PARTICIPATION_CANCEL_NOT_ALLOWED`를 반환하는 초기안이다.

추적: `PAR-002` → 인증·Match·본인 관계·시간·활성 상태 확인 → 단일 상태 전이 → 204 → 없음/시간/상태 오류.

### 6.3 내가 참가한 매치 — `PAR-003`

| 항목 | 계약 |
|---|---|
| 목적 | 회원이 자신의 참가 관계와 연결된 매치를 확인한다. |
| Method / URI / 접근 | `GET /api/members/me/participating-matches` / 로그인 필요 |
| Query | `participationStatus`, `matchStatus`, `startFrom`, `startTo`, `page`, `size`; 선택 |
| Request body | 없음 |
| 성공 | `200 OK`, `{ participation, match }` 항목 Page. 기본은 `CONFIRMED`, `startAt ASC, id ASC` 초기안 |
| 상태 변화 | 없음 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `VALIDATION_FAILED` |

요청 JSON: 본문 없음. 예: `GET /api/members/me/participating-matches?participationStatus=CONFIRMED&page=0&size=20`.

성공 응답 JSON:

```json
{"content":[{"participation":{"id":81,"matchId":42,"memberId":9,"status":"CONFIRMED","createdAt":"2026-08-02T10:00:00Z","updatedAt":"2026-08-02T10:00:00Z"},"match":{"id":42,"title":"토요일 오전 풋살","status":"SCHEDULED","recruitmentStatus":"OPEN","startAt":"2026-08-15T10:00:00+09:00","endAt":"2026-08-15T12:00:00+09:00","remainingSlots":7,"currentMemberParticipating":true}}],"page":0,"size":20,"totalElements":1,"totalPages":1,"first":true,"last":true}
```

목록의 `match`는 `MatchResponse`의 축약 표현 **초기안**이며 구현에서 전체 응답으로 통일할 수 있다. 필드 선택이 바뀌어도 참가 상태와 Match 상태를 혼동하지 않는다.

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":400,"code":"VALIDATION_FAILED","message":"요청 값이 올바르지 않습니다.","path":"/api/members/me/participating-matches","fieldErrors":[{"field":"participationStatus","rejectedValue":"ACTIVE","message":"지원하지 않는 참가 상태입니다."}]}
```

추적: `PAR-003` → 세션 회원·필터 검증 → `match_participations.member_id` 관계와 Match 조회 → Page 200.

### 6.4 주최자의 참가자 목록 — `PAR-004`

| 항목 | 계약 |
|---|---|
| 목적 | 주최자가 해당 매치의 참가 관계를 확인한다. |
| Method / URI / 접근 | `GET /api/matches/{matchId}/participations` / 로그인 + 해당 주최자 |
| Path | `matchId`: 존재하는 매치 ID |
| Query | `status`, `page`, `size`; 상태 기본값 `CONFIRMED`, 기본 정렬 `createdAt ASC, id ASC` 초기안 |
| Request body | 없음 |
| 성공 | `200 OK`, 회원 요약을 포함한 참가 항목 Page |
| 상태 변화 | 없음 |
| 대표 오류 | `AUTHENTICATION_REQUIRED`, `MATCH_NOT_FOUND`, `MATCH_ACCESS_DENIED`, `VALIDATION_FAILED` |

요청 JSON: 본문 없음. 예: `GET /api/matches/42/participations?status=CONFIRMED&page=0&size=20`.

성공 응답 JSON:

```json
{"content":[{"id":81,"matchId":42,"member":{"id":9,"nickname":"참가자"},"status":"CONFIRMED","createdAt":"2026-08-02T10:00:00Z","updatedAt":"2026-08-02T10:00:00Z"}],"page":0,"size":20,"totalElements":1,"totalPages":1,"first":true,"last":true}
```

대표 오류 JSON:

```json
{"timestamp":"2026-08-02T10:00:00Z","status":403,"code":"MATCH_ACCESS_DENIED","message":"해당 매치의 주최자만 참가자를 조회할 수 있습니다.","path":"/api/matches/42/participations","fieldErrors":[]}
```

주최자가 `CONFIRMED` 참가 관계를 가지면 일반 참가자와 같은 목록에 포함한다. 참가자 응답에는 비밀번호 해시, 이메일과 세션 정보 등 불필요한 개인정보를 포함하지 않는다.

추적: `PAR-004` → 인증·Match 존재·주최자 소유권·필터 확인 → `match_participations.match_id` 조회 → Page 200 또는 403/404.

## 7. HTTP 상태 코드 정책

| HTTP | 사용 조건 | 대표 API·오류 |
|---:|---|---|
| `200 OK` | 일반 조회, 본문이 필요한 수정·로그인, 재참가 | GET 전반, `AUTH-001`, `MEM-003`, `MAT-004`, PAR-001 재참가 |
| `201 Created` | 새 리소스 생성 | `MEM-001`, `MAT-003`, PAR-001 최초 관계 생성 |
| `204 No Content` | 본문 없는 명령 성공 | `AUTH-002`, `MAT-005`~`007`, `PAR-002` 첫 취소 |
| `400 Bad Request` | 깨진 JSON, 필드·query 검증, 지원하지 않는 enum, 잘못된 시간 범위 | `INVALID_REQUEST`, `VALIDATION_FAILED`, `MATCH_INVALID_TIME_RANGE` |
| `401 Unauthorized` | 로그인 실패 또는 인증되지 않은 보호 요청 | `INVALID_CREDENTIALS`, `AUTHENTICATION_REQUIRED` |
| `403 Forbidden` | 인증됐으나 매치 소유권 없음, CSRF 검증 실패 | `MATCH_ACCESS_DENIED`, `CSRF_TOKEN_INVALID` |
| `404 Not Found` | 회원·구장·코트·매치·참가 관계가 없음 | 리소스별 `*_NOT_FOUND` |
| `409 Conflict` | 현재 상태와 명령 충돌 | 이미 참가, 모집 마감, 정원 마감, 시작·취소, 불가한 재참가·정원 감소 |
| `500 Internal Server Error` | 예상하지 못한 서버 오류 | `INTERNAL_SERVER_ERROR` |

**초기 추천:** `400`과 `409`로 형식·값 오류와 상태 충돌을 구분할 수 있으므로 1차 MVP에서 `422 Unprocessable Entity`를 추가하지 않는다. 필요성이 확인되면 모든 관련 API와 클라이언트 분기에 미치는 영향을 검토한다.

## 8. 오류 코드 카탈로그

| 코드 | HTTP | 의미 | 발생 조건 | 관련 API | 결정 상태 |
|---|---:|---|---|---|---|
| `INVALID_REQUEST` | 400 | 요청 문법 오류 | 깨진 JSON, 읽을 수 없는 본문 | body가 있는 API | 초기안 |
| `VALIDATION_FAILED` | 400 | 필드·path·query 검증 실패 | 누락, null, 공백, 형식, 범위, 지원하지 않는 enum | 전체 | 초기안 |
| `INTERNAL_SERVER_ERROR` | 500 | 예상하지 못한 오류 | 내부 오류를 안전하게 감춤 | 전체 | 초기안 |
| `AUTHENTICATION_REQUIRED` | 401 | 로그인 필요 | 유효한 인증 세션 없음 | 보호 API | 초기안 |
| `INVALID_CREDENTIALS` | 401 | 로그인 실패 | 이메일·비밀번호 불일치; 계정 존재 여부 비공개 | `AUTH-001` | 확정 의미, 표현 초기안 |
| `CSRF_TOKEN_INVALID` | 403 | CSRF 검증 실패 | 토큰 없음·불일치·만료 | 상태 변경 API | CSRF 방식과 함께 초기안 |
| `MEMBER_NOT_FOUND` | 404 | 회원 없음 | 인증 식별자에 대응하는 회원 없음 | `MEM-002`, `MEM-003`, `AUTH-003` | 초기안 |
| `MEMBER_EMAIL_DUPLICATED` | 409 | 이메일 충돌 | 가입 이메일이 이미 사용됨 | `MEM-001` | 초기안 |
| `MEMBER_NICKNAME_DUPLICATED` | 409 | 닉네임 충돌 | 닉네임 유일 정책을 선택한 경우 | `MEM-001`, `MEM-003` | **열린 결정 조건부** |
| `VENUE_NOT_FOUND` | 404 | 구장 없음 | path의 구장 ID 없음 | `VEN-002`, `VEN-003` | 초기안 |
| `COURT_NOT_FOUND` | 404 | 코트 없음 | 요청 body의 Court ID 없음 | `MAT-003`, `MAT-004` | 초기안 |
| `MATCH_NOT_FOUND` | 404 | 매치 없음 | path의 Match ID 없음 | MAT-002, MAT-004~007, PAR-001~002, PAR-004 | 초기안 |
| `MATCH_ACCESS_DENIED` | 403 | 매치 소유권 부족 | 인증 회원이 해당 주최자가 아님 | MAT-004~007, PAR-004 | 초기안 |
| `MATCH_ALREADY_STARTED` | 409 | 이미 시작함 | `현재 시각 >= startAt`에서 금지 명령 | MAT-004~007, PAR-001 | 초기안 |
| `MATCH_ALREADY_CANCELLED` | 409 | 이미 취소됨 | Match가 `CANCELLED` | MAT-004~007, PAR-001 | 초기안 |
| `MATCH_RECRUITMENT_CLOSED` | 409 | 모집 마감 | 참가 시 `CLOSED`, 또는 중복 마감 | MAT-005, PAR-001 | 초기안 |
| `MATCH_FULL` | 409 | 남은 자리 없음 | 참가·재참가 시 활성 수가 정원에 도달 | PAR-001 | 확정 의미, 구현 해법 열린 결정 |
| `MATCH_INVALID_TIME_RANGE` | 400 | 매치 시간 범위 오류 | `endAt <= startAt` 등 | MAT-003, MAT-004 | 초기안 |
| `MATCH_CAPACITY_BELOW_CONFIRMED_COUNT` | 409 | 정원 감소 충돌 | 변경 정원이 `CONFIRMED` 수보다 작음 | `MAT-004` | 확정 의미, 코드 초기안 |
| `MATCH_REOPEN_NOT_ALLOWED` | 409 | 모집 재개 불가 | 이미 OPEN, 남은 자리 없음 또는 재개 불가 상태 | `MAT-006` | 초기안 |
| `PARTICIPATION_ALREADY_CONFIRMED` | 409 | 이미 참가 중 | 기존 관계가 `CONFIRMED` | `PAR-001` | 초기안 |
| `PARTICIPATION_NOT_FOUND` | 404 | 참가 관계 없음 | 본인 참가 관계를 찾지 못함 | `PAR-002` | 초기안 |
| `PARTICIPATION_NOT_ACTIVE` | 409 | 활성 참가 아님 | 반복 취소를 충돌로 정할 경우 | `PAR-002` | **열린 결정 조건부** |
| `PARTICIPATION_CANCEL_NOT_ALLOWED` | 409 | 참가 취소 불가 | 시작 이후 등 취소 금지 조건 | `PAR-002` | 초기안 |
| `PARTICIPATION_REJOIN_NOT_ALLOWED` | 409 | 재참가 불가 | `CANCELLED_BY_MATCH` 등 재참가 금지 상태 | `PAR-001` | 초기안 |

`ACCESS_DENIED`는 검토했지만 1차 MVP에 전역 역할로 제한하는 API가 없으므로 활성 카탈로그에서 제외한다. 매치 주최자 소유권 부족에는 `MATCH_ACCESS_DENIED`를 사용하며, 향후 ADMIN 전용 API가 생기면 일반 권한 코드를 다시 검토한다. `COURT_NOT_AVAILABLE`도 실제 대관 가능 여부를 보장하는 코드로 오해되므로 제외한다. 1차 MVP Court는 정보용이며 존재하지 않으면 `COURT_NOT_FOUND`만 사용한다. 필드마다 별도 top-level 오류 코드를 만들지 않고 `VALIDATION_FAILED.fieldErrors`로 표현한다. Java 예외 클래스와 오류 코드를 일대일로 고정하지 않는다.

## 9. 요청·응답·오류 추적성 요약

| 요구사항 | 요청 검증·도메인 규칙 | DB 제약 후보 | 성공 | 대표 오류 |
|---|---|---|---|---|
| `MEM-001` | 필수값, 이메일 형식·중복, 비밀번호 해시 | `members.email` UNIQUE·NOT NULL | 201 | `VALIDATION_FAILED`, `MEMBER_EMAIL_DUPLICATED` |
| `VEN-003` | 구장 존재, 구장별 코트 관계 | `courts.venue_id` FK·NOT NULL | 200 Page | `VENUE_NOT_FOUND` |
| `MAT-003` | Court, 시간, 정원, 선택적 주최자 참가 | Match FK·NOT NULL, 참가 UNIQUE 후보 | 201 | `COURT_NOT_FOUND`, `MATCH_INVALID_TIME_RANGE` |
| `MAT-004` | 주최자·상태·시간·필드 정책·활성 참가 수 | Match FK·NOT NULL; 교차 행 불변식은 앱 트랜잭션 | 200 | `MATCH_ACCESS_DENIED`, `MATCH_CAPACITY_BELOW_CONFIRMED_COUNT` |
| `MAT-007` | 주최자·상태·시간, 활성 참가 일괄 전이 | 상태 허용 값 후보 | 204 | `MATCH_ALREADY_CANCELLED`, `MATCH_ALREADY_STARTED` |
| `PAR-001` | Match 존재·참가 가능·기존 상태·정원 | `UNIQUE(match_id, member_id)`, FK·NOT NULL | 최초 201 / 재참가 200 | `PARTICIPATION_ALREADY_CONFIRMED`, `MATCH_FULL` |
| `PAR-002` | 본인 활성 참가·시작 전 | 단일 관계와 허용 상태 코드 | 204 | `PARTICIPATION_NOT_FOUND`, `PARTICIPATION_CANCEL_NOT_ALLOWED` |

나머지 조회 API도 API별 `추적` 문장에서 기능 ID, 검증, 관계, 성공과 오류를 연결한다. 일반 DB CHECK만으로 참가 수와 정원의 교차 행 불변식을 보장한다고 가정하지 않는다.

## 10. 열린 결정과 변경 영향

| 항목 | 현재 상황 | 선택지 | 현재 추천안 | 결정 시점 | 변경 영향 |
|---|---|---|---|---|---|
| 현재 참가 수 | 응답·정원 검증에 필요 | 매번 집계 / Match 저장 / 혼합 | 추천안 없음 | PAR-001 조회·동시성 테스트 전 | ERD, 쿼리, 트랜잭션, 응답 파생 값 |
| 마지막 자리 경합 | 선조회만으로 불변식 보장 불가 | 비관·낙관 락 / 조건부 갱신 등 | 추천안 없음 | 재현 테스트 작성 시 | Repository, 실패·재시도, 성능 |
| 반복 참가 취소 | 상태 중복 변화 금지만 확정 | 멱등 204 / 409 충돌 | 추천안 없음 | PAR-002 구현 전 | 오류 코드, 클라이언트 재시도 |
| 닉네임 중복 | 유일 정책 미승인 | 허용 / 전역 유일 | 추천안 없음 | 회원 검증·제약 구현 전 | 요청 검증, UNIQUE, 조건부 오류 코드 |
| 시간·타임존 | 외부 offset 형식만 초기 제안 | UTC 저장 / offset·zone 보존 등 | 추천안 없음 | Match 영속화 전 | DB 타입, 직렬화, 비교·테스트 |
| 페이지 방식 | 초기 계약은 0-based Page | 0/1 기반, Page/Slice | 0-based Page | 목록 구현 전 | 모든 query와 Page 본문 |
| 크기·정렬 | size 20/100, endpoint별 정렬 초기안 | 사용성·쿼리 측정 후 조정 | 현재 값을 출발점으로 사용 | 조회 구현·실행 계획 확인 시 | API, 인덱스, 클라이언트 |
| 성공 래퍼 | 직접 본문 초기안 | 직접 반환 / `data` 래퍼 | 직접 반환 | 첫 endpoint 구현 전 | 모든 성공 응답 |
| CSRF 초기화 | 클라이언트·저장소 미정 | 별도 GET / 쿠키·헤더 활용 | 추천안 없음 | 프론트엔드 인증 통합 전 | AUTH-004, 보안 설정, 클라이언트 |
| 422 | 400·409로 현재 구분 가능 | 미사용 / 도입 | 미사용 | 새 오류 분류 필요 시 | HTTP·오류 매핑 전반 |
| 표시·권한 편의 필드 | 클라이언트 편의를 위해 파생 필드 제안 | 서버 계산 / 클라이언트 계산 / 일부 제거 | 현재 `MatchResponse`를 검증 | 첫 목록·상세 구현 시 | 조회 쿼리, 응답 크기, 공개·인증 계약 |
| `Location` | 대응 GET이 있는 Match만 명확 | 생성마다 사용 / 조회 URI가 있을 때만 사용 | Match 개설에만 우선 사용 | 생성 API 구현 시 | 헤더 계약, 리소스 URI |

열린 결정의 결론은 [결정 기록](./01-decisions.md), 요구사항, ERD와 테스트에 함께 반영한다. 구현 중에는 문제·불변식·재현 조건을 먼저 확인하고 특정 락, 멱등성 키, 캐시나 메시징을 이 계약만으로 도입하지 않는다.
