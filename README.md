# 카카오톡 서브디바이스 기능 루팅없이 전기종 활성화

## 개요

카카오톡의 **서브디바이스(SubDevice)** 기능은 태블릿 등의 보조 기기에서 메인 스마트폰과 함께 카카오톡을 사용할 수 있게 해주는 기능입니다. 기기 모델명을 서버 AllowList API로 확인하여 허용된 기기에서 활성화됩니다.

https://github.com/user-attachments/assets/ef10ef2f-f46c-4d5b-b252-de297de831bf

---

## 전체 흐름도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        앱 로그인 화면 진입                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ① Build.MODEL 가져오기 (Sq/C17355a.java)                                │
│     - 현재 기기 모델명 추출                                               │
│     - 공백 제거 + 대문자 변환                                             │
│     - 예: "SM-X710" (갤럭시 탭 S9)                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ② AllowList API 호출 (SubDeviceLoginService.java)                       │
│     GET https://katalk.kakao.com/android/account/allowlist.json          │
│         ?model_name={모델명}                                             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ③ 서버 응답 (SubDeviceAllowListResponse.java)                           │
│     { "allowlisted": true }  또는  { "allowlisted": false }              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
            allowlisted = true              allowlisted = false
                    │                               │
                    ▼                               ▼
┌─────────────────────────────┐     ┌─────────────────────────────┐
│  ④ 체크박스 표시              │     │  체크박스 숨김 (GONE)        │
│  "다른 기기와 함께 사용"       │     │  일반 로그인만 가능          │
│  (Use with Main Device)     │     └─────────────────────────────┘
│  자동으로 체크됨              │
└─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ⑤ 사용자가 로그인 버튼 클릭                                              │
│     - accountKey (이메일/전화번호)                                        │
│     - password                                                           │
│     - isSubdeviceLogin = chk_subdevice.isChecked()                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
            isChecked = true                isChecked = false
                    │                               │
                    ▼                               ▼
┌─────────────────────────────┐     ┌─────────────────────────────┐
│  ⑥ 서브디바이스 로그인 API    │     │  일반 로그인 API             │
│  (SubDeviceLoginService)    │     │  (CreateAccountService)     │
│  model_name = null 전송     │     └─────────────────────────────┘
└─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ⑦ 로그인 성공 → 서브디바이스 모드로 카카오톡 사용                          │
│     - 메인 기기와 동시 로그인 유지                                         │
│     - 메시지 동기화                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 핵심 파일 분석

### 1. 모델명 추출

**파일**: `Sq/C17355a.java` (원본: `Build.kt`)

```java
public static final String a(b bVar) {
    String MODEL = Build.MODEL;              // 기기 모델명
    String strK = new Regex("\\s").k(MODEL, "");  // 공백 제거
    return strK.toUpperCase(Locale.US);      // 대문자 변환
}
```

### 2. AllowList API 정의

**파일**: `com/kakao/talk/net/retrofit/service/SubDeviceLoginService.java`

```java
public static final String BASE_URL =
    C12890j.m(C12890j.f41219b) + "/android/account/";
// 프로덕션: https://katalk.kakao.com/android/account/

@Qz0.f("allowlist.json")  // GET 요청
Object d(
    @Qz0.t("model_name") String str,  // 모델명 쿼리 파라미터
    Continuation<? super retrofit2.w<SubDeviceAllowListResponse>> continuation
);
```

### 3. API 응답 모델

**파일**: `com/kakao/talk/net/retrofit/service/subdevice/SubDeviceAllowListResponse.java`

```java
public final class SubDeviceAllowListResponse {
    private final boolean allowlisted;  // 허용 여부

    public final boolean getAllowlisted() {
        return this.allowlisted;
    }
}
```

### 4. ViewModel - 핵심 로직

**파일**: `ee/C27639t.java` (원본: `LoginAccountViewModel.kt`)

```java
// checkSubDeviceMode() - AllowList 확인
SubDeviceLoginService subDeviceLoginService = this.subDeviceLoginService;
String strA = C17355a.a(V30.b.f100556a);  // Build.MODEL 가져옴
Object objD = subDeviceLoginService.d(strA, this);  // API 호출

SubDeviceAllowListResponse r9 = (SubDeviceAllowListResponse) r9;
boolean r1 = r9.getAllowlisted();  // ★ 허용 여부
// → LoginAccountUiState.isSubdeviceModeEnabled 업데이트

// x2() - 로그인 분기
public final void x2(String accountKey, String password, boolean isSubdeviceLogin) {
    if (isSubdeviceLogin) {
        // 서브디바이스 로그인 API
        new m(accountKey, password, this, null);
    } else {
        // 일반 로그인 API
        new n(accountKey, password, null);
    }
}
```

### 5. Fragment - UI 처리

