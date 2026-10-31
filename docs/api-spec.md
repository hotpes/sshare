# LastEat API 명세 v0.1

> 2026-10-31 팀 회의 결정 반영: **리뷰·쿠폰 기능 전부 삭제.** 별점 표시도 함께 제거.
> 기준 화면: 손님 앱 UI 초안 7개 화면, 사장님 앱 UI (`Owner` 브랜치).

## 목차

- [0. 범위](#0-범위)
- [1. 공통 규칙](#1-공통-규칙)
- [2. 인증·회원](#2-인증회원)
- [3. 가게 (손님)](#3-가게-손님)
- [4. 혜택 (손님)](#4-혜택-손님)
- [5. 저장](#5-저장)
- [6. 사장님](#6-사장님)
- [7. 이미지 업로드](#7-이미지-업로드)
- [8. 코드 값](#8-코드-값)
- [9. 에러 코드](#9-에러-코드)
- [10. 명세 기준 엔티티 변경 사항](#10-명세-기준-엔티티-변경-사항)

---

## 0. 범위

| 포함 | 제외 |
|---|---|
| 카카오 로그인, 내 정보 | 리뷰, 별점, 리뷰 답글 (회의 결정) |
| 주변 가게 검색, 키워드 검색, 가게 상세, 메뉴 | 쿠폰 받기, 내 쿠폰, 매장 쿠폰 사용 처리 (회의 결정) |
| 주변 혜택 목록·상세 | 결제, 환불, 정산 |
| 가게 저장 | 저장 컬렉션 (2차) |
| 사장님 가게·메뉴·영업시간·영업 상태 관리 | 사장님 소식, 통계, 알림 (2차) |
| 혜택 등록 신청 (검토 후 노출) | 관리자 화면 (혜택 승인은 운영진이 DB에서 처리) |
| 이미지 업로드 | 공지사항, 자주 묻는 질문 (외부 페이지 링크) |

> 마감할인·예약 API는 이 문서에 포함하지 않았다. 엔티티(`ClosingDeal`, `Reservation`)는 남아 있고, 진행이 결정되면 별도 섹션으로 추가한다.

---

## 1. 공통 규칙

### 1.1 기본

| 항목 | 값 |
|---|---|
| Base URL | `/api/v1` |
| 형식 | JSON, UTF-8 (`Content-Type: application/json`) |
| 인증 | `Authorization: Bearer {accessToken}` |
| 시간대 | 한국 시간 (Asia/Seoul) |
| 날짜 | `2026-11-01` |
| 시각 | `14:00` (24시간, 분 단위) |
| 날짜+시각 | `2026-11-01T14:00:00+09:00` |
| 거리 | 미터(m) 정수 |
| 좌표 | `lat` 위도, `lng` 경도 (소수) |

인증 표시: 🔓 로그인 없이 가능 · 🔑 로그인 필요 · 🏪 사장님(`OWNER`) 로그인 필요

🔓 API에 토큰을 함께 보내면, 응답의 `bookmarked`처럼 로그인한 사람 기준 값이 채워진다. 토큰이 없으면 `false`.

### 1.2 응답 형식

성공

```json
{
  "success": true,
  "data": { },
  "error": null
}
```

실패

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "STORE_NOT_FOUND",
    "message": "가게를 찾을 수 없습니다."
  }
}
```

입력값 오류는 `error.fields`에 항목별 사유를 담는다.

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "INVALID_INPUT",
    "message": "입력값을 확인해 주세요.",
    "fields": [
      { "field": "title", "reason": "30자 이하로 입력해 주세요." }
    ]
  }
}
```

`message`는 앱 화면에 그대로 보여 줘도 되는 문장이다.

### 1.3 목록 (페이지)

요청: `page` (0부터, 기본 0), `size` (기본 20, 최대 100)

```json
{
  "success": true,
  "data": {
    "content": [ ],
    "page": 0,
    "size": 20,
    "totalElements": 24,
    "hasNext": true
  },
  "error": null
}
```

---

## 2. 인증·회원

로그인 흐름: 앱이 카카오 SDK로 로그인 → 받은 카카오 액세스 토큰을 서버에 전달 → 서버가 카카오에 확인 → 서버 자체 토큰(access/refresh) 발급. 처음 로그인하면 자동 가입된다.

| 토큰 | 유효 기간 |
|---|---|
| accessToken | 1시간 |
| refreshToken | 14일 |

### 2.1 카카오 로그인 🔓

`POST /auth/kakao`

요청

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| kakaoAccessToken | string | O | 카카오 SDK에서 받은 액세스 토큰 |
| fcmToken | string | | 푸시 알림 토큰 (2차 알림 기능용, 미리 저장) |

응답

```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "eyJhbGciOi...",
  "newMember": true,
  "member": {
    "id": 12,
    "nickname": "맛집탐험가",
    "role": "CONSUMER"
  }
}
```

에러: `KAKAO_AUTH_FAILED`

### 2.2 토큰 재발급 🔓

`POST /auth/refresh`

| 필드 | 타입 | 필수 |
|---|---|---|
| refreshToken | string | O |

응답: 2.1과 같은 `accessToken`, `refreshToken` (refresh도 새로 발급, 이전 것은 폐기)

에러: `INVALID_TOKEN`, `EXPIRED_TOKEN` → 다시 카카오 로그인

### 2.3 로그아웃 🔑

`POST /auth/logout`

요청 본문 없음. 해당 기기의 refreshToken을 폐기한다. 응답 `data`: `null`

### 2.4 내 정보 🔑

`GET /members/me`

```json
{
  "id": 12,
  "nickname": "맛집탐험가",
  "email": "user@kakao.com",
  "role": "OWNER",
  "ownedStoreCount": 1
}
```

`email`은 카카오 동의 항목에 따라 `null`일 수 있다. `ownedStoreCount`가 1 이상이면 마이 화면에 사장님 메뉴를 보여 준다.

### 2.5 푸시 토큰 갱신 🔑

`PATCH /members/me/fcm-token`

| 필드 | 타입 | 필수 |
|---|---|---|
| fcmToken | string | O |

### 2.6 회원 탈퇴 🔑

`DELETE /members/me`

저장한 가게와 토큰을 함께 삭제한다. 사장님으로 연결된 가게가 있으면 `OWNER_HAS_STORE` 에러 (운영진에게 연결 해제 요청).

---

## 3. 가게 (손님)

### 3.1 주변 가게 목록·검색 🔓

`GET /stores`

화면: 주변 탐색 (지도 마커 + 하단 목록), 검색 결과 (지도·목록)

| 파라미터 | 타입 | 필수 | 기본 | 설명 |
|---|---|---|---|---|
| lat | number | O | | 기준 위도 (내 위치 또는 지도 중심) |
| lng | number | O | | 기준 경도 |
| radius | int | | 500 | 반경(m), 최대 3000 |
| keyword | string | | | 가게 이름·카테고리 이름·메뉴 이름 검색 (2자 이상) |
| category | string | | | `KOREAN` 등, 여러 개는 쉼표 (`KOREAN,CAFE`) |
| openNow | boolean | | false | 지금 영업 중인 곳만 |
| hasBenefit | boolean | | false | 지금 이용 가능한 혜택이 있는 곳만 |
| sort | string | | DISTANCE | `DISTANCE` 가까운 순 (1차는 이것만) |
| page, size | int | | 0, 20 | 지도 마커를 한 번에 그릴 때는 `size=100` |

"이 지역 다시 찾기"는 지도 중심 좌표로 다시 호출한다.

응답 `content[]`

```json
{
  "id": 3,
  "name": "연남 온기",
  "category": "KOREAN",
  "signatureMenuName": "연어 아보카도 덮밥",
  "thumbnailUrl": "https://cdn.lasteat.app/stores/3/main.jpg",
  "lat": 37.5622,
  "lng": 126.9255,
  "distance": 240,
  "businessStatus": "OPEN",
  "recommended": true,
  "mainBenefit": {
    "id": 11,
    "title": "점심특가 10% 할인"
  },
  "bookmarked": false
}
```

- `signatureMenuName`: 대표 메뉴 중 첫 번째, 없으면 `null`
- `mainBenefit`: 지금 노출 중인 혜택 중 마감이 가장 가까운 1개, 없으면 `null`
- `businessStatus`: [8.2 영업 상태](#82-영업-상태) 참고
- "지금 영업 중인 24곳"은 `openNow=true`로 호출한 `totalElements`

### 3.2 가게 상세 🔓

`GET /stores/{storeId}`

화면: 식당 상세 (정보 탭, 대표 메뉴, 챙겨가는 혜택)

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| lat, lng | number | | 보내면 `distance` 계산, 없으면 `null` |

```json
{
  "id": 3,
  "name": "연남 온기",
  "category": "KOREAN",
  "description": "제철 재료로 채운 든든한 한 그릇.",
  "address": "서울 마포구 동교로38길 12, 1층",
  "phone": "02-332-0428",
  "lat": 37.5622,
  "lng": 126.9255,
  "distance": 240,
  "imageUrls": [
    "https://cdn.lasteat.app/stores/3/main.jpg",
    "https://cdn.lasteat.app/stores/3/2.jpg"
  ],
  "tags": ["혼밥 좋아요", "포장 가능", "주차 불가"],
  "recommended": true,
  "businessStatus": "OPEN",
  "businessHours": {
    "openTime": "11:00",
    "closeTime": "21:00",
    "breakStart": "15:30",
    "breakEnd": "17:00",
    "closedDays": ["MON"]
  },
  "signatureMenus": [
    {
      "id": 21,
      "name": "연어 아보카도 덮밥",
      "description": "구운 연어와 고소한 아보카도의 조화",
      "price": 15000,
      "imageUrl": "https://cdn.lasteat.app/menus/21.jpg",
      "soldOut": false
    }
  ],
  "menuCount": 8,
  "benefits": [
    {
      "id": 11,
      "title": "식사 금액 10% 할인",
      "type": "DISCOUNT",
      "partner": true,
      "summary": "평일 14~17시 · 2만원 이상 · 최대 5천원"
    }
  ],
  "bookmarked": false
}
```

- `imageUrls[0]`이 대표 사진, 사진 개수는 배열 길이 ("사진 24")
- 마지막 주문 시간은 1차 범위에서 제외 (필요하면 `lastOrderTime` 추가)
- `benefits`는 지금 노출 중인 혜택 전체의 요약. 상세는 3.4 또는 4.2

에러: `STORE_NOT_FOUND`

### 3.3 메뉴 목록 🔓

`GET /stores/{storeId}/menus`

화면: 식당 상세 › 메뉴 탭, "전체 메뉴 8"

```json
[
  {
    "id": 21,
    "name": "연어 아보카도 덮밥",
    "description": "구운 연어와 고소한 아보카도의 조화",
    "price": 15000,
    "imageUrl": "https://cdn.lasteat.app/menus/21.jpg",
    "signature": true,
    "soldOut": false
  }
]
```

대표 메뉴 먼저, 그다음 등록 순. 페이지 없음.

### 3.4 가게 혜택 목록 🔓

`GET /stores/{storeId}/benefits`

화면: 식당 상세 › 혜택 탭

응답: [4.2 혜택 상세](#42-혜택-상세-) 객체의 배열. 지금 노출 중인 혜택만 (승인 완료, 노출 기간 안, 사장님이 중지하지 않은 것).

---

## 4. 혜택 (손님)

### 4.1 주변 혜택 목록 🔓

`GET /benefits`

화면: 혜택·이벤트 탭 "지금, 내 주변 혜택"

| 파라미터 | 타입 | 필수 | 기본 | 설명 |
|---|---|---|---|---|
| lat, lng | number | O | | 기준 좌표 |
| radius | int | | 1000 | 반경(m), 최대 3000 |
| type | string | | | `DISCOUNT`, `FREE_ITEM`, `ETC` |
| partner | boolean | | | `true`면 제휴 혜택만 |
| sort | string | | DISTANCE | `DISTANCE` 가까운 순, `ENDING_SOON` 곧 마감 순 |
| page, size | int | | 0, 20 | |

화면 필터 매핑: 전체 → 조건 없음 · 할인 → `type=DISCOUNT` · 제휴 혜택 → `partner=true` · 곧 마감 → `sort=ENDING_SOON`

응답 `content[]`

```json
{
  "id": 11,
  "title": "식사 금액 10% 할인",
  "type": "DISCOUNT",
  "partner": true,
  "summary": "평일 14~17시 · 2만원 이상 · 최대 5천원",
  "startDate": "2026-10-01",
  "endDate": "2026-10-31",
  "store": {
    "id": 3,
    "name": "연남 온기",
    "category": "KOREAN",
    "thumbnailUrl": "https://cdn.lasteat.app/stores/3/main.jpg",
    "distance": 240
  }
}
```

"맛있는 동네, 반가운 혜택 · 제휴 식당 12곳" 배너 숫자는 `partner=true`의 `totalElements`를 쓴다.

### 4.2 혜택 상세 🔓

`GET /benefits/{benefitId}`

```json
{
  "id": 11,
  "title": "식사 금액 10% 할인",
  "type": "DISCOUNT",
  "partner": true,
  "description": "여유로운 평일 오후, 식사 메뉴 주문 시 결제 금액의 10%를 할인해 드려요.",
  "discount": {
    "discountType": "RATE",
    "value": 10,
    "maxDiscount": 5000
  },
  "freeItem": null,
  "minOrderAmount": 20000,
  "startDate": "2026-11-01",
  "endDate": "2026-11-30",
  "days": ["MON", "TUE", "WED", "THU", "FRI"],
  "availableFrom": "14:00",
  "availableTo": "17:00",
  "restrictions": ["ONE_PER_PERSON", "NO_DUPLICATE", "DINE_IN_ONLY"],
  "extraNotice": "주류 및 포장 주문은 제외됩니다. 결제 전 직원에게 혜택 화면을 보여 주세요.",
  "summary": "평일 14~17시 · 2만원 이상 · 최대 5천원",
  "store": {
    "id": 3,
    "name": "연남 온기",
    "category": "KOREAN",
    "thumbnailUrl": "https://cdn.lasteat.app/stores/3/main.jpg",
    "distance": null
  }
}
```

- `discount`: `type=DISCOUNT`일 때만, 그 외 `null`
- `freeItem`: `type=FREE_ITEM`일 때 제공 메뉴 (예: `"아메리카노 1잔"`), 그 외 `null`
- `restrictions`는 안내 문구용이다. 쿠폰이 없으므로 서버가 사용 여부를 검증하지 않는다.
- `summary`는 서버가 조건을 조합해 만든 한 줄 문구 (목록 카드용)
- 담당자 연락처는 손님 응답에 포함하지 않는다.

에러: `BENEFIT_NOT_FOUND` (노출 중이 아닌 혜택 포함)

---

## 5. 저장

### 5.1 가게 저장 🔑

`POST /stores/{storeId}/bookmark`

이미 저장한 가게면 그대로 성공 (중복 저장 안 됨). 응답 `data`: `{ "bookmarked": true }`

### 5.2 저장 취소 🔑

`DELETE /stores/{storeId}/bookmark`

저장하지 않은 가게여도 성공. 응답 `data`: `{ "bookmarked": false }`

### 5.3 내 저장 목록 🔑

`GET /members/me/bookmarks`

화면: 저장 탭

| 파라미터 | 타입 | 필수 | 기본 | 설명 |
|---|---|---|---|---|
| category | string | | | 카테고리 필터 |
| openNow | boolean | | false | 영업 중만 |
| lat, lng | number | | | 보내면 `distance` 계산 |
| sort | string | | RECENT | `RECENT` 최근 저장 순, `DISTANCE` (좌표 필요) |
| page, size | int | | 0, 20 | |

응답 `content[]`: [3.1](#31-주변-가게-목록검색-) 목록 항목과 같은 형식 + `bookmarkedAt`

---

## 6. 사장님

모든 API 🏪. `{storeId}`가 본인 가게가 아니면 `NOT_STORE_OWNER` (403).

사장님 계정과 가게 연결은 1차에서는 운영진이 직접 처리한다 (사장님 인증 = 운영진이 연결해 준 상태).

### 6.1 내 가게 목록

`GET /owner/stores`

화면: 혜택 등록 › 식당 선택, 사장님 앱 진입

```json
[
  {
    "id": 3,
    "name": "연남 온기",
    "address": "서울 마포구 동교로38길 12, 1층",
    "thumbnailUrl": "https://cdn.lasteat.app/stores/3/main.jpg",
    "businessStatus": "OPEN"
  }
]
```

### 6.2 가게 정보 조회

`GET /owner/stores/{storeId}`

[3.2 가게 상세](#32-가게-상세-) 형식 + 사장님 전용 필드 `businessStatusOverride` (아래 6.5)

### 6.3 가게 정보 수정

`PUT /owner/stores/{storeId}`

화면: 사장님 앱 › 가게 정보

| 필드 | 타입 | 필수 | 제약 |
|---|---|---|---|
| name | string | O | 1~30자 |
| description | string | | 300자 이하 |
| address | string | O | |
| lat, lng | number | O | 주소를 바꾸면 앱이 지도 SDK로 좌표를 구해 함께 보낸다 |
| phone | string | | 숫자와 `-` |
| category | string | O | [8.1](#81-가게-카테고리) |
| imageUrls | string[] | | 최대 30장, 첫 번째가 대표 사진. [7](#7-이미지-업로드)에서 받은 URL |
| tags | string[] | | 최대 5개, 각 10자 이하 |

응답: 6.2와 같은 형식

### 6.4 영업시간 수정

`PUT /owner/stores/{storeId}/hours`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| openTime | string | O | `11:00` |
| closeTime | string | O | `21:00`. 자정을 넘기면 `openTime`보다 이른 시각 (예: `02:00`) |
| breakStart | string | | 쉬는 시간 시작, `breakEnd`와 함께 |
| breakEnd | string | | |
| closedDays | string[] | | 정기 휴무 요일 `["MON"]` |

사장님 앱은 요일을 숫자 집합(1=월 ~ 7=일)으로 가지고 있으므로 보낼 때 [8.3 요일](#83-요일) 코드로 바꾼다.

### 6.5 영업 상태 변경

`PATCH /owner/stores/{storeId}/business-status`

화면: 사장님 앱 › 영업 상태 (영업 중 / 준비 중 / 오늘 휴무)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| status | string | O | `OPEN`, `BREAK`, `CLOSED`, `AUTO` |

- `OPEN`·`BREAK`·`CLOSED`: 영업시간과 상관없이 **오늘 하루** 이 상태로 표시. 다음 날 0시에 자동으로 `AUTO`로 돌아간다.
- `AUTO`: 영업시간 기준으로 자동 계산

응답

```json
{
  "businessStatus": "CLOSED",
  "businessStatusOverride": "CLOSED"
}
```

### 6.6 메뉴 추가·수정·삭제

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/owner/stores/{storeId}/menus` | 메뉴 목록 (3.3과 같은 형식) |
| POST | `/owner/stores/{storeId}/menus` | 메뉴 추가 |
| PUT | `/owner/stores/{storeId}/menus/{menuId}` | 메뉴 수정 |
| DELETE | `/owner/stores/{storeId}/menus/{menuId}` | 메뉴 삭제 |
| PATCH | `/owner/stores/{storeId}/menus/{menuId}/sold-out` | 품절 표시 `{ "soldOut": true }` |

추가·수정 요청

| 필드 | 타입 | 필수 | 제약 |
|---|---|---|---|
| name | string | O | 1~30자 |
| price | int | O | 0 이상 |
| description | string | | 100자 이하 |
| imageUrl | string | | [7](#7-이미지-업로드)에서 받은 URL |
| signature | boolean | | 대표 메뉴 여부, 기본 false |

응답: 메뉴 객체 (3.3 항목). 에러: `MENU_NOT_FOUND`

### 6.7 혜택 등록 신청

`POST /owner/stores/{storeId}/benefits`

화면: 혜택 등록 (혜택 등록 신청하기). 신청하면 `PENDING`(검토 중)이 되고, 운영진이 승인하면 손님에게 노출된다.

| 필드 | 타입 | 필수 | 제약 / 설명 |
|---|---|---|---|
| contactPhone | string | O | 담당자 연락처. 검토 안내용, 손님에게 공개 안 함 |
| type | string | O | `DISCOUNT` 금액 할인 · `FREE_ITEM` 메뉴 제공 · `ETC` 기타 혜택 |
| title | string | O | 30자 이하 |
| description | string | O | 300자 이하 |
| discountType | string | DISCOUNT일 때 O | `RATE` 비율(%), `AMOUNT` 금액(원) |
| discountValue | int | DISCOUNT일 때 O | RATE면 1~100, AMOUNT면 1 이상 |
| maxDiscount | int | RATE일 때 O | 최대 할인 금액(원) |
| freeItem | string | FREE_ITEM일 때 O | 제공 메뉴, 30자 이하 |
| startDate | string | O | 오늘 이후 |
| endDate | string | O | `startDate` 이후. 지나면 노출 자동 종료 |
| days | string[] | O | 이용 가능 요일, 1개 이상 |
| availableFrom | string | O | 이용 시작 시각 |
| availableTo | string | O | 이용 종료 시각 (`availableFrom`보다 늦게) |
| minOrderAmount | int | O | 최소 주문 금액(원), 없으면 0 |
| restrictions | string[] | O | [8.6 사용 제한](#86-혜택-사용-제한), 없으면 빈 배열 |
| extraNotice | string | | 추가 안내, 200자 이하 |
| agreed | boolean | O | "입력한 혜택 정보가 사실이며…" 동의, `true`만 허용 |

응답: 6.8 목록 항목 1개 (`status: "PENDING"`)

"임시저장"은 앱 기기에 저장한다 (서버 저장 없음).

### 6.8 내 가게 혜택 목록

`GET /owner/stores/{storeId}/benefits`

```json
[
  {
    "id": 11,
    "title": "식사 금액 10% 할인",
    "type": "DISCOUNT",
    "status": "APPROVED",
    "rejectReason": null,
    "paused": false,
    "visible": true,
    "startDate": "2026-11-01",
    "endDate": "2026-11-30",
    "summary": "평일 14~17시 · 2만원 이상 · 최대 5천원"
  }
]
```

- `status`: [8.5 혜택 검토 상태](#85-혜택-검토-상태)
- `visible`: 지금 손님에게 보이는지 (승인 완료 + 기간 안 + 중지 안 함)

상세 조회는 `GET /owner/stores/{storeId}/benefits/{benefitId}` (4.2 형식 + `contactPhone`, `status`, `rejectReason`, `paused`)

### 6.9 혜택 수정·중지·삭제

| 메서드 | 경로 | 설명 |
|---|---|---|
| PUT | `/owner/stores/{storeId}/benefits/{benefitId}` | 수정. 요청은 6.7과 같음. **수정하면 다시 `PENDING`** (검토 후 노출) |
| PATCH | `/owner/stores/{storeId}/benefits/{benefitId}/pause` | 노출 중지·재개 `{ "paused": true }` |
| DELETE | `/owner/stores/{storeId}/benefits/{benefitId}` | 삭제 |

에러: `BENEFIT_NOT_FOUND`

---

## 7. 이미지 업로드

### 7.1 이미지 업로드 🏪

`POST /uploads/images` (`multipart/form-data`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| file | file | O | jpg, png, webp, 10MB 이하 |
| purpose | string | O | `STORE`, `MENU` |

응답

```json
{ "url": "https://cdn.lasteat.app/uploads/2026/11/1f3a...jpg" }
```

받은 URL을 가게 수정(6.3)이나 메뉴 저장(6.6)에 담아 보낸다. 업로드만 하고 저장하지 않은 이미지는 서버가 주기적으로 지운다.

에러: `INVALID_FILE`, `FILE_TOO_LARGE`

---

## 8. 코드 값

### 8.1 가게 카테고리

| 코드 | 화면 |
|---|---|
| KOREAN | 한식 |
| WESTERN | 양식 |
| JAPANESE | 일식 |
| CHINESE | 중식 |
| CAFE | 카페 |
| BAKERY | 베이커리 |
| ETC | 기타 |

### 8.2 영업 상태

| 코드 | 화면 | 계산 |
|---|---|---|
| OPEN | 영업 중 | 영업시간 안, 쉬는 시간 아님 |
| BREAK | 준비 중 | 쉬는 시간 |
| CLOSED | 영업 종료 / 오늘 휴무 | 영업시간 밖 또는 정기 휴무 요일 |

사장님이 6.5로 오늘 상태를 직접 정했으면 그 값이 우선한다. `openNow=true` 필터는 `OPEN`만 포함한다.

### 8.3 요일

`MON` `TUE` `WED` `THU` `FRI` `SAT` `SUN` (사장님 앱 숫자 1~7과 순서 같음)

### 8.4 혜택 유형

| 코드 | 화면 | 필수 추가 필드 |
|---|---|---|
| DISCOUNT | 금액 할인 | discountType, discountValue (RATE면 maxDiscount) |
| FREE_ITEM | 메뉴 제공 | freeItem |
| ETC | 기타 혜택 | 없음 (description으로 설명) |

`partner`(제휴 혜택 여부)는 사장님이 신청할 때 정하지 않고, 운영진이 승인할 때 지정한다.

### 8.5 혜택 검토 상태

| 코드 | 화면 |
|---|---|
| PENDING | 검토 중 |
| APPROVED | 노출 승인 |
| REJECTED | 반려 (`rejectReason` 확인 후 수정해서 다시 신청) |

### 8.6 혜택 사용 제한

| 코드 | 화면 |
|---|---|
| ONE_PER_PERSON | 1인 1회 사용 가능 |
| NO_DUPLICATE | 다른 할인과 중복 사용 불가 |
| DINE_IN_ONLY | 매장 방문 시에만 사용 가능 |

### 8.7 회원 권한

| 코드 | 설명 |
|---|---|
| CONSUMER | 손님 (가입 기본값) |
| OWNER | 사장님 (운영진이 가게와 연결할 때 변경) |

---

## 9. 에러 코드

| HTTP | code | 상황 |
|---|---|---|
| 400 | INVALID_INPUT | 필수값 누락, 형식·길이 오류 (`fields`에 상세) |
| 400 | INVALID_FILE | 지원하지 않는 이미지 형식 |
| 401 | UNAUTHORIZED | 토큰 없음 (🔑·🏪 API) |
| 401 | INVALID_TOKEN | 잘못된 토큰 |
| 401 | EXPIRED_TOKEN | accessToken 만료 → 2.2로 재발급 |
| 401 | KAKAO_AUTH_FAILED | 카카오 토큰 확인 실패 |
| 403 | FORBIDDEN | 사장님 권한 필요 |
| 403 | NOT_STORE_OWNER | 본인 가게가 아님 |
| 404 | STORE_NOT_FOUND | 가게 없음 |
| 404 | MENU_NOT_FOUND | 메뉴 없음 |
| 404 | BENEFIT_NOT_FOUND | 혜택 없음 또는 노출 중 아님 |
| 409 | OWNER_HAS_STORE | 가게가 연결된 사장님의 탈퇴 |
| 413 | FILE_TOO_LARGE | 이미지 10MB 초과 |
| 500 | INTERNAL_ERROR | 서버 오류 |

---

## 10. 명세 기준 엔티티 변경 사항

Sprint 1 엔티티(PR #11)에서 이 명세를 구현하려면 아래를 바꾼다. Sprint 2 첫 작업.

| 엔티티 | 변경 |
|---|---|
| Store | `ratingAvg`, `reviewCount` 삭제 (리뷰 삭제) · `thumbnailUrl` → `StoreImage` 테이블 (여러 장, 순서) · `tags` 추가 · `recommended` 추가 · `businessStatusOverride` + 적용 날짜 추가 · `closedDays`를 요일 코드 목록으로 |
| StoreImage | 신규: store, imageUrl, sortOrder |
| Menu | `soldOut` 추가 |
| Benefit | `type`을 `DISCOUNT`/`FREE_ITEM`/`ETC`로 재정의 · `partner`, `description`, `freeItem`, `days`, `restrictions`, `extraNotice`, `contactPhone`, `status`, `rejectReason`, `paused` 추가 · `conditionText` 삭제 (`description`, `extraNotice`로 대체) |
| Member | 변경 없음 (refreshToken은 별도 저장) |
| Bookmark | 변경 없음 |
| ClosingDeal, Reservation | 변경 없음 (마감할인 진행 여부 결정 대기) |
