---
title: "Frida 2 - 네이티브(.so) 레이어 탐지 우회: 루팅·디버깅 탐지 무력화 실전"
date: 2026-09-06 00:00:00 +0900
categories: [Mobile Hacking, Android]
tags: [android, frida, native, so, root-detection, anti-debugging, jni, jadx, apktool, dynamic-analysis]
toc: true
---

지난 글([frida-first-hooking](https://so-sung.github.io/posts/frida-first-hooking/))에서는 Java 레이어 함수 하나(`checkPassword`)의 `implementation`을 통째로 갈아치우는 방식으로 후킹의 첫걸음을 뗐다. 그런데 실제로 점검을 나가보면, 특히 금융 앱들은 이 정도로 만만하지 않다. 대부분 루팅 탐지·디버깅 탐지 로직을 자바가 아니라 **네이티브(.so) 레이어**에 심어두고, 이 소를 다시 무결성 체크로 감싸서 후킹 자체를 어렵게 만든다.

점검자 입장에서 이 절차(APK를 뜯어서 so 파일이 뭘 하는지 파악하고, 그 함수를 Frida로 후킹해서 우회하는 것)는 실무에서 반드시 거쳐야 하는 과정이다. 그래서 오늘은 이 절차를 직접 재현해보기로 했다. 지난번 만든 테스트 앱(`com.sosung.friatest`)에 네이티브 루팅/디버깅 탐지 로직을 새로 심고, 그걸 정적으로 분석한 다음 Frida로 우회하는 데까지 가봤다.

## 0. 오늘 하려는 것 — 왜 네이티브 레이어인가

시중 보안 솔루션(상용 앱 보호 SDK)을 뜯어보면 자바 레벨에서 탐지 로직을 그대로 노출하는 경우는 거의 없다. 대부분 아래와 같은 구조를 취한다.

- 자바 코드에는 `native boolean isRooted()`처럼 **선언만** 있고, 실제 로직은 `.so` 안에 컴파일되어 들어가 있다.
- `.so` 파일 자체도 별도의 시그니처/룰 파일과 해시를 대조해서, 파일이 변조됐는지 자체적으로 검증한다.
- 탐지 로직이 `ptrace`, `access()`, `/proc/self/status`의 `TracerPid` 값처럼 시스템 콜/파일 레벨에서 동작하기 때문에, 자바 레이어에서 아무리 후킹해도 소용이 없다.

즉, "자바 함수 후킹"만으로는 이런 앱을 우회할 수 없고, **so 파일 안의 함수를 직접 찾아서 후킹**해야 한다. 오늘 글은 이 지점을 손으로 직접 만들어보고 뚫어보는 데 목적이 있다.

## 1. 테스트 앱에 네이티브 탐지 로직 심기

먼저 테스트 앱에 실제로 탐지할 대상을 만들어야 한다. `app/src/main/cpp/` 아래에 CMake 기반 네이티브 모듈을 추가했다.

**CMakeLists.txt**

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("friatest")

add_library(friatest SHARED
    native-lib.cpp
)

find_library(log-lib log)

target_link_libraries(friatest ${log-lib})
```

**native-lib.cpp — 실제 탐지 로직**

```cpp
#include <jni.h>
#include <sys/ptrace.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

// 디버거(및 Frida attach) 탐지: 자기 자신에 PTRACE_TRACEME를 걸어본다.
// 이미 다른 프로세스(디버거/Frida)가 붙어있으면 이 호출 자체가 실패한다.
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_sosung_friatest_MainActivity_nativeIsDebugged(JNIEnv *env, jobject /* this */) {
    if (ptrace(PTRACE_TRACEME, 0, 1, 0) < 0) {
        return JNI_TRUE;
    }
    ptrace(PTRACE_DETACH, 0, 1, 0);
    return JNI_FALSE;
}

// 루팅 탐지: su 바이너리가 존재하는 흔한 경로들을 순서대로 확인한다.
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_sosung_friatest_MainActivity_nativeIsRooted(JNIEnv *env, jobject /* this */) {
    const char *paths[] = {
        "/system/bin/su",
        "/system/xbin/su",
        "/sbin/su",
        "/system/app/Superuser.apk"
    };
    for (const char *path : paths) {
        if (access(path, F_OK) == 0) {
            return JNI_TRUE;
        }
    }
    return JNI_FALSE;
}
```

**MainActivity.kt — UI와 연결**

```kotlin
package com.sosung.friatest

import android.graphics.Color
import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    companion object {
        init {
            System.loadLibrary("friatest")
        }
    }

    external fun nativeIsDebugged(): Boolean
    external fun nativeIsRooted(): Boolean

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val statusBox = findViewById<TextView>(R.id.statusBox)
        val checkBtn = findViewById<Button>(R.id.checkBtn)

        checkBtn.setOnClickListener {
            val debugged = nativeIsDebugged()
            val rooted = nativeIsRooted()
            val detected = debugged || rooted

            if (detected) {
                statusBox.text = "우회 실패\n(디버깅:$debugged / 루팅:$rooted)"
                statusBox.setBackgroundColor(Color.parseColor("#D32F2F"))
            } else {
                statusBox.text = "우회 성공\n(디버깅:$debugged / 루팅:$rooted)"
                statusBox.setBackgroundColor(Color.parseColor("#388E3C"))
            }
        }
    }
}
```

빌드하면 `libfriatest.so`가 생성되고, 테스트에 쓰는 Nox Player가 x86 에뮬레이터이기 때문에 APK 안에는 `lib/x86/libfriatest.so` 형태로 패키징된다.

[![1](https://so-sung.github.io/assets/img/posts/2026-09-06/1.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/1.png)
*(캡처 1: Android Studio에서 `native-lib.cpp` 코드와 `BUILD SUCCESSFUL` 빌드 로그 — 오늘 분석/우회할 대상의 "원본"을 보여주는 베이스라인 컷)*

## 2. 후킹 전 베이스라인 확인

Nox Player 에뮬레이터에 설치하고 `탐지 우회 체크` 버튼을 눌러본다. 실제로 눌러보니 `디버깅:true / 루팅:true`로 둘 다 탐지됐다 — "우회 실패" 빨간불이다.

여기서 하나 짚을 게 있다. Nox Player는 **기본적으로 su 바이너리가 미리 심어져 있는, 처음부터 루팅된 형태의 에뮬레이터**다. 그래서 별다른 조작 없이도 `nativeIsRooted()`가 `true`로 잡히는 게 정상이다. `디버깅:true` 쪽은, adb 연결이나 백그라운드에 남아있던 디버그 브리지 때문에 `/proc/self/status`의 `TracerPid`가 이미 값을 가지고 있었을 가능성이 있다. 즉 일반 순정 에뮬레이터(AVD)라면 이 베이스라인 화면이 `false/false`(초록불)로 뜨는 게 더 자연스럽고, Nox처럼 루팅된 에뮬레이터를 쓰면 이렇게 처음부터 빨간불이 뜨는 게 오히려 "이 앱이 탐지를 제대로 하고 있다"는 걸 보여주는 좋은 예시가 된다.

[![2](https://so-sung.github.io/assets/img/posts/2026-09-06/2.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/2.png)
*(캡처 2: Nox Player에서 `탐지 우회 체크`를 누른 직후, `statusBox`가 빨간색 "우회 실패\n(디버깅:true / 루팅:true)"로 표시된 화면)*

## 3. 실전 분석 관점 — APK를 뜯어서 이 구조를 파악한다면?

지금은 내가 직접 짠 코드라 구조를 알고 있지만, 실무에서는 남의 앱을 받아서 이 구조를 처음부터 파악해야 한다. 순서는 이렇다.

### 3-1. APK 추출

```bash
adb shell pm path com.sosung.friatest
adb pull /data/app/~~<random>==/com.sosung.friatest-<random>==/base.apk
```

### 3-2. jadx로 자바 레이어 확인

```bash
jadx-gui base.apk
```

`MainActivity` 클래스를 열어보면 아래처럼 **본문 없이 시그니처만 있는 native 함수**가 그대로 보인다. jadx는 Kotlin 소스가 아니라 컴파일된 dex를 Java 스타일로 역컴파일하기 때문에, 실제로는 이런 형태로 나온다.

```java
public final class MainActivity extends AppCompatActivity {
    public final native boolean nativeIsDebugged();
    public final native boolean nativeIsRooted();

