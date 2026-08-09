---
title: "APK는 어떻게 만들어지고 실행되는가"
date: 2026-08-09 19:00:00 +0900
categories: [Android Security]
tags: [android, apk, dex, art, reverse-engineering]
toc: true
---

Android 앱 리버싱을 시작하기로 하고 처음 든 생각은, Frida부터 켜고 후킹 스크립트를 돌려보고 싶다는 거였다. 근데 막상 APK를 열어보니 `classes.dex`, `AndroidManifest.xml`, `resources.arsc` 같은 파일들이 뭔지도 모르는 채로 도구만 붙잡고 있으면 결국 "왜 되는지 모르는 우회"만 반복하게 될 것 같았다. 그래서 도구 얘기를 하기 전에, APK가 애초에 어떻게 만들어지고 어떻게 실행되는지부터 정리해보기로 했다.

## APK를 열면 보이는 것들

APK는 사실 ZIP 파일이다. 그냥 `unzip`으로 풀어보면 된다.

```bash
unzip target.apk -d target
```

풀어보면 대략 이런 구성이 나온다.

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

물론 앱마다 조금씩 다르다. Native 코드를 안 쓰면 `lib/`가 없을 수도 있고, DEX가 하나로 충분하면 `classes2.dex`도 없다. 문제는 이 파일들이 왜 이런 형태로 존재하는지, 그리고 이게 어떻게 실행까지 이어지는지를 모르면 나중에 JADX나 Ghidra를 돌려도 "결과가 나오긴 했는데 이게 뭘 의미하는지" 감이 안 잡힌다는 거다.

## 왜 `.class`가 아니라 DEX인가

Kotlin이나 Java로 짠 코드는 컴파일되면 원래 JVM이 쓰는 `.class` 파일이 된다. 근데 Android는 이 `.class`를 그대로 안 쓰고 DEX(Dalvik Executable)라는 별도 포맷으로 바꿔서 넣는다.

```text
Kotlin/Java Source → Compiler → .class 파일들 → R8 → DEX bytecode → APK
```

여기서 R8이라고 썼는데, 사실 D8/R8을 같이 병기하는 글이 많다. 근데 정확히는 역할이 다르다. D8은 `.class`를 DEX로 바꾸는(dexing) 것만 담당하는 컴파일러고, R8은 거기에 코드 축소, 난독화, 최적화까지 더 하는 상위 컴파일러다. 요즘 빌드 툴체인에서는 R8이 D8의 dexing 역할까지 흡수해서 처리하기 때문에, 둘을 나란히 놓고 "이거 아니면 저거" 식으로 이해하기보다는 R8이 D8을 포함한다고 보는 게 맞다.

Android가 왜 굳이 별도 포맷을 쓰냐면, 애초에 데스크톱과는 다른 조건에서 돌아가야 했기 때문이다. 메모리도 저장공간도 배터리도 제한적이고, 여러 앱이 동시에 떠 있는 환경. 이런 조건에 맞춰서 자체 실행 환경을 만든 게 처음엔 Dalvik Virtual Machine(DVM)이었고, 지금은 그 자리를 ART(Android Runtime)가 대신하고 있다. Android 5.0부터 ART가 기본이 됐고 Dalvik은 이제 안 쓴다.

## DEX 파일이 여러 개인 이유

앱이 커지면 `classes.dex` 하나가 아니라 `classes2.dex`, `classes3.dex`처럼 여러 개로 쪼개지는 걸 보게 된다. "앱이 커져서 자동으로 나뉜다"는 설명은 절반만 맞다. 정확히는 DEX 파일 하나가 담을 수 있는 메서드 레퍼런스가 65,536개(64K)로 제한돼 있어서다. 이 상한을 넘으면 Multi-Dex 구성이 강제된다. 그래서 분석할 때 `classes.dex` 하나만 보고 끝내면 안 되고, 있는 DEX 파일을 전부 확인해야 로직을 놓치지 않는다.

## DEX는 그냥 실행되지 않는다

