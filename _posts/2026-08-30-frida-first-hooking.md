---
title: "Frida 1 - 모바일 후킹 첫걸음, 옵션과 실습 정리"
date: 2026-08-30 00:00:00 +0900
categories: [Mobile Hacking, Android]
tags: [android, frida, dynamic-analysis, hooking, apk-analysis]
toc: true
---

지난 글들([implicit-intent-hijacking](https://so-sung.github.io/posts/implicit-intent-hijacking/), [intent-redirect-chaining](https://so-sung.github.io/posts/intent-redirect-chaining/), [apk-build-and-execution](https://so-sung.github.io/posts/apk-build-and-execution/))에서는 APK를 정적으로 뜯어보고, Manifest와 코드를 읽어서 취약점을 찾고, `am start`로 직접 컴포넌트를 강제 호출해보는 데까지 다뤘다. 그런데 이 방식에는 한계가 있다. 코드를 읽고 "이 함수가 이렇게 동작하겠구나"까지는 추측할 수 있어도, **그 함수가 실행되는 순간 실제로 어떤 값이 들어오고 나가는지**는 정적 분석만으로는 확인이 안 된다. 지난 글에서 검증하려던 것 중 일부도 결국 "코드상으로는 이렇게 보이는데, 실행 중에 진짜 그런지 찍어봐야 안다"는 지점에서 막혔다. 그래서 이번엔 동적 분석 도구인 Frida를 처음부터 정리해보기로 했다.

이번 편은 스크립트 심화(Interceptor.attach 등 JS API)보다는, **설치부터 첫 후킹 성공까지 반드시 거쳐야 하는 절차와 옵션**을 정직하게 정리하는 데 집중했다.

## 0. Frida는 왜 만들어졌나

Frida는 리버싱 연구자 Ole André Vadla Ravnås가 만든 DBI(Dynamic Binary Instrumentation) 프레임워크다. 핵심 아이디어는 "바이너리를 다시 컴파일하거나 패치하지 않고, 실행 중인 프로세스에 코드를 주입해서 함수 호출을 가로채고 조작한다"는 것이다.

기존에는 프로그램이 실제로 어떻게 동작하는지 확인하려면 소스코드가 있거나, 디스어셈블해서 직접 패치하는 수밖에 없었다. Frida는 JavaScript로 짧은 스크립트만 작성하면 실행 중인 프로세스에 실시간으로 꽂아 넣을 수 있게 만들었다는 점에서, 정적 분석만으로는 답이 안 나오는 지점을 뚫어주는 도구다. 안드로이드 해킹 관점에서는 다음과 같은 상황에서 쓰게 된다.

- 난독화된 코드라 정적으로는 로직을 읽기 힘들 때, 실행 중 실제 리턴값·인자값을 찍어서 역으로 로직을 추론할 때
- 클라이언트 측 검증(비밀번호 체크, 결제 검증, 루팅 탐지, SSL Pinning 등)을 실시간으로 우회해서 "이 검증이 클라이언트에만 있는지, 서버에도 있는지"를 확인할 때
- 정적 분석으로 찾은 의심 지점(예: 지난 글의 암시적 Intent 하이재킹)이 실제 런타임에서도 재현되는지 검증할 때

## 1. Frida의 동작 모드 — Server / Gadget / Inject

Frida는 상황에 따라 세 가지 방식으로 붙는다.

| 모드 | 위치 | 필요 조건 | 특징 |
|---|---|---|---|
| frida-server | 디바이스에 상주하는 데몬 | 루팅 필요 | 가장 강력함. 어떤 앱이든 spawn/attach 자유 |
| frida-gadget | 앱 안에 내장되는 공유 라이브러리(.so) | 루팅 불필요, APK 리패키징 + 재서명 필요 | 무결성 체크·서명 검증이 있는 앱엔 한계 있음 |
| frida-inject | 호스트에서 실행되는 CLI 클라이언트 | 기존 Frida 연결 필요 | 짧은 스크립트를 한 번 주입할 때 |

루팅된 테스트 단말을 갖고 있으면 server가 제일 자유롭고, 루팅을 못 하는 실제 서비스 앱을 분석해야 하면 APK를 리패키징해서 gadget을 심는 방식을 쓴다. 이번 편은 가장 기본이 되는 **frida-server** 기준으로 다룬다.

## 2. 설치하기 전에 — 모바일 관점에서 확인해야 할 것

여기서 순서가 중요하다. **frida-server를 받기 전에** 먼저 확인해야 하는 게 두 가지 있다.

**① PC에 클라이언트부터 설치하고 버전을 기억해둔다**

```bash
pip install frida-tools
frida --version
```

이 시점의 버전 숫자를 반드시 기억해야 한다. 뒤에서 받을 서버 바이너리가 이 버전과 **정확히** 일치해야 하기 때문이다. Frida는 클라이언트(PC의 frida-tools/frida-python)와 서버(디바이스의 frida-server) 버전이 어긋나면 연결이 실패하거나 이상하게 동작한다. 예를 들어 17.16 클라이언트가 17.17 서버와 통신하면 정상적으로 붙지 않는 케이스가 보고돼 있다.

**② 타겟(안드로이드) 아키텍처를 확인한다**

```bash
adb shell getprop ro.product.cpu.abi
```

결과가 `arm64-v8a`면 서버도 arm64용을, `x86_64`(에뮬레이터 대부분)면 x86_64용을 받아야 한다. 여기서 아키텍처를 잘못 고르면 실행 자체가 안 되고 `not executable: magic 7F...` 같은 에러로 바로 막힌다.

| 확인 항목 | 명령어 |
|---|---|
| PC의 frida 클라이언트 버전 | `frida --version` |
| 디바이스 아키텍처(Android) | `adb shell getprop ro.product.cpu.abi` |

이 두 가지(클라이언트 버전, 디바이스 아키텍처)를 먼저 못 박고 GitHub releases에서 `frida-server-<버전>-android-<arch>.xz` 형식의 파일을 받는다. Android용은 `.xz`로 압축돼 있어서 `unxz` 또는 `xz -d`로 풀어야 한다.

## 3. frida-server 설치 & 실행 — 직접 해본 과정

준비된 `frida-server` 바이너리를 디바이스로 옮기고 실행 권한을 준 뒤 실행한다. 일반적으로 `/data/local/tmp/` 경로에 올린다.

```bash
adb push frida-server-<ver>-android-<arch> /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server &"
```

여기서 막히는 지점이 있다. **frida-server는 루팅된 셸(root 권한)에서 실행해야 한다.** 일반 권한으로 실행하면 다른 앱의 프로세스에 attach할 수 없거나 애초에 데몬이 제대로 뜨지 않는다. 에뮬레이터 기준으로는 `adb root`로 adbd 자체를 루트 권한으로 재시작해줘야 한다.

실제로 확인한 순서는 이랬다.

1. `adb devices`로 연결된 에뮬레이터 확인
2. `frida --version`으로 클라이언트 버전 재확인
3. 일반 셸에서 `frida-server` 실행 시도 → SELinux 정책 로드 실패로 권한 거부됨
4. `adb root`로 루트 권한 재시작
5. `adb shell`로 들어가서 `/data/local/tmp` 경로에 `frida-server`가 올라와 있는지 확인
6. 다시 `adb shell "/data/local/tmp/frida-server &"`로 백그라운드 실행

[![1](https://so-sung.github.io/assets/img/posts/2026-08-30/1.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/1.png)
*(캡처 1: adb devices로 에뮬레이터 확인 → frida --version 확인 → 일반 권한으로 frida-server 실행 시 SELinux 관련 권한 거부 → adb root로 루트 권한 재시작 → /data/local/tmp에 frida-server가 있는지 확인 → 루트 권한으로 재실행)*

일반 권한으로 먼저 실행을 시도했을 때 `Unable to load SELinux policy from the kernel: ... Permission denied`가 뜨는 걸 그대로 캡처에 남겨뒀다. **루트 권한 없이 frida-server를 실행하려고 하면 이런 식으로 막힌다는 걸 직접 보여주는 게 이 단계에서 제일 중요하다고 생각했다.**

## 4. 서버가 떠 있는지 — 프로세스 확인

frida-server가 정상적으로 떠 있으면, PC 쪽에서 `frida-ps`로 디바이스의 프로세스 목록을 조회할 수 있다.

| 명령어 | 용도 |
|---|---|
| `frida-ps -U` | USB 연결된 디바이스의 프로세스 목록 |
| `frida-ps -Ua` | 실행 중인 앱만 |
| `frida-ps -Uai` | 설치된 전체 앱까지 포함 |

이번 테스트에서는 미리 만들어둔 테스트 앱(`com.sosung.friatest`)이 실행 중인 상태에서, `frida-ps -U`와 `frida-ps -Ua`로 필터링해서 PID와 패키지명이 잡히는지 확인했다.

[![2](https://so-sung.github.io/assets/img/posts/2026-08-30/2.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/2.png)
*(캡처 2: frida-ps -U | findstr friatest, frida-ps -Ua | findstr friatest 결과 — PID 7358, 패키지명 com.sosung.friatest가 정상적으로 잡힘)*

여기서 PID와 패키지명이 잡히면, frida-server ↔ 클라이언트 연결이 정상이라는 뜻이다. 이 확인을 건너뛰고 바로 후킹 스크립트를 실행했다가 연결 오류를 만나면 원인을 좁히기 더 어려워지므로, 실습 순서상 꼭 넣는 게 맞다고 봤다.

## 5. 테스트 앱 준비 — 후킹 대상 만들기

실제 후킹 대상은 Android Studio에서 새로 만든 최소한의 테스트 앱(`com.sosung.friatest`)이다. 비밀번호를 입력받아서, 정해진 값과 일치하면 잠금을 해제하는 아주 단순한 로직만 넣었다. 실제 서비스 앱 로직을 흉내 내되, 후킹 포인트가 명확히 보이도록 일부러 단순화했다.

```kotlin
package com.sosung.friatest

import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.TextView
import android.graphics.Color
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val pwInput = findViewById<EditText>(R.id.pwInput)
        val loginBtn = findViewById<Button>(R.id.loginBtn)
        val statusBox = findViewById<TextView>(R.id.statusBox)

        loginBtn.setOnClickListener {
            val input = pwInput.text.toString()
            if (checkPassword(input)) {
                statusBox.text = "잠김 해제"
                statusBox.setBackgroundColor(Color.parseColor("#388E3C"))
            } else {
                statusBox.text = "잠김"
                statusBox.setBackgroundColor(Color.parseColor("#D32F2F"))
            }
        }
    }

    // 후킹 대상이 될 함수 — 일부러 하드코딩된 검증 로직
    private fun checkPassword(input: String): Boolean {
        return input == "correct_password_1234"
    }
}
```

포인트는 `checkPassword()`라는 함수 하나가 `true`/`false`를 리턴하고, 그 결과에 따라 화면 상태(`statusBox`)가 "잠김"/"잠김 해제"로 바뀐다는 것이다. 이 리턴값 하나가 이번 후킹의 목표 지점이다.

`1234`처럼 틀린 값을 넣으면 당연히 `checkPassword()`가 `false`를 리턴하고, 화면은 빨간색 "잠김" 상태로 남는다. 이게 후킹 전 베이스라인이다.

[![3](https://so-sung.github.io/assets/img/posts/2026-08-30/3.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/3.png)
*(캡처 3: Android Studio의 MainActivity.kt 코드(빨간 박스가 checkPassword 결과에 따라 statusBox를 바꾸는 부분)와, 에뮬레이터에서 1234를 입력했지만 후킹 전이라 "잠김" 상태 그대로인 화면)*

## 6. 후킹 스크립트 작성 — checkPassword 리턴값 가로채기

이제 이 함수를 Frida로 가로챈다. Java 레이어 함수를 후킹할 때는 `Java.perform` 안에서 `Java.use`로 클래스를 가져오고, 원하는 메서드의 `implementation`을 통째로 교체하는 방식을 쓴다.

```javascript
// hook.js
Java.perform(function () {
    var MainActivity = Java.use("com.sosung.friatest.MainActivity");

    MainActivity.checkPassword.implementation = function (input) {
        var originalResult = this.checkPassword(input);
        console.log("[*] 입력값: " + input);
        console.log("[*] 원래 리턴값: " + originalResult);

        // 무조건 true로 강제 변경
        return true;
    };
});
```

이 스크립트가 하는 일은 두 가지다.

1. 원본 `checkPassword()`를 그대로 호출해서, 실제로 어떤 값이 들어오고 원래 리턴값이 무엇인지 콘솔에 찍는다. (가로채기)
2. 그 값을 무시하고 **무조건 `true`를 리턴**하도록 함수 자체를 바꿔치기한다. (조작)

이 두 가지를 한 스크립트에서 동시에 보여주는 이유는, "값을 그냥 훔쳐보는 것"과 "그 값을 실제로 조작해서 앱 동작 자체를 바꾸는 것"이 Frida에서는 결국 같은 지점(`implementation` 교체)에서 일어난다는 걸 보여주고 싶었기 때문이다.

## 7. 실행 — spawn으로 앱을 새로 띄우면서 후킹 걸기

```bash
frida -U -f com.sosung.friatest -l hook.js --no-pause
```

`-f`는 spawn 옵션이다. 이미 떠 있는 프로세스에 붙는 게 아니라, 앱을 처음부터 다시 실행시키면서 그 순간부터 후킹을 건다는 뜻이다. `checkPassword()`처럼 앱 초기 로직에 걸려 있는 함수는 attach(`-n`)로 붙으면 이미 지나간 시점을 놓칠 수 있어서, 이런 경우엔 spawn이 더 안전하다.

실행하면 콘솔에 Frida REPL이 뜨고, 앱이 새로 실행된다. 이 상태에서 똑같이 `1234`를 입력하고 로그인 버튼을 눌렀다.

[![4](https://so-sung.github.io/assets/img/posts/2026-08-30/4.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/4.png)
*(캡처 4: `frida -U -f com.sosung.friatest -l hook.js` 실행 터미널 — 콘솔에 "입력값: 1234", "원래 리턴값: false"가 찍히고, 동시에 에뮬레이터 화면은 초록색 "잠김 해제"로 바뀜)*

콘솔에는 원래 리턴값이 `false`라고 그대로 찍혀 있다. 즉 `1234`는 여전히 틀린 비밀번호가 맞다. 그런데 화면은 초록색 "잠김 해제"로 바뀌었다. `checkPassword()`가 반환하는 실제 값과 무관하게, Frida가 `implementation`을 통째로 바꿔서 앱 로직이 항상 `true`를 받도록 만들었기 때문이다.

## 8. 참고 — 네이티브(.so) 레이어도 후킹 대상이 된다

지금까지는 Java 레이어(Kotlin/Java 코드) 함수를 후킹했지만, Frida는 `Interceptor.attach`를 쓰면 앱이 로드하는 네이티브 라이브러리(`.so`)의 export된 함수도 똑같이 가로챌 수 있다. 실무에서 자주 나오는 사례로는 다음과 같은 것들이 있다.

- `libssl.so`의 `SSL_read` / `SSL_write`를 후킹해서, 암호화되기 전/복호화된 후의 평문 트래픽을 확인하는 방식 (SSL Pinning 자체를 우회하지 않고도 평문을 볼 수 있음)
- SSL Pinning 검증 로직이 네이티브에 구현돼 있을 때, 해당 검증 함수의 리턴값을 강제로 통과시키는 방식
- 루팅 탐지 로직이 `.so` 안에서 `access()`, `stat()` 같은 시스템 콜을 호출해 특정 파일 존재 여부를 확인할 때, 그 시스템 콜의 리턴값을 조작해서 탐지를 우회하는 방식

Java 레이어 후킹은 `Java.use`로 접근할 클래스명·메서드 시그니처만 알면 되지만, 네이티브 레이어는 함수의 심볼 이름이나 오프셋을 먼저 찾아야 하기 때문에 난이도가 한 단계 올라간다. 이 부분은 다음 편 이후에 별도로 다룰 예정이다.

## 마무리 — 오늘 정리한 것과 다음 편

오늘은 스크립트 문법보다 "Frida를 처음 쓰는 사람이 설치 단계에서 왜 막히는지"에 집중했다. 정리하면 이렇다.

- frida-server를 받기 전에 **클라이언트 버전**과 **디바이스 아키텍처**를 먼저 확인해야 한다.
- frida-server는 **루트 권한** 없이는 정상적으로 실행되지 않는다.
- `frida-ps`로 서버-클라이언트 연결을 먼저 확인하고 후킹을 시도하는 게 디버깅에 유리하다.
- Java 레이어 후킹은 `implementation` 교체 한 줄로 리턴값을 완전히 바꿔치기할 수 있다.
- 네이티브(.so) 레이어도 `Interceptor.attach`로 같은 방식의 후킹이 가능하다.

다음 편은 오늘 다룬 `implementation` 교체를 조금 더 확장해서, 인자값 자체를 바꾸는 방식과 `Interceptor.attach`를 이용한 네이티브 함수 후킹을 다뤄볼 예정이다.

---

# Frida 1 - First Steps into Mobile Hooking: Options and a Hands-On Walkthrough

In previous posts ([implicit-intent-hijacking](https://so-sung.github.io/posts/implicit-intent-hijacking/), [intent-redirect-chaining](https://so-sung.github.io/posts/intent-redirect-chaining/), [apk-build-and-execution](https://so-sung.github.io/posts/apk-build-and-execution/)), I covered static analysis — reading manifests and code to find vulnerabilities, and force-launching components directly with `am start`. But static analysis has a ceiling. You can read code and guess how a function behaves, but **you can't see what values actually flow in and out at the moment the function runs** — not without executing it. Some of what I tried to verify in earlier posts hit exactly this wall: "the code looks like this, but I need to see it happen at runtime to be sure." So this time I'm starting from scratch with Frida, a dynamic analysis tool.

Rather than diving into advanced scripting (`Interceptor.attach` and the rest of the JS API), this post focuses on the procedure and options you have to get through before your **first successful hook** — as honestly as I can lay it out.

## 0. Why Frida exists

Frida is a Dynamic Binary Instrumentation (DBI) framework created by reverse-engineering researcher Ole André Vadla Ravnås. The core idea: inject code into a running process — without recompiling or patching the binary — to intercept and manipulate function calls.

Previously, seeing how a program actually behaved meant either having the source code or disassembling it and patching by hand. Frida lets you write a short JavaScript snippet and inject it into a live process, which fills exactly the gap static analysis can't. From an Android hacking perspective, this is useful when:

- Code is obfuscated and hard to read statically, so you print actual return values and arguments at runtime to infer the logic instead
- You want to bypass client-side checks in real time (password checks, payment validation, root detection, SSL pinning) to confirm whether a given check lives only on the client, or on the server too
- You want to confirm at runtime that something you found statically (like the implicit-intent hijack from an earlier post) actually reproduces during execution

## 1. Frida's three modes — Server / Gadget / Inject

Frida attaches in one of three ways, depending on the situation.

| Mode | Where it lives | Requirement | Notes |
|---|---|---|---|
| frida-server | A daemon that stays on the device | Requires root | Most powerful — spawn/attach freely into any app |
| frida-gadget | A shared library (.so) embedded inside the app | No root, but requires repackaging + re-signing the APK | Limited against apps with integrity/signature checks |
| frida-inject | A CLI client run from the host | Requires an existing Frida connection | For injecting a short script once |

If you have a rooted test device, `server` gives you the most freedom. If you need to analyze a real production app on an unrooted device, you repackage the APK to embed `gadget` instead. This post covers the most basic case: **frida-server**.

## 2. Before installing — what to check from a mobile perspective

The order here matters. There are two things to check **before** downloading frida-server.

**① Install the client on your PC first, and note the version**

```bash
pip install frida-tools
frida --version
```

Remember this version number exactly — the server binary you download later must match it **precisely**. If the client (frida-tools/frida-python on the PC) and the server (frida-server on the device) versions drift apart, the connection fails or behaves erratically. There are reported cases of a 17.16 client failing to properly connect to a 17.17 server, for example.

**② Check the target (Android) architecture**

```bash
adb shell getprop ro.product.cpu.abi
```

If the result is `arm64-v8a`, you need the arm64 server build; if it's `x86_64` (most emulators), you need the x86_64 build. Pick the wrong architecture and the binary simply won't run — you'll hit a `not executable: magic 7F...` error immediately.

| Item to check | Command |
|---|---|
| Frida client version (PC) | `frida --version` |
| Device architecture (Android) | `adb shell getprop ro.product.cpu.abi` |

Nail down these two things (client version, device architecture) first, then grab `frida-server-<version>-android-<arch>.xz` from GitHub releases. The Android build ships compressed as `.xz`, so you'll need `unxz` or `xz -d` to extract it.

## 3. Installing & running frida-server — what actually happened

Push the prepared `frida-server` binary to the device, make it executable, and run it. It typically goes under `/data/local/tmp/`.

```bash
adb push frida-server-<ver>-android-<arch> /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server &"
```

Here's where it gets stuck: **frida-server has to run from a rooted shell.** Run it without root, and it either can't attach to other apps' processes or the daemon doesn't come up properly at all. On an emulator, that means restarting adbd itself with root via `adb root`.

Here's the actual sequence I went through:

1. Confirm the connected emulator with `adb devices`
2. Re-confirm the client version with `frida --version`
3. Try running `frida-server` from a normal shell → permission denied due to a failed SELinux policy load
4. Restart with root privileges via `adb root`
5. Drop into `adb shell` and confirm `frida-server` is actually present under `/data/local/tmp`
6. Run `adb shell "/data/local/tmp/frida-server &"` again, this time as a root shell, in the background

[![1](https://so-sung.github.io/assets/img/posts/2026-08-30/1.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/1.png)
*(Capture 1: Checking the emulator with adb devices → confirming the client version with frida --version → a permission error running frida-server without root (SELinux-related) → restarting with root via adb root → confirming frida-server exists under /data/local/tmp → re-running it with root)*

I kept the `Unable to load SELinux policy from the kernel: ... Permission denied` error from the first, non-root attempt in the capture on purpose. **Showing exactly how it fails without root felt like the most important thing to get across at this step.**

## 4. Confirming the server is actually up — process check

Once frida-server is running properly, you can query the device's process list from the PC side with `frida-ps`.

| Command | Purpose |
|---|---|
| `frida-ps -U` | Process list on the USB-connected device |
| `frida-ps -Ua` | Running apps only |
| `frida-ps -Uai` | All installed apps |

With the test app (`com.sosung.friatest`) running, I filtered `frida-ps -U` and `frida-ps -Ua` to confirm the PID and package name showed up correctly.

[![2](https://so-sung.github.io/assets/img/posts/2026-08-30/2.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/2.png)
*(Capture 2: Results of `frida-ps -U | findstr friatest` and `frida-ps -Ua | findstr friatest` — PID 7358 and package name com.sosung.friatest show up correctly)*

Seeing the PID and package name here confirms the frida-server ↔ client connection is healthy. Skipping this check and jumping straight to the hook script makes any connection error much harder to diagnose, so I think it's worth keeping as a step in its own right.

## 5. Setting up the test app — building something to hook

The actual hook target is a minimal test app (`com.sosung.friatest`) I built fresh in Android Studio. The logic is intentionally simple: take a password, and unlock if it matches a fixed value. It's meant to mimic real app logic while keeping the hook point obvious.

```kotlin
package com.sosung.friatest

import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.TextView
import android.graphics.Color
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val pwInput = findViewById<EditText>(R.id.pwInput)
        val loginBtn = findViewById<Button>(R.id.loginBtn)
        val statusBox = findViewById<TextView>(R.id.statusBox)

        loginBtn.setOnClickListener {
            val input = pwInput.text.toString()
            if (checkPassword(input)) {
                statusBox.text = "Unlocked"
                statusBox.setBackgroundColor(Color.parseColor("#388E3C"))
            } else {
                statusBox.text = "Locked"
                statusBox.setBackgroundColor(Color.parseColor("#D32F2F"))
            }
        }
    }

    // The function we're going to hook — deliberately hardcoded validation logic
    private fun checkPassword(input: String): Boolean {
        return input == "correct_password_1234"
    }
}
```

The key point: one function, `checkPassword()`, returns `true`/`false`, and the UI state (`statusBox`) flips between "Locked"/"Unlocked" based on that return value. That single return value is today's hook target.

Enter a wrong value like `1234`, and `checkPassword()` naturally returns `false`, leaving the screen in its red "Locked" state. This is the baseline, before any hooking.

[![3](https://so-sung.github.io/assets/img/posts/2026-08-30/3.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/3.png)
*(Capture 3: The MainActivity.kt code in Android Studio (red box marking where statusBox is set based on checkPassword's result), alongside the emulator screen — 1234 was entered, but since no hook is active yet, it's still "Locked")*

## 6. Writing the hook — intercepting checkPassword's return value

Now let's intercept this function with Frida. To hook a Java-layer function, you grab the class with `Java.use` inside `Java.perform`, then replace the target method's `implementation` entirely.

```javascript
// hook.js
Java.perform(function () {
    var MainActivity = Java.use("com.sosung.friatest.MainActivity");

    MainActivity.checkPassword.implementation = function (input) {
        var originalResult = this.checkPassword(input);
        console.log("[*] Input: " + input);
        console.log("[*] Original return value: " + originalResult);

        // Force it to always return true
        return true;
    };
});
```

This script does two things:

1. Calls the original `checkPassword()` as-is, and logs the actual input and the original return value to the console. (interception)
2. Ignores that value and forces the function to **always return `true`** instead. (manipulation)

I wanted both in one script deliberately: "just peeking at a value" and "actually changing app behavior by manipulating it" happen at exactly the same point in Frida — replacing the `implementation`.

## 7. Running it — spawning the app fresh with the hook attached

```bash
frida -U -f com.sosung.friatest -l hook.js --no-pause
```

`-f` is the spawn option. Instead of attaching to an already-running process, it relaunches the app from scratch with the hook active from that moment on. A function like `checkPassword()` that fires early in the app's lifecycle can be missed if you attach (`-n`) after the fact — spawn is the safer choice here.

Running this brings up the Frida REPL and relaunches the app. I entered `1234` again and tapped the login button, same as before.

[![4](https://so-sung.github.io/assets/img/posts/2026-08-30/4.png)](https://so-sung.github.io/assets/img/posts/2026-08-30/4.png)
*(Capture 4: The terminal running `frida -U -f com.sosung.friatest -l hook.js` — the console prints "Input: 1234", "Original return value: false", while the emulator screen flips to a green "Unlocked")*

The console still prints the original return value as `false` — `1234` is still the wrong password. But the screen turned into green "Unlocked" anyway. Regardless of what `checkPassword()` actually returns, Frida replaced its `implementation` wholesale, so the app's logic always receives `true`.

## 8. Note — the native (.so) layer is a hook target too

So far this was a Java-layer (Kotlin/Java code) hook, but Frida can just as easily intercept exported functions in the native libraries (`.so`) an app loads, using `Interceptor.attach`. Common real-world examples include:

- Hooking `SSL_read` / `SSL_write` in `libssl.so` to view plaintext traffic before encryption or after decryption — without needing to bypass SSL pinning itself
- When pinning validation logic is implemented natively, forcing the validation function's return value to always pass
- When root-detection logic calls system calls like `access()` or `stat()` from within a `.so` to check whether specific files exist, manipulating the return value of that system call to defeat the detection

Java-layer hooking only requires knowing the class name and method signature to reach via `Java.use`, but native-layer hooking requires first locating the function's symbol name or offset — a step up in difficulty. I'll cover this separately in a later post.

## Wrap-up — what this covered, and what's next

Today was less about script syntax and more about "where a first-time Frida user actually gets stuck during setup." To summarize:

- Before downloading frida-server, check the **client version** and the **device architecture** first.
- frida-server won't run properly without **root**.
- Checking the server-client connection with `frida-ps` before attempting a hook makes debugging much easier.
- Java-layer hooking can completely swap a return value with a single `implementation` replacement.
- The native (.so) layer supports the same kind of hooking via `Interceptor.attach`.

Next up, I'll extend today's `implementation`-replacement approach further — modifying argument values directly, and hooking native functions with `Interceptor.attach`.
