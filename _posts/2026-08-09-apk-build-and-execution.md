---
title: "APK 분석 1"
date: 2026-08-09 20:00:00 +0900
categories: [Android Security]
tags: [android, apk, dex, art, apktool, static-analysis, reverse-engineering]
toc: true
---

안드로이드 앱을 해킹해보겠다고 마음먹었을 때 제일 먼저 하고 싶었던 건 Frida를 켜고 후킹 스크립트를 돌려보는 거였다. 근데 생각해보면 순서가 좀 이상하다. 뭘 후킹할지 알아야 후킹을 하지, 그 앱이 어떤 파일들로 이루어져 있고 어떻게 실행되는지도 모르는 채로 스크립트부터 돌리면 결국 "왜 되는지 모르는 우회"만 반복하게 된다. 그래서 도구를 켜기 전에 APK 파일 자체를 뜯어보는 것부터 시작하기로 했다. APK는 어떻게 열어볼 수 있는지, 그냥 unzip으로 풀어보는 것과 APKTool로 디코딩하는 게 뭐가 다른지, 그리고 그 안에 있는 파일들이 실제로 앱을 실행시킬 때 어떤 역할을 하는지를 오늘 정리해본다.

## APK를 여는 두 가지 방법 — unzip과 APKTool

APK는 사실 ZIP 포맷이다. 그래서 가장 단순하게는 그냥 압축 프로그램으로 열어볼 수 있다.

```bash
unzip target.apk -d target_zip
```

근데 이렇게 풀어보면 `AndroidManifest.xml`을 열었을 때 사람이 읽을 수 있는 XML이 아니라 이상한 바이너리 덩어리가 나온다. 처음엔 이게 왜 이런가 싶었는데, 이유는 간단하다. Android는 Manifest와 각종 레이아웃 XML을 텍스트 그대로 패키징하지 않고, 빌드 시점에 AXML(Android Binary XML)이라는 컴파일된 바이너리 포맷으로 바꿔서 넣는다. `resources.arsc`도 마찬가지로 리소스 ID와 실제 값을 연결하는 바이너리 테이블이라서 그냥 열어보면 의미를 알 수 없는 바이트 덩어리로 보인다.

그래서 실제 분석에서는 unzip 대신 APKTool을 쓴다.

```bash
apktool d target.apk -o target_apktool
```

APKTool은 AXML을 다시 읽을 수 있는 XML로 디코딩하고, `resources.arsc`를 원래의 `res/` 폴더 구조로 복원하고, `classes.dex`는 Smali라는 사람이 읽고 수정할 수 있는 형태로 역어셈블해준다. 심지어 `apktool b`로 다시 빌드해서 원래 APK와 거의 동일한(서명만 다시 해야 하는) 형태로 재조립하는 것도 가능하다. 즉 unzip은 "이 안에 뭐가 들어있나" 구조만 빠르게 훑어볼 때 쓰고, 실제로 Manifest를 읽거나 코드를 수정하고 싶으면 APKTool로 디코딩해야 한다.