    static {
        System.loadLibrary("friatest");
    }

    public static final void onCreate$lambda$0(MainActivity this$0, TextView $statusBox, View it) {
        boolean debugged = this$0.nativeIsDebugged();
        boolean rooted = this$0.nativeIsRooted();
        boolean detected = debugged || rooted;
        if (detected) {
            $statusBox.setText("우회 실패\n(디버깅:" + debugged + " / 루팅:" + rooted + ")");
            $statusBox.setBackgroundColor(Color.parseColor("#D32F2F"));
        } else {
            $statusBox.setText("우회 성공\n(디버깅:" + debugged + " / 루팅:" + rooted + ")");
            $statusBox.setBackgroundColor(Color.parseColor("#388E3C"));
        }
    }
}
```

`public final native boolean nativeIsDebugged();`, `nativeIsRooted();` — **본문이 아예 없다.** 이게 "이 로직은 네이티브에 있다"는 확실한 신호다.

[![3](https://so-sung.github.io/assets/img/posts/2026-09-06/3.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/3.png)
*(캡처 3: jadx-gui에서 `MainActivity` 클래스를 열어, `native` 키워드가 붙은 두 함수 선언(본문 없음)과 `onCreate$lambda$0`에서 그 함수들을 호출하는 부분이 함께 보이는 화면)*

### 3-3. apktool로 리소스/구조 확인

```bash
apktool d base.apk -o friatest_decoded
```

디코딩된 폴더 트리를 보면(테스트 환경이 Nox Player=x86 단일 아키텍처라서 `lib/` 아래 아키텍처가 하나만 있다):

```
friatest_decoded/
|   AndroidManifest.xml
|   apktool.yml
|
+---lib
|   \---x86
|           libfriatest.so
|           liblog.so
|
+---original
|       AndroidManifest.xml
|
+---res
    +---anim
    |       abc_fade_in.xml
    |       abc_fade_out.xml
    |       ...
