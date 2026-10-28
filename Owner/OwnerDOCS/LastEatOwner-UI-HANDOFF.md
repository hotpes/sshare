# LastEat 사장님 UI 소스 전달

GitHub에는 `LastEatOwner-UI` 폴더의 **Kotlin 파일 7개만** 넣으면 됩니다. Gradle 파일, Manifest, Activity, 샘플 데이터, 사진·폰트·drawable 리소스는 전달 폴더에 없습니다. 기존 메인 앱의 Kotlin 소스 경로에 복사하고, 패키지 이름은 프로젝트 규칙에 맞게 변경해도 됩니다.

## 실행 환경

이 소스는 기존 **Kotlin · Jetpack Compose · Material3** 앱에 통합하는 화면 코드입니다. Compose 자체까지 없애고 실행되는 코드는 아닙니다. 기존 앱이 제공하는 Compose Runtime / UI / Foundation / Material3와 Kotlin·Java 표준 API만 사용합니다. 추가 이미지 로더, 아이콘 패키지, Activity Compose, Navigation, ViewModel, 네트워크 라이브러리를 요구하지 않습니다. 별도의 Gradle 파일이나 버전을 강제하지 않습니다. `java.time` 사용으로 기존 앱의 minSdk 26 이상 또는 해당 앱의 desugaring 설정을 사용합니다.

## 연결

```kotlin
// 메인 앱에서 이미 적용하는 MaterialTheme 안에서 호출하세요.
OwnerScreen(
    state = ownerState,
    onAction = { action -> handleOwnerAction(action) },
    imageContent = { url, description, modifier ->
        // 메인 앱의 기존 서버 이미지 컴포넌트를 사용하세요.
        // url, contentDescription, modifier를 전달하고 Crop으로 표시합니다.
        // 로딩/오류/빈 URL 표시는 이 컴포넌트에서 처리합니다.
    },
    onPickImage = { currentUrl, onSelectedUrl ->
        // 메인 앱에서 사진 선택 → 업로드를 처리합니다.
        // 성공한 서버 URL을 onSelectedUrl(uploadedUrl)로 반환합니다.
        // 취소/실패 시 콜백을 호출하지 않고 메인 앱에서 안내합니다.
    },
)
```

예시의 `handleOwnerAction` 및 이미지 컴포넌트·선택·업로드 함수는 메인 앱에서 구현합니다.

- **이미지:** `Shop.photoUrl`, `MenuItem.photoUrl`에 서버 URL을 넣습니다. UI는 URL을 `imageContent`로 전달할 뿐 다운로드·디코딩·캐시를 수행하지 않습니다. `imageContent`를 생략하면 사진 자리만 표시합니다. 전달받은 `modifier`와 접근성 설명을 실제 이미지에 적용하세요.
- **사진 변경:** `onPickImage`에서 최종 업로드 URL을 돌려주면 편집 중인 초안만 바뀝니다. 저장 버튼의 `ShopDraft.photoUrl` / `MenuDraft.photoUrl`에 이 URL이 전달됩니다. 선택기·업로드 진행·오류는 호스트가 처리합니다. 콜백을 생략하면 사진 변경 버튼은 비활성화됩니다.
- **글꼴:** 메인 앱 `MaterialTheme.typography`의 각 `fontFamily`를 그대로 사용합니다. UI는 크기·굵기·색 등 화면 스타일만 적용합니다. 폰트 파일과 `R.font` 참조가 없습니다.
- **아이콘:** `OwnerIcons.kt`의 Compose 벡터 코드로 그립니다. drawable 파일이나 material-icons 라이브러리가 필요 없습니다.
- **실행 진입점:** `OwnerScreen`입니다. 기존 `MainActivity`나 앱 테마를 교체하지 않습니다.

## 파일

| 파일 | 역할 |
|---|---|
| OwnerScreen.kt | 화면 진입점, 창 열기·닫기, 외부 연결 콜백 |
| OwnerUi.kt | 메인 화면, 스타일, 이미지 표시 슬롯, 메인 앱 폰트 상속 |
| OwnerForms.kt | 가게·메뉴·영업시간·혜택 입력 UI |
| OwnerSheets.kt | 쿠폰·리뷰·소식·통계·영업 상태 UI |
| OwnerState.kt | 외부 화면 데이터·로딩·결과 모델과 표시 형식 |
| OwnerActions.kt | 요청 콜백·입력 DTO |
| OwnerIcons.kt | 리소스 파일 없이 그리는 벡터 아이콘 |

## 데이터 처리 경계

UI는 API·인증·저장소·서버 이미지 로딩·사진 업로드·쿠폰 판정·할인 계산·사용 처리·통계 집계를 수행하지 않습니다. 일반 저장 요청 후 목록·영업 상태·품절·답글을 자체 변경하지 않습니다.

`OwnerState`로 서버 조회 결과를 전달하고 `OwnerAction`을 메인 앱의 연동 계층에서 처리하세요. `isLoading`, `isSubmitting`, `result`도 외부에서 제공합니다. 저장 성공 시 최신 데이터와 새로운 `OwnerResult(id, true, message)`를 제공하면 편집 창이 닫힙니다. 실패 결과면 초안을 유지합니다.

| 이벤트 | 메인 앱·백엔드 처리 |
|---|---|
| LoadSection | 필요한 영역 조회 |
| OpenCoupon | 이전 쿠폰·견적 결과 초기화 |
| SaveShop / SaveHours / SetBusinessStatus | 권한·정책 확인 및 매장 저장 |
| SaveMenu / SetMenuSoldOut / DeleteMenu | 메뉴 생성·수정·품절·삭제 |
| SaveBenefit / SetBenefitIssuance | 혜택 정책 확인, 생성·수정·발급 중지·재개 |
| SaveReviewReply / PublishNotice | 답글·소식 저장 |
| CheckCoupon | 쿠폰 조회와 매장·만료·상태·사용 조건 확인 |
| QuoteCoupon | 할인액·결제 예정액 계산 및 견적 발행 |
| RedeemCoupon | 견적 재확인, 중복 방지, 실제 사용 처리 |

쿠폰 화면 상태는 `Idle → Checking → Ready → Redeemed`로 외부에서 전달합니다. 실패 시 `Rejected` 또는 `Ready.error`를 사용합니다. `CouponQuote`의 주문액·할인액·결제액·사용 가능 여부도 서버 결과를 그대로 표시합니다. 신규 ID 생성과 모든 업무 검증은 서버 담당입니다. UI는 필수값·숫자·시간·날짜 등 입력 형태만 확인합니다.

## 별도 확인용 결과

`LastEatOwner-UI-preview.apk`는 전달한 **동일한 7개 소스**를 별도 테스트 호스트에 연결해 빌드한 레이아웃 확인용 앱입니다. 이미지·폰트 리소스를 포함하지 않고 시스템 기본 글꼴과 사진 자리 표시를 사용합니다. 샘플 데이터와 호스트·빌드 설정은 전달 소스 ZIP에 없습니다. 확인용 APK의 저장·검증 버튼은 실제 서버 처리를 하지 않습니다.

`LastEatOwner-UI-layouts.zip`에는 실제 Compose 렌더링으로 캡처한 화면 PNG가 들어 있습니다. 메인 화면, 가게 정보, 영업 상태, 메뉴 목록·편집, 영업시간, 혜택 세 종류, 리뷰·답글, 소식, 통계, 알림, 쿠폰 입력·견적·완료 레이아웃을 확인할 수 있습니다. 쿠폰 견적·완료 캡처는 테스트 호스트가 지정한 상태를 렌더링한 것입니다.
