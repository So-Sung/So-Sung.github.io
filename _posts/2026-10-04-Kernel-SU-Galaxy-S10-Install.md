---
title: "Kernel-Su Porting Log 2 - Galaxy S10(Exynos 9820) 루팅, 그리고 Frida가 막히는 지점을 직접 찾아본 기록"
date: 2026-10-04 00:00:00 +0900
categories: [Mobile Hacking, Android]
tags: [android, kernelsu, susfs, exynos9820, galaxy-s10, odin, twrp, rooting, frida, anti-frida, research-note]
toc: true
---

지난 [S10e 포팅 로그](https://so-sung.github.io/posts/Kernel-SU-Galaxy-S10E-Install/)에서 Galaxy S10e(beyond0lteks)에 KernelSU-Next + SuSFS를 심는 과정을 정리했다. 이번 주는 같은 작업을 서랍 속 Galaxy S10(beyond1, 동일 Exynos 9820 계열)에 그대로 적용해봤다. 전체 흐름은 지난주와 거의 동일하지만, 기기가 다르다 보니 지난주엔 안 겪었던 실수를 두 번 했다. 하나는 벽돌까지 갈 뻔한 실수였고, 하나는 "코드네임만 비슷해서 헷갈린" 가벼운 실수였다. 그리고 루팅 자체는 끝냈는데, 정작 그 다음 단계인 **Frida 연결**에서 막혀서, 이번 글 후반부는 그 원인을 추적한 기록으로 채웠다.

## 0. 오늘 하려는 것

- **대상 기기**: Samsung Galaxy S10 (보드명 beyond1lte / beyond1lteks)
- **참고한 지난주 기기**: Galaxy S10e (beyond0lteks) — [지난 글](https://so-sung.github.io/posts/Kernel-SU-Galaxy-S10E-Install/) 참고
- **목표**: 지난주와 동일하게 KernelSU-Next + SuSFS가 통합된 커널을 S10에 플래싱해서 루팅된 실기기를 확보하는 것
- **이번 주에 새로 생긴 목표**: 루팅은 됐는데 Frida가 제대로 안 붙어서, 왜 안 붙는지 원인을 정적으로 조사해보는 것

기기가 바뀌면 똑같은 절차도 미묘하게 달라질 수 있다는 걸 이번 주에 제대로 체감했다.

## 1. S10e 때와 뭐가 똑같고 뭐가 다른가

먼저 지난주 절차를 그대로 재사용할 수 있는 부분과, 기기가 달라서 손봐야 하는 부분을 구분해뒀다.

**그대로 재사용 가능했던 것**
- TWRP + vbmeta를 콤바인 tar로 묶어서 Odin에 올리는 방식
- `Wipe → Format Data`로 암호화 해제 후 `Reboot → Recovery`로 MTP 재인식시키는 순서
- KernelSU-Next 매니저 앱에서 `Superuser` 탭에 들어가 `ADB Shell`에 권한을 직접 허용해줘야 하는 것 (su 바이너리가 없는 KernelSU 구조이므로)

**기기 차이로 손봐야 했던 것**
- 커널 통합 이미지: S10e는 `beyond0lteks`용 `GoRhanHee_Kernel_for_beyond0lteks_ksun.zip`을 썼지만, S10은 당연히 `beyond1lteks`용 이미지를 따로 받아야 했다.
- TWRP 이미지: 이 부분에서 첫 번째 삽질이 나왔다(아래 2번 참고).
- vbmeta 이미지: 이 부분에서 진짜 벽돌 직전까지 갔다(아래 3번 참고).

## 2. 삽질 1 — "lteKS"라는 이름 때문에 헷갈린 TWRP 코드네임

[GoRhanHee/android_kernel_samsung_exynos9820](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) 릴리스 페이지에서 커널 zip 파일명이 `beyond1lteks`로 돼 있는 걸 보고, 처음엔 TWRP도 당연히 "lteks" 코드네임으로 찾아야 하는 줄 알았다. [twrp.me](https://dl.twrp.me/)에서 `beyond1lteks`를 검색해봤는데 원하는 결과가 바로 나오지 않아서 한참 헤맸다.

다시 찾아보니, 커널 zip의 `ks`는 **KernelSU를 가리키는 접미사**였고, TWRP 쪽 기기 코드네임과는 관계가 없었다. 즉 S10의 TWRP는 그냥 베이스 코드네임인 **`beyond1lte`** 로 찾으면 되는 거였다 — S10e 때 `beyond0lte`로 TWRP를 받았던 것과 완전히 같은 패턴이다. `ks`가 붙은 이름은 어디까지나 "KernelSU가 통합된 커널 빌드"라는 뜻이었을 뿐, 리커버리 이미지를 찾을 때는 무시해도 되는 부분이었다.

```
커널 zip 파일명:  beyond1lteks  (ks = KernelSU 통합 빌드라는 표시)
TWRP 검색용 코드네임:  beyond1lte   (ks 떼고 검색)
```

별거 아닌 혼동인데, 이름 하나 잘못 읽은 채로 몇 번을 더 검색하다가 시간을 꽤 썼다. 지난주엔 애초에 S10e 코드네임을 한 번에 맞게 찾았어서 이런 문제 자체가 없었는데, 이번엔 커널 zip 이름에 낚여서 괜히 돌아갔다.

> 참고: [GoRhanHee/android_kernel_samsung_exynos9820 v3.2.0 — GitHub Releases](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) · [TWRP for beyond1lte — twrp.me](https://dl.twrp.me/beyond1lte/)

## 3. 삽질 2 — vbmeta가 삼성 공통인 줄 알았는데, 기기 전용이었다

지난주 S10e 작업에서는 `vbmeta_disabled.img`를 TWRP와 콤바인해서 별 문제 없이 넘어갔었다. 그래서 막연히 "vbmeta는 삼성 기기면 어느 정도 공통으로 쓰이는 파일이겠지"라고 생각하고 있었는데, 이번에 찾아보니 그게 아니었다. **vbmeta도 커널 zip이나 TWRP처럼 기기 코드네임마다 정확히 맞는 걸 따로 구해야 하는 파일이었다** — 같은 Exynos 9820, 같은 beyond 시리즈라고 해서 S10e용과 S10용이 서로 호환되는 게 아니었던 것.

이걸 모른 채로 지난주에 썼던 vbmeta를 그대로 재사용해서 TWRP와 콤바인 플래싱을 했다. 그런데 이번엔 증상이 지난주와 달랐다. Odin 로그는 중간에 끊기지 않고 **초록색 `PASS!`까지 정상적으로 떴다.** "어, 이번엔 문제없이 넘어가나" 싶어서 안심하고 재부팅했는데, 그대로 **무한 재부팅 루프**에 빠져버렸다. 지난주처럼 `RQT_CLOSE !!`로 Odin 단계에서부터 거부당한 게 아니라, **Odin은 PASS를 찍었지만 실제 부팅 검증(AVB) 단계에서 걸려서 계속 재부팅만 반복하는 패턴**이었다. 플래싱 자체는 "성공"으로 떴는데 결과물은 벽돌이라, 오히려 지난주보다 더 헷갈렸다 — 로그만 보면 뭐가 잘못됐는지 바로 알 수가 없었다.

복구 과정도 지난주보다 손이 더 많이 갔다. 이번엔 Frija와 Bifrost 둘 다 시도해봤는데, 둘 다 모델명/CSC만 넣는 일반적인 방식으로는 바로 순정 펌웨어를 받아오질 못했다. 결국 방법을 바꿔서, **벽돌이 난 폰의 TWRP(다행히 리커버리 자체는 살아있었다)로 들어가서 `adb shell`로 붙은 다음, `getprop`으로 기기 고유 식별값을 직접 뽑아내는 쪽으로 돌아갔다.** 정확히 어떤 프로퍼티였는지는 기억이 가물가물한데(`ro.serialno` 계열의 시리얼 넘버였던 것 같고, IMEI 쪽 값도 같이 확인했던 것 같다), 아무튼 이 값을 Frija/Bifrost 쪽에 직접 넣어주고 나서야 겨우 이 기기에 정확히 맞는 순정 롬을 찾아서 받을 수 있었다. 모델명·CSC 조합 검색만으로는 안 되고, 기기 자체에서 뽑아낸 식별값이 있어야 넘어가는 케이스가 있다는 걸 이번에 처음 알았다.

> 참고: [Bifrost — zwander.dev](https://bifrost.zwander.dev/) · [TWRP for beyond1lte — twrp.me](https://dl.twrp.me/beyond1lte/)

## 4. 이후 절차는 지난주와 동일

OEM 언락 → TWRP/vbmeta 콤바인 재플래싱 → TWRP 강제 진입 → Format Data → 커널 zip 플래싱 → KernelSU-Next 매니저 설치 → Superuser 탭에서 ADB Shell 권한 허용, 이 순서는 지난주 글과 완전히 동일해서 반복해서 적지는 않는다. 다만 이번엔 `beyond1lteks` 커널 zip과 `beyond1lte` TWRP라는, S10에 맞는 올바른 파일로 진행했고, 최종적으로 `adb shell` → `su`에서 프롬프트가 `beyond1:/ #`로 바뀌는 걸 확인했다. 루팅 자체는 성공.

## 5. 그런데 Frida가 안 붙는다

루팅이 끝났으니 지난주 작업([SO 무결성 실험](https://so-sung.github.io/posts/zygote-2/))을 이 기기에서 이어가려고 Frida부터 붙여봤는데, 여기서 막혔다. 증상을 정리하면 이렇다.

1. KernelSU-Next 모듈 형태로 frida-server를 올려서 부팅 시 자동 기동시키는 방식까지는 됐다.
2. 그런데 **WebView가 자꾸 깨진다.** 앱 내 웹뷰 컴포넌트가 로드되다가 크래시하거나, 렌더링 자체가 멈춰버리는 증상이 반복됐다. 정확한 원인은 아직 특정 못 했는데, 커널 레벨 모듈로 주입되는 frida-server와 WebView 프로세스(크로미움 기반) 사이에서 뭔가 충돌이 나는 것 같다는 심증만 있다.
3. WebView 문제를 어찌어찌 피해서 Frida가 붙는 상황을 만들어도, **이 기기가 Android 12 이하**라서 그런지 보안 솔루션이 들어간 앱에서는 거의 바로 탐지됐다. GKI 2.0 기준(커널 5.10+, Android 12+)보다 아래 환경이다 보니, 최신 안드로이드 버전을 전제로 한 은폐 기법들이 제대로 안 먹히는 건지도 모르겠다.

지난주 글 8번에서 `Unmount modules by default` + ZygiskNext + Shamiko 조합을 측정해보겠다고 남겨뒀었는데, 이걸 켜봐도 체감상 탐지까지 걸리는 시간이 크게 늘지 않았다. 그래서 이번엔 "어떤 설정을 켜고 끌 것인가" 수준이 아니라, **Frida 자체가 정확히 어느 지점에서 꼬리를 밟히는지**부터 다시 짚어봐야겠다고 생각했다.

## 6. Frida가 어디서 탐지되는가 — 알려진 탐지 포인트 정리

다음 편에서 직접 테스트해볼 걸 전제로, 이번 주는 일단 "보안 솔루션들이 보통 Frida를 어디서 잡아내는가"를 공부 삼아 정리해봤다. 상용 RASP SDK들과 공개된 탐지 프로젝트들(OWASP MASTG, DetectFrida, EnvScope 등)을 찾아보니, 크게 아래 범주로 나뉘는 것 같다.

### 6.1 프로세스/파일 이름 기반 탐지

가장 기본적인 방식. `/proc/*/cmdline`이나 `ps` 결과를 훑어서 `frida-server`, `frida-helper`, `frida-agent` 같은 문자열이 그대로 박혀 있는 프로세스나 파일을 찾는다. `/data/local/tmp/frida-server`, `/system/bin/frida-server`처럼 흔히 올려두는 경로도 같이 스캔 대상이 된다.

### 6.2 네트워크 포트 기반 탐지

frida-server는 기본적으로 D-Bus 프로토콜 기반으로 **27042번 포트**(일부 구현은 27043도 같이)를 열고 대기한다. 보안 솔루션은 `127.0.0.1:27042`로 직접 `connect()`를 시도해보거나, `/proc/net/tcp` / `/proc/net/tcp6`를 읽어서 해당 포트가 열려 있는지 확인한다. 단순히 포트가 열려있다는 것만으로는 다른 정상 프로세스일 수도 있으니, 한 단계 더 나아가 그 포트에 **D-Bus AUTH 메시지**를 직접 보내서 `REJECT` 응답이 오는지까지 확인하는 방식도 흔하다고 한다 — Frida가 D-Bus 프로토콜을 쓰는데, 이게 안드로이드 환경에서는 흔치 않은 프로토콜이라 오히려 이게 더 확실한 시그니처가 되는 셈이다.

### 6.3 스레드 이름 기반 탐지

Frida가 attach되면 프로세스 안에 `gmain`, `gdbus`, `gum-js-loop`, `pool-frida` 같은 이름의 스레드가 새로 생긴다. `/proc/self/task/*/comm`을 순회하면서 이런 이름이 있는지 확인하는 방식인데, 함수 하나하나를 후킹하는 것보다 훨씬 포괄적으로 걸린다는 게 특징인 것 같다.

### 6.4 D-Bus 서비스/심볼 이름 기반 탐지

포트 스캔보다 더 깊이 들어가면, D-Bus 서비스 식별자 자체가 `re.frida.server`처럼 고정된 이름으로 노출돼 있어서 D-Bus introspection으로 바로 잡힌다는 내용도 봤다. 또 `dlsym`이나 메모리 문자열 스캔으로 `frida_agent_main` 같은 내부 심볼 이름, `frida-agent-64.so` / `frida-agent-32.so`처럼 `/proc/self/maps`에 그대로 찍히는 모듈 이름도 전형적인 탐지 포인트라고 한다.

### 6.5 메모리/매핑 비교 기반 탐지 (서명 문자열에 의존하지 않는 방식)

여기가 좀 더 까다로운 부분인 것 같다. Frida가 어떤 함수를 인라인 후킹하면, 디스크에 있는 원본 `.so`의 `.text` 섹션과 실제로 메모리에 로드된 `.text` 섹션 사이에 차이가 생긴다. 이걸 그대로 바이트 단위로 비교(memdisk comparison)하는 방식은 "frida"라는 이름 자체에 의존하지 않기 때문에, 단순히 바이너리 이름이나 포트를 바꾸는 식의 회피로는 안 통한다고 알려져 있다. `.text` 세그먼트의 메모리 보호 속성(`r-xp` 같은 권한)이 비정상적으로 바뀌어 있는지 확인하는 것도 같은 계열이다.

### 6.6 종합

정리하면 Frida 탐지는 대략 다섯 레이어로 쌓여 있는 셈이다.

| 레이어 | 확인 대상 | 이름/포트 변경으로 회피 가능한가 |
|---|---|---|
| 프로세스/파일명 | `frida-server`, `frida-agent` 등 문자열 | 가능 |
| 네트워크 포트 | 27042/27043, D-Bus AUTH 응답 | 가능(포트 변경 시) |
| 스레드 이름 | `gmain`, `gum-js-loop`, `pool-frida` 등 | 가능 |
| 심볼/D-Bus 서비스명 | `frida_agent_main`, `re.frida.server` 등 | 가능 |
| 메모리 바이트 비교 | `.text` 섹션 디스크 vs 메모리, 권한 비트 | **불가능(이름과 무관)** |

맨 아래 메모리 비교 레이어를 빼면, 나머지 네 레이어는 전부 "frida"라는 **이름**(프로세스명, 파일 경로, 포트, 스레드명, 심볼명, D-Bus 서비스명)에 의존하고 있다는 공통점이 있다. 다르게 말하면, 이 네 레이어는 이론적으로는 **이름을 통째로 바꿔서 빌드**하면 우회가 될 수 있다는 뜻이다. 실제로 이런 접근으로 소스에서부터 `frida` 문자열 자체를 치환해서 빌드하는 프로젝트들(anti-detection frida-server 빌드)이 이미 있다는 것도 확인했다.

## 7. 다음 편 계획 — 이름을 바꾼 frida-server를 직접 빌드해보기

그래서 다음 주는, 지금 막힌 지점을 뚫어보기 위해 **Frida 소스를 직접 받아서 이름을 바꾼 버전으로 빌드**해보려고 한다. 지금까지 조사한 걸 토대로, 바꿔야 할 지점을 아래처럼 잡아뒀다.

1. **프로세스/바이너리 이름**: `frida-server` → 임의의 다른 실행 파일명으로 변경해서 빌드. `/proc/*/cmdline` 스캔 자체를 무력화.
2. **리스닝 포트**: 기본 27042 대신 임의 포트로 변경. 표준 frida 클라이언트는 `adb forward`로 포트를 맞춰주면 접속 자체에는 문제가 없다고 하니, 이 부분은 빌드 옵션 레벨에서 처리 가능할 것 같다.
3. **D-Bus 서비스 식별자 / 내부 심볼명**: `re.frida.server`, `frida_agent_main` 같은, 고정 문자열로 박혀있는 서비스명·심볼명을 변경. 다만 D-Bus 프로토콜 인터페이스 자체(`re.frida.HostSession` 등)는 표준 클라이언트와의 호환성 때문에 건드리면 안 될 것 같다 — 여기까지 바꾸면 내 PC의 일반 `frida` 클라이언트가 아예 못 붙을 수 있다.
4. **스레드 이름**: `gmain`, `gum-js-loop`, `pool-frida` 같은 하드코딩된 스레드 이름도 가능하면 변경. 다만 이건 Frida 코어(GLib/GDBus 쪽) 내부에 꽤 깊이 박혀있는 이름들이라, 소스 레벨에서 어디까지 안전하게 건드릴 수 있을지는 직접 빌드해보면서 확인해야 할 것 같다.
5. **임시 경로/바이너리 잔여 문자열**: 설치 과정에서 `.frida`, `frida-` 같은 접두사가 붙는 임시 경로나, 빌드 결과물 안에 남아있는 `frida` 문자열 잔여물도 별도로 스윕해서 지워야 할 것 같다. 바이너리 안에 이름을 다 바꿨는데 문자열 레벨에서 "frida"가 그대로 남아있으면 의미가 없으니, 빌드 후에 strings로 한 번 더 훑어보는 검증 단계를 넣어야겠다.

이렇게 다섯 군데를 손본 빌드를 하나 만들고, 이걸 KernelSU 모듈로 다시 패키징해서 이 S10에 올려본 다음, 지금 막혔던 WebView 깨짐 문제가 이 과정에서 같이 해결되는지도 지켜볼 생각이다(사실 이건 frida-server 이름 문제와는 별개의 원인일 가능성이 높다고 보고 있어서, 이쪽은 따로 원인을 더 파봐야 할 것 같다). 그리고 6번에서 정리한 표의 마지막 줄 — **메모리 바이트 비교 기반 탐지**는 이름을 아무리 바꿔도 못 피한다는 걸 알고 들어가는 거라, 이름 변경 빌드로 어디까지 탐지를 피할 수 있고 어디서부터는 다시 막히는지를 직접 측정해서 다음 글에 정리해볼 예정이다.

## 8. 아직 확실치 않은 부분

- Odin이 `PASS!`까지 찍었는데도 실제로는 부팅 검증에서 걸려 무한 재부팅에 빠진 게 정확히 vbmeta 서명 불일치 때문인지, 다른 파티션(예: TWRP 쪽 호환성)까지 같이 얽힌 문제인지 명확히 분리하지 못했다.
- 복구 과정에서 TWRP의 `getprop`으로 뽑아낸 값이 정확히 어떤 프로퍼티였는지 기억이 분명하지 않다 — 시리얼 넘버였는지 IMEI 계열 값이었는지 다음에 다시 겪으면 정확히 기록해둬야 할 것 같다.
- WebView가 깨지는 게 정확히 frida-server 모듈 때문인지, KernelSU 자체 설정 때문인지, 아니면 완전히 별개의 원인인지 아직 못 좁혔다.
- Android 12 이하 환경이라서 탐지가 더 쉬운 건지, 아니면 그냥 이 기기/솔루션 조합의 문제인지도 아직 분리해서 확인하지 못했다.
- 6번에 정리한 다섯 가지 탐지 포인트가 실제로 내가 테스트하려는 보안 솔루션에도 전부 해당하는지는 직접 붙여보기 전까지는 추정일 뿐이다.
- D-Bus 프로토콜 인터페이스까지 건드리지 않고 서비스 식별자만 바꾸는 선에서 표준 frida 클라이언트와 호환이 유지되는지는 아직 직접 빌드해서 확인한 게 아니다.

## 마무리 — 오늘 정리한 것과 다음 편

- Galaxy S10(beyond1lteks)에 KernelSU-Next + SuSFS 심는 과정은 큰 틀에서 지난주 S10e와 같았지만, 커널 zip 이름의 "ks" 접미사에 낚여 TWRP 코드네임을 잘못 찾아본 것, 그리고 vbmeta가 삼성 공통 파일인 줄 알고 지난주 것을 재사용했다가 Odin은 `PASS!`를 찍고도 무한 재부팅에 빠지는 벽돌을 또 낸 것, 이렇게 두 번의 실수가 있었다. 복구는 Frija/Bifrost만으로는 안 돼서, 벽돌 상태에서도 살아있던 TWRP로 들어가 `adb shell` + `getprop`으로 기기 식별값을 직접 뽑아낸 뒤에야 순정 롬을 받을 수 있었다.
- 루팅 자체는 성공했지만, 그 다음 단계인 Frida 연결에서 WebView 크래시와 빠른 탐지라는 새 문제를 만났다.
- 이번 주는 일단 "Frida가 보통 어디서 탐지되는가"를 다섯 레이어(프로세스/파일명, 포트, 스레드명, 심볼/D-Bus 서비스명, 메모리 바이트 비교)로 정리했고, 이 중 메모리 비교를 제외한 네 레이어는 이름을 바꾸는 빌드로 이론상 우회 가능하다는 걸 확인했다.
- 다음 편은 이 다섯 지점을 실제로 건드린 커스텀 frida-server를 직접 빌드해서, 이 S10 위에서 얼마나 버티는지 측정해보는 걸로 이어갈 예정이다.

---
---

# Kernel-Su Porting Log 2 - Rooting a Galaxy S10(Exynos 9820), and Tracking Down Exactly Where Frida Gets Caught

In the [last S10e porting log](https://so-sung.github.io/posts/Kernel-SU-Galaxy-S10E-Install/), I wrote up the process of getting KernelSU-Next + SuSFS onto a Galaxy S10e (beyond0lteks). This week I applied the same work to a Galaxy S10 (beyond1, same Exynos 9820 family) that had been sitting in a drawer. The overall flow was nearly identical to last week's, but since it's a different device, I ran into two mistakes I didn't have last time — one nearly bricked the phone, and one was a lighter "the codenames just looked similar" mix-up. Rooting itself finished fine, but the very next step — **getting Frida to actually connect** — is where things got stuck, so the back half of this post is a record of chasing down why.

## 0. What I set out to do today

- **Target device**: Samsung Galaxy S10 (board name beyond1lte / beyond1lteks)
- **Last week's reference device**: Galaxy S10e (beyond0lteks) — see the [previous post](https://so-sung.github.io/posts/Kernel-SU-Galaxy-S10E-Install/)
- **Goal**: same as last week — flash a kernel with KernelSU-Next + SuSFS integrated onto the S10 to get a rooted real device
- **A new goal that came up this week**: rooting succeeded, but Frida won't connect properly, so dig into why — statically, for now

I genuinely felt this week that even "the same procedure" can shift in subtle ways once the device changes.

## 1. What carried over from the S10e, and what didn't

I separated out what I could reuse directly from last week's procedure versus what needed adjusting because the device was different.

**Carried over as-is**
- Combining TWRP + vbmeta into a single tar and flashing it via Odin
- The `Wipe → Format Data` → `Reboot → Recovery` sequence to clear encryption and re-detect MTP
- Having to manually grant `ADB Shell` permission under the `Superuser` tab in the KernelSU-Next manager app (since KernelSU has no `su` binary structurally)

**Had to be adjusted for the device difference**
- The integrated kernel image: S10e used `GoRhanHee_Kernel_for_beyond0lteks_ksun.zip` for `beyond0lteks`, but the S10 obviously needed its own separate `beyond1lteks` image.
- The TWRP image: this is where the first stumble happened (see section 2).
- The vbmeta image: this is where things nearly bricked for real (see section 3).

## 2. Stumble 1 — confused about TWRP's codename because of the "lteKS" suffix

Seeing the kernel zip's filename as `beyond1lteks` on the [GoRhanHee/android_kernel_samsung_exynos9820](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) release page, I naturally assumed TWRP would also need to be looked up under the "lteks" codename. Searching `beyond1lteks` on [twrp.me](https://dl.twrp.me/) didn't turn up what I wanted right away, and I spent a while stuck on it.

Digging further, it turned out the `ks` in the kernel zip name is just a **suffix indicating KernelSU**, and has nothing to do with TWRP's device codename. The S10's TWRP just needed to be found under the base codename, **`beyond1lte`** — exactly the same pattern as grabbing TWRP under `beyond0lte` for the S10e last week. The `ks` suffix only ever meant "this is a kernel build with KernelSU integrated" — something to ignore entirely when looking for the recovery image.

```
Kernel zip filename:    beyond1lteks  (ks = marker for a KernelSU-integrated build)
TWRP search codename:   beyond1lte    (search with ks dropped)
```

A trivial mix-up, but misreading one name cost a fair bit of time re-searching. Last week I nailed the S10e codename on the first try, so this problem didn't even come up — this time I got baited by the kernel zip's filename and went in a circle for nothing.

> Reference: [GoRhanHee/android_kernel_samsung_exynos9820 v3.2.0 — GitHub Releases](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) · [TWRP for beyond1lte — twrp.me](https://dl.twrp.me/beyond1lte/)

## 3. Stumble 2 — I thought vbmeta was shared across Samsung devices; it was device-specific

Last week's S10e work combined `vbmeta_disabled.img` with TWRP without any real trouble. That left me with a vague assumption that vbmeta was probably common across Samsung devices to some extent — but digging into it this time, that wasn't the case at all. **vbmeta, just like the kernel zip or TWRP, turns out to be a file you need to source separately, exactly matched to the device codename** — the same Exynos 9820, the same beyond series, doesn't mean the S10e's and the S10's are interchangeable.

Not knowing that, I reused last week's vbmeta as-is and flashed it combined with TWRP. But this time the symptom was different from last week. The Odin log didn't cut off partway through — it ran all the way to a clean green **`PASS!`**. Relieved, thinking "oh, no issues this time," I rebooted — and dropped straight into an **endless reboot loop**. Unlike last week, where it got rejected right at the Odin stage with `RQT_CLOSE !!`, this time **Odin reported PASS, but the actual boot verification (AVB) stage was what failed, so it just kept rebooting over and over.** The flash itself said "success" while the result was still a brick — honestly more confusing than last week, since the log alone gave no immediate clue what had gone wrong.

Recovery also took more effort than last week. I tried both Frija and Bifrost this time, and neither worked with the usual approach of just entering model name/CSC — neither pulled the correct stock firmware right away. I ended up switching tactics: **I booted into the bricked phone's TWRP (thankfully the recovery itself was still alive), connected with `adb shell`, and pulled a device-identifying value directly via `getprop`.** I don't remember exactly which property it was (I think it was a serial-number-type value, something in the `ro.serialno` family, and I think I also cross-checked something IMEI-related), but either way, only after feeding that value directly into Frija/Bifrost was I finally able to find and download the stock ROM that actually matched this specific device. This was the first time I learned there are cases where a model-name-and-CSC search alone isn't enough — you need an identifier pulled from the device itself to get past it.

> Reference: [Bifrost — zwander.dev](https://bifrost.zwander.dev/) · [TWRP for beyond1lte — twrp.me](https://dl.twrp.me/beyond1lte/)

## 4. Everything after this matched last week

OEM unlock → combined TWRP/vbmeta reflash → forcing entry into TWRP → Format Data → flashing the kernel zip → installing the KernelSU-Next manager → granting ADB Shell permission under the Superuser tab — this sequence matched last week's post exactly, so I won't repeat it. The only difference was using the correct files for the S10 this time — the `beyond1lteks` kernel zip and `beyond1lte` TWRP — and I confirmed the prompt in `adb shell` → `su` flipped to `beyond1:/ #`. Rooting itself: success.

## 5. But Frida won't connect

With rooting done, I went to pick up last week's work ([the SO-integrity experiments](https://so-sung.github.io/posts/zygote-2/)) on this device, starting by attaching Frida — and got stuck right there. Here's a summary of the symptoms:

1. Getting frida-server loaded as a KernelSU module and auto-starting at boot worked fine.
2. But **WebView keeps breaking.** The in-app WebView component kept either crashing mid-load or just freezing its rendering outright. I haven't pinned down the exact cause yet — just a hunch that something's conflicting between the kernel-level injected frida-server and the WebView process (Chromium-based).
3. Even when I managed to work around the WebView issue and get Frida attached, it got detected almost immediately in apps with a security solution — possibly because **this device is Android 12 or below.** Sitting below the GKI 2.0 baseline (kernel 5.10+, Android 12+), I wonder if the concealment techniques that assume a more recent Android version just don't work properly here.

I'd left a note in section 8 of last week's post about measuring the `Unmount modules by default` + ZygiskNext + Shamiko combination — turning that on didn't meaningfully extend the time before detection kicked in, as far as I could tell. So this time I figured I needed to step back from "which settings to toggle" and instead pin down, from scratch, **exactly where Frida is leaving a trail in the first place.**

## 6. Where Frida actually gets detected — a rundown of known detection points

With next week's hands-on testing in mind, this week I just studied up — pulling together "where security solutions typically catch Frida." Looking through commercial RASP SDKs and public detection projects (OWASP MASTG, DetectFrida, EnvScope, and others), it seems to break down roughly into the following categories.

### 6.1 Process/file-name-based detection

The most basic method. Scanning `/proc/*/cmdline` or `ps` output for processes or files with strings like `frida-server`, `frida-helper`, `frida-agent` baked right in. Common install paths like `/data/local/tmp/frida-server` or `/system/bin/frida-server` get scanned too.

### 6.2 Network-port-based detection

By default, frida-server listens on **port 27042** (some implementations also cover 27043) over the D-Bus protocol. Security solutions either attempt a direct `connect()` to `127.0.0.1:27042`, or read `/proc/net/tcp` / `/proc/net/tcp6` to check whether that port is open. Since an open port alone could just be some other legitimate process, it's apparently common to go one step further and send a **D-Bus AUTH message** directly to that port to check for a `REJECT` response — since Frida speaks the D-Bus protocol, which is unusual enough in an Android context that this ends up being a fairly reliable signature on its own.

### 6.3 Thread-name-based detection

Once Frida attaches, new threads named things like `gmain`, `gdbus`, `gum-js-loop`, `pool-frida` appear inside the process. Walking `/proc/self/task/*/comm` to check for these names seems to catch things far more broadly than hooking individual functions one at a time.

### 6.4 D-Bus service / symbol-name-based detection

Going a level deeper than port scanning, I found that the D-Bus service identifier itself is exposed under a fixed name like `re.frida.server`, catchable directly via D-Bus introspection. There's also `dlsym`-based or memory-string-scan detection for internal symbol names like `frida_agent_main`, and module names like `frida-agent-64.so` / `frida-agent-32.so` that show up plainly in `/proc/self/maps` — all apparently typical detection points.

### 6.5 Memory/mapping-comparison-based detection (the kind that doesn't rely on a signature string)

This looks like the trickier part. When Frida inline-hooks a function, a difference appears between the `.text` section of the original `.so` on disk and the `.text` section actually loaded into memory. A byte-level comparison of the two (memdisk comparison) doesn't depend on the name "frida" at all, so it's known not to be defeated by simply renaming the binary or changing the port. Checking whether the `.text` segment's memory-protection attributes (permission bits like `r-xp`) have been abnormally altered falls into the same family.

### 6.6 Putting it together

In short, Frida detection seems to stack up across roughly five layers.

| Layer | What's checked | Evadable by renaming/changing ports? |
|---|---|---|
| Process/file name | Strings like `frida-server`, `frida-agent` | Yes |
| Network port | 27042/27043, D-Bus AUTH response | Yes (with a port change) |
| Thread name | `gmain`, `gum-js-loop`, `pool-frida`, etc. | Yes |
| Symbol/D-Bus service name | `frida_agent_main`, `re.frida.server`, etc. | Yes |
| Memory byte comparison | `.text` section disk vs. memory, permission bits | **No (independent of names)** |

Setting aside the bottom memory-comparison layer, the remaining four layers all have one thing in common: they rely on the **name** "frida" in some form — process name, file path, port, thread name, symbol name, D-Bus service name. Put differently, those four layers could, in theory, be evaded by **building with the name swapped out entirely**. I also confirmed this approach already exists in practice — there are projects (anti-detection frida-server builds) that replace the `frida` string itself starting from source.

## 7. Plan for next time — building a renamed frida-server myself

So next week, to try to break through where I'm currently stuck, I'm planning to **grab the Frida source directly and build a version with the name changed**. Based on what I've looked into so far, here's where I've marked as needing changes.

1. **Process/binary name**: build with `frida-server` renamed to some arbitrary executable name. Neutralizes `/proc/*/cmdline` scanning outright.
2. **Listening port**: swap the default 27042 for an arbitrary port. Apparently the standard frida client has no issue connecting as long as the port is matched via `adb forward`, so this should be handleable at the build-option level.
3. **D-Bus service identifier / internal symbol names**: change hardcoded strings like `re.frida.server`, `frida_agent_main`. The D-Bus protocol interface itself (`re.frida.HostSession`, etc.), though, probably has to stay untouched for compatibility with the standard client — going that far risks my own PC's regular `frida` client failing to connect at all.
4. **Thread names**: change hardcoded thread names like `gmain`, `gum-js-loop`, `pool-frida` where possible. These are apparently buried fairly deep inside Frida's core (the GLib/GDBus side), though, so I'll need to actually attempt the build to see how far I can safely go at the source level.
5. **Temp paths / leftover strings in the binary**: temp paths carrying prefixes like `.frida`, `frida-` during installation, and any leftover `frida` string residue inside the build output, probably need a separate sweep to clean up. Renaming everything inside the binary means nothing if the string "frida" is still sitting there at the string level, so I'll add a verification pass post-build — running `strings` over it once more to check.

Once I have a build with these five spots addressed, I plan to repackage it as a KernelSU module, load it onto this S10, and also watch whether the WebView-breaking issue I'm currently stuck on happens to resolve along the way (though I suspect that's actually unrelated to the frida-server naming issue, so that one probably needs its own separate investigation). And going in with the understanding — per the last row of the table in section 6 — that **memory-byte-comparison-based detection** can't be evaded no matter how much renaming I do, I plan to actually measure how far a renamed build can get before detection resumes, and write that up next time.

## 8. What's still unclear

- Whether Odin reporting `PASS!` while still falling into an endless reboot loop at actual boot verification was precisely a vbmeta signature mismatch, or something tangled up with another partition (TWRP compatibility, for instance) as well — I haven't clearly separated the two.
- I'm not entirely sure exactly which property I pulled via `getprop` in TWRP during recovery — whether it was a serial number or something IMEI-related — something worth recording precisely if I run into this again.
- Whether WebView breaking is specifically caused by the frida-server module, by a KernelSU setting itself, or by something else entirely hasn't been narrowed down yet.
- Whether detection is easier specifically because this is an Android 12-or-below environment, or just a quirk of this particular device/solution combination, also hasn't been separated out and confirmed.
- Whether the five detection points laid out in section 6 actually all apply to the security solution I'm planning to test against is just an assumption until I actually attach to it.
- Whether changing only the service identifier — without touching the D-Bus protocol interface itself — keeps compatibility with the standard frida client hasn't been confirmed by actually building it yet.

## Wrap-up — what today covered, and what's next

- Getting KernelSU-Next + SuSFS onto the Galaxy S10 (beyond1lteks) matched last week's S10e process in broad strokes, but came with two mistakes: getting baited by the kernel zip name's "ks" suffix into looking up the wrong TWRP codename, and assuming vbmeta was a Samsung-wide shared file, reusing last week's, and bricking the phone again — this time with Odin reporting `PASS!` while still falling into an endless reboot. Recovery needed more than just Frija/Bifrost — only after booting into the still-alive TWRP and pulling a device identifier via `adb shell` + `getprop` was I able to get the stock ROM.
- Rooting itself succeeded, but the next step — getting Frida connected — turned up new problems: WebView crashes and fast detection.
- This week I focused on cataloging "where Frida typically gets detected" across five layers (process/file name, port, thread name, symbol/D-Bus service name, memory byte comparison), and confirmed that, aside from memory comparison, the other four layers are theoretically evadable through a renamed build.
- Next time, I plan to pick this up by actually building a custom frida-server that addresses these five points, and measuring how long it holds up on this S10.
