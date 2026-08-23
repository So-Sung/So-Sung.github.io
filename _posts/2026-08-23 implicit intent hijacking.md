---
title: "APK 분석 3 - 암시적 Intent 하이재킹 테스트"
date: 2026-08-23 00:00:00 +0900
categories: [Mobile Hacking, Android]
tags: [android, intent, implicit-intent, explicit-intent, apk-analysis]
---

지난 글에서는 `exported="true"`인 컴포넌트를 `am start`로 직접 강제호출하고, 그 안에서 nested Intent를 검증 없이 재실행하는 리디렉션 체이닝을 다뤘다. 그런데 그 글을 쓰면서 계속 걸리는 게 하나 있었다. 내가 만든 `QRConnectActivity`는 목적지(컴포넌트)를 코드에서 정확히 지정하는 **명시적 Intent**를 썼는데, 실무에서는 목적지를 아예 안 정하고 "이 액션 처리할 수 있는 앱 아무나 나와라" 하는 **암시적 Intent**도 많이 쓴다. 그럼 이 둘의 위험도가 진짜 다른지, 암시적 Intent에 민감 데이터를 실어보내면 실제로 누가 가로챌 수 있는지 이번엔 직접 테스트 앱 두 개(피해 앱 + 공격 앱)를 만들어서 확인해봤다.

## 0. 명시적 Intent와 암시적 Intent란

| 구분 | 명시적 Intent (Explicit) | 암시적 Intent (Implicit) |
|---|---|---|
| 정의 | 실행할 컴포넌트를 클래스명 등으로 정확히 지정 | 수행할 작업(Action)만 명시, 수신자는 시스템이 결정 |
| 코드 예시 | `Intent(this, TargetActivity::class.java)` | `Intent("com.example.ACTION_XXX")` |
| 수신자 결정 | 개발자가 코드에서 확정 | 시스템이 Manifest의 `intent-filter` 매칭으로 결정 |
| 주 용도 | 앱 내부 화면 전환, 우리 앱 컴포넌트끼리 통신 | 다른 앱 기능 호출 (카메라, 공유, 브라우저 열기 등) |
| 위험 포인트 | 상대적으로 안전 (수신자가 확정적) | 동일한 액션을 등록한 악성 앱이 있으면 가로챌 수 있음 |

정의만 보면 당연한 얘기 같은데, "그래서 실제로 얼마나 쉽게 가로채지는지"는 직접 테스트 앱을 만들어봐야 감이 온다. 그래서 이번엔 민감 데이터(`secret`)를 담은 암시적 Intent를 쏘는 피해 앱과, 그 액션을 가로채는 공격 앱을 각각 만들어봤다.

## 1. 기존 코드에 테스트용 버튼 추가

피해 앱(`test_intent`)의 로그인 화면(`MainActivity`)에 버튼을 하나 추가해서, 누르면 민감 데이터를 담은 암시적 Intent를 쏘도록 만들었다.

