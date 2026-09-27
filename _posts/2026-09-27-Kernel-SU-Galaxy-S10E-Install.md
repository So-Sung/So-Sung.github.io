---
title: "커널 수 설치- Galaxy S10e(Exynos 9820)에 KernelSU-Next + SuSFS 심어보기: 빌드 실패에서 벽돌, 복구까지"
date: 2026-09-27 00:00:00 +0900
categories: [Mobile Hacking, Android]
tags: [android, kernelsu, susfs, exynos9820, galaxy-s10e, odin, twrp, rooting, kernel-build, research-note]
toc: true
---

지난 [Zygote 2](https://so-sung.github.io/posts/zygote-2/) 글 7번에서 "커널 소스 확보 가능성 조사"를 다음 과제로 남겨뒀었다. 원래 계획대로면 이번 주는 4.1~4.4에서 세운 SO 무결성 실험을 시작해야 했는데, 막상 시작하려고 노트북을 열어놓고 앉으니 뭔가 앞뒤가 안 맞았다. 그 실험들은 전부 "안정적으로 루팅된 실기기가 이미 손에 있다"는 걸 전제로 깔고 있는데, 정작 나한테는 그런 기기가 없었다. `friatest`야 에뮬레이터에서 돌리면 그만이지만, SO 무결성 얘기는 결국 링커·서명 검증·부트 체인까지 실기기 레벨에서 확인해야 의미가 있는 얘기라서, 에뮬레이터로 어설프게 흉내 내고 넘어가면 나중에 "이거 실기기에서도 똑같이 되는지 확인 안 했잖아"라는 말을 스스로에게 하게 될 게 뻔했다.

그래서 이번 주는 SO 얘기를 한 주 미뤄두고, 서랍에 방치돼 있던 Galaxy S10e(Exynos 9820)를 꺼내서 KernelSU-Next + SuSFS를 직접 심는 과정 자체를 파보기로 했다. 결론부터 미리 말해두면, 소스를 직접 빌드해서 넣겠다는 원래 계획은 실패했고, 그 실패를 정리하는 과정에서 기기를 한 번 벽돌 냈다가 복구했고, 최종적으로는 계획과는 다른 경로로 루팅에 성공했다. 이번 글은 성공담이라기보다는 "계획이 두 번 틀어진 기록"에 가깝다. 그래도 실패한 경로까지 그대로 남겨야 나중에 똑같은 삽질을 반복하지 않을 것 같아서, 시간 순서 그대로 정리한다.

## 0. 오늘 하려는 것

- **대상 기기**: Samsung Galaxy S10e (SM-G970N / 보드명 beyond0lteks)
- **타겟 SoC**: Samsung Exynos 9820 (arm64-v8a)
- **작업 환경**: WSL2 Ubuntu 22.04 LTS (x86_64), Windows 11 호스트
- **사용 도구**: Samsung Odin v3.14.4, TWRP 3.7.0_9-2, Bifrost, (시도 후 실패한) Frija
- **최종 목표**: KernelSU-Next + SuSFS 서브시스템이 통합된 커널을 올려서, ADB 디버깅·루트 감지를 우회할 수 있는 "안정적인 루팅 실기기"를 확보하는 것

한 가지 미리 짚어두고 싶은 건, 오늘 하는 작업이 지난 Zygote 2 글의 5번 섹션("커널수를 Android 9에 맞게 직접 빌드해볼 수 있을까")에서 세워둔 세 가지 전제조건 — 기기와 정확히 일치하는 커널 소스, 보안 패치 레벨 일치, 원본 boot.img 백업 — 을 실제로 갖추고 시작하는 첫 시도라는 점이다. 다만 대상 기기는 그 글에서 다루던 Android 9 기기가 아니라 S10e(One UI 4.1 기준)라서, 커널 버전 자체는 KernelSU가 "호환 가능"이라고 명시한 범위 안에 이미 들어와 있다는 차이가 있다. 그러니까 이번엔 "지원 범위 밖이라 안 되는" 문제가 아니라, "지원은 되는데 빌드 과정 자체가 험난한" 문제라는 걸 미리 알고 들어간 셈이다.

## 1. WSL2에서 소스 직접 빌드 시도

시작은 [KernelKreation/exynos9820_SUSFS_kernel](https://github.com/KernelKreation/exynos9820_SUSFS_kernel) 저장소였다. README에 "STOCK Galaxy S10 and N10 Series OneUI4.1"이라고 명시돼 있어서, 내 S10e에 정확히 맞는 타겟이라는 확신을 갖고 시작했다.

### 1.1 초기 환경 구축

`git clone`으로 저장소를 받아오는 것 자체는 금방 끝났는데, 저장소 안에 14GB가 넘는 크로스 컴파일 툴체인 바이너리가 Git LFS로 관리되고 있어서 이걸 받아오는 데만 상당한 시간이 걸렸다. WSL2 네트워크 속도가 딱히 느린 건 아닌데도, `git lfs pull`이 대용량 바이너리를 하나씩 순차적으로 받아오는 방식이라 체감상 꽤 오래 걸렸다. 커피 한 잔 타는 동안 진행률 바만 쳐다보고 있었던 기억이 난다.

다음으로 신경 쓴 건 텍스트 인코딩이었다. 저장소가 원래 윈도우 환경에서 관리된 흔적이 있었는지, `Kconfig`나 일부 `Makefile`에 DOS/Windows식 줄바꿈(CRLF)이 섞여 있었다. 이걸 그대로 두면 빌드 스크립트가 파싱하다가 이상한 지점에서 죽을 수 있다는 걸 이전에 다른 프로젝트에서 겪어본 적이 있어서, 미리 저장소 전체에 `dos2unix`를 일괄 적용해뒀다. 여기까지는 순조로웠다.

### 1.2 이슈 1 — GCC 툴체인 심볼릭 링크 단선

준비를 마치고 툴체인 바이너리부터 검증해보려고 아래 명령을 쳤다.

```bash
./toolchain/aarch64-linux-android-4.9/bin/aarch64-linux-android-gcc --version
```

결과는 `No such file or directory`. 처음엔 그냥 경로를 잘못 친 줄 알고 몇 번을 다시 확인했는데, `file` 명령으로 찍어보니 해당 경로가 심볼릭 링크이긴 한데 대상 파일이 존재하지 않는 깨진 링크(broken symlink)였다. 저장소 구조를 좀 더 파보니 실제 GCC 바이너리는 `toolchain/gcc-cfp/gcc-cfp-jopp-only/...`라는 훨씬 깊은 서브디렉터리에 있었고, 루트의 심볼릭 링크는 이 경로를 상대경로로 가리키도록 만들어져 있었는데 그 상대경로 계산이 어긋나 있었다. 아마 저장소를 만든 사람의 로컬 환경과 실제로 클론해서 쓰는 환경의 디렉터리 깊이가 미묘하게 달랐던 게 아닐까 싶다.

조치는 단순했다. 깨진 링크를 지우고 실제 경로를 기준으로 다시 잡아줬다.

```bash
cd ~/exynos9820_SUSFS_kernel/toolchain
rm -f aarch64-linux-android-4.9 arm-linux-androideabi-4.9
ln -s gcc-cfp/gcc-cfp-jopp-only/aarch64-linux-android-4.9 aarch64-linux-android-4.9
ln -s gcc-cfp/gcc-cfp-jopp-only/arm-linux-androideabi-4.9 arm-linux-androideabi-4.9
```

다시 버전 확인을 해보니 `gcc (GCC) 4.9.x`가 정상적으로 출력됐다. 64비트용, 32비트용 둘 다 확인. 별거 아닌 거 같은데 여기서만 거의 한 시간을 썼다.

### 1.3 이슈 2 — prepare-compiler-check 및 Stack Protector 검사 실패

링크를 고치고 `build_kernel.sh`를 실행했더니 이번엔 다른 지점에서 멈췄다.

```
Cannot use CONFIG_CC_STACKPROTECTOR_STRONG: -fstack-protector-strong not supported by compiler
make: *** [Makefile:1250: prepare-compiler-check] 오류 1
```

원인을 찾으려고 `build_kernel.sh` 내부를 열어봤다. 스크립트가 `make`를 호출하는 부분에서 `ARCH`, `CROSS_COMPILE`, `CC` 같은 변수들을 명시적으로 인자에 넘기지 않고 그냥 단독으로 `make`를 부르고 있었다. 이렇게 되면 Makefile의 `prepare-compiler-check` 타겟이 `-fstack-protector-strong` 옵션을 검사할 때, 타겟 아키텍처(`--target=aarch64-linux-gnu`)나 크로스 어셈블러 경로를 전달받지 못한 채로 호스트(x86_64) 컴파일 환경 기준으로 검사를 돌려버린다. 당연히 이 옵션 지원 여부 판단 자체가 틀어지고, 오진 상태로 빌드가 중단되는 것이었다.

부수적으로 이 시점에 WSL2 환경이 순수 64비트로만 구성돼 있어서, 구형 GCC 툴체인 내부의 32비트 실행 파일이나 그게 의존하는 `libc6:i386` 같은 32비트 호환 라이브러리가 아예 없는 상태라는 것도 함께 확인했다. 이게 당장 이 에러의 직접 원인은 아니었지만, 이후 단계에서 분명히 발목을 잡을 거라는 예감이 들어서 메모만 해뒀다.

### 1.4 이슈 3 — External Assembler의 LSE 옵션 거부

`build_kernel.sh` 대신 환경변수를 직접 셸에 export한 뒤 `make`를 수동으로 호출하는 식으로 우회해서 다시 돌렸다. `prepare-compiler-check`는 통과했는데, 이번엔 C 컴파일 단계인 `scripts/mod/empty.o` 직전에서 또 걸렸다.

```
clang-6.0: warning: argument unused during compilation: '-Xassembler -march=armv8-a+lse'
Assembler messages:
Fatal error: invalid -march= option: `armv8-a+lse'
clang-6.0: error: assembler command failed with exit code 1
```

`arch/arm64/Makefile`을 열어보니 `-Wa,-march=armv8-a+lse` 플래그가 어셈블러로 넘어가도록 돼 있었다. Clang 6.0은 어셈블리 처리를 자체적으로 하지 않고 외부 어셈블러(external assembler)를 호출하는 구조인데, 이 프로젝트에서는 그 외부 어셈블러가 GCC 4.9의 `as`였다. 문제는 이 GCC 4.9 시절의 `as`가 ARMv8.1부터 추가된 LSE(Large System Extensions) 명령어 확장 구문 자체를 모른다는 것이었다. `+lse`라는 문자열을 아예 이해하지 못하니 옵션 파싱 단계에서 그대로 에러를 내뱉은 것.

당장 떠오른 우회책은 소스에서 `+lse`를 그냥 지워버리는 것이었는데, 이건 좀 위험한 선택 같았다. 커널 소스 안에는 LSE 원자적 명령어를 전제로 한 인라인 어셈블리 코드가 이미 들어가 있을 가능성이 높고, 만약 그렇다면 어셈블러 플래그만 지운다고 해결되는 게 아니라 컴파일 과정 어딘가에서 "이 명령어를 모른다"는 형태로 다시 터질 게 뻔했다. 소스를 임의로 건드리는 대신, Clang과 GCC 어셈블러 사이의 `CROSS_COMPILE` / `PATH` 매핑을 좀 더 정확하게 잡아주는 방향으로 접근해야 한다는 결론을 내렸다.

이 세 개의 이슈를 하나씩 잡아나가면서 든 생각은, 오픈소스 커널 저장소 하나를 받아서 빌드 스크립트를 한 번 돌리는 것도 이 정도로 손이 많이 가는구나 싶었다는 것이다. 지난 글에 "커널 소스를 직접 빌드해서 심어야 한다"고 이론으로 적어 넣을 때랑, 실제로 이 세 개의 에러를 하나씩 붙잡고 앉아 있는 건 정말 다른 얘기였다.

> 참고: [KernelKreation/exynos9820_SUSFS_kernel — GitHub](https://github.com/KernelKreation/exynos9820_SUSFS_kernel) · [KernelSU-Next — GitHub](https://github.com/KernelSU-Next/KernelSU-Next)

## 2. 방향 전환 — 소스 빌드를 접다

세 번째 이슈까지 원인 분석을 마친 시점에서 잠깐 노트북 화면에서 손을 떼고 생각을 정리했다. 솔직히 말하면 처음엔 "소스를 처음부터 끝까지 직접 빌드해야 진짜 연구라고 부를 수 있다"는 고집이 있었다. 근데 곰곰이 다시 보니, 이번 실험의 진짜 목적은 "커널을 내가 직접 컴파일하는 경험 자체"가 아니라 "SO 무결성 실험을 돌릴 수 있는, 안정적으로 루팅된 실기기를 확보하는 것"이었다. 목적과 수단이 살짝 뒤바뀌어 있었던 셈이다.

그러던 중 같은 기기(beyond0lteks)를 대상으로 이미 빌드와 테스트까지 끝난 통합 이미지가 공개돼 있다는 걸 알게 됐다. [GoRhanHee/android_kernel_samsung_exynos9820](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) 릴리스 페이지에 KernelSU-Next v3.2.0과 SuSFS가 통합된 `GoRhanHee_Kernel_for_beyond0lteks_ksun.zip` 패키지가 올라와 있었고, 릴리스 노트에 어떤 커밋 기준으로 빌드됐는지, 어떤 SuSFS 버전을 물고 있는지까지 비교적 상세하게 적혀 있어서 "이건 최소한 한 번은 검증된 결과물이구나"라는 판단이 섰다.

인라인 어셈블리 불일치 문제를 계속 손으로 고쳐가면서 혼자 빌드를 완주하는 리스크와, 이미 검증된 패키지를 TWRP로 플래싱하는 상대적 안정성을 비교했을 때, 이번엔 완벽주의를 좀 내려놓고 후자로 방향을 틀기로 했다. 목표를 다시 명확히 정의하니 판단도 훨씬 쉬워진다는 걸 새삼 느꼈다. 다만 마음 한구석엔 "이건 숙제를 남이 대신 풀어준 느낌"이라는 찜찜함도 살짝 남아서, 소스 빌드 쪽은 나중에 GCC 툴체인을 최신 버전으로 교체하거나 서브모듈 분리형 저장소로 다시 시도해볼 여지를 남겨두기로 했다.

## 3. 첫 Odin 플래싱 — RQT_CLOSE와 함께 온 벽돌

방향을 정했으니 이제 순서는 간단해 보였다. TWRP를 올리고, `vbmeta` 검증을 끄고, 커널 zip을 플래싱하면 끝이라고 생각했다. 실제로는 전혀 간단하지 않았다.

우선 [TWRP 공식 다운로드 페이지](https://dl.twrp.me/beyond0lte/twrp-3.7.0_9-2-beyond0lte.img.tar.html)에서 `beyond0lte` 코드네임에 맞는 `twrp-3.7.0_9-2-beyond0lte.img.tar`를 받았다. 여기서 `beyond0lte`가 정확히 S10e에 해당하는 코드네임이 맞는지 한 번 더 확인했는데(S10은 `beyond1`, S10+는 `beyond2` 계열이라 헷갈리기 쉬웠다), 다행히 제대로 찾았다. 여기에 `vbmeta_disabled.img`를 함께 준비해서 Odin으로 플래싱을 시도했다.

로그 창에는 이렇게 찍혔다.

```
SetupConnection..
Get PIT for mapping..
recovery.img
vbmeta.img
RQT_CLOSE !!
```

그리고 `Complete(Write) operation failed`. 화면은 그대로 무한 재부팅 루프에 들어갔다. 원인을 찾아보니, 순정 상태로 복귀하거나 파티션을 건드리는 과정에서 삼성의 보안 솔루션인 VaultKeeper(정확히는 KG State라는 커널 게이트 상태값)가 활성화되면서 서명되지 않은 커스텀 바이너리를 강제로 차단한 것이었다. RQT_CLOSE는 딱 봐도 "요청을 강제로 닫았다"는 의미라서, 이게 단순 통신 오류가 아니라 의도적인 차단이라는 게 명확하게 느껴졌다.

솔직히 이 화면을 보고 있을 때 등 뒤로 식은땀이 좀 났다. "이거 진짜 벽돌인가" 하는 생각이 스쳤다. 다행히 볼륨 버튼 조합으로 다운로드 모드 자체는 여전히 진입이 됐고, 이게 완전한 하드 브릭은 아니라는 첫 번째 안도감이었다.

복구는 펌웨어 다운로드 툴을 이용해 SM-G970N용 최신 순정 4종 바이너리(AP/BL/CP/CSC)를 통째로 받아서 다시 플래싱하는 방식으로 진행했다. 이 과정에서 도구 선택에 약간의 시행착오가 있었다.

### 3.1 Frija 시도, 실패

처음엔 [Frija](https://github.com/SlackingVeteran/frija/releases)를 먼저 써봤다. 예전에 다른 삼성 기기에서 순정 펌웨어를 받을 때 써본 적이 있어서 익숙한 도구였는데, 이번엔 모델명(SM-G970N)과 CSC 코드를 넣고 펌웨어 조회를 돌렸더니 다운로드 서버 쪽에서 응답이 제대로 오지 않는 증상이 계속 반복됐다. 몇 번 재시도하고 CSC 코드를 다른 조합으로도 바꿔봤지만 똑같았다. 로그를 봐도 정확히 어느 단계에서 막히는지 명확하게 잡히지 않아서, 시간을 더 쓰기보다는 다른 도구로 넘어가는 쪽을 택했다. (이 부분은 왜 실패했는지 아직 정확히 특정하지 못했고, 나중에 다시 붙잡을 생각이다.)

### 3.2 Bifrost로 전환, 순정 복구 성공

대신 [Bifrost](https://bifrost.zwander.dev/)를 써봤다. 웹 기반 도구라 별도 설치 없이 바로 모델명/CSC로 조회가 됐고, SM-G970N 최신 순정 바이너리 4종을 문제없이 받을 수 있었다. 받은 파일을 단말을 다운로드 모드로 진입시킨 뒤 Odin v3.14.4의 각 슬롯(AP/BL/CP/CSC)에 정확히 할당해서 플래싱했다.

결과는 성공이었다. 무한 재부팅 상태를 벗어나 삼성 순정 시스템으로 완전히 복구됐다. 이 과정에서 배운 건 명확했다 — **삼성 기기에서 부트/리커버리 파티션을 건드리기 전에는, "혹시 벽돌이 나면 순정 4종 세트로 되돌릴 수 있는가"를 먼저 확인하고, 그 순정 세트를 안정적으로 받을 수 있는 도구를 최소 두 개는 미리 확보해두고 시작해야 한다는 것.** Frija가 막혔을 때 Bifrost라는 대안이 없었다면 이 시점에서 완전히 멈췄을 것이다.

> 참고: [TWRP for beyond0lte — twrp.me](https://dl.twrp.me/beyond0lte/twrp-3.7.0_9-2-beyond0lte.img.tar.html) · [SlackingVeteran/frija — GitHub](https://github.com/SlackingVeteran/frija/releases) · [Bifrost — zwander.dev](https://bifrost.zwander.dev/)

## 4. 부트로더 완전 언락 — OEM LOCK 스위치의 함정

순정 복구 후 다운로드 모드에 다시 들어가 보니 상단에 `OEM LOCK: ON(U)`이라는 문구가 떠 있었다. 개발자 옵션에서 "OEM 잠금 해제" 스위치는 분명히 켜놨었는데도 이 상태였다. 처음엔 이게 왜 이러나 싶었는데, 찾아보니 개발자 옵션의 스위치는 어디까지나 "나중에 부트로더를 언락하는 걸 허용하겠다"는 사전 동의일 뿐이고, 실제 부트로더 언락은 다운로드 모드 안에서 별도의 물리적 승인 절차를 거쳐야 최종적으로 완료된다는 걸 알게 됐다.

이 절차를 진행하면서 삼성 Exynos 계열 기기의 부트로더 언락 방식이 기종마다 미묘하게 다르다는 게 걱정돼서, 같은 Exynos 계열인 Note 10 5G(SM-N976N)를 다루는 [XDA의 부트로더 언락/루팅 가이드](https://xdaforums.com/t/guide-bootloader-unlock-and-root-note-10-5g-sm-n976n-exynos-version.4439103/)를 교차 참고했다. 기기는 다르지만 같은 시기 같은 계열 Exynos 모델이라 다운로드 모드 진입 방식, 버튼 조합 타이밍, "Unlock bootloader?" 확인 화면의 UI 흐름까지 상당히 비슷했고, 이 가이드 덕분에 내가 밟고 있는 절차가 이 기기군 전체에서 공통적으로 쓰이는 정상적인 흐름이라는 확신을 가지고 진행할 수 있었다.

실제로 밟은 절차는 이랬다.

1. 단말 전원을 완전히 끈다.
2. 음량 낮춤 버튼 + 빅스비 버튼을 동시에 누른 채로 PC와 USB 케이블을 연결한다.
3. 민트색 배경의 Warning(경고) 화면에 진입한다.
4. 화면 안내에 따라 음량 높임 버튼을 3초 이상 길게 눌러 "Device unlock mode" 메뉴로 들어간다.
5. "Unlock bootloader?"라는 확인 화면이 뜨면, 음량 높임 버튼을 짧게 한 번 눌러 Yes를 선택한다.
6. 자동으로 완전 공장초기화가 진행되고, 그대로 재부팅된다.
7. 재부팅 후 다시 다운로드 모드로 진입해서 좌측 상단 문구가 `OEM LOCK: OFF`로 바뀐 걸 확인한다.

이 시점에 내부 저장소 데이터는 전부 날아갔다. 애초에 새 기기로 가정하고 시작한 작업이라 데이터 손실 자체는 예상했던 부분이지만, 그래도 공장초기화가 자동으로 진행되는 순간 화면을 보면서 "이거 되돌릴 방법이 있나" 하는 생각이 잠깐 스치긴 했다. 다행히 정상적으로 재부팅됐다.

## 5. TWRP + vbmeta 콤바인 패키징과 강제 진입

`OEM LOCK: OFF`를 확인한 뒤엔 TWRP 이미지와 `vbmeta_disabled.img`를 하나로 묶어서 다시 플래싱을 시도했다.

### 5.1 콤바인 tar 만들기

TWRP 단독 이미지만 플래싱하면 AVB(Android Verified Boot) 검증에 걸려서 부팅 자체가 막힐 가능성이 있다는 걸 미리 알고 있었기 때문에, `twrp-3.7.0_9-2-beyond0lte.img`와 `vbmeta_disabled.img`를 7-Zip으로 묶어서 `twrp_vbmeta_combined.tar`라는 단일 아카이브로 만들었다. 두 파일을 굳이 하나로 묶는 이유는, Odin이 tar 파일 내부의 파일명을 보고 각 파티션(recovery, vbmeta)에 자동으로 매핑해서 순서대로 써주는 방식으로 동작하기 때문에, 별도로 두 번 플래싱하는 것보다 한 번에 같이 밀어 넣는 게 더 안전하다고 판단해서였다.

### 5.2 Odin 플래싱

다운로드 모드(`OEM LOCK: OFF` 상태) 진입 후 Odin의 AP 슬롯에 콤바인 tar 파일을 넣고, `Auto Reboot` 옵션을 체크 해제한 상태로 `Start`를 눌렀다. 이번엔 로그 창에 `RQT_CLOSE`가 뜨지 않았고, 초록색 `PASS!`로 정상 종료됐다. 지난번 벽돌 사태를 겪은 직후라 이 초록색 글자 하나가 이렇게 반가울 수가 없었다.

### 5.3 TWRP 강제 진입 — 타이밍이 생각보다 까다로웠다

`Auto Reboot`을 꺼뒀기 때문에, PASS 이후에도 기기는 자동으로 재부팅되지 않고 대기 상태로 남아 있었다. 여기서 그냥 아무 버튼이나 눌러 재부팅하면 스톡 리커버리나 일반 부팅으로 들어가 버릴 수 있어서, TWRP로 강제 진입하는 별도의 버튼 조합이 필요했다.

- 케이블 연결을 유지한 상태에서 음량 낮춤 + 전원 버튼을 동시에 7초간 누른다.
- 화면이 완전히 꺼지는 순간, 즉시 손을 떼고 곧바로 음량 높임 + 빅스비 + 전원 버튼을 동시에 꾹 누른다.

이 타이밍이 생각보다 예민했다. 화면이 꺼지고 손을 떼는 타이밍이 0.5초만 늦어도 그냥 일반 부팅으로 넘어가 버렸다. 처음 두 번은 실패해서 일반 안드로이드 부팅 화면(삼성 로고)까지 봤다가 다시 전원을 눌러 재시도해야 했다. 세 번째 시도에서야 TWRP 로고가 뜨는 걸 확인했다.

### 5.4 데이터 포맷과 파티션 재인식

TWRP에 정상 진입한 뒤엔 `Wipe → Format Data` 메뉴로 들어가서 `yes`를 입력해 내부 저장소의 암호화를 해제하고 파일시스템을 초기화했다. 이 단계를 건너뛰면 이후 MTP로 파일을 전송할 때 저장소가 암호화된 상태 그대로 잡혀서 파일 복사 자체가 막히거나 이상하게 동작할 수 있다는 걸 이전 경험으로 알고 있었기 때문에, 이번엔 처음부터 순서에 넣어뒀다.

포맷이 끝난 뒤 TWRP 메인 메뉴에서 `Reboot → Recovery`를 눌러서 리커버리 자체를 한 번 더 재시작시켰다. 이렇게 하면 MTP 파티션이 새로 인식되면서, PC에서 내부 저장소로 파일을 정상적으로 전송할 수 있는 상태가 된다.

> 참고: [TWRP for beyond0lte — twrp.me](https://dl.twrp.me/beyond0lte/twrp-3.7.0_9-2-beyond0lte.img.tar.html) · [Guide: Bootloader Unlock and Root, Note 10 5G SM-N976N Exynos — XDA Forums](https://xdaforums.com/t/guide-bootloader-unlock-and-root-note-10-5g-sm-n976n-exynos-version.4439103/)

## 6. 커널 플래싱과 루트 권한 획득

TWRP가 MTP 모드로 정상 인식된 상태에서, PC에서 내부 저장소 최상위 경로로 두 파일을 복사했다.

- `GoRhanHee_Kernel_for_beyond0lteks_ksun.zip` — [GoRhanHee/android_kernel_samsung_exynos9820 v3.2.0 릴리스](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0)에서 받은 커널 패키지
- `KernelSU_Next_v3.2.0_33129-release.apk` — 대응하는 매니저 앱

TWRP `Install` 메뉴로 들어가서 `GoRhanHee_Kernel_...zip`을 선택했다. `ZIP signature check` 같은 옵션은 기본값(비활성) 그대로 뒀는데, 애초에 서명되지 않은 커스텀 패키지를 검증 없이 그대로 신뢰하고 플래싱하는 게 이 과정의 전제라서 굳이 켤 이유가 없었다. 하단 슬라이더를 밀어서 플래싱을 진행했고, `Successful` 메시지를 확인한 뒤 `Reboot System`을 눌렀다.

부팅 화면을 지켜보는 그 몇 십 초가 은근히 긴장됐다. 지난번 벽돌 경험 때문인지 삼성 로고가 평소보다 오래 떠 있는 것처럼 느껴졌는데, 다행히 정상적으로 One UI 홈 화면까지 올라왔다.

### 6.1 매니저 앱 설치 및 상태 확인

부팅 후 '내 파일' 앱으로 들어가서 `KernelSU_Next_v3.2.0_...apk`를 실행해 설치를 마쳤다. 앱을 켜보니 상단에 `Working`이라는 상태 텍스트가 떠 있었고, 커널 정보 항목에서 SuSFS 지원이 활성화돼 있는 것도 확인됐다. 여기까지만 보면 다 끝난 것 같았다.

### 6.2 ADB Shell su 거부 — 예상치 못한 마지막 관문

안심하고 PC 터미널에서 `adb shell`로 들어가 습관적으로 `su`를 쳤는데, 돌아온 건 이거였다.

```
/system/bin/sh: su: inaccessible or not found
```

에러 코드는 127. 순간 "설치가 제대로 안 됐나" 하고 당황했는데, 찾아보니 이건 실패가 아니라 KernelSU의 구조적 특징이었다. Magisk 계열과 달리 KernelSU는 전역적으로 존재하는 `/system/bin/su` 바이너리 자체가 없다. 대신 어떤 프로세스가 루트 권한을 요청하면 커널 레벨에서 그 요청을 가로채서, 앱/프로세스 단위로 개별 허용 여부를 판단하는 구조로 동작한다. 즉 `su`가 실행이 안 된 게 아니라, ADB Shell이라는 프로세스에 대해 아직 아무도 권한을 허용해주지 않은 상태였던 것이다.

KernelSU-Next 앱의 `Superuser`(두 번째 탭) 메뉴로 들어가서 목록을 훑어보니 `ADB Shell`(혹은 `Shell` / UID 1000으로 표시되는 항목) 항목이 있었다. 이 항목의 `Allow Superuser` 스위치를 켜고 나서 터미널로 돌아가 다시 `su`를 입력하니, 프롬프트가 `beyond0:/ $`에서 `beyond0:/ #`으로 바뀌는 걸 확인할 수 있었다. 루트 확보 완료.

> 참고: [GoRhanHee/android_kernel_samsung_exynos9820 v3.2.0 — GitHub Releases](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) · [KernelSU-Next — GitHub](https://github.com/KernelSU-Next/KernelSU-Next)

## 7. 전체 진행 타임라인

| 순서 | 단계 | 실행 내용 요약 | 상태 |
|---|---|---|---|
| 01 | 소스 & LFS 동기화 | git-lfs로 14GB+ 툴체인 수급, CRLF 정리 | 완료 |
| 02 | 심볼릭 링크 복구 | 깨진 GCC 툴체인 링크 재설정, 4.9.x 검증 | 완료 |
| 03 | 빌드 디버깅 | prepare-compiler-check 인자 누락, LSE 어셈블러 충돌 원인 규명 | 완료(원인만 규명) |
| 04 | 방향 전환 | GoRhanHee 기성 통합 이미지로 전환 결정 | 완료 |
| 05 | 1차 Odin 플래싱 | TWRP+vbmeta 시도 중 RQT_CLOSE, 무한 부팅 | 실패 |
| 06 | Frija 시도 | 순정 펌웨어 조회 시도, 서버 응답 없음 | 실패 |
| 07 | Bifrost 복구 | 순정 4종(AP/BL/CP/CSC) 수급 및 플래싱, 완전 복구 | 완료 |
| 08 | 부트로더 완전 언락 | Device unlock mode 진입, OEM LOCK: OFF 확정 | 완료 |
| 09 | TWRP/vbmeta 콤바인 | tar 생성 후 Odin 플래싱, PASS 달성 | 완료 |
| 10 | TWRP 강제 진입 | 볼륨+전원 조합, 3회 시도 끝에 성공 | 완료 |
| 11 | Format Data | 내부 저장소 암호화 해제 및 초기화 | 완료 |
| 12 | 커널 zip 플래싱 | GoRhanHee 통합 커널 설치, 부팅 성공 | 완료 |
| 13 | 매니저 & SU 승인 | KernelSU-Next 설치, ADB Shell 권한 허용 | 최종 성공 |

## 8. 최종 상태와 다음에 켜볼 것들

지금은 Exynos 9820 Galaxy S10e에서 KernelSU-Next v3.2.0 + SuSFS가 `Working` 상태로 정상 동작 중이고, ADB Shell에서 완전한 루트 권한(`beyond0:/ #`)을 확보한 상태다. 후속으로 켜두면 좋을 설정들도 조사해서 정리해뒀다.

- KernelSU-Next `Settings` 메뉴의 `Unmount modules by default` — 일반 앱을 실행할 때마다 모듈 마운트 상태와 커널 흔적을 자동으로 언마운트해준다고 한다.
- `Modules` 탭에 ZygiskNext와 Shamiko를 얹으면, ADB 디버깅 플래그·개발자 옵션 활성화 상태·커널 흔적을 일반 금융/게임 앱으로부터 은폐할 수 있다고 한다.

다만 이 두 가지는 아직 실제로 켜보고 "정말 탐지가 우회되는지"까지는 측정하지 않은 상태다. 말 그대로 다음 실험 거리로 남겨둔다.

## 9. 아직 확실치 않은 부분

- Odin의 `RQT_CLOSE !!`가 정확히 VaultKeeper의 어떤 내부 조건에서 트리거되는지는 로그 몇 줄만으로 유추한 것이라, 재현 실험을 통해 확정한 건 아니다.
- Frija가 정확히 왜 응답을 받지 못했는지(서버 문제였는지, 모델/CSC 코드 조합 문제였는지)는 아직 특정하지 못했다.
- `Unmount modules by default` + ZygiskNext + Shamiko 조합이 실제로 어느 수준까지 탐지를 우회하는지는 측정 전이다.
- 소스 직접 빌드(1번)의 이슈 3(LSE 어셈블러 충돌)을 `CROSS_COMPILE`/`PATH` 재매핑만으로 실제로 해결할 수 있는지는 아직 검증하지 못했다 — 방향 전환 전에 이 접근을 끝까지 밀어붙여보지 않았기 때문이다.

## 10. 앞으로 직접 검증해보고 싶은 것들

1. **소스 빌드 재도전**: 최신 버전의 Clang/binutils로 툴체인을 교체하거나, 서브모듈 분리형 저장소 구조를 가진 다른 Exynos 9820 커널 트리로 동일한 빌드를 다시 시도해서, 이번에 막혔던 LSE 이슈가 재현되는지 확인.
2. **Frija 실패 원인 재현**: 다른 모델명/CSC 조합으로도 Frija를 다시 테스트해서, 이번 실패가 이 기기(SM-G970N)에 국한된 문제였는지 확인.
3. **은폐 설정 실측**: `Unmount modules by default` + ZygiskNext + Shamiko를 켠 상태와 끈 상태에서, 루팅 탐지 라이브러리(RootBeer 등 테스트용)를 붙여 실제 탐지 결과가 달라지는지 비교.
4. **SO 무결성 실험 본격 시작**: 이번에 확보한 루팅 실기기 위에서, Zygote 2에서 미뤄뒀던 4.1~4.4 실험을 드디어 순서대로 진행.

## 마무리 — 오늘 정리한 것과 다음 편

소스를 직접 빌드해서 심겠다는 원래 계획은 실패했지만, 그 실패 덕분에 이 기기에서 정확히 뭐가 어디까지 막혀있는지는 오히려 더 분명해졌다. 벽돌을 한 번 내고 순정으로 복구까지 해보니, 이제 이 기기로 뭘 시도하든 최소한 "최악의 경우 되돌릴 수 있다"는 확신이 생겼다는 게 이번 주 가장 큰 소득인 것 같다. 다음 편은 원래 이번 주에 하려다 미뤄둔 SO 무결성 실험(Zygote 2의 4.1~4.4)을, 이제야 확보한 이 루팅된 기기 위에서 실제로 돌려보는 걸로 이어갈 예정이다.

---
---

# Kernel-Su Porting Log 1 - Getting KernelSU-Next + SuSFS onto a Galaxy S10e (Exynos 9820): From Build Failure to a Brick, and Back

In [Zygote 2](https://so-sung.github.io/posts/zygote-2/), item 7 left "check whether kernel source is even obtainable" as the next task. The original plan was to open this week's post with the SO-integrity experiments laid out in 4.1–4.4, but once I actually sat down with the laptop open, something felt out of order. All of those experiments assume a stable, already-rooted real device sitting there as a given — and I didn't have one. `friatest` is fine to run on an emulator, but the whole point of the SO-integrity question is the linker, signature verification, and boot chain, which only mean anything at the real-device level — faking it on an emulator would just mean telling myself later "you never actually confirmed this holds on real hardware."

So this week I set the SO topic aside for a week and pulled the Galaxy S10e (Exynos 9820) out of a drawer where it had been sitting, and dug into the process of getting KernelSU-Next + SuSFS onto it directly. Short version, up front: the original plan of building from source and flashing it myself failed. In the process of untangling that failure I bricked the device once and had to recover it, and I ended up rooted through a completely different path than planned. This post is less a success story and more a record of "the plan going sideways twice." Still, I think leaving the failed path intact is worth more than cleaning it up after the fact — so here it is, in the order it actually happened.

## 0. What I set out to do

- **Target device**: Samsung Galaxy S10e (SM-G970N / board name beyond0lteks)
- **Target SoC**: Samsung Exynos 9820 (arm64-v8a)
- **Build environment**: WSL2 Ubuntu 22.04 LTS (x86_64), Windows 11 host
- **Tools used**: Samsung Odin v3.14.4, TWRP 3.7.0_9-2, Bifrost, and (failed) Frija
- **Goal**: flash a kernel with KernelSU-Next + SuSFS integrated, to get a stable, rooted real device capable of bypassing ADB debugging and root detection

Worth noting up front: this is the first attempt actually starting with the three prerequisites laid out in section 5 of Zygote 2 ("could I build a custom kernel matching Android 9 myself") — kernel source that exactly matches the device, matching security patch level, and a backup of the original boot.img — already in hand from the start. The difference is that the target device this time isn't the Android 9 device from that post, but the S10e (running One UI 4.1), so the kernel version itself already falls within the range KernelSU officially calls "compatible." Which means this time the problem going in wasn't "unsupported, therefore blocked" — it was "supported, but the build process itself is rough."

## 1. Attempting a source build in WSL2

I started with [KernelKreation/exynos9820_SUSFS_kernel](https://github.com/KernelKreation/exynos9820_SUSFS_kernel). The README explicitly states "STOCK Galaxy S10 and N10 Series OneUI4.1," so I went in confident this was exactly the right target for my S10e.

### 1.1 Initial environment setup

The `git clone` itself finished quickly, but the repo's cross-compile toolchain — over 14GB, tracked with Git LFS — took a noticeably long time to pull down. WSL2's network speed itself wasn't the bottleneck; `git lfs pull` just fetches large binaries sequentially, one at a time, so it felt slower than it probably was. I remember making a cup of coffee and just watching the progress bar for most of it.

Next I dealt with text encoding. The repo had clearly been managed from a Windows environment at some point — CRLF line endings had crept into `Kconfig` and some `Makefile`s. I'd run into build scripts choking on this in other projects before, so I ran `dos2unix` across the whole repo up front, before touching anything else. Smooth sailing so far.

### 1.2 Issue 1 — Broken GCC toolchain symlinks

With setup done, I checked the toolchain binary first:

```bash
./toolchain/aarch64-linux-android-4.9/bin/aarch64-linux-android-gcc --version
```

Result: `No such file or directory`. My first assumption was a typo in the path, and I double-checked it a few times before running `file` on it — the path was indeed a symlink, but a broken one pointing at a target that didn't exist. Digging further into the repo structure, the actual GCC binary lived under a much deeper subdirectory, `toolchain/gcc-cfp/gcc-cfp-jopp-only/...`, and the symlink at the root was meant to point there via a relative path — except that relative-path calculation was off. My guess is a subtle difference between the directory depth on the repo maintainer's local machine and the depth you get from a fresh clone.

The fix was simple enough — delete the broken link and re-create it against the actual path:

```bash
cd ~/exynos9820_SUSFS_kernel/toolchain
rm -f aarch64-linux-android-4.9 arm-linux-androideabi-4.9
ln -s gcc-cfp/gcc-cfp-jopp-only/aarch64-linux-android-4.9 aarch64-linux-android-4.9
ln -s gcc-cfp/gcc-cfp-jopp-only/arm-linux-androideabi-4.9 arm-linux-androideabi-4.9
```

Re-checking the version this time printed `gcc (GCC) 4.9.x` cleanly, for both the 64-bit and 32-bit variants. Sounds trivial in hindsight, but this alone ate close to an hour.

### 1.3 Issue 2 — prepare-compiler-check and the Stack Protector check failing

With the symlinks fixed, running `build_kernel.sh` got further, but stopped somewhere new:

```
Cannot use CONFIG_CC_STACKPROTECTOR_STRONG: -fstack-protector-strong not supported by compiler
make: *** [Makefile:1250: prepare-compiler-check] Error 1
```

I opened up `build_kernel.sh` to trace the cause. In the section where the script calls `make`, it wasn't explicitly passing `ARCH`, `CROSS_COMPILE`, or `CC` as arguments — just invoking `make` bare. With that missing, the `prepare-compiler-check` target in the Makefile, when testing whether `-fstack-protector-strong` was supported, never received the target-architecture flag (`--target=aarch64-linux-gnu`) or a cross-assembler path, and ended up running the check against the host (x86_64) compilation environment instead. Naturally, the resulting support determination was wrong, and the build aborted on a false negative.

As a side note at this point, I also confirmed that the WSL2 environment was purely 64-bit, with none of the 32-bit compatibility libraries (like `libc6:i386`) that the older GCC toolchain's own 32-bit executables would depend on. That wasn't the direct cause of this particular error, but it had the unmistakable feel of something that was going to bite me later, so I just noted it and moved on.

### 1.4 Issue 3 — the external assembler rejecting the LSE option

Instead of relying on `build_kernel.sh`, I exported the environment variables directly into the shell and called `make` manually to work around it, then re-ran. `prepare-compiler-check` passed this time, but it stalled again, right before the C-compile step at `scripts/mod/empty.o`:

```
clang-6.0: warning: argument unused during compilation: '-Xassembler -march=armv8-a+lse'
Assembler messages:
Fatal error: invalid -march= option: `armv8-a+lse'
clang-6.0: error: assembler command failed with exit code 1
```

Opening `arch/arm64/Makefile`, I found the `-Wa,-march=armv8-a+lse` flag being passed through to the assembler. Clang 6.0 doesn't handle assembly itself — it calls out to an external assembler — and in this project's setup, that external assembler was GCC 4.9's `as`. The problem: that era's `as` has no idea what the LSE (Large System Extensions) instruction-set extension added in ARMv8.1 even is. It doesn't recognize the `+lse` string at all, and just fails during option parsing.

The obvious quick fix would've been stripping `+lse` from the source directly, but that felt like a bad move. The kernel source almost certainly has inline assembly written on the assumption LSE atomic instructions are available, and if so, just removing the assembler flag wouldn't fix anything — it'd just move the failure somewhere else, into a "this instruction is unknown" error mid-compile. Rather than touching the source, I concluded this called for more carefully mapping `CROSS_COMPILE` / `PATH` between Clang and the GCC assembler instead.

Working through these three issues one at a time, one thing became clear: even just pulling down an open-source kernel repo and running its build script once takes far more effort than that description suggests. There's a real gap between writing "you need to build the kernel yourself" as a line of theory in a previous post, and actually sitting there chasing these three errors down, one by one.

> Reference: [KernelKreation/exynos9820_SUSFS_kernel — GitHub](https://github.com/KernelKreation/exynos9820_SUSFS_kernel) · [KernelSU-Next — GitHub](https://github.com/KernelSU-Next/KernelSU-Next)

## 2. Pivoting — giving up on the source build

After finishing the root-cause analysis on issue three, I stepped back from the laptop for a minute to think it through. Honestly, at first I had a bit of stubbornness about it — a feeling that "real research" meant seeing a source build through from start to finish myself. But looking at it again more clearly, the actual goal of this experiment was never "the experience of compiling the kernel myself" — it was "getting a stable, rooted real device to run the SO-integrity experiments on." Means and ends had quietly swapped places on me.

Around then I found out that a fully built and tested integrated image already existed for this exact device (beyond0lteks). The [GoRhanHee/android_kernel_samsung_exynos9820 v3.2.0 release page](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) had a `GoRhanHee_Kernel_for_beyond0lteks_ksun.zip` package with KernelSU-Next v3.2.0 and SuSFS integrated, and the release notes went into reasonable detail — which commit it was built from, which SuSFS version it was tracking — enough that I felt confident this was, at minimum, something that had been verified once already.

Weighing the risk of continuing to hand-patch inline-assembly mismatches to finish a solo build, against the relative safety of flashing an already-verified package via TWRP, I let go of the perfectionism this time and pivoted to the latter. Redefining the actual goal made the call a lot easier than I expected. There was still a small nagging feeling that this was "letting someone else finish my homework," though, so I filed the source-build path away as something worth revisiting later — either with a newer Clang/binutils toolchain, or against a differently structured, submodule-separated repo.

## 3. The first Odin flash — a brick, courtesy of RQT_CLOSE

With direction settled, the rest looked simple: flash TWRP, disable `vbmeta` verification, flash the kernel zip, done. In practice it was nothing of the sort.

I grabbed `twrp-3.7.0_9-2-beyond0lte.img.tar` for the `beyond0lte` codename from the [official TWRP download page](https://dl.twrp.me/beyond0lte/twrp-3.7.0_9-2-beyond0lte.img.tar.html). I double-checked that `beyond0lte` really was the correct codename for the S10e specifically — the S10 proper uses `beyond1`, the S10+ uses `beyond2`, and it's easy to mix them up — but confirmed I had the right one. Along with a prepared `vbmeta_disabled.img`, I attempted a flash through Odin.

The log window showed:

```
SetupConnection..
Get PIT for mapping..
recovery.img
vbmeta.img
RQT_CLOSE !!
```

followed by `Complete(Write) operation failed`. The screen dropped straight into an endless reboot loop. Digging into the cause, it turned out that during a return-to-stock or partition-modifying operation, Samsung's own security layer — VaultKeeper (specifically a kernel gate state value called KG State) — had activated and was actively blocking the unsigned custom binaries. `RQT_CLOSE` reads unmistakably like "the request was forcibly closed" — this wasn't some generic comms glitch, it was a deliberate block.

I'll be honest, watching that screen was a bit of a cold-sweat moment. "Is this actually bricked" crossed my mind more than once. Thankfully, the volume-button combo still got me into download mode — the first sign this wasn't a full hard brick.

Recovery meant grabbing the four latest stock binaries for SM-G970N (AP/BL/CP/CSC) with a firmware download tool and re-flashing. This part had a bit of trial and error over which tool to use.

### 3.1 Trying Frija, and failing

I tried [Frija](https://github.com/SlackingVeteran/frija/releases) first — a tool I'd used before on other Samsung devices for stock firmware, so it felt like the familiar choice. This time, entering the model number (SM-G970N) and a CSC code and running a firmware lookup kept returning no meaningful response from the download server. I retried a few times and tried different CSC code combinations, with the same result every time. The logs didn't make it obvious exactly where the failure was happening, so rather than sink more time into it, I switched tools. (I still haven't pinned down exactly why this failed — something to come back to later.)

### 3.2 Switching to Bifrost, successful stock recovery

I used [Bifrost](https://bifrost.zwander.dev/) instead. Being web-based, there was no install step — I could look up firmware by model/CSC directly, and this time I got the four latest stock SM-G970N binaries without issue. With the device in download mode, I loaded the four files into Odin v3.14.4's respective slots (AP/BL/CP/CSC) and flashed them.

It worked. The endless-reboot state ended, and the device came back fully restored to stock. The lesson from this was unambiguous: **before touching a boot/recovery partition on a Samsung device, confirm up front that you can fall back to a full stock four-file set if it bricks — and have at least two tools lined up that can reliably fetch that stock set.** Without Bifrost as a fallback once Frija stalled, this is exactly where the whole attempt would have ground to a permanent halt.

> Reference: [TWRP for beyond0lte — twrp.me](https://dl.twrp.me/beyond0lte/twrp-3.7.0_9-2-beyond0lte.img.tar.html) · [SlackingVeteran/frija — GitHub](https://github.com/SlackingVeteran/frija/releases) · [Bifrost — zwander.dev](https://bifrost.zwander.dev/)

## 4. Fully unlocking the bootloader — the OEM LOCK toggle trap

Back in download mode after the stock recovery, the top of the screen read `OEM LOCK: ON(U)` — despite having already flipped "OEM unlocking" on in developer options beforehand. I wasn't sure why at first, but it turned out that developer-options toggle is only prior consent to allow an unlock later on — the bootloader itself isn't actually unlocked until a separate, physical confirmation step is completed inside download mode.

Worried that the exact unlock procedure might differ subtly between Exynos-family Samsung models, I cross-referenced an [XDA guide for bootloader unlock and rooting on the Note 10 5G (SM-N976N), Exynos version](https://xdaforums.com/t/guide-bootloader-unlock-and-root-note-10-5g-sm-n976n-exynos-version.4439103/) — a different device, but the same generation and same Exynos family. The download-mode entry method, the button-combo timing, and even the UI flow of the "Unlock bootloader?" confirmation screen matched closely enough that the guide gave me real confidence I was following the standard procedure shared across this whole device family, rather than improvising something device-specific and risky.

The actual steps taken:

1. Power the device off completely.
2. Hold Volume Down + Bixby together while connecting the USB cable to a PC.
3. Land on the mint-colored Warning screen.
4. Following the on-screen prompt, long-press Volume Up for 3+ seconds to enter "Device unlock mode."
5. On the "Unlock bootloader?" confirmation screen, a short press of Volume Up selects Yes.
6. A full factory reset runs automatically, followed by a reboot.
7. Re-entering download mode after reboot confirms the top-left text now reads `OEM LOCK: OFF`.

Internal storage was completely wiped at this point. I'd gone in assuming this as effectively a fresh device, so data loss itself wasn't a surprise — but watching the automatic factory reset kick off, there was still a brief flicker of "is there any way to undo this." Thankfully it rebooted cleanly.

## 5. Packaging TWRP + vbmeta as a combined tar, and forcing entry

With `OEM LOCK: OFF` confirmed, I tried flashing again, this time combining the TWRP image with `vbmeta_disabled.img`.

### 5.1 Building the combined tar

I already knew flashing TWRP alone risked tripping AVB (Android Verified Boot) verification and blocking the boot entirely, so I combined `twrp-3.7.0_9-2-beyond0lte.img` and `vbmeta_disabled.img` into a single archive, `twrp_vbmeta_combined.tar`, using 7-Zip. The reason for combining them rather than flashing separately: Odin maps files inside a tar to their respective partitions (recovery, vbmeta) by filename and writes them in order automatically, and pushing both at once felt more reliable than two separate flash operations.

### 5.2 The Odin flash

With the device in download mode (`OEM LOCK: OFF`), I loaded the combined tar into Odin's AP slot, unchecked `Auto Reboot`, and hit `Start`. No `RQT_CLOSE` in the log this time — a clean green `PASS!`. Coming right off the previous brick, that green text was a genuinely welcome sight.

### 5.3 Forcing entry into TWRP — the timing was trickier than expected

Since `Auto Reboot` had been unchecked, the device sat idle in place after the PASS instead of rebooting automatically. Pressing any random button here risked dropping into stock recovery or a normal boot, so I needed a specific combo to force entry into TWRP instead.

- With the cable still connected, hold Volume Down + Power together for 7 seconds.
- The instant the screen goes completely dark, let go immediately and hold Volume Up + Bixby + Power together at once.

This timing was more sensitive than I expected. Being off by even half a second on when I let go after the screen went dark just dropped it into a normal boot. The first two attempts failed and I ended up watching the Samsung logo of a normal boot both times, then had to power off and retry. Third attempt was the one that landed on the TWRP logo.

### 5.4 Formatting data and re-detecting partitions

Once properly inside TWRP, I went to `Wipe → Format Data`, typed `yes` to remove the encryption on internal storage and reset the filesystem. I already knew from a previous experience that skipping this step can leave storage detected as encrypted afterward, which either blocks file transfer entirely or behaves erratically over MTP — so this time I built it into the sequence from the start.

After the format finished, going back to the TWRP main menu and selecting `Reboot → Recovery` restarted the recovery itself once more. This re-detects the MTP partition properly, putting the device in a state where files can actually be transferred in from a PC.

> Reference: [TWRP for beyond0lte — twrp.me](https://dl.twrp.me/beyond0lte/twrp-3.7.0_9-2-beyond0lte.img.tar.html) · [Guide: Bootloader Unlock and Root, Note 10 5G SM-N976N Exynos — XDA Forums](https://xdaforums.com/t/guide-bootloader-unlock-and-root-note-10-5g-sm-n976n-exynos-version.4439103/)

## 6. Flashing the kernel and getting root

With TWRP properly recognized in MTP mode, I copied two files from the PC to the root of internal storage:

- `GoRhanHee_Kernel_for_beyond0lteks_ksun.zip` — the kernel package from the [GoRhanHee/android_kernel_samsung_exynos9820 v3.2.0 release](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0)
- `KernelSU_Next_v3.2.0_33129-release.apk` — the matching manager app

I went into TWRP's `Install` menu and selected `GoRhanHee_Kernel_...zip`. I left options like `ZIP signature check` at their default (disabled) — the whole premise of this step is trusting and flashing an unsigned custom package without verification, so there was no real reason to turn that on. I slid the confirmation slider, saw `Successful`, and hit `Reboot System`.

Those next thirty-odd seconds watching the boot screen were oddly tense. Maybe it was the earlier brick still fresh in my mind, but the Samsung logo felt like it stayed up longer than usual. It resolved cleanly into the One UI home screen, though.

### 6.1 Installing the manager app and checking status

After boot, I opened 'My Files' and ran `KernelSU_Next_v3.2.0_...apk` to finish the install. Opening the app showed `Working` at the top, with SuSFS support listed as active under the kernel info section. At this point it looked like everything was done.

### 6.2 ADB Shell su denied — an unexpected last hurdle

Feeling confident, I dropped into `adb shell` on the PC and typed `su` out of habit. What came back:

```
/system/bin/sh: su: inaccessible or not found
```

Error code 127. My first reaction was "did the install actually fail" — but digging into it, this wasn't a failure at all, just a structural quirk of KernelSU. Unlike Magisk-family root, KernelSU has no globally present `/system/bin/su` binary. Instead, when a process requests root, the kernel itself intercepts that request and decides, per app/process, whether to grant it. In other words, `su` wasn't failing to run — no one had yet granted permission to the `ADB Shell` process specifically.

Going into KernelSU-Next's `Superuser` tab (the second tab) and scrolling the list, I found an entry for `ADB Shell` (listed under `Shell` / UID 1000). Flipping that entry's `Allow Superuser` switch on and going back to the terminal to retry `su` flipped the prompt from `beyond0:/ $` to `beyond0:/ #`. Root, confirmed.

> Reference: [GoRhanHee/android_kernel_samsung_exynos9820 v3.2.0 — GitHub Releases](https://github.com/GoRhanHee/android_kernel_samsung_exynos9820/releases/tag/v3.2.0) · [KernelSU-Next — GitHub](https://github.com/KernelSU-Next/KernelSU-Next)

## 7. Full timeline

| # | Stage | Summary | Status |
|---|---|---|---|
| 01 | Source & LFS sync | Pulled 14GB+ toolchain via git-lfs, normalized CRLF | Done |
| 02 | Fixing symlinks | Re-linked broken GCC toolchain symlinks, verified 4.9.x | Done |
| 03 | Build debugging | Found root cause of prepare-compiler-check gap and LSE assembler conflict | Done (root cause only) |
| 04 | Pivot | Decided to switch to GoRhanHee's prebuilt integrated image | Done |
| 05 | First Odin flash | TWRP+vbmeta attempt hit RQT_CLOSE, endless reboot | Failed |
| 06 | Trying Frija | Stock firmware lookup attempted, no server response | Failed |
| 07 | Bifrost recovery | Fetched and flashed stock four-file set (AP/BL/CP/CSC), full recovery | Done |
| 08 | Full bootloader unlock | Entered Device unlock mode, confirmed OEM LOCK: OFF | Done |
| 09 | TWRP/vbmeta combined | Built tar, flashed via Odin, got PASS | Done |
| 10 | Forcing TWRP entry | Volume+Power combo, succeeded on the 3rd attempt | Done |
| 11 | Format Data | Removed internal storage encryption, reset filesystem | Done |
| 12 | Kernel zip flash | Installed GoRhanHee's integrated kernel, boot succeeded | Done |
| 13 | Manager & SU approval | Installed KernelSU-Next, granted ADB Shell permission | Final success |

## 8. Where things stand, and what to flip on next

The Galaxy S10e (Exynos 9820) is now running KernelSU-Next v3.2.0 + SuSFS in a `Working` state, with full root confirmed over ADB Shell (`beyond0:/ #`). I looked into a couple of follow-up settings worth turning on next.

- `Unmount modules by default` under KernelSU-Next's `Settings` — reportedly auto-unmounts module state and kernel traces every time a regular app launches.
- Stacking ZygiskNext and Shamiko under the `Modules` tab — reportedly hides the ADB debugging flag, developer-options state, and kernel traces from regular finance/game apps.

Neither of these has actually been turned on and measured for whether detection is genuinely bypassed yet, though. That's literally next on the list.

## 9. What's still unclear

- Exactly what internal condition inside VaultKeeper triggers Odin's `RQT_CLOSE !!` is something I've only inferred from a handful of log lines, not confirmed through a deliberate reproduction.
- Why Frija specifically failed to get a response (server-side issue vs. a bad model/CSC code combination) hasn't been pinned down.
- How well `Unmount modules by default` + ZygiskNext + Shamiko actually holds up against detection hasn't been measured.
- Whether issue 3 from the source build (the LSE assembler conflict) is actually solvable through `CROSS_COMPILE`/`PATH` remapping alone remains unverified — I never pushed that approach all the way through before pivoting away from it.

## 10. What I want to verify by hand, next

1. **Retry the source build**: swap in a newer Clang/binutils toolchain, or try the same build against a different, submodule-separated Exynos 9820 kernel tree, to see if the LSE issue reproduces.
2. **Reproduce the Frija failure**: retest Frija with different model/CSC combinations to see whether this failure was specific to this device (SM-G970N) or something broader.
3. **Measure the hiding settings**: compare actual detection results with a test root-detection library (e.g. RootBeer) attached, with `Unmount modules by default` + ZygiskNext + Shamiko toggled on versus off.
4. **Actually start the SO-integrity experiments**: finally run the 4.1–4.4 experiments from Zygote 2, in order, on this now-rooted real device.

## Wrap-up — what today covered, and what's next

The original plan of building the kernel from source and flashing it myself didn't pan out — but the failure made it a lot clearer exactly where, and how, this device pushes back. Having bricked it once and recovered fully back to stock, the biggest takeaway from this week is probably the confidence that whatever gets tried on this device next, worst case, it can be brought back. Next up: finally running the SO-integrity experiments from Zygote 2 (4.1–4.4) that got set aside this week, now actually on a rooted real device.