DEX는 ARM64 CPU가 바로 돌릴 수 있는 네이티브 코드가 아니다. 그래서 ART라는 런타임이 중간에서 처리를 해줘야 하는데, 여기서 오해하기 쉬운 부분이 있다. ART가 실행하는 시점에 인터프리터로 한 줄씩 읽어서 처리한다고 생각하기 쉬운데, 실제로는 훨씬 미리 준비가 된다.

앱을 설치하면(혹은 백그라운드에서) `dex2oat`이라는 게 DEX를 미리 네이티브 코드(OAT/ELF 형태)로 컴파일해둔다. 이게 AOT(Ahead-Of-Time) 컴파일이다. 그래서 실행 시점에는

- 이미 AOT로 컴파일된 코드는 바로 실행되고
- 아직 컴파일 안 된 부분은 인터프리터가 처리하고
- 반복적으로 실행되는 핫코드는 JIT가 따로 최적화한다

이 세 가지가 상황에 맞게 섞여서 돌아간다. 이 얘기는 나중에 `.so` 파일을 Ghidra로 열어볼 때 다시 나올 텐데, 지금은 "DEX가 그대로 실행되는 게 아니라 ART가 미리든 그때그때든 네이티브 코드로 바꿔서 실행한다" 정도만 기억하면 된다.

## Manifest와 4대 컴포넌트

`AndroidManifest.xml`은 시스템이 이 앱을 어떻게 실행하고 관리할지 알려주는 파일이다. 권한, 패키지 정보, 그리고 Activity/Service/BroadcastReceiver/ContentProvider 같은 컴포넌트 선언이 여기 들어간다.

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

## 코드 말고 리소스 쪽도

`res/`에는 layout, drawable 같은 리소스가 들어가고, `resources.arsc`는 그 리소스 ID랑 실제 리소스를 연결해주는 컴파일된 테이블이다. `assets/`에는 앱이 직접 읽는 JSON이나 HTML, 설정 파일 같은 게 들어간다. 분석할 때는 이쪽에 API 키나 인증서, DB 파일 같은 민감한 정보가 평문으로 박혀있는 경우가 은근히 있어서 코드만큼 눈여겨봐야 한다.

## 서명은 왜 중요한가

`META-INF/`에는 APK 서명 정보가 들어있다. 예전 APK는 JAR 서명 기반 v1 방식을 썼는데, 지금은 APK 전체의 무결성을 더 강하게 검증하려고 v2, v3 서명 스킴이 추가됐다. 그래서 APK를 풀었다가 코드 좀 고치고 다시 zip으로 묶는다고 원래 앱이랑 똑같은 게 되는 건 아니다. 수정하는 순간 원래 서명은 무효가 되고, 설치하려면 다시 서명해야 한다. 이 부분은 나중에 서명 검증 우회를 직접 다뤄볼 때 더 깊게 볼 예정이다.

## 여기까지가 정적분석의 영역

지금까지 얘기한 건 전부 앱을 실행 안 시키고도 할 수 있는 것들, 즉 정적분석이다. 압축 풀고, 구조 보고, 디컴파일해서 로직 읽고. 이것만으로도 꽤 많은 걸 알 수 있지만 한계는 있다. 코드가 난독화됐거나, 로직이 런타임에 동적으로 결정되는 경우엔 정적분석만으론 막힌다.

그래서 다음 글부터는 실제로 앱을 돌려놓고 함수 호출을 가로채는 동적분석으로 넘어간다. 첫 타깃은 SSL Pinning이고, 도구는 Frida다.

---
---

# How an APK Gets Built and Executed

When I decided to get into Android reversing, my first instinct was to just fire up Frida and start hooking things. But once I actually opened an APK, I realized I didn't really know what `classes.dex`, `AndroidManifest.xml`, or `resources.arsc` even were. Poking around with tools without understanding the underlying structure felt like it would just turn into "bypassing things without knowing why they work." So before touching any tooling, I wanted to nail down how an APK is actually built and how it runs.

## What's Inside an APK

An APK is, underneath it all, a ZIP file. You can just unzip it.