```

`liblog.so`가 같이 들어있는 건 `CMakeLists.txt`에서 `find_library(log-lib log)`로 안드로이드 로그 라이브러리를 링크했기 때문이다.

[![4](https://so-sung.github.io/assets/img/posts/2026-09-06/4.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/4.png)
*(캡처 4: `apktool d`로 디코딩한 폴더 트리 — `lib/x86/` 아래 `libfriatest.so`, `liblog.so`가 들어있는 구조)*

## 4. so 파일은 어떻게 보나 — 심볼 확인

여기서부터가 오늘 글의 핵심 질문("so 파일은 어떻게 보지?")에 대한 답이다. `.so`는 결국 ELF 바이너리이기 때문에, 어떤 함수가 익스포트되어 있는지부터 확인하는 게 순서다.

**Windows 환경 팁**: 리눅스처럼 `nm`/`readelf`가 기본으로 깔려있지 않기 때문에, Android Studio가 이미 설치해둔 NDK 툴체인 안의 `llvm-nm.exe` / `llvm-readelf.exe`를 그대로 쓰면 된다. 보통 아래 경로에 있으므로, 이 경로를 환경변수 `PATH`에 등록해두면 어디서든 `llvm-nm`, `llvm-readelf` 명령으로 바로 쓸 수 있다.

```
%LOCALAPPDATA%\Android\Sdk\ndk\<버전>\toolchains\llvm\prebuilt\windows-x86_64\bin
```

이렇게 등록해두고 실제로 뽑아본 결과:

```
C:\study\frida\friatest_decoded\lib\x86>llvm-nm -D --defined-only libfriatest.so
000005f0 T Java_com_sosung_friatest_MainActivity_nativeIsDebugged
00000680 T Java_com_sosung_friatest_MainActivity_nativeIsRooted

C:\study\frida\friatest_decoded\lib\x86>llvm-readelf -sW libfriatest.so | findstr FUNC
     1: 00000000     0 FUNC    GLOBAL DEFAULT   UND __cxa_atexit@LIBC
     2: 00000000     0 FUNC    GLOBAL DEFAULT   UND __cxa_finalize@LIBC
     3: 00000000     0 FUNC    GLOBAL DEFAULT   UND __register_atfork@LIBC
     4: 00000000     0 FUNC    GLOBAL DEFAULT   UND __stack_chk_fail@LIBC
     5: 00000000     0 FUNC    GLOBAL DEFAULT   UND access@LIBC
     6: 00000000     0 FUNC    GLOBAL DEFAULT   UND ptrace@LIBC
     7: 000005f0   139 FUNC    GLOBAL DEFAULT    13 Java_com_sosung_friatest_MainActivity_nativeIsDebugged
     8: 00000680   223 FUNC    GLOBAL DEFAULT    13 Java_com_sosung_friatest_MainActivity_nativeIsRooted
