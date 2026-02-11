# SubDevice Java-Smali 매핑 참고

Java 디컴파일 소스와 Smali 코드 간의 대응 관계 정리.

---

## 1. 핵심 흐름 요약

```
[1] 앱 시작 -> Build.MODEL 획득 (Sq/a.smali)
[2] AllowList API 호출 (SubDeviceLoginService -> allowlist.json)
[3] 서버 응답 (SubDeviceAllowListResponse -> allowlisted: true/false)
[4] UI 업데이트 (LoginAccountViewModel -> checkSubDeviceMode)
[5] 체크박스 표시/숨김 (LoginAccountFragment -> subdeviceCheckLayout)
[6] 로그인 시 체크 상태에 따라 SubDevice/일반 로그인 분기
```

---

## 2. Java 소스 - 스말리 코드 매핑 테이블

### 2.1 Build.MODEL 획득 (기기 모델명)

| 구분      | 경로                        | 핵심 라인                                                               |
| --------- | --------------------------- | ----------------------------------------------------------------------- |
| **Java**  | `sources/Sq/C17355a.java`   | `Build.MODEL` 접근, 공백 제거, 대문자 변환                              |
| **Smali** | `smali_classes8/Sq/a.smali` | 라인 46: `sget-object p0, Landroid/os/Build;->MODEL:Ljava/lang/String;` |

**스말리 메서드:**

```smali
# smali_classes8/Sq/a.smali (라인 33-115)
.method public static final a(LV30/b;)Ljava/lang/String;
    # 라인 46: Build.MODEL 접근
    sget-object p0, Landroid/os/Build;->MODEL:Ljava/lang/String;
    # 공백 -> "-" 변환, 대문자 변환 후 반환
.end method
```

---

### 2.2 AllowList API 서비스

| 구분      | 경로                                                                             | 핵심 라인                                             |
| --------- | -------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Java**  | `sources/com/kakao/talk/net/retrofit/service/SubDeviceLoginService.java`         | 메서드 `d()` - GET allowlist.json?model_name={모델명} |
| **Smali** | `smali_classes9/com/kakao/talk/net/retrofit/service/SubDeviceLoginService.smali` | 라인 243-271                                          |

**스말리 메서드:**

```smali
# SubDeviceLoginService.smali (라인 243-271)
.method public abstract d(Ljava/lang/String;Lkotlin/coroutines/Continuation;)Ljava/lang/Object;
    .annotation runtime LQz0/f;
        value = "allowlist.json"   # <- 라인 250
    .end annotation

    .annotation runtime LQz0/t;
        value = "model_name"       # <- 라인 246
    .end annotation
.end method
```

**API 엔드포인트:**

```
GET https://katalk.kakao.com/android/account/allowlist.json?model_name={모델명}
```

---

### 2.3 AllowList 응답 처리

| 구분      | 경로                                                                                            | 핵심 라인                 |
| --------- | ----------------------------------------------------------------------------------------------- | ------------------------- |
| **Java**  | `sources/com/kakao/talk/net/retrofit/service/subdevice/SubDeviceAllowListResponse.java`         | `getAllowlisted()` 메서드 |
| **Smali** | `smali_classes9/com/kakao/talk/net/retrofit/service/subdevice/SubDeviceAllowListResponse.smali` | 라인 72, 213-222          |

**스말리 필드 & 메서드:**

```smali
# SubDeviceAllowListResponse.smali

# 필드 정의 (라인 72)
.field public final a:Z    # allowlisted (boolean)

# Getter 메서드 (라인 213-222)
.method public final a()Z
    iget-boolean v0, p0, Lcom/kakao/talk/net/retrofit/service/subdevice/SubDeviceAllowListResponse;->a:Z
    return v0
.end method
```

---

### 2.4 checkSubDeviceMode (ViewModel)

| 구분      | 경로                                                 | 핵심 라인                                                              |
| --------- | ---------------------------------------------------- | ---------------------------------------------------------------------- |
| **Java**  | `sources/ee/C27639t.java` (LoginAccountViewModel.kt) | `checkSubDeviceMode()` 메서드 - AllowList API 호출 및 UI 상태 업데이트 |
| **Smali** | `smali_classes6/ee.1/t.smali`                        | 라인 1461: 메서드 `m2()`                                               |

**관련 Inner 클래스:**