**파일**: `ee/C27635p.java` (원본: `LoginAccountFragment.kt`)

```java
// z4() - 체크박스 표시/숨김
public final void z4(boolean canSubdevice) {
    e40.h.p(subdeviceCheckLayout, canSubdevice);  // 레이아웃 표시/숨김
    X3().f277603d.setChecked(canSubdevice);       // 체크박스 자동 체크
}

// p4() - 로그인 버튼 클릭
public final void p4() {
    String text = X3().f277601b.getText();   // accountKey
    String text2 = X3().f277607h.getText();  // password
    // ★ 체크박스 상태를 isSubdeviceLogin으로 전달
    Y3().x2(text, text2, X3().f277603d.isChecked());
}
```

### 6. 레이아웃

**파일**: `resources/res/layout-sw600dp/auth_login_account.xml`

```xml
<ConstraintLayout android:id="@+id/subdevice_check_layout"
    android:visibility="gone">  <!-- 기본 숨김 -->

    <TdCheckBox android:id="@+id/chk_subdevice"/>
    <TextView android:text="@string/text_check_subdevice"/>
    <!-- "Use with Main Device" (다른 기기와 함께 사용) -->
</ConstraintLayout>
```

---

## 참고 사항

### 모델명 확인은 AllowList API에서만 수행

| 단계           | model_name 전송      | 서버 검증 |
| -------------- | -------------------- | --------- |
| AllowList 확인 | `Build.MODEL` → 전송 | 검증함    |
| 실제 로그인    | **`null`로 전송**    | 검증 안함 |

**파일**: `ee/C27639t.java` 라인 1030

```java
SubDeviceLoginParams subDeviceLoginParams = new SubDeviceLoginParams(
    email,
    password,
    (String) null,  // device_uuid
    (String) null,  // device_name
    ...
    (String) null,  // ★ model_name = null !!
    ...
);
```

---

## 수정 지점

`allowlisted` 값 반환 위치: `ee/C27639t.java`의 `checkSubDeviceMode`

```java
boolean r1 = r9.getAllowlisted();
```

SubDeviceAllowListResponse.smali

```
# virtual methods
.method public final a()Z
    .locals 0

    .line 1
     iget-boolean p0, p0, Lcom/kakao/talk/net/retrofit/service/subdevice/SubDeviceAllowListResponse;->a:Z
    .line 2
    .line 3
    return p0
.end method

```

무조건 true를 반환하도록 아래와 같이 수정

```
# virtual methods
.method public final a()Z
    .locals 0

    .line 1
    const/4 p0, 0x1

    .line 2
    .line 3
    return p0
.end method

```

---

## 서버 URL 환경별 설정

**파일**: `Iq/C12890j.java`

| 환경                | `f41219b` 값                       |
| ------------------- | ---------------------------------- |
| Sandbox             | `sandbox{동적값}-katalk.kakao.com` |
| Alpha               | `alpha{동적값}-katalk.kakao.com`   |
| Beta                | `beta-katalk.kakao.com`            |
| CBT                 | `cbt-katalk.kakao.com`             |
| **Real (프로덕션)** | `katalk.kakao.com`                 |

---

## 파일 요약표

| 파일                                                                            | 원본명                     | 역할                              |
| ------------------------------------------------------------------------------- | -------------------------- | --------------------------------- |
| `Sq/C17355a.java`                                                               | `Build.kt`                 | `Build.MODEL` 추출                |
| `com/kakao/talk/net/retrofit/service/SubDeviceLoginService.java`                | -                          | AllowList API 정의                |
| `com/kakao/talk/net/retrofit/service/subdevice/SubDeviceAllowListResponse.java` | -                          | API 응답 모델                     |
| `ee/C27639t.java`                                                               | `LoginAccountViewModel.kt` | AllowList 확인 + 로그인 분기      |
| `ee/C27635p.java`                                                               | `LoginAccountFragment.kt`  | UI 처리 (체크박스)                |
| `ee/LoginAccountUiState.java`                                                   | -                          | `isSubdeviceModeEnabled` 상태     |
| `resources/res/layout-sw600dp/auth_login_account.xml`                           | -                          | 체크박스 레이아웃                 |
| `XE/InterfaceC19583d0.java`                                                     | -                          | `getIsSubDeviceMode()` 인터페이스 |
| `Iq/C12890j.java`                                                               | -                          | 서버 URL 환경 설정                |

---

## 요약

1. 서브디바이스 기능은 기기 모델명을 서버 AllowList API에 전송하여 허용 여부를 확인
2. AllowList API 응답(`allowlisted`)에 따라 "다른 기기와 함께 사용" 체크박스 표시/숨김
3. 실제 로그인 시에는 `model_name`이 `null`로 전송됨