```

`extern "C"`로 선언했기 때문에 이름이 맹글링되지 않고, JNI 네이밍 규칙 그대로 심볼이 잡힌다. `readelf` 결과의 `UND` 항목(`access@LIBC`, `ptrace@LIBC`)을 보면, 이 so가 실제로 libc의 `access`/`ptrace`를 가져다 쓰고 있다는 것도 정적으로 미리 확인할 수 있다 — 소스 없이도 "이 함수가 대충 뭘 검사하는지" 추측 가능한 지점이다.

[![5](https://so-sung.github.io/assets/img/posts/2026-09-06/5.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/5.png)
*(캡처 5: `llvm-nm -D --defined-only`와 `llvm-readelf -sW ... findstr FUNC` 실행 결과 — `nativeIsDebugged`(0x5f0, 139바이트), `nativeIsRooted`(0x680, 223바이트) 심볼과 `access@LIBC`/`ptrace@LIBC` import가 보이는 터미널 화면)*

## 5. Frida로 네이티브 함수 후킹 — 우회 스크립트 작성

심볼 이름을 확보했으니, 이제 Java 레이어가 아니라 **모듈(so) 안의 함수 주소**를 직접 찾아서 `Interceptor.attach`로 리턴값을 조작한다.

처음에 짠 스크립트는 spawn 직후 곧바로 모듈을 찾으려다가 `Error: unable to find module 'libfriatest.so'` 에러를 만났다. 원인은 타이밍이다 — `libfriatest.so`는 프로세스가 뜬 직후가 아니라, **`MainActivity` 클래스가 실제로 로드되는 시점**(companion object의 `System.loadLibrary` 호출 시점)에야 메모리에 올라온다. 그런데 스크립트가 `setTimeout(fn, 0)`으로 "거의 즉시" 모듈을 찾으려 했으니 아직 로드 전이라 실패한 것이다.

그래서 **so가 로드될 때까지 재시도(폴링)** 하도록 고쳤다.

```javascript
// native_bypass.js
function waitForModuleAndHook() {
    const libName = "libfriatest.so";
    const lib = Process.findModuleByName(libName);

    if (lib === null) {
        // 아직 so가 로드되지 않음 -> 50ms 후 다시 시도
        setTimeout(waitForModuleAndHook, 50);
        return;
    }

    const targets = [
        "Java_com_sosung_friatest_MainActivity_nativeIsDebugged",
        "Java_com_sosung_friatest_MainActivity_nativeIsRooted"
    ];

    targets.forEach(function (symbol) {
        const addr = lib.getExportByName(symbol);
        if (addr) {
            Interceptor.attach(addr, {
                onLeave: function (retval) {
                    console.log("[*] " + symbol + " original return: " + retval);
                    retval.replace(0); // JNI_FALSE로 강제 조작 -> 탐지 무력화
                    console.log("[*] " + symbol + " forced return: 0 (bypassed)");
                }
            });
            console.log("[+] hooked: " + symbol + " @ " + addr);
        } else {
            console.log("[-] symbol not found: " + symbol);
        }
    });
}

waitForModuleAndHook();
```

`Java.perform` 대신 그냥 폴링 함수로 시작한 이유는, 이 스크립트가 자바 레이어를 전혀 건드리지 않고 **모듈 로드 이후 네이티브 심볼만** 다루기 때문이다. `retval.replace(0)`은 `jboolean`(사실상 `uint8_t`) 리턴값을 `JNI_FALSE`(0)로 강제 치환하는 부분으로, 오늘 우회의 핵심 라인이다.

## 6. 실행 — spawn으로 걸어서 실제로 뚫어보기

```bash
frida -Uf com.sosung.friatest -l native_bypass.js
```

실제로 실행한 환경은 Frida 16.3.3, 대상은 Nox Player가 물려있는 `SM G965N`(Nox가 갤럭시 S9로 디바이스 정보를 스푸핑한 것)이었다.

**느낀 점 — 첫 시도의 실수와 해결 과정**

맨 처음 짠 스크립트는 spawn하자마자(`setTimeout(fn, 0)`) 바로 `libfriatest.so` 모듈을 찾으려 했다. 실행해보니 아래처럼 곧바로 에러가 떴다.

```
C:\study\frida>frida -Uf com.sosung.friatest -l native_bypass.js
...
Connected to SM G965N (id=127.0.0.1:62001)
Spawned `com.sosung.friatest`. Resuming main thread!

Error: unable to find module 'libfriatest.so'
    at value (frida/runtime/core.js:370)
    at <anonymous> (C:\study\frida\native_bypass.js:4)
```

처음엔 "심볼 이름을 잘못 적었나?" 싶었는데, 원인은 그게 아니었다. **`libfriatest.so`는 프로세스가 뜨는 순간이 아니라, `MainActivity` 클래스가 실제로 로드되는 시점(companion object의 `System.loadLibrary` 호출 시점)에야 메모리에 올라온다.** spawn 직후와 그 시점 사이엔 시간차가 있는데, 스크립트는 그 틈을 기다리지 않고 "거의 즉시" 모듈을 찾으려다 실패한 것이다. 정적 분석으로 심볼까지 다 확인해놓고도 정작 타이밍 문제로 막힌 경험이라, "so 파일이 언제 메모리에 올라오는지"까지 신경 써야 한다는 걸 직접 겪고 나서야 체감했다.

해결은 5단계에서 소개한 대로, so가 로드될 때까지 **재시도(폴링)**하도록 스크립트를 고치는 것이었다. 고친 뒤 다시 실행하니 정상적으로 붙었다.

```
C:\study\frida>frida -Uf com.sosung.friatest -l native_bypass.js
     ____
    / _  |   Frida 16.3.3 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to SM G965N (id=127.0.0.1:62001)