```bash
unzip target.apk -d target
```

Do that, and you'll typically see something like this:

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

This varies from app to app, of course — no native code means no `lib/`, and a single DEX file means no `classes2.dex`. The real issue is that if you don't understand why these files exist in this form, and how they eventually get executed, running JADX or Ghidra later just produces output you can't actually interpret.

## Why DEX Instead of `.class`

Code written in Kotlin or Java compiles into `.class` files — the format the JVM normally uses. But Android doesn't ship those directly; it converts them into a separate format called DEX (Dalvik Executable).

```text
Kotlin/Java Source → Compiler → .class files → R8 → DEX bytecode → APK
```

I wrote "R8" there, though a lot of write-ups list D8 and R8 side by side as if they're interchangeable. They're not quite the same thing. D8 is the compiler responsible only for dexing — converting `.class` into DEX. R8 does that plus shrinking, obfuscation, and optimization. In the modern build toolchain, R8 absorbs D8's dexing role entirely, so it's more accurate to think of R8 as encompassing D8 rather than treating them as parallel choices.

The reason Android bothers with a separate format at all comes down to the constraints it was built under from the start — limited memory, limited storage, battery life, multiple apps running at once. The runtime that first grew out of those constraints was the Dalvik Virtual Machine (DVM), later replaced by ART (Android Runtime). ART's been the default since Android 5.0, and Dalvik is gone at this point.

## Why There's More Than One DEX File

As an app grows, you'll see multiple DEX files — `classes2.dex`, `classes3.dex`, and so on — instead of just one. "It splits automatically as the app gets bigger" is only half the story. The real reason is that a single DEX file can hold at most 65,536 (64K) method references. Cross that ceiling and Multi-Dex becomes mandatory. Which means when you're analyzing an app, stopping at `classes.dex` alone isn't enough — you need to check every DEX file present or you'll miss logic.

## DEX Doesn't Just "Run"

DEX isn't native code an ARM64 CPU can execute directly. That's where ART comes in — but there's a common misconception here. It's easy to assume ART interprets DEX line-by-line at runtime, but in reality far more happens ahead of time.

When an app is installed (or in the background afterward), something called `dex2oat` pre-compiles the DEX into native code (OAT/ELF format). That's Ahead-Of-Time (AOT) compilation. So at runtime:

- code already AOT-compiled just runs directly
- anything not yet compiled gets handled by the interpreter
- frequently-executed "hot" code gets further optimized by the JIT

All three work together depending on the situation. This comes back up later when opening `.so` files in Ghidra, but for now the key point is: DEX doesn't execute as-is — ART turns it into native code, either ahead of time or on the fly, and runs that.

## The Manifest and the Four Components

`AndroidManifest.xml` tells the system how to run and manage the app — permissions, package info, and declarations of Activity/Service/BroadcastReceiver/ContentProvider all live here.

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

## It's Not Just Code

`res/` holds resources like layouts and drawables, and `resources.arsc` is the compiled table that maps resource IDs to the actual resources. `assets/` holds files the app reads directly — JSON, HTML, config files. Worth checking here too, since API keys, certificates, or database files sometimes end up sitting in plaintext in this area — just as easy to overlook as the code itself.

## Why Signing Matters

`META-INF/` holds the APK's signing information. Older APKs used JAR-signing-based v1 signatures, but v2 and v3 schemes were added later to verify the integrity of the whole APK more strongly. So unzipping an APK, tweaking some code, and re-zipping it doesn't give you the same app back — the moment you modify it, the original signature becomes invalid, and you have to re-sign it to install. I'll get into bypassing signature verification directly in a later post.

## This Is All Still Static Analysis

Everything above is stuff you can do without ever running the app — unzip it, map the structure, decompile the logic. That gets you pretty far, but it has limits. Once obfuscation is involved, or logic gets decided dynamically at runtime, static analysis alone hits a wall.

So the next post moves into dynamic analysis — running the app for real and intercepting function calls as they happen. First target: SSL Pinning. First tool: Frida.