| 스말리 파일                     | 역할                | 핵심 라인                                                                       |
| ------------------------------- | ------------------- | ------------------------------------------------------------------------------- |
| `smali_classes6/ee.1/t$c.smali` | Coroutine 상태 머신 | 라인 32: DebugMetadata "checkSubDeviceMode"                                     |
| `smali_classes6/ee.1/t$d.smali` | 결과 처리 콜백      | 라인 234: SubDeviceLoginService.d() 호출, 라인 246: Sq/a.a() 호출 (Build.MODEL) |

**스말리 메서드 (t.smali):**

```smali
# ee.1/t.smali (라인 1461)
.method public final m2(Lkotlin/coroutines/Continuation;)Ljava/lang/Object;
    # 메타데이터: checkSubDeviceMode
    # SubDeviceLoginService.d() 호출하여 allowlist 확인
    # 결과를 LoginAccountUiState에 반영
.end method
```

---

### 2.5 체크박스 UI (Fragment)

| 구분      | 경로                                                | 핵심 라인                               |
| --------- | --------------------------------------------------- | --------------------------------------- |
| **Java**  | `sources/ee/C27635p.java` (LoginAccountFragment.kt) | `z4()` 메서드 - 체크박스 표시/숨김      |
| **Smali** | `smali_classes6/ee.1/p.smali`                       | 라인 1040, 3370: "subdeviceCheckLayout" |

**XML 레이아웃:**

- 파일: `resources/res/layout-sw600dp/auth_login_account.xml`
- 체크박스 ID: `@+id/chk_subdevice`
- 컨테이너: `@+id/subdevice_check_layout` (기본 visibility: GONE)

**스말리 메서드:**

```smali
# ee.1/p.smali
.method public final z4(Z)V
    # 파라미터: canSubdevice (boolean)
    # subdevice_check_layout visibility 설정
    # chk_subdevice 체크 상태 설정
    # divider, QR 로그인 버튼 visibility 설정
.end method
```

---

### 2.6 SubDevice 로그인 파라미터

| 구분      | 경로                                                                                      | 핵심 라인              |
| --------- | ----------------------------------------------------------------------------------------- | ---------------------- |
| **Java**  | `sources/com/kakao/talk/net/retrofit/service/subdevice/SubDeviceLoginParams.java`         | `model_name` 필드 포함 |
| **Smali** | `smali_classes9/com/kakao/talk/net/retrofit/service/subdevice/SubDeviceLoginParams.smali` | 라인 35, 1543          |

**중요:** 실제 로그인 시 `model_name`은 `null`로 전송됨 -> AllowList 확인 이후 모델명 재검증 없음

---

## 3. 스말리 코드 위치 요약

| 기능               | 스말리 파일 경로                                      | 핵심 라인  |
| ------------------ | ----------------------------------------------------- | ---------- |
| Build.MODEL 획득   | `smali_classes8/Sq/a.smali`                           | 46         |
| AllowList API      | `smali_classes9/.../SubDeviceLoginService.smali`      | 243-271    |
| allowlisted 필드   | `smali_classes9/.../SubDeviceAllowListResponse.smali` | 72, 213    |
| checkSubDeviceMode | `smali_classes6/ee.1/t.smali`                         | 1461       |
| 체크박스 UI        | `smali_classes6/ee.1/p.smali`                         | 1040, 3370 |
| 로그인 파라미터    | `smali_classes9/.../SubDeviceLoginParams.smali`       | 35, 1543   |

---

## 4. 코드 위치 참고

### 4.1 Build.MODEL 반환 위치 (Sq/a.smali)

**위치:** `smali_classes8/Sq/a.smali` 라인 46

**원본:**

```smali
sget-object p0, Landroid/os/Build;->MODEL:Ljava/lang/String;
```

---

### 4.2 allowlisted 반환 위치 (SubDeviceAllowListResponse.smali)

**위치:** `smali_classes9/.../SubDeviceAllowListResponse.smali` 라인 213-222

**원본:**

```smali
.method public final a()Z
    iget-boolean v0, p0, Lcom/kakao/talk/net/retrofit/service/subdevice/SubDeviceAllowListResponse;->a:Z
    return v0
.end method
```

---

### 4.3 checkSubDeviceMode 결과 처리 (t$d.smali)

**위치:** `smali_classes6/ee.1/t$d.smali` - invokeSuspend 메서드 내

---

## 5. 검증 방법

수정 후 다음을 확인:

1. 앱 실행 시 "다른 기기와 함께 사용" 체크박스가 표시되는지
2. 체크박스 체크 후 로그인이 정상 작동하는지
3. 서브디바이스 모드로 채팅이 가능한지

---