Spawned `com.sosung.friatest`. Resuming main thread!
[SM G965N::com.sosung.friatest ]-> [+] hooked: Java_com_sosung_friatest_MainActivity_nativeIsDebugged @ 0xd1e1f5f0
[+] hooked: Java_com_sosung_friatest_MainActivity_nativeIsRooted @ 0xd1e1f680
[*] Java_com_sosung_friatest_MainActivity_nativeIsDebugged original return: 0x0
[*] Java_com_sosung_friatest_MainActivity_nativeIsDebugged forced return: 0 (bypassed)
[*] Java_com_sosung_friatest_MainActivity_nativeIsRooted original return: 0x1
[*] Java_com_sosung_friatest_MainActivity_nativeIsRooted forced return: 0 (bypassed)
```

[![6](https://so-sung.github.io/assets/img/posts/2026-09-06/6.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/6.png)
*(캡처 6: 수정된 native_bypass.js 실행 터미널 — `hooked: ...nativeIsDebugged @ 0xd1e1f5f0`, `hooked: ...nativeIsRooted @ 0xd1e1f680` 후킹 성공 로그와, 각각의 `original return` → `forced return: 0 (bypassed)` 조작 로그)*

두 함수 모두 정상적으로 후킹됐고, `nativeIsDebugged`의 원래 리턴값은 `0x0`(Frida 붙은 시점인데도 이번엔 디버깅 탐지가 안 걸렸다), `nativeIsRooted`의 원래 리턴값은 `0x1`(Nox Player가 루팅된 상태라 정상적으로 탐지됨)이었다. 두 값 모두 강제로 `0`으로 바꿔치기(bypassed)한 뒤, 앱에서 다시 `탐지 우회 체크` 버튼을 눌러봤다.

[![7](https://so-sung.github.io/assets/img/posts/2026-09-06/7.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/7.png)
*(캡처 7: 후킹된 상태에서 `탐지 우회 체크`를 눌렀을 때 `statusBox`가 초록색 "우회 성공\n(디버깅:false / 루팅:false)"으로 바뀐 화면 — 캡처 2의 빨간불 "우회 실패"와 대비됨)*

원래대로라면 `nativeIsRooted()`는 su 바이너리가 있으니 무조건 `true`가 나와야 하는데, 네이티브 함수의 리턴값 자체를 가로채서 `0`으로 바꿔치기했기 때문에 앱은 "탐지 안 됨"으로 오판하고 초록불을 띄웠다. Java 레이어에서 아무리 뒤져봐도 이 로직 자체가 안 보이는 이유가 바로 이거다 — 검증이 애초에 `.so` 안에서만 일어나기 때문이다.

## 마무리 — 오늘 정리한 것과 다음 편

- 실제 앱(특히 금융권)의 루팅/디버깅 탐지는 자바가 아니라 **네이티브(.so) 레이어**에 구현되는 경우가 많고, 이 경우 자바 레벨 후킹만으로는 우회가 안 된다.
- APK를 뜯을 때는 **jadx로 자바 시그니처 확인 → apktool로 so 파일 위치 파악 → llvm-nm/llvm-readelf로 심볼 확인**의 순서를 거치면, "어떤 함수를 후킹해야 하는지"가 정적으로 먼저 확정된다.
- Frida에서 네이티브 함수는 `Java.use` 대신 `Process.findModuleByName` + `Module.getExportByName` + `Interceptor.attach` 조합으로 후킹하고, `retval.replace()`로 리턴값을 조작한다.
- so가 프로세스 시작과 동시에 로드되지 않는다는 걸 에러로 직접 겪었다 — spawn 직후 바로 모듈을 찾으려 하지 말고, **모듈이 로드될 때까지 폴링**하는 방어적인 스크립트를 짜야 안전하다는 걸 몸으로 배웠다.

다음 편에서는 오늘 만든 탐지 로직에 간단한 무결성 체크(예: 자기 자신의 해시를 검증하는 로직)를 추가해보고, 그걸 Frida로 다시 우회하는 것까지 다뤄볼 예정이다.

---

# Frida 2 - Bypassing Native (.so) Layer Detection: Defeating Root & Debugger Checks in Practice

In the previous post ([frida-first-hooking](https://so-sung.github.io/posts/frida-first-hooking/)), I took my first step into hooking by swapping out the entire `implementation` of a single Java-layer function (`checkPassword`). But real-world apps — financial apps especially — aren't that forgiving. Most of them implement root and debugger detection not in Java, but in the **native (.so) layer**, and often wrap that library in an integrity check to make hooking itself harder.

From a tester's perspective, this whole procedure — tearing apart an APK to figure out what a `.so` file is actually doing, then hooking that function with Frida to bypass it — is something you have to go through in real assessments. So today I reproduced it myself: I added native root/debugger detection logic to the test app from before (`com.sosung.friatest`), analyzed it statically, and bypassed it with Frida.

## 0. What I set out to do today — why the native layer

Looking at commercial security SDKs (app-protection products), detection logic is almost never exposed plainly at the Java level. The typical structure looks like this:

- The Java code only has a **declaration**, like `native boolean isRooted()` — the real logic is compiled into a `.so`.
- The `.so` itself is validated against a separate signature/rule file and a hash, checking whether the file has been tampered with.
- Detection logic operates at the syscall/file level — things like `ptrace`, `access()`, or the `TracerPid` field in `/proc/self/status` — so hooking anything in the Java layer accomplishes nothing.

In other words, hooking Java functions alone can't bypass this kind of app — you have to **find and hook the actual function inside the `.so`**. Today's post is about building that exact scenario by hand and breaking through it.

## 1. Adding native detection logic to the test app

First, I needed something real to detect. I added a CMake-based native module under `app/src/main/cpp/`.

**CMakeLists.txt**

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("friatest")

add_library(friatest SHARED
    native-lib.cpp
)

find_library(log-lib log)

target_link_libraries(friatest ${log-lib})
```