> 참고: [Apktool 공식 GitHub](https://github.com/iBotPeaches/Apktool) · [resources.arsc 내부 구조 — Apktool Wiki](https://apktool.org/wiki/advanced/resources-arsc/)

## unzip으로 본 APK 구조

구조 확인만 할 거면 unzip으로도 충분하다. 실제로 풀어보면 대략 이런 모양이 나온다.

```text
target.apk
├── AndroidManifest.xml
├── classes.dex
├── classes2.dex
├── resources.arsc
├── lib/
│   ├── arm64-v8a/
│   ├── armeabi-v7a/
│   └── x86_64/
├── assets/
├── res/
└── META-INF/
```

앱마다 조금씩 다르다. Native 코드를 안 쓰면 `lib/`가 없을 수도 있고, DEX가 하나로 충분하면 `classes2.dex`도 없다. 여기서부터는 이 파일들이 각각 왜 이런 형태로 존재하고, 실행까지 어떻게 이어지는지를 하나씩 따라가 본다.

## 왜 `.class`가 아니라 DEX인가

Kotlin이나 Java로 짠 코드는 컴파일되면 원래 JVM이 쓰는 `.class` 파일이 된다. 그런데 Android는 이 `.class`를 그대로 패키징하지 않고 DEX(Dalvik Executable)라는 별도 포맷으로 바꿔서 넣는다.

```text
Kotlin/Java Source → Compiler → .class 파일들 → R8 → DEX bytecode → APK
```

여기서 R8이라고만 썼는데, D8과 R8을 뭉뚱그려 같은 걸로 이해하기 쉽다. 정확히는 역할이 나뉘어 있다. D8은 `.class`를 DEX로 바꾸는(dexing) 것만 담당하는 컴파일러고, R8은 거기에 사용하지 않는 코드를 걷어내는 축소(shrinking), 클래스/메서드 이름을 짧게 바꾸는 난독화(obfuscation), 메서드 인라이닝 같은 최적화까지 더 하는 상위 컴파일러다. 요즘 빌드 툴체인에서는 R8이 D8의 dexing 역할까지 흡수해서 한 번에 처리하기 때문에, 둘을 나란히 놓고 이해하기보다는 R8이 D8을 포함한다고 보는 게 정확하다.

Android가 굳이 별도 포맷을 쓰는 이유는 애초에 데스크톱과는 다른 조건에서 돌아가야 했기 때문이다. 메모리도 저장공간도 배터리도 제한적이고, 여러 앱이 동시에 떠 있는 환경. 이런 조건에 맞춰서 자체 실행 환경을 만든 게 처음엔 Dalvik Virtual Machine(DVM)이었고, 지금은 그 자리를 ART(Android Runtime)가 대신하고 있다. Android 5.0부터 ART가 기본이 됐고 Dalvik은 이제 안 쓴다.

> 참고: [Enable app optimization with R8 — Android Developers](https://developer.android.com/topic/performance/app-optimization/enable-app-optimization) · [Android Runtime and Dalvik — AOSP](https://source.android.com/docs/core/runtime)

## DEX 파일이 여러 개인 이유

앱이 커지면 `classes.dex` 하나가 아니라 `classes2.dex`, `classes3.dex`처럼 여러 개로 쪼개지는 걸 보게 된다. "앱이 커져서 자동으로 나뉜다"는 설명은 절반만 맞다. 정확히는 DEX 파일 하나가 담을 수 있는 메서드 레퍼런스가 65,536개(64K)로 제한돼 있어서다. 이 제한은 DEX 명세 자체에서 오는 건데, `invoke-*` 계열 명령어가 대상 메서드를 16비트 인덱스로 참조하기 때문에 표현 가능한 범위가 딱 여기까지다. 이 상한을 넘으면 Multi-Dex 구성이 강제된다. 그래서 분석할 때 `classes.dex` 하나만 보고 끝내면 안 되고, 있는 DEX 파일을 전부 확인해야 로직을 놓치지 않는다.

> 참고: [Enable multidex for apps with over 64K methods — Android Developers](https://developer.android.com/build/multidex)

## DEX는 그냥 실행되지 않는다

DEX는 ARM64 CPU가 바로 돌릴 수 있는 네이티브 코드가 아니다. 그래서 ART라는 런타임이 중간에서 처리를 해줘야 하는데, 여기서 오해하기 쉬운 부분이 있다. ART가 실행 시점에 인터프리터로 한 줄씩 읽어서 처리한다고 생각하기 쉬운데, 실제로는 훨씬 미리 준비가 된다.

앱을 설치하면(혹은 백그라운드에서) `dex2oat`이라는 게 DEX를 미리 네이티브 코드(OAT/ELF 형태)로 컴파일해둔다. 이게 AOT(Ahead-Of-Time) 컴파일이다. 그래서 실행 시점에는

- 이미 AOT로 컴파일된 코드는 바로 실행되고
- 아직 컴파일 안 된 부분은 인터프리터가 처리하고
- 반복적으로 실행되는 핫코드는 JIT가 따로 최적화한다

이 세 가지가 상황에 맞게 섞여서 돌아간다. 이 얘기는 나중에 `.so` 파일을 Ghidra로 열어볼 때 다시 나올 텐데, 지금은 "DEX가 그대로 실행되는 게 아니라 ART가 미리든 그때그때든 네이티브 코드로 바꿔서 실행한다" 정도만 기억하면 된다.

> 참고: [Android Runtime and Dalvik — AOSP](https://source.android.com/docs/core/runtime) · [ART JIT Compiler — AOSP](https://source.android.com/docs/core/runtime/jit-compiler)

## Manifest와 4대 컴포넌트

`AndroidManifest.xml`은 시스템이 이 앱을 어떻게 실행하고 관리할지 알려주는 파일이다. 권한, 패키지 정보, 그리고 Activity/Service/BroadcastReceiver/ContentProvider 같은 컴포넌트 선언이 여기 들어간다. 앞서 본 것처럼 unzip으로 풀면 바이너리로 나오지만, APKTool로 디코딩하면 아래처럼 정상적인 XML로 볼 수 있다.

```xml
<activity
    android:name=".PaymentActivity"
    android:exported="true" />
```

컴포넌트 네 개의 역할은 대충 이렇다.

| Component | 역할 |
|---|---|
| Activity | 화면 하나 |
| Service | 백그라운드 작업 |
| BroadcastReceiver | 브로드캐스트 이벤트 수신 |
| ContentProvider | 앱 간 데이터 공유 |

이게 보안 관점에서 중요한 이유는, 이 컴포넌트들이 다른 앱이나 시스템과 통신하는 진입점이 될 수 있기 때문이다. 위 예시처럼 `exported="true"`인 Activity를 보면 보통 "노출됐으니 위험하다"고 바로 생각하기 쉬운데, 이건 성급한 결론이다. `exported=true` 자체는 문제가 아니다. 정상적인 앱도 외부 연동을 위해 컴포넌트를 노출해야 하는 경우가 많다. 진짜 봐야 할 건 그 컴포넌트가 어떤 Intent를 받는지, 어떤 입력을 처리하는지, 호출자의 권한을 검증하는지, 민감한 데이터를 돌려주는지다. 이 질문에 답이 나와야 실제 위험도를 판단할 수 있다.

> 참고: [App Manifest Overview — Android Developers](https://developer.android.com/guide/topics/manifest/manifest-intro) · [App Fundamentals — Android Developers](https://developer.android.com/guide/components/fundamentals) · [Android exported components — Android Developers](https://developer.android.com/privacy-and-security/risks/android-exported)

## 코드 말고 리소스 쪽도

`res/`에는 layout, drawable 같은 리소스가 들어가고, `resources.arsc`는 그 리소스 ID랑 실제 리소스를 연결해주는 컴파일된 테이블이다. `assets/`에는 앱이 직접 읽는 JSON이나 HTML, 설정 파일 같은 게 들어간다. 분석할 때는 이쪽에 API 키나 인증서, DB 파일 같은 민감한 정보가 평문으로 박혀있는 경우가 은근히 있어서 코드만큼 눈여겨봐야 한다.

> 참고: [App resources overview — Android Developers](https://developer.android.com/guide/topics/resources/providing-resources)

## 서명은 왜 중요한가

`META-INF/`에는 APK 서명 정보가 들어있다. 예전 APK는 JAR 서명 기반 v1 방식을 썼는데, 지금은 APK 전체의 무결성을 더 강하게 검증하려고 v2, v3 서명 스킴이 추가됐다. 그래서 APK를 풀었다가 코드 좀 고치고 다시 zip으로 묶는다고 원래 앱이랑 똑같은 게 되는 건 아니다. 수정하는 순간 원래 서명은 무효가 되고, 설치하려면 다시 서명해야 한다. 이 부분은 나중에 서명 검증 우회를 직접 다뤄볼 때 더 깊게 볼 예정이다.

> 참고: [APK Signature Scheme v2 — Android Developers](https://source.android.com/docs/security/features/apksigning/v2) · [APK Signature Scheme v3 — Android Developers](https://source.android.com/docs/security/features/apksigning/v3)

## 정적분석이라는 것

여기까지 한 건 전부 앱을 단 한 번도 실행하지 않고 한 작업이다. 압축을 풀거나 디코딩하고, 파일 구조를 보고, Manifest를 읽고, DEX가 실행되는 원리를 이해하는 것. 이걸 정적분석(Static Analysis)이라고 부른다. OWASP MASTG에서는 정적분석을 "앱을 실행하지 않고 소스코드나 바이너리를 분석하는 것"으로 정의하는데, 실무에서는 여기에 몇 가지가 더 포함된다.

- **구조 분석**: 오늘 한 것처럼 APK 안에 어떤 파일이 있는지, Manifest에 어떤 컴포넌트와 권한이 선언돼 있는지 파악하는 것
- **코드 리딩**: `classes.dex`를 JADX로 Java 형태로, 혹은 baksmali로 Smali 형태로 디컴파일해서 실제 로직을 읽는 것
- **문자열/시크릿 탐색**: 하드코딩된 API 키, URL, 인증서 같은 게 코드나 `assets/` 안에 그대로 박혀있는지 찾는 것
- **Native 바이너리 분석**: `lib/` 안의 `.so` 파일을 Ghidra나 IDA로 열어서 어셈블리 수준에서 로직을 확인하는 것

이 네 가지만 해도 상당히 많은 걸 알 수 있다. 근데 정적분석에는 명확한 한계가 있다. R8로 난독화된 코드는 클래스/메서드 이름이 다 `a`, `b`, `c`로 바뀌어 있어서 로직 흐름은 보여도 의미를 파악하기 어렵다. 문자열이 런타임에 복호화되는 경우엔 코드만 봐서는 실제 값을 알 수 없다. 리플렉션으로 클래스를 동적으로 로드하거나, 서버에서 설정값을 받아와 그때그때 로직을 바꾸는 경우도 정적분석만으로는 잡아낼 수 없다. SSL Pinning처럼 "이 조건이 맞아야만 통과한다"는 로직도, 코드를 읽어서 조건을 이해하는 것과 실제로 그 조건을 우회해서 통과시키는 건 다른 얘기다.

그래서 다음 글부터는 실제로 앱을 돌려놓고 함수 호출을 가로채는 동적분석(Dynamic Analysis)으로 넘어간다. 오늘 정적분석으로 구조를 파악했다면, 다음은 그 구조 위에서 실제로 동작하는 로직을 실시간으로 조작해보는 단계다. 첫 타깃은 SSL Pinning이고, 도구는 Frida다.

> 참고: [MASTG-TECH-0025: Automated Static Analysis — OWASP](https://mas.owasp.org/MASTG/techniques/android/MASTG-TECH-0025/) · [MASTG-TECH-0014: Static Analysis on Android — OWASP](https://mas.owasp.org/MASTG/techniques/android/MASTG-TECH-0014/)

---
---

# APK Analysis 1

When I decided to get into Android hacking, my first instinct was to just fire up Frida and start hooking things. But thinking about it, that's backwards. You need to know what to hook before you can hook it, and you can't know that without first understanding what files make up the app and how it actually runs. So before touching any tooling, I wanted to start by taking the APK itself apart — how you can even open one, what the difference is between just unzipping it and decoding it with APKTool, and what role each file inside actually plays once the app runs.

## Two Ways to Open an APK — unzip and APKTool

An APK is, underneath it all, a ZIP file. So the simplest way to look inside is to just unzip it.

```bash
unzip target.apk -d target_zip
```

But do that, and opening `AndroidManifest.xml` doesn't give you readable XML — it gives you a weird blob of binary data. At first I wasn't sure why, but the reason is simple: Android doesn't package the Manifest and layout XML files as plain text. At build time, they get compiled into a binary format called AXML (Android Binary XML). `resources.arsc` is the same story — it's a binary table linking resource IDs to actual values, so opening it directly just gives you a meaningless byte blob.

This is why real analysis uses APKTool instead of raw unzip.

```bash
apktool d target.apk -o target_apktool
```

APKTool decodes AXML back into readable XML, reconstructs `resources.arsc` back into a normal-looking `res/` folder, and disassembles `classes.dex` into Smali — a human-readable, editable intermediate format. You can even rebuild it with `apktool b` into something nearly identical to the original APK (you'll just need to re-sign it). So unzip is for a quick structural glance at what's inside; if you actually want to read the Manifest or modify the code, you decode it with APKTool.

> Reference: [Apktool official GitHub](https://github.com/iBotPeaches/Apktool) · [Inside resources.arsc — Apktool Wiki](https://apktool.org/wiki/advanced/resources-arsc/)

## What unzip Shows You

If all you need is the structure, unzip is enough. Typically you'll see something like this:

```text
target.apk
├── AndroidManifest.xml
├── classes.dex
├── classes2.dex
├── resources.arsc
├── lib/
│   ├── arm64-v8a/
│   ├── armeabi-v7a/
│   └── x86_64/
├── assets/
├── res/
└── META-INF/
```

This varies from app to app — no native code means no `lib/`, and a single DEX file means no `classes2.dex`. From here, let's walk through why each of these files exists in this form, and how it eventually leads to execution.

## Why DEX Instead of `.class`

Code written in Kotlin or Java compiles into `.class` files — the format the JVM normally uses. But Android doesn't package those directly; it converts them into a separate format called DEX (Dalvik Executable).

```text
Kotlin/Java Source → Compiler → .class files → R8 → DEX bytecode → APK
```

I wrote just "R8" there, and it's easy to lump D8 and R8 together as the same thing. They're not quite the same. D8 is the compiler responsible only for dexing — converting `.class` into DEX. R8 does that plus shrinking (stripping out unused code), obfuscation (shortening class/method names), and optimizations like method inlining. In the modern build toolchain, R8 absorbs D8's dexing role entirely, so it's more accurate to think of R8 as encompassing D8 rather than treating them as two separate things.

The reason Android bothers with a separate format at all comes down to the constraints it was built under from the start — limited memory, limited storage, battery life, multiple apps running at once. The runtime that first grew out of those constraints was the Dalvik Virtual Machine (DVM), later replaced by ART (Android Runtime). ART's been the default since Android 5.0, and Dalvik is gone at this point.

> Reference: [Enable app optimization with R8 — Android Developers](https://developer.android.com/topic/performance/app-optimization/enable-app-optimization) · [Android Runtime and Dalvik — AOSP](https://source.android.com/docs/core/runtime)

## Why There's More Than One DEX File

As an app grows, you'll see multiple DEX files — `classes2.dex`, `classes3.dex`, and so on — instead of just one. "It splits automatically as the app gets bigger" is only half the story. The real reason is that a single DEX file can hold at most 65,536 (64K) method references. That limit comes directly from the DEX spec — `invoke-*` instructions reference the target method through a 16-bit index, so that's the ceiling of what's representable. Cross it and Multi-Dex becomes mandatory. Which means when you're analyzing an app, stopping at `classes.dex` alone isn't enough — you need to check every DEX file present or you'll miss logic.

> Reference: [Enable multidex for apps with over 64K methods — Android Developers](https://developer.android.com/build/multidex)

## DEX Doesn't Just "Run"

DEX isn't native code an ARM64 CPU can execute directly. That's where ART comes in — but there's a common misconception here. It's easy to assume ART interprets DEX line-by-line at runtime, but in reality far more happens ahead of time.

When an app is installed (or in the background afterward), something called `dex2oat` pre-compiles the DEX into native code (OAT/ELF format). That's Ahead-Of-Time (AOT) compilation. So at runtime:

- code already AOT-compiled just runs directly
- anything not yet compiled gets handled by the interpreter
- frequently-executed "hot" code gets further optimized by the JIT

All three work together depending on the situation. This comes back up later when opening `.so` files in Ghidra, but for now the key point is: DEX doesn't execute as-is — ART turns it into native code, either ahead of time or on the fly, and runs that.

> Reference: [Android Runtime and Dalvik — AOSP](https://source.android.com/docs/core/runtime) · [ART JIT Compiler — AOSP](https://source.android.com/docs/core/runtime/jit-compiler)

## The Manifest and the Four Components

`AndroidManifest.xml` tells the system how to run and manage the app — permissions, package info, and declarations of Activity/Service/BroadcastReceiver/ContentProvider all live here. As shown earlier, raw unzip gives you binary, but decoded with APKTool it looks like ordinary XML:

```xml
<activity
    android:name=".PaymentActivity"
    android:exported="true" />
```

Roughly, the four components do this:

| Component | Role |
|---|---|
| Activity | A single screen |
| Service | Background work |
| BroadcastReceiver | Receives broadcast events |
| ContentProvider | Shares data between apps |

Why this matters for security: these components can serve as entry points for other apps or the system to communicate with yours. Seeing an Activity with `exported="true"` like above, it's tempting to jump straight to "exposed, therefore dangerous" — but that's jumping the gun. `exported=true` on its own isn't the problem. Plenty of legitimate apps need to expose components for external integration. What actually matters is what Intent it accepts, what input it processes, whether it checks the caller's permissions, and whether it hands back sensitive data. Those questions are what determine real risk.

> Reference: [App Manifest Overview — Android Developers](https://developer.android.com/guide/topics/manifest/manifest-intro) · [App Fundamentals — Android Developers](https://developer.android.com/guide/components/fundamentals) · [Android exported components — Android Developers](https://developer.android.com/privacy-and-security/risks/android-exported)

## It's Not Just Code

`res/` holds resources like layouts and drawables, and `resources.arsc` is the compiled table that maps resource IDs to the actual resources. `assets/` holds files the app reads directly — JSON, HTML, config files. Worth checking here too, since API keys, certificates, or database files sometimes end up sitting in plaintext in this area — just as easy to overlook as the code itself.

> Reference: [App resources overview — Android Developers](https://developer.android.com/guide/topics/resources/providing-resources)

## Why Signing Matters

`META-INF/` holds the APK's signing information. Older APKs used JAR-signing-based v1 signatures, but v2 and v3 schemes were added later to verify the integrity of the whole APK more strongly. So unzipping an APK, tweaking some code, and re-zipping it doesn't give you the same app back — the moment you modify it, the original signature becomes invalid, and you have to re-sign it to install. I'll get into bypassing signature verification directly in a later post.

> Reference: [APK Signature Scheme v2 — Android Developers](https://source.android.com/docs/security/features/apksigning/v2) · [APK Signature Scheme v3 — Android Developers](https://source.android.com/docs/security/features/apksigning/v3)

## What Static Analysis Actually Is

Everything covered so far was done without ever running the app once — unzipping or decoding it, reading the file structure, reading the Manifest, understanding how DEX gets executed. This is Static Analysis. OWASP's MASTG defines it as examining an app's source code or binary without executing it, and in practice it usually covers a few distinct things:

- **Structural analysis**: figuring out what files are inside the APK and what components/permissions are declared in the Manifest, like we did today
- **Reading the code**: decompiling `classes.dex` with JADX into Java-like source, or with baksmali into Smali, to actually read the logic
- **Hunting for strings and secrets**: checking whether hardcoded API keys, URLs, or certificates are sitting in the code or in `assets/`
- **Native binary analysis**: opening `.so` files in `lib/` with Ghidra or IDA to inspect logic at the assembly level

These four alone reveal a lot. But static analysis has clear limits. Code obfuscated by R8 has its class and method names rewritten to `a`, `b`, `c`, so you can follow the logic's shape without understanding what it means. If strings are decrypted only at runtime, reading the code alone won't reveal the actual values. Reflection-based dynamic class loading, or logic that changes based on config fetched from a server, can't be caught through static analysis alone. And something like SSL Pinning — "this only passes if a specific condition is met" — reading the condition in code and actually bypassing it at runtime are two different problems entirely.

That's why the next post moves into Dynamic Analysis — running the app for real and intercepting function calls as they happen. If today was about mapping the structure through static analysis, the next step is manipulating the logic that actually runs on top of that structure, in real time. First target: SSL Pinning. First tool: Frida.

> Reference: [MASTG-TECH-0025: Automated Static Analysis — OWASP](https://mas.owasp.org/MASTG/techniques/android/MASTG-TECH-0025/) · [MASTG-TECH-0014: Static Analysis on Android — OWASP](https://mas.owasp.org/MASTG/techniques/android/MASTG-TECH-0014/)