[![1](https://so-sung.github.io/assets/img/posts/2026-08-23/1.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/1.png)
*(캡처 1: Android Studio에서 MainActivity.kt의 btnImplicitLab 클릭 리스너 부분)*

```kotlin
// MainActivity.kt
btnImplicitLab.setOnClickListener {
    val intent = Intent("com.sosung.test_intent.ACTION_LAB").apply {
        putExtra("secret", "VERY_SECRET_DATA")
    }
    startActivity(intent)
}
```

여기서 포인트는 `Intent(this, XXXActivity::class.java)`처럼 컴포넌트를 지정한 게 아니라, `Intent("com.sosung.test_intent.ACTION_LAB")`처럼 **액션 문자열만 넣고** 던졌다는 것이다. "이 액션을 처리할 수 있는 아무 컴포넌트나 실행해라"는 뜻이 된다.

정상적으로 이 액션을 받도록 만들어둔 `ImplicitLabActivity`는 Manifest에 이렇게 등록돼 있다.

```xml
<!-- AndroidManifest.xml (test_intent) -->
<activity
    android:name=".ImplicitLabActivity"
    android:exported="true"
    android:windowSoftInputMode="adjustResize">
    <intent-filter>
        <action android:name="com.sosung.test_intent.ACTION_LAB" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

## 2. 하이재킹을 시도할 공격 앱 코드 작성

이제 별도 프로젝트(`intent_attacker`)로 공격 앱을 만들었다. 여기가 핵심인데, **피해 앱이 쓴 것과 똑같은 액션 문자열**로 intent-filter를 등록하기만 하면 된다.

[![2](https://so-sung.github.io/assets/img/posts/2026-08-23/2.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/2.png)
*(캡처 2: intent_attacker 프로젝트의 AndroidManifest.xml — AttackerActivity가 test_intent와 똑같은 액션 문자열로 등록돼 있다)*

```xml
<!-- AndroidManifest.xml (intent_attacker) -->
<activity
    android:name=".AttackerActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="com.sosung.test_intent.ACTION_LAB" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

그리고 이 액션으로 진입하면 전달받은 extra를 그대로 읽어서 보여주는 코드를 작성했다.

```kotlin
// AttackerActivity.kt
class AttackerActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val hijackedSecret = intent.getStringExtra("secret")
        Log.d("AttackerApp", "Hijacked Data: $hijackedSecret")
        Toast.makeText(this, "🚨 [공격 성공] 가로챈 데이터: $hijackedSecret", Toast.LENGTH_LONG).show()
    }
}
```

여기서 하이재킹이 성립하는 코드적 특징은 딱 하나다: **피해 앱과 공격 앱이 액션 문자열(`com.sosung.test_intent.ACTION_LAB`)만 똑같이 맞추면, 시스템 입장에서는 둘 다 "이 Intent를 처리할 자격이 있는 컴포넌트"로 취급한다.** 패키지명이 다르든, 서명이 다르든, 시스템은 상관하지 않는다. 오직 `action` + `category` 매칭만 본다.

## 3. 실제 실행하면 이렇게 하이재킹된다

두 앱을 같은 기기(에뮬레이터)에 같이 설치한 상태에서, 피해 앱의 "Implicit Intent 테스트 실행" 버튼을 눌러봤다.

[![3](https://so-sung.github.io/assets/img/posts/2026-08-23/3.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/3.png)
*(캡처 3: 클릭 직전의 로그인 화면. 빨간 박스가 이번에 테스트할 "Implicit Intent 테스트 실행" 버튼)*

이 버튼을 누르는 순간, 같은 액션을 처리할 수 있는 앱이 두 개(정상 앱의 `ImplicitLabActivity`, 공격 앱의 `AttackerActivity`)라서 시스템이 선택 다이얼로그를 띄운다.

[![4](https://so-sung.github.io/assets/img/posts/2026-08-23/4.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/4.png)
*(캡처 4: 버튼 클릭 시 뜨는 앱 선택 다이얼로그. 상단에 "Complete action using AttackerApp"이 `Just once` / `Always`와 함께 먼저 제안되고, 정상 앱인 test_intent는 하단 "Use a different app"을 펼쳐야 나온다)*

여기서 눈에 띄는 게, 정상 앱과 공격 앱이 나란히 리스트로 뜨는 게 아니라 **시스템이 기본 후보 하나를 먼저 추천하는 형태**라는 점이다. 그리고 하필 그 추천 후보가 `AttackerApp`이다. 사용자가 무심코 `Just once`를 누르면, 정상 앱 목록은 펼쳐보지도 않은 채 바로 데이터가 넘어간다. 실제로 `Just once`를 눌러봤다.

[![5](https://so-sung.github.io/assets/img/posts/2026-08-23/5.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/5.png)
*(캡처 5: `Just once` 선택 후 AttackerApp 화면에 뜨는 토스트 — "🚨 [공격 성공] 가로챈 데이터: VERY_SECRET_DATA")*

`secret` extra에 담아 보냈던 `VERY_SECRET_DATA`가 그대로 공격 앱에 전달된 게 확인된다. `adb logcat`으로도 확인했는데, `AttackerApp: Hijacked Data: VERY_SECRET_DATA` 라인이 그대로 찍혔다.

## 4. AttackerApp만 후보인 상황이면 어떻게 될까

선택 다이얼로그는 후보가 2개 이상일 때만 뜬다. 그럼 정상 앱의 컴포넌트가 아예 후보에서 빠지면 어떻게 될까 — 를 재현하려고 `test_intent` 자체를 지우는 대신, **`AndroidManifest.xml`에서 `ImplicitLabActivity`의 intent-filter 블록만 통째로 주석 처리**했다. (test_intent 자체를 삭제하면 버튼을 누를 방법이 없어지니, "정상 컴포넌트만 사라진 상태"를 이렇게 재현했다.)

[![6](https://so-sung.github.io/assets/img/posts/2026-08-23/6.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/6.png)
*(캡처 6: `ImplicitLabActivity`의 `<activity>` 블록 전체를 주석 처리한 AndroidManifest.xml. 이제 test_intent 앱 자신도 `ACTION_LAB`을 처리할 컴포넌트를 갖고 있지 않다)*

이 상태로 Clean Project 후 재빌드·재설치했다. `intent_attacker`는 지우지 않고 그대로 뒀다.

[![7](https://so-sung.github.io/assets/img/posts/2026-08-23/7.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/7.png)
*(캡처 7: 주석 처리 후 재설치한 test_intent 앱을 다시 실행해서, 아까와 같은 버튼을 눌렀다)*

이번엔 어떻게 되는지 확인했다.

[![8](https://so-sung.github.io/assets/img/posts/2026-08-23/8.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/8.png)
*(캡처 8: 선택 다이얼로그 자체가 뜨지 않고 곧바로 AttackerApp 화면으로 진입 — "🚨 [공격 성공] 가로챈 데이터: VERY_SECRET_DATA")*

**선택 다이얼로그 없이 바로** `AttackerActivity`로 넘어갔다. 사용자는 뭔가 선택할 기회조차 없이 그냥 데이터를 뺏기는 셈이다.

이걸 찍어보고 나서 든 생각인데, 이게 오히려 지금까지 본 것 중 제일 현실적으로 위험한 케이스다. 캡처 4에서는 그래도 다이얼로그가 뜨고, 잘 보면 "Use a different app"을 눌러 정상 앱을 선택할 여지가 있었다. 근데 캡처 8처럼 **후보가 하나뿐인 상황**은 실제 피싱 시나리오에서 이렇게 만들어질 수 있다:

- 사용자가 정상 앱을 아직 설치하지 않은 상태에서, 똑같은 액션을 처리하는 가짜 앱을 먼저 설치하도록 유도한 경우 (스미싱 링크로 위장 앱 먼저 깔게 하기)
- 사용자 기기에서 정상 앱이 삭제됐거나 업데이트 중 잠깐 비활성화된 사이, 가짜 앱이 그 자리를 대신 차지한 경우
- 가짜 앱이 아이콘·이름·화면 UI를 정상 앱과 비슷하게 꾸며놨다면, 사용자는 다이얼로그도 없이 넘어갔으니 "지금 내가 다른 앱으로 이동했다"는 사실조차 인지하지 못한다

즉 후보가 여러 개라 다이얼로그가 뜨는 상황보다, **후보가 하나뿐이라 조용히 넘어가는 상황이 사용자 입장에서는 의심할 기회 자체가 없어서 더 위험하다.** 다이얼로그가 뜨는지 여부에 기대서 "어차피 사용자가 알아챌 것"이라고 안심하면 안 된다는 걸 이번 테스트로 확인했다.

## 5. 명시적 Intent로 바꾸면 막히는지 확인

마지막으로, 주석 처리했던 `ImplicitLabActivity`를 다시 복구하고, `MainActivity.kt`의 버튼 코드를 명시적 Intent로 바꿔서 실제로 하이재킹이 막히는지 확인했다.

[![9](https://so-sung.github.io/assets/img/posts/2026-08-23/9.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/9.png)
*(캡처 9: MainActivity.kt에서 액션 문자열 대신 `ImplicitLabActivity::class.java`로 컴포넌트를 직접 지정하도록 수정한 부분(빨간 박스))*

```kotlin
// 보완 코드
btnImplicitLab.setOnClickListener {
    val intent = Intent(this, ImplicitLabActivity::class.java).apply {
        putExtra("secret", "VERY_SECRET_DATA")
    }
    startActivity(intent)
}
```

이 상태로 재빌드·재설치했다. 이번에도 `AttackerApp`은 지우지 않고 그대로 뒀다 — 공격 앱이 살아있어도 막히는지가 핵심이니까.

[![10](https://so-sung.github.io/assets/img/posts/2026-08-23/10.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/10.png)
*(캡처 10: 코드 수정 후 재설치한 test_intent를 다시 실행해서 같은 버튼을 눌렀다. AttackerApp은 여전히 기기에 설치돼 있는 상태)*

결과는 이렇다.

[![11](https://so-sung.github.io/assets/img/posts/2026-08-23/11.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/11.png)
*(캡처 11: 다이얼로그 없이 곧바로 정상 컴포넌트로 진입 — "[정상 수신] secret: VERY_SECRET_DATA" 토스트. AttackerApp이 설치돼 있어도 아예 후보로 고려되지 않았다)*

컴포넌트를 코드에서 확정해버리니 시스템이 다른 후보를 고려할 여지 자체가 없어졌다. `AttackerApp`이 여전히 설치돼 있어도 아무 영향이 없다.

## 암시적/명시적 Intent, 정리하면서 중요하다고 느낀 점

- 암시적 Intent의 위험은 "액션 문자열이 같으면 시스템이 패키지명·서명 상관없이 둘 다 후보로 본다"는 매칭 방식 자체에서 나온다. 코드 실수가 아니라 **메커니즘 자체가 그렇게 설계돼 있다.**
- 선택 다이얼로그가 뜬다고 안전한 게 아니다. 캡처 4처럼 다이얼로그가 떠도 공격 앱이 기본 추천으로 먼저 뜨는 경우가 있고, 캡처 8처럼 후보가 하나뿐이면 다이얼로그 자체가 안 뜨고 바로 전달된다. 사용자가 알아챌 거라는 가정 자체가 틀릴 수 있다.
- 정상 앱이 아직 설치 안 됐거나 삭제된 틈에 이름·아이콘을 비슷하게 꾸민 가짜 앱만 후보로 남는 시나리오는, 다이얼로그도 안 뜨고 사용자가 의심할 여지도 없다는 점에서 여러 후보가 뜨는 상황보다 오히려 더 위험하다.
- 반대로 명시적 Intent는 애초에 후보를 좁혀버리기 때문에, **앱 내부 통신이나 민감 데이터 전달에는 예외 없이 명시적 Intent를 써야 한다**는 게 이번 테스트로 체감됐다. "가능하면"이 아니라 "반드시"에 가깝다.
- 암시적 Intent 자체를 쓰지 말라는 얘기는 아니다. 브라우저 열기, 공유하기처럼 "정말 아무 앱이나 처리해도 상관없는 작업"에는 여전히 암시적 Intent가 맞는 도구다. 문제는 **민감 데이터를 실어서 보낼 때 암시적 Intent를 쓰는 것**이다.

다음 편은 PendingIntent 쪽으로 넘어가서, `FLAG_MUTABLE`로 만든 PendingIntent를 제3자가 받아서 내부 Intent를 변조하는 케이스를 테스트해볼 예정이다.

---

# (English) APK Analysis 3 - Implicit Intents: What Happens When You Don't Specify a Receiver

In the last post, I force-called an `exported="true"` component directly with `am start` and chained it into re-executing a nested Intent without validation. While writing that post, one thing kept bothering me: the `QRConnectActivity` I built used an **explicit Intent**, where the target component is specified precisely in code. But in real-world apps, it's also common to use an **implicit Intent** — one that doesn't specify a target at all, and instead just says "whichever app can handle this action, come forward." So this time I built two test apps (a victim app and an attacker app) to see whether the risk level is actually different, and whether sensitive data sent through an implicit Intent can really be intercepted.

## 0. Explicit Intent vs. Implicit Intent

| | Explicit Intent | Implicit Intent |
|---|---|---|
| Definition | Precisely specifies the target component (e.g. by class name) | Only specifies the action to perform; the system decides the receiver |
| Code example | `Intent(this, TargetActivity::class.java)` | `Intent("com.example.ACTION_XXX")` |
| Receiver resolution | Fixed by the developer in code | Resolved by the system matching against `intent-filter` entries in manifests |
| Typical use | Screen transitions within an app, communication between your own app's components | Invoking another app's functionality (camera, share, opening a browser, etc.) |
| Risk | Relatively safe (receiver is deterministic) | Can be intercepted by any malicious app that registers the same action |

The definitions alone sound obvious, but "how easily can this actually be intercepted in practice" only really clicks once you build a test app and try it. So I built a victim app that fires an implicit Intent carrying sensitive data (`secret`), and a separate attacker app that intercepts that action.

## 1. Adding a test button to the existing code

I added a button to the login screen (`MainActivity`) of the victim app (`test_intent`) that, when tapped, fires an implicit Intent carrying sensitive data.

[![1](https://so-sung.github.io/assets/img/posts/2026-08-23/1.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/1.png)
*(Capture 1: The btnImplicitLab click listener in MainActivity.kt, in Android Studio)*

```kotlin
// MainActivity.kt
btnImplicitLab.setOnClickListener {
    val intent = Intent("com.sosung.test_intent.ACTION_LAB").apply {
        putExtra("secret", "VERY_SECRET_DATA")
    }
    startActivity(intent)
}
```

The key point here is that instead of specifying a component like `Intent(this, XXXActivity::class.java)`, I threw an Intent with just `Intent("com.sosung.test_intent.ACTION_LAB")` — **only an action string, no target**. This effectively says "run whatever component can handle this action."

The `ImplicitLabActivity` that was built to legitimately receive this action is registered in the manifest like this:

```xml
<!-- AndroidManifest.xml (test_intent) -->
<activity
    android:name=".ImplicitLabActivity"
    android:exported="true"
    android:windowSoftInputMode="adjustResize">
    <intent-filter>
        <action android:name="com.sosung.test_intent.ACTION_LAB" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

## 2. Writing the attacker app's hijacking code

Next, I built the attack app as a separate project (`intent_attacker`). This is the crux of it: all you need to do is register an intent-filter with **the exact same action string** the victim app uses.

[![2](https://so-sung.github.io/assets/img/posts/2026-08-23/2.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/2.png)
*(Capture 2: AndroidManifest.xml of the intent_attacker project — AttackerActivity is registered with the exact same action string as test_intent)*

```xml
<!-- AndroidManifest.xml (intent_attacker) -->
<activity
    android:name=".AttackerActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="com.sosung.test_intent.ACTION_LAB" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

Then I wrote code that, upon entry via this action, simply reads and displays whatever extra was passed in.

```kotlin
// AttackerActivity.kt
class AttackerActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val hijackedSecret = intent.getStringExtra("secret")
        Log.d("AttackerApp", "Hijacked Data: $hijackedSecret")
        Toast.makeText(this, "🚨 [Attack Succeeded] Hijacked data: $hijackedSecret", Toast.LENGTH_LONG).show()
    }
}
```

There's exactly one code-level characteristic that makes this hijack possible: **as long as the victim app and the attacker app match on the action string (`com.sosung.test_intent.ACTION_LAB`), the system treats both as "components eligible to handle this Intent."** It doesn't matter that the package names differ or the signing keys differ — the system only checks the `action` + `category` match.

## 3. Running it — this is what the hijack actually looks like

With both apps installed on the same device (emulator), I tapped the victim app's "Run Implicit Intent Test" button.

[![3](https://so-sung.github.io/assets/img/posts/2026-08-23/3.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/3.png)
*(Capture 3: The login screen right before tapping — the red box marks the "Run Implicit Intent Test" button being tested here)*

The moment this button is tapped, two apps can handle the same action (the legitimate app's `ImplicitLabActivity` and the attacker app's `AttackerActivity`), so the system shows a disambiguation dialog.

[![4](https://so-sung.github.io/assets/img/posts/2026-08-23/4.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/4.png)
*(Capture 4: The app-selection dialog that appears on tap. "Complete action using AttackerApp" is offered first, along with `Just once` / `Always` — you have to expand "Use a different app" at the bottom to find the legitimate test_intent app)*

What stands out here is that instead of listing the legitimate app and the attacker app side by side, **the system proposes a single default candidate first** — and that candidate happens to be `AttackerApp`. If a user carelessly taps `Just once`, the data goes straight to the attacker without the legitimate app's option ever being expanded. I tried tapping `Just once`.

[![5](https://so-sung.github.io/assets/img/posts/2026-08-23/5.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/5.png)
*(Capture 5: After choosing `Just once`, the toast that appears in the AttackerApp screen — "🚨 [Attack Succeeded] Hijacked data: VERY_SECRET_DATA")*

The `VERY_SECRET_DATA` I had sent in the `secret` extra was delivered straight to the attacker app. I also confirmed this via `adb logcat` — the line `AttackerApp: Hijacked Data: VERY_SECRET_DATA` showed up exactly as expected.

## 4. What happens when AttackerApp is the only candidate

The disambiguation dialog only appears when there are 2 or more candidates. So what happens if the legitimate app's component simply isn't in the running? Instead of uninstalling `test_intent` entirely (which would make the button unreachable), I **commented out just the intent-filter block for `ImplicitLabActivity`** in `AndroidManifest.xml`, to simulate "the legitimate component is gone, but the app itself is still installed."

[![6](https://so-sung.github.io/assets/img/posts/2026-08-23/6.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/6.png)
*(Capture 6: AndroidManifest.xml with the entire `<activity>` block for `ImplicitLabActivity` commented out. Now the test_intent app itself has no component that can handle `ACTION_LAB`)*

I did a clean rebuild and reinstalled in this state, leaving `intent_attacker` installed as-is.

[![7](https://so-sung.github.io/assets/img/posts/2026-08-23/7.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/7.png)
*(Capture 7: Relaunching the reinstalled test_intent app after commenting out the block, and tapping the same button again)*

Here's what happened this time.

[![8](https://so-sung.github.io/assets/img/posts/2026-08-23/8.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/8.png)
*(Capture 8: No disambiguation dialog at all — straight into the AttackerApp screen: "🚨 [Attack Succeeded] Hijacked data: VERY_SECRET_DATA")*

It went **straight to** `AttackerActivity` with no selection dialog whatsoever. The user has no chance to choose anything — the data is simply taken.

Looking at this after capturing it, I think this is actually the most realistically dangerous case I've seen so far. In Capture 4, a dialog did appear, and there was at least a chance to tap "Use a different app" and pick the legitimate one. But a situation like Capture 8, where **there's only one candidate**, can be engineered in a real phishing scenario in ways like:

- Tricking a user into installing a fake app that registers the same action *before* they ever install the real one (e.g. via a smishing link)
- A window where the real app has been uninstalled, or is briefly inactive during an update, and a fake app takes its place as the sole handler
- If the fake app's icon, name, and UI are dressed up to resemble the real app, the user — having seen no dialog at all — may never even realize they've been redirected to a different app

In other words, a situation where there's only one candidate and the flow proceeds silently is **more dangerous than a multi-candidate situation with a dialog**, precisely because the user never gets a chance to be suspicious. This test made it clear that you can't rely on "the dialog will appear and the user will notice" as a safety net.

## 5. Confirming that switching to an explicit Intent blocks the hijack

Finally, I restored the commented-out `ImplicitLabActivity` block and changed the button code in `MainActivity.kt` to use an explicit Intent, to confirm the hijack is actually blocked.

[![9](https://so-sung.github.io/assets/img/posts/2026-08-23/9.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/9.png)
*(Capture 9: The part of MainActivity.kt (red box) changed to directly specify `ImplicitLabActivity::class.java` as the component instead of an action string)*

```kotlin
// Fixed code
btnImplicitLab.setOnClickListener {
    val intent = Intent(this, ImplicitLabActivity::class.java).apply {
        putExtra("secret", "VERY_SECRET_DATA")
    }
    startActivity(intent)
}
```

I rebuilt and reinstalled with this change. I left `AttackerApp` installed this time too — the whole point is to confirm the fix holds even while the attacker app is still present.

[![10](https://so-sung.github.io/assets/img/posts/2026-08-23/10.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/10.png)
*(Capture 10: Relaunching the reinstalled test_intent app after the code fix, and tapping the same button again. AttackerApp is still installed on the device)*

Here's the result.

[![11](https://so-sung.github.io/assets/img/posts/2026-08-23/11.png)](https://so-sung.github.io/assets/img/posts/2026-08-23/11.png)
*(Capture 11: Straight into the legitimate component with no dialog — the toast "[Normal receipt] secret: VERY_SECRET_DATA". AttackerApp wasn't even considered as a candidate, despite being installed)*

Once the component is fixed in code, the system no longer has any room to consider other candidates. `AttackerApp` being installed makes no difference at all.

## What I took away about implicit vs. explicit Intents

- The risk of implicit Intents comes from the matching mechanism itself: if the action string matches, the system treats both apps as eligible candidates regardless of package name or signing certificate. **It's not a coding mistake — it's how the mechanism is designed to work.**
- A disambiguation dialog appearing doesn't mean you're safe. As Capture 4 showed, even when a dialog appears, the attacker app can be offered as the default suggestion; and as Capture 8 showed, if there's only one candidate, no dialog appears at all and the data goes straight through. You can't assume the user will notice.
- A scenario where the legitimate app isn't installed yet (or has been removed), leaving only a fake app disguised with a similar name/icon as the sole candidate, is arguably more dangerous than a multi-candidate scenario — there's no dialog and no opportunity for the user to become suspicious.
- Conversely, explicit Intents narrow the field of candidates from the start, so this test made it clear that **explicit Intents should be used without exception for internal app communication or transferring sensitive data.** It's less "when possible" and more "mandatory."
- This isn't an argument against implicit Intents altogether. For tasks where "any app handling this is genuinely fine" — opening a browser, sharing content — implicit Intents remain the right tool. The problem is **using an implicit Intent to carry sensitive data.**

Next up, I'm moving on to PendingIntents — testing the case where a `PendingIntent` created with `FLAG_MUTABLE` is received by a third party who then tampers with the Intent it wraps.