**native-lib.cpp — the actual detection logic**

```cpp
#include <jni.h>
#include <sys/ptrace.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

// Debugger (and Frida-attach) detection: attempt PTRACE_TRACEME on itself.
// If another process (a debugger/Frida) is already attached, this call fails.
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_sosung_friatest_MainActivity_nativeIsDebugged(JNIEnv *env, jobject /* this */) {
    if (ptrace(PTRACE_TRACEME, 0, 1, 0) < 0) {
        return JNI_TRUE;
    }
    ptrace(PTRACE_DETACH, 0, 1, 0);
    return JNI_FALSE;
}

// Root detection: check common paths where an su binary would exist.
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_sosung_friatest_MainActivity_nativeIsRooted(JNIEnv *env, jobject /* this */) {
    const char *paths[] = {
        "/system/bin/su",
        "/system/xbin/su",
        "/sbin/su",
        "/system/app/Superuser.apk"
    };
    for (const char *path : paths) {
        if (access(path, F_OK) == 0) {
            return JNI_TRUE;
        }
    }
    return JNI_FALSE;
}
```

**MainActivity.kt — wiring it to the UI**

```kotlin
package com.sosung.friatest

import android.graphics.Color
import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    companion object {
        init {
            System.loadLibrary("friatest")
        }
    }

    external fun nativeIsDebugged(): Boolean
    external fun nativeIsRooted(): Boolean

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val statusBox = findViewById<TextView>(R.id.statusBox)
        val checkBtn = findViewById<Button>(R.id.checkBtn)

        checkBtn.setOnClickListener {
            val debugged = nativeIsDebugged()
            val rooted = nativeIsRooted()
            val detected = debugged || rooted

            if (detected) {
                statusBox.text = "Bypass failed\n(debug:$debugged / root:$rooted)"
                statusBox.setBackgroundColor(Color.parseColor("#D32F2F"))
            } else {
                statusBox.text = "Bypass succeeded\n(debug:$debugged / root:$rooted)"
                statusBox.setBackgroundColor(Color.parseColor("#388E3C"))
            }
        }
    }
}
```

Building this produces `libfriatest.so`. Since my test device is Nox Player, an x86 emulator, it's packaged into the APK as `lib/x86/libfriatest.so`.

[![1](https://so-sung.github.io/assets/img/posts/2026-09-06/1.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/1.png)
*(Capture 1: Android Studio showing the `native-lib.cpp` code alongside a `BUILD SUCCESSFUL` log — the baseline shot establishing today's "original" target)*

## 2. Confirming the baseline before hooking anything

Installed on Nox Player, then tapped `Check`. In practice, both came back `true` — debug:true / root:true — showing red, "bypass failed."

Worth noting: Nox Player is **rooted out of the box**, with an `su` binary already present. So `nativeIsRooted()` reading `true` here is expected, with no extra setup. The `debug:true` result may be down to an existing adb connection or leftover debug bridge already leaving a value in `/proc/self/status`'s `TracerPid`. On a plain AVD emulator, this baseline would more naturally read `false/false` (green); using a pre-rooted emulator like Nox instead makes for a good demonstration that the detection is actually working correctly from the start.

[![2](https://so-sung.github.io/assets/img/posts/2026-09-06/2.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/2.png)
*(Capture 2: Nox Player right after tapping `Check`, showing `statusBox` in red, "Bypass failed\n(debug:true / root:true)")*

## 3. From an analyst's perspective — reconstructing this by tearing apart the APK

I already know the structure since I wrote the code, but in a real assessment you'd have to figure this out from scratch. Here's the order.

### 3-1. Extract the APK

```bash
adb shell pm path com.sosung.friatest
adb pull /data/app/~~<random>==/com.sosung.friatest-<random>==/base.apk
```

### 3-2. Check the Java layer with jadx

```bash
jadx-gui base.apk
```

Opening `MainActivity` shows the native functions as bodiless signatures, exactly as below. jadx decompiles compiled dex into Java-style syntax rather than the original Kotlin, so this is what it actually looks like:

```java
public final class MainActivity extends AppCompatActivity {
    public final native boolean nativeIsDebugged();
    public final native boolean nativeIsRooted();

    static {
        System.loadLibrary("friatest");
    }

    public static final void onCreate$lambda$0(MainActivity this$0, TextView $statusBox, View it) {
        boolean debugged = this$0.nativeIsDebugged();
        boolean rooted = this$0.nativeIsRooted();
        boolean detected = debugged || rooted;
        if (detected) {
            $statusBox.setText("Bypass failed\n(debug:" + debugged + " / root:" + rooted + ")");
            $statusBox.setBackgroundColor(Color.parseColor("#D32F2F"));
        } else {
            $statusBox.setText("Bypass succeeded\n(debug:" + debugged + " / root:" + rooted + ")");
            $statusBox.setBackgroundColor(Color.parseColor("#388E3C"));
        }
    }
}
```

`public final native boolean nativeIsDebugged();`, `nativeIsRooted();` — **no body at all**. That's the definitive signal that this logic lives natively.

[![3](https://so-sung.github.io/assets/img/posts/2026-09-06/3.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/3.png)
*(Capture 3: jadx-gui with `MainActivity` open, showing both `native` function declarations (bodiless) alongside `onCreate$lambda$0`, where they're actually called)*

### 3-3. Check the structure with apktool

```bash
apktool d base.apk -o friatest_decoded
```

The decoded folder tree (only one architecture appears here, since the test environment is Nox Player = x86 only):

```
friatest_decoded/
|   AndroidManifest.xml
|   apktool.yml
|
+---lib
|   \---x86
|           libfriatest.so
|           liblog.so
|
+---original
|       AndroidManifest.xml
|
+---res
    +---anim
    |       abc_fade_in.xml
    |       abc_fade_out.xml
    |       ...
```

`liblog.so` shows up alongside it because `CMakeLists.txt` links against Android's log library via `find_library(log-lib log)`.

[![4](https://so-sung.github.io/assets/img/posts/2026-09-06/4.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/4.png)
*(Capture 4: The decoded folder tree from `apktool d`, showing `libfriatest.so` and `liblog.so` under `lib/x86/`)*

## 4. How do you actually look inside a .so? — checking symbols

This is the real answer to today's core question — "how do I even look at a `.so` file?" Since a `.so` is ultimately an ELF binary, the first step is checking what functions it exports.

**A Windows note**: unlike Linux, `nm`/`readelf` aren't installed by default — but Android Studio already ships `llvm-nm.exe` / `llvm-readelf.exe` as part of its NDK toolchain, and those work just as well. They're usually found here, so adding this path to your `PATH` environment variable lets you run `llvm-nm`, `llvm-readelf` from anywhere:

```
%LOCALAPPDATA%\Android\Sdk\ndk\<version>\toolchains\llvm\prebuilt\windows-x86_64\bin
```

With that set up, here's the actual output:

```
C:\study\frida\friatest_decoded\lib\x86>llvm-nm -D --defined-only libfriatest.so
000005f0 T Java_com_sosung_friatest_MainActivity_nativeIsDebugged
00000680 T Java_com_sosung_friatest_MainActivity_nativeIsRooted

C:\study\frida\friatest_decoded\lib\x86>llvm-readelf -sW libfriatest.so | findstr FUNC
     1: 00000000     0 FUNC    GLOBAL DEFAULT   UND __cxa_atexit@LIBC
     2: 00000000     0 FUNC    GLOBAL DEFAULT   UND __cxa_finalize@LIBC
     3: 00000000     0 FUNC    GLOBAL DEFAULT   UND __register_atfork@LIBC
     4: 00000000     0 FUNC    GLOBAL DEFAULT   UND __stack_chk_fail@LIBC
     5: 00000000     0 FUNC    GLOBAL DEFAULT   UND access@LIBC
     6: 00000000     0 FUNC    GLOBAL DEFAULT   UND ptrace@LIBC
     7: 000005f0   139 FUNC    GLOBAL DEFAULT    13 Java_com_sosung_friatest_MainActivity_nativeIsDebugged
     8: 00000680   223 FUNC    GLOBAL DEFAULT    13 Java_com_sosung_friatest_MainActivity_nativeIsRooted
```

Since these were declared `extern "C"`, the names aren't mangled — they appear exactly as the JNI naming convention dictates. Looking at the `UND` entries in the `readelf` output (`access@LIBC`, `ptrace@LIBC`), you can already tell, statically, that this `.so` is pulling in libc's `access`/`ptrace` — enough to guess roughly what it's checking, even without the source.

[![5](https://so-sung.github.io/assets/img/posts/2026-09-06/5.png)](https://so-sung.github.io/assets/img/posts/2026-09-06/5.png)
*(Capture 5: Terminal output of `llvm-nm -D --defined-only` and `llvm-readelf -sW ... findstr FUNC`, showing the `nativeIsDebugged` (0x5f0, 139 bytes) and `nativeIsRooted` (0x680, 223 bytes) symbols along with the `access@LIBC`/`ptrace@LIBC` imports)*

## 5. Hooking the native function with Frida — writing the bypass script

Now that I have the symbol names, instead of the Java layer I go straight for the **function address inside the module (`.so`)** and manipulate its return value with `Interceptor.attach`.

My first version tried to find the module immediately after spawn, and hit `Error: unable to find module 'libfriatest.so'`. The cause was timing — `libfriatest.so` doesn't load the instant the process starts; it only loads when **the `MainActivity` class is actually loaded** (when the companion object's `System.loadLibrary` call runs). But the script tried to find the module "almost immediately" via `setTimeout(fn, 0)`, so it ran before the library had loaded.

So I rewrote it to **poll and retry until the `.so` loads**.

```javascript
// native_bypass.js
function waitForModuleAndHook() {
    const libName = "libfriatest.so";
    const lib = Process.findModuleByName(libName);

    if (lib === null) {
        // Not loaded yet -> retry in 50ms
        setTimeout(waitForModuleAndHook, 50);
        return;
    }

    const targets = [
        "Java_com_sosung_friatest_MainActivity_nativeIsDebugged",
        "Java_com_sosung_friatest_MainActivity_nativeIsRooted"
    ];

    targets.forEach(function (symbol) {
        const addr = lib.getExportByName(symbol);
        if (addr) {
            Interceptor.attach(addr, {
                onLeave: function (retval) {
                    console.log("[*] " + symbol + " original return: " + retval);
                    retval.replace(0); // Force JNI_FALSE -> defeat the check
                    console.log("[*] " + symbol + " forced return: 0 (bypassed)");
                }
            });
            console.log("[+] hooked: " + symbol + " @ " + addr);
        } else {
            console.log("[-] symbol not found: " + symbol);
        }
    });
}

waitForModuleAndHook();
```

I started with a plain polling function instead of `Java.perform`, since this script never touches the Java layer at all — it only deals with native symbols after the module loads. `retval.replace(0)` is the line that forcibly rewrites the `jboolean` (effectively a `uint8_t`) return value to `JNI_FALSE` — the crux of today's bypass.

## 6. Running it — spawning the app and actually breaking through

```bash
frida -Uf com.sosung.friatest -l native_bypass.js
```

The actual environment was Frida 16.3.3, targeting `SM G965N` — Nox Player spoofing itself as a Galaxy S9.

```
C:\study\frida>frida -Uf com.sosung.friatest -l native_bypass.js
     ____
    / _  |   Frida 16.3.3 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to SM G965N (id=127.0.0.1:62001)
Spawned `com.sosung.friatest`. Resuming main thread!
```

With the original (buggy) script, this is exactly where the error showed up:

```
Error: unable to find module 'libfriatest.so'
    at value (frida/runtime/core.js:370)
    at <anonymous> (C:\study\frida\native_bypass.js:4)
```

With the polling version instead, `[+] hooked: Java_..._nativeIsDebugged @ 0x...` and `[+] hooked: Java_..._nativeIsRooted @ 0x...` should appear once the `.so` loads. Tapping `Check` from there should print `original return: 1` followed by `forced return: 0 (bypassed)` in the console, and the screen should flip to green, "bypass succeeded."

> **📷 Capture 6 (to be recaptured)**: Terminal running the fixed `native_bypass.js` — the `hooked: ...` logs, plus `original return: 1` → `forced return: 0 (bypassed)` after tapping the button.

> **📷 Capture 7 (to be recaptured)**: Nox Player with the hook active, after tapping `Check` again — `statusBox` now shows green, "Bypass succeeded." (Worth placing side-by-side with the red screenshot from Capture 2 for contrast.)

Normally, the instant Frida attaches to the process, `ptrace(PTRACE_TRACEME)` should fail and the app should always read "detected (true)." But because the native function's return value itself was intercepted and rewritten to `0`, the app is fooled into believing "not detected," and shows green. This is exactly why digging through the Java layer never reveals this logic — the check happens entirely inside the `.so`.

## Wrap-up — what today covered, and what's next

- Root/debugger detection in real apps — financial apps especially — is often implemented at the **native (.so) layer**, not in Java, and Java-level hooking alone can't bypass it in that case.
- When tearing apart an APK, the sequence — **jadx for the Java signatures → apktool to locate the `.so` files → llvm-nm/llvm-readelf to confirm symbols** — lets you pin down exactly which function to hook, statically, before you ever touch Frida.
- For native functions, Frida hooks with `Process.findModuleByName` + `Module.getExportByName` + `Interceptor.attach` instead of `Java.use`, and manipulates the return value with `retval.replace()`.
- I ran directly into the fact that a `.so` doesn't necessarily load the instant a process starts — don't try to grab the module right after spawn; write a defensive script that **polls until the module loads**.

Next time, I'll add a simple integrity check (something that validates its own hash) to today's detection logic, and cover bypassing that with Frida as well.
