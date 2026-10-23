# MicroBlaze Standalone SDK 구축 계획서 (Vitis 2025.2)

Oct 15, 2026 · @김도현

## 1. 개요

목표는 `make` 명령만으로 XSA에서 download.bit 생성과 JTAG 펌웨어 업로드까지 재현되고, 저장소에는 IDE 구성물이 전혀 없는 MicroBlaze standalone SDK를 만드는 것이다. Vitis 2025.2는 Linux 셸의 PATH에 항상 있다고 가정하고, BSP 생성, 비트스트림 병합, JTAG 업로드에만 사용한다. 플래시용 이미지는 만들지 않는다.

핵심 결정은 여섯 가지다.

- Vitis 워크스페이스는 `build/` 아래의 일회용 산출물로만 만들고 커밋하지 않는다. 저장소가 유일한 원본(single source of truth)이다. 단, XSA는 기본적으로 git에서 제외하고 SHA-256 해시만 추적한다.
- BSP는 XSCT Tcl 스크립트(`xsct`)로 생성하고, 애플리케이션은 저장소에서 관리하는 `CMakeLists.txt`와 `lscript.ld`로 CMake가 직접 빌드한다.
- download.bit는 XSA에서 꺼낸 bit·mmi 파일과 앱 ELF를 `updatemem`으로 병합해 만든다.

* 펌웨어 업로드는 xsdb로 JTAG 케이블을 자동 감지한 뒤, 비트스트림을 다시 프로그래밍하지 않고 시스템 리셋(Reset entire system)만 적용해 ELF를 올리고 실행한다.
* 코드는 보드(`m1`, `m1p`) 아래 앱 폴더로 나누고 `BOARD=m1 DESIGN=m1_app`처럼 선택한다. 디자인 하나에는 앱 하나만 둔다.

- SDK에는 Python을 쓰지 않는다. Vitis Unified의 BSP 생성(Lopper)은 내부적으로 Python을 쓰므로, BSP는 Tcl 기반 XSCT로 만들고 보드 접근은 xsdb(Tcl)로 한다. XSCT는 2026.1에서 제거되므로 Vitis는 2025.2로 고정한다.

## 2. 범위와 전제

범위는 Linux 호스트에서 `m1`, `m1p` 보드의 MicroBlaze(클래식) standalone 앱을 빌드하고 보드에 올리는 것까지다.

| 구분 | 내용 |
| --- | --- |
| 포함 | 보드·앱 신규 생성(기본 템플릿 Empty Application), BOARD·DESIGN 선택, XSA 업데이트, BSP·앱 빌드와 clean, download.bit 생성, JTAG 펌웨어 업로드(비트스트림 재프로그래밍 없음), download.bit JTAG 프로그래밍 |
| 제외 | 플래시용 이미지(MCS/BIN)와 부트로더, Windows 호스트, MicroBlaze V, FreeRTOS, 멀티 프로세서, 디자인당 복수 앱, 사용자 정의 드라이버 저장소, XSCT가 제거된 Vitis 2026.1 이상 |
| 도구 전제 | `settings64.sh`가 적용된 셸에서 `xsct`, `xsdb`, `updatemem`이 PATH에 있고 `XILINX_VITIS`가 설정되어 있다. 그 밖의 도구는 CMake(3.20 이상)와 GNU make뿐이다. 보조 스크립트는 모두 CMake 스크립트 모드(cmake -P), xsct, xsdb로 실행하므로 Ninja, Python, unzip, sha256sum 같은 별도 도구가 필요 없고, SDK에는 Python 코드가 없다 |
| 하드웨어 전제 | Linux용 JTAG 케이블 드라이버(udev 규칙)가 설치되어 있다. 업로드 전에 FPGA에는 해당 보드의 비트스트림이 이미 올라가 있어야 한다 |
| 입력 | Vivado에서 비트스트림 포함으로 export한 XSA. .bit와 .mmi가 있어야 download.bit를 만들 수 있다 |

## 3. 요구사항

요구사항은 9개이며, 각 항목은 수용 기준을 만족해야 완료로 본다.

| ID | 요구사항 | 수용 기준 |
| --- | --- | --- |
| R1 | XSA로 보드 신규 생성 | `make new-board BOARD=m1 XSA=<경로>` 한 번으로 `boards/m1/board.mk`와 `hw/xsa.sha256`이 생기고 BSP 빌드가 성공한다 |
| R2 | 보드 아래 앱(디자인) 신규 생성 | `make new BOARD=m1 DESIGN=m1_app`이 Empty Application 기반으로 `boards/m1/m1_app/`을 만들고, 추가 수작업 없이 `make build`가 성공한다 |
| R3 | BOARD·DESIGN 폴더 구조와 선택 | 앱은 `boards/<BOARD>/<DESIGN>/`에 있고 두 변수로 선택한다. DESIGN이 해당 BOARD 아래에 없으면 오류로 중단하며, 앱 간 산출물이 섞이지 않는다 |
| R4 | 보드 XSA 업데이트 | `make update-xsa BOARD=m1 XSA=<경로>`가 XSA 교체, 해시 갱신, BSP 재생성을 하고, 그 보드의 모든 앱에 대해 xparameters.h와 링커 스크립트 변경점을 보고한다. 추적 중인 lscript.ld는 자동으로 덮어쓰지 않는다 |
| R5 | 빌드와 clean | XSA·BSP 설정이 그대로면 `make build`가 Vitis를 띄우지 않고 변경된 소스만 증분 빌드한다. clean은 앱/보드 BSP/전체 단위로 나뉜다 |
| R6 | 앱 ELF와 XSA 비트스트림을 합친 download.bit 생성 | `make bit`가 `build/<BOARD>/<DESIGN>/download.bit`를 만든다. ELF가 BRAM 밖에 로드 세그먼트를 가지거나 XSA에 mmi가 없으면 오류로 중단한다 |
| R7 | JTAG 펌웨어 업로드 | `make upload`가 JTAG 케이블을 자동 감지하고, 비트스트림 프로그래밍 없이 시스템 리셋만 적용한 뒤 ELF를 올려 실행한다. 케이블이 없거나 여러 개, MicroBlaze 타깃 없음, 디바이스 불일치일 때 원인을 밝힌 오류로 중단한다 |
| R8 | 링커 스크립트와 CMake 설정 형상 관리 | 앱별 `lscript.ld`와 `CMakeLists.txt`, 공통 CMake 모듈, 툴체인 파일이 모두 git으로 추적된다 |
| R9 | IDE 구성물 금지, XSA 기본 무시 | 어떤 make 타깃을 실행해도 `build/` 밖에 미추적 파일이 생기지 않는다. 저장소에 Vitis 워크스페이스 메타데이터가 없고, XSA는 무시되며 해시 파일만 추적된다 |

## 4. 설계 원칙과 아키텍처

Vitis는 BSP 생성기(XSCT)와 보드 접근 도구(updatemem, xsdb)로만 쓰고, 일상적인 빌드는 Vitis 프로젝트 없이 수행한다. 이를 위해 세 가지 원칙을 둔다.

- **저장소가 유일한 원본이다.** 사람이 편집하는 파일만 추적하고, 나머지는 저장소 파일과 XSA로부터 재생성할 수 있어야 한다. XSA 자체는 무시하되 해시로 식별한다.
- **Vitis 워크스페이스는 일회용이다.** `build/<BOARD>/ws/` 안에서만 만들고, 입력 해시가 바뀔 때만 실행하며, 산출물 중 BSP만 꺼내 쓴다.
- **하드웨어 의존 값은 생성물로 다룬다.** CPU 플래그, 드라이버, 주소는 BSP와 함께 생성한다. 유일한 예외인 lscript.ld는 추적하되, XSA 변경 시 비교 보고로 보호한다.

&#91;embedded content: SDK 구성 · BSP 생성, 앱 빌드, download.bit와 JTAG 업로드\]

BSP는 보드마다 하나이며 XSA나 BSP 설정이 바뀔 때만 다시 만들어진다. 일상적인 소스 수정은 CMake 빌드만 거치고, 결과 ELF는 download.bit로 합치거나 xsdb로 바로 업로드한다.

## 5. 디렉터리 구조와 형상 관리 정책

앱은 `boards/<BOARD>/<DESIGN>/`에 두고, XSA와 BSP는 보드 단위로 하나씩 공유한다. 재생성 가능한 모든 것은 `build/` 아래로 모은다.

```
mbsdk/
├── Makefile                 # 진입점: BOARD·DESIGN 해석, 타깃 정의
├── local.mk                 # (무시) 개인별 기본 BOARD·DESIGN, JTAG_CABLE
├── mk/
│   ├── config.mk            # 도구 확인, board.mk·design.mk 로드
│   ├── bsp.mk               # BSP 생성 규칙(보드 단위), mkdir 잠금
│   ├── app.mk               # CMake 구성·빌드 규칙(앱 단위)
│   └── hw.mk                # bit, upload, program 규칙
├── cmake/
│   ├── toolchain-microblaze.cmake
│   └── mbsdk.cmake          # mbsdk_add_app() 등 공통 함수
├── scripts/                 # 실행 도구: xsct | cmake -P | xsdb (Python 없음)
│   ├── xsct_bsp.tcl         # xsct: 플랫폼 생성, BSP 설정, 빌드
│   ├── xsct_seed.tcl        # xsct: Empty Application으로 lscript 시드
│   ├── export_bsp.cmake     # cmake -P: BSP 산출물 복사, flags.cmake 생성
│   ├── xsa_extract.cmake    # cmake -P: XSA(zip)에서 bit·mmi·디바이스 정보 추출
│   ├── check_elf_bram.cmake # cmake -P: mb-readelf로 ELF가 BRAM 범위 안인지 검사
│   ├── jtag_upload.tcl      # xsdb: 케이블 감지, rst -system, dow, con
│   └── jtag_program.tcl     # xsdb: 케이블 감지, download.bit 프로그래밍
├── templates/
│   ├── board/               # board.mk.in
│   └── app/                 # design.mk.in, CMakeLists.txt.in, src/main.c
├── boards/
│   ├── m1/
│   │   ├── board.mk         # 보드별 define(CPU, BSP 설정)
│   │   ├── hw/
│   │   │   ├── m1.xsa       # (무시)
│   │   │   └── xsa.sha256   # 기대하는 XSA 해시
│   │   └── m1_app/
│   │       ├── design.mk
│   │       ├── CMakeLists.txt
│   │       ├── lscript.ld
│   │       └── src/
│   └── m1p/
│       ├── board.mk
│       ├── hw/
│       └── m1p_app/
└── build/                   # (무시) 전부 재생성 가능
    └── <BOARD>/
        ├── ws/              # XSCT 임시 워크스페이스
        ├── hw/              # 추출한 bit, mmi
        ├── bsp/             # include/, lib/libxil.a, flags.cmake, .stamp
        ├── ref/             # 참조 lscript.ld, 이전 xparameters.h
        └── <DESIGN>/
            ├── app/         # CMake 빌드 트리(Unix Makefiles), <DESIGN>.elf
            └── download.bit
```

| 대상 | 형상 관리 | 근거 |
| --- | --- | --- |
| `Makefile`, `mk/`, `cmake/`, `scripts/`, `templates/` | 추적 | SDK 자체 |
| `boards/<BOARD>/board.mk`, `boards/<BOARD>/hw/xsa.sha256` | 추적 | 보드 설정과 기대하는 XSA의 식별 |
| `boards/<BOARD>/<DESIGN>/` 전체 | 추적 | 앱 원본(R8) |
| `boards/<BOARD>/hw/*.xsa` | 무시 | 기본 정책(R9). 꼭 필요하면 `git add -f`로 예외 추적 |
| `build/` | 무시 | 워크스페이스, BSP, 빌드 트리, download.bit 모두 재생성 가능 |
| `local.mk` | 무시 | 개인 설정과 JTAG 케이블 지정 |

`.gitignore`는 `/build/`, `/local.mk`, `*.xsa` 세 줄이다. XSA가 저장소에 없으므로 새로 clone한 뒤에는 `make update-xsa`로 XSA를 배치해야 한다. 빌드는 실제 XSA의 해시를 `xsa.sha256`과 비교해 다르면 중단하므로, 바이너리를 추적하지 않고도 모두가 같은 하드웨어로 빌드하게 된다. `make check-tree`는 `git status --porcelain`으로 `build/` 밖 미추적 파일을 검사한다.

## 6. 설정 체계

보드 공통 설정은 `boards/<BOARD>/board.mk`, 앱 설정은 `boards/<BOARD>/<DESIGN>/design.mk`에 make 변수로 정의하고, 작업 대상은 `BOARD`와 `DESIGN` 두 변수로 고른다.

**BOARD·DESIGN 해석 순서**는 다음과 같다. 위에서 먼저 정해진 값이 이긴다.

1. 명령줄: `make BOARD=m1 DESIGN=m1_app build`
2. `local.mk`의 `BOARD := m1`, `DESIGN := m1_app` (`make use BOARD=m1 DESIGN=m1_app`이 이 파일을 갱신한다)
3. 값이 없거나 `boards/$(BOARD)/$(DESIGN)/design.mk`가 없으면 오류로 중단하고 `보드/앱` 목록을 출력한다. 묵시적 기본값은 두지 않는다.

보드 단위 타깃(`new-board`, `update-xsa`, `bsp`, `clean-bsp`)은 `BOARD`만 있으면 된다. XSA 경로는 `boards/<BOARD>/hw/<BOARD>.xsa`로 고정해 변수로 두지 않는다.

**board.mk 변수**

| 변수 | 필수 | 예시 | 용도 |
| --- | --- | --- | --- |
| `CPU` | 필수 | `microblaze_0` | 플랫폼 생성 시 프로세서 인스턴스 |
| `VITIS_VERSION` | 필수 | `2025.2` | 실행 중인 Vitis와 다르면 경고(드라이버 버전 고정 목적) |
| `PROC_PATH` | 선택 | `design_1_i/microblaze_0` | updatemem `-proc` 값. 비우면 mmi의 InstPath에서 자동 추출 |
| `BSP_STDIN`, `BSP_STDOUT` | 선택 | `axi_uartlite_0` | standalone OS의 stdin/stdout |
| `BSP_LIBS` | 선택 | `xilffs` | 도메인에 추가할 라이브러리 |
| `BSP_CONFIG` | 선택 | `os:standalone_xxx=val` | 그 밖의 `bsp config` 항목(option:param=value 목록) |

**design.mk 변수**

| 변수 | 필수 | 예시 | 용도 |
| --- | --- | --- | --- |
| `APP_NAME` | 선택 | 기본값 = DESIGN | ELF 파일 이름 |
| `BUILD_TYPE` | 선택 | `Release` | `CMAKE_BUILD_TYPE` |
| `UPLOAD_VERIFY` | 선택 | `1` | 업로드 후 메모리 내용을 ELF와 대조 |

**local.mk 또는 명령줄 전용 변수**

| 변수 | 예시 | 용도 |
| --- | --- | --- |
| `JTAG_CABLE` | `210251A08870` | 케이블 일련번호(일부 일치). 케이블이 여러 개일 때만 필요 |
| `HW_SERVER` | `tcp:labpc:3121` | 원격 hw\_server. 비우면 로컬 hw\_server를 자동으로 띄운다 |

```make
# boards/m1/board.mk
CPU           := microblaze_0
VITIS_VERSION := 2025.2
BSP_STDIN     := axi_uartlite_0
BSP_STDOUT    := axi_uartlite_0

# boards/m1/m1_app/design.mk
BUILD_TYPE    := Release
```

Makefile은 이 값들을 환경 변수로 export해 xsct·xsdb Tcl 스크립트와 `cmake -P` 스크립트에 넘긴다. `BSP_STDIN` 등은 XSCT의 `bsp config`, `BSP_LIBS`는 `bsp setlib`로 적용된다. BSP 재생성 여부는 XSA, `BSP_*` 변수, `VITIS_VERSION`, `xsct_bsp.tcl`을 합친 SHA-256 해시를 `build/<BOARD>/bsp/.stamp`와 비교해 판단한다. 파일 시각이 아닌 해시를 쓰므로 브랜치 전환만으로 BSP가 다시 만들어지지 않는다.

## 7. 기능별 상세 설계

모든 기능은 "저장소의 보드·앱 폴더를 읽어 `build/`에 결과를 만든다"는 패턴을 따른다. 저장소 파일을 바꾸는 타깃은 `new-board`, `new`, `update-xsa`, `accept-lscript`, `use` 다섯뿐이다.

### 7.1 보드 생성: `make new-board` (R1)

1. `BOARD`가 이미 있으면 중단한다. XSA(zip) 안에 .bit와 .mmi가 있는지 확인하고, 없으면 경고를 낸 뒤 계속하되 `bit` 타깃은 쓸 수 없음을 알린다.
2. `templates/board/`로 `boards/<BOARD>/board.mk`를 만든다. `CPU=`를 주지 않으면 mmi의 프로세서가 하나일 때 그 이름을 쓰고, 여럿이면 목록을 보여주고 중단한다.
3. XSA를 `boards/<BOARD>/hw/<BOARD>.xsa`로 복사하고 `cmake -E sha256sum`으로 구한 해시를 `hw/xsa.sha256`에 기록한다.
4. `xsct scripts/xsct_bsp.tcl`로 플랫폼 생성, 도메인 설정(`bsp config`, 라이브러리 추가), 빌드를 수행하고, `cmake -P scripts/export_bsp.cmake`로 `build/<BOARD>/bsp/`를 채운다.

### 7.2 앱 생성: `make new` (R2)

1. `BOARD`는 있어야 하고 `DESIGN`은 그 아래에 없어야 한다. DESIGN 이름은 폴더 이름으로 안전한 문자만 허용한다.
2. `templates/app/`을 `boards/<BOARD>/<DESIGN>/`으로 복사하고 `design.mk.in`, `CMakeLists.txt.in`을 치환한다.
3. `xsct scripts/xsct_seed.tcl`가 보드의 임시 워크스페이스에 **Empty Application** 템플릿으로 앱 프로젝트를 만들고, 생성된 `lscript.ld`만 앱 폴더로 복사한다. 이 앱 프로젝트는 버린다.
4. Empty Application에는 소스가 없으므로, 빌드가 성립하도록 빈 `main()`만 있는 `src/main.c` 하나를 둔다.
5. 생성 결과와 다음 명령(`make use BOARD=<BOARD> DESIGN=<DESIGN>`)을 출력한다.

### 7.3 선택과 조회: `make use`, `make list`, `make info` (R3)

`make use`는 `local.mk`에 현재 BOARD·DESIGN을 기록한다. `make list`는 `boards/*/*/design.mk`를 찾아 `보드/앱` 목록과 현재 선택을 보여준다. `make info`는 해석된 변수, 실제 XSA 해시와 기대 해시, BSP stamp 상태, 실행 중인 Vitis 버전을 출력한다.

### 7.4 XSA 업데이트: `make update-xsa` (R4)

1. 새 XSA의 해시가 `xsa.sha256`과 같으면 파일만 배치하고 끝낸다. 새로 clone한 뒤 XSA를 놓는 경우다.
2. 해시가 다르면 기존 `xparameters.h`를 `build/<BOARD>/ref/`에 백업하고, XSA를 덮어쓰고 `xsa.sha256`을 갱신한다. 이 파일을 커밋하면 팀원들의 다음 빌드가 해시 불일치로 멈춰 새 XSA가 필요함을 알린다.
3. `build/<BOARD>/ws`, `bsp`, `hw`를 지우고 7.1의 4단계를 다시 실행한다. XSCT의 하드웨어 갱신 기능 대신 **재생성**을 택한다. BSP 설정이 모두 `board.mk`에 있어 잃는 것이 없고 결과가 결정적이다.
4. 변경 보고를 출력한다. `xparameters.h`는 이전본과 diff하고, `xsct_seed.tcl`로 만든 참조 `lscript.ld`는 그 보드 **모든 앱**의 `lscript.ld`와 MEMORY 블록을 비교한다.
5. MEMORY 블록이 다른 앱은 경고만 하고, 반영은 앱별 `make accept-lscript`로 명시적으로 한다. 사용자가 손본 스택·힙 크기와 섹션 배치를 보호하기 위해서다.
6. 그 보드 모든 앱의 CMake 빌드 트리를 무효화해 다음 빌드에서 재구성되게 한다.

### 7.5 빌드와 clean: `make build`, `make clean*` (R5)

`make build`(기본 타깃)는 먼저 실제 XSA의 해시를 `xsa.sha256`과 비교하고, XSA가 없거나 다르면 `make update-xsa`를 안내하며 중단한다. 이어서 BSP stamp 해시를 확인해 필요할 때만 XSCT를 실행한다. 그다음 `cmake -S boards/<BOARD>/<DESIGN> -B build/<BOARD>/<DESIGN>/app -G "Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-microblaze.cmake -DMBSDK_BSP_DIR=build/<BOARD>/bsp`로 구성하고, `cmake --build build/<BOARD>/<DESIGN>/app -j`로 빌드한 뒤 `mb-size`로 ELF 크기를 보고한다. CMake가 생성한 Makefile을 최상위 make가 `cmake --build`로 호출하는 구조이며, 평소 개발 루프에서는 Vitis가 뜨지 않는다.

| 타깃 | 지우는 범위 | 영향 |
| --- | --- | --- |
| `clean` | `build/<BOARD>/<DESIGN>/app` | 그 앱만 재컴파일 |
| `distclean` | `build/<BOARD>/<DESIGN>/` | 그 앱의 빌드 트리와 download.bit |
| `clean-bsp` | `build/<BOARD>/`의 `ws`, `bsp`, `hw`, `ref` | 그 보드 BSP 재생성, 그 보드 모든 앱 재구성 |
| `clean-all` | `build/` 전체 | 모든 보드·앱 처음부터 |

### 7.6 download.bit 생성과 프로그래밍: `make bit`, `make program` (R6)

1. `cmake -P scripts/xsa_extract.cmake`가 CMake의 아카이브 해제 기능으로 XSA에서 .bit, .mmi, 디바이스 정보를 `build/<BOARD>/hw/`로 꺼낸다. XSA는 zip이므로 별도 압축 도구가 필요 없다.
2. `PROC_PATH`를 정한다. `board.mk` 값이 있으면 쓰고, 없으면 mmi의 `InstPath`에서 가져온다.
3. `cmake -P scripts/check_elf_bram.cmake`가 Vitis 툴체인의 `mb-readelf`로 ELF 로드 세그먼트를 읽어, mmi에 정의된 BRAM 범위 안인지 검사한다. 벗어나면 섹션 이름과 주소를 출력하고 중단한다.
4. `updatemem -force -meminfo <mmi> -bit <bit> -data <elf> -proc <PROC_PATH> -out build/<BOARD>/<DESIGN>/download`을 실행한다. [UG1580](https://docs.amd.com/r/en-US/ug1580-updatemem/Arguments-for-updatemem)에 따르면 `-out`에는 확장자를 빼고 주며 .bit가 자동으로 붙고, 암호화되지 않은 비트스트림만 처리한다.
5. `make program`은 7.7의 1\~3단계와 같은 JTAG 감지 후 `fpga -file`로 download.bit를 FPGA에 프로그래밍한다. 하드웨어가 바뀌었거나 FPGA가 비어 있을 때 쓴다.

### 7.7 펌웨어 업로드: `make upload` (R7)

`make upload`는 `build`를 먼저 수행한 뒤 `xsdb scripts/jtag_upload.tcl`을 실행한다. FPGA는 다시 프로그래밍하지 않고, Vitis 실행 설정의 "Reset entire system"에 해당하는 시스템 리셋만 적용한 뒤 펌웨어만 올린다.

1. **연결**: `connect`로 hw\_server에 붙는다. `HW_SERVER`가 없으면 로컬 hw\_server를 자동으로 띄운다.
2. **케이블 자동 감지**: `jtag targets -filter {level == 0}`으로 케이블 목록을 얻는다. 0개면 "케이블 없음(USB 연결과 케이블 드라이버 확인)"으로 중단한다. 1개면 그것을 쓴다. 여러 개면 `JTAG_CABLE`과 일련번호가 하나만 일치할 때 그것을 쓰고, 아니면 목록을 출력하고 중단한다.
3. **디바이스 확인**: 선택한 케이블 체인의 FPGA 이름(`jtag_device_name`)을 XSA에서 읽은 디바이스 이름과 비교해 다르면 중단한다. 다른 보드에 잘못 올리는 것을 막는다.
4. **MicroBlaze 타깃 확인**: 그 케이블 아래에서 `targets -filter`로 이름이 MicroBlaze로 시작하는 타깃을 찾는다. 없으면 FPGA가 비어 있거나 MDM이 없는 비트스트림으로 판단하고 `make program`을 안내하며 중단한다. 여러 개면 범위 밖이므로 중단한다.
5. **시스템 리셋**: `rst -system`으로 MDM의 시스템 리셋을 걸어 프로세서와 주변장치를 함께 초기화한다. 비트스트림은 그대로 유지된다. [UG1725](https://docs.amd.com/r/en-US/ug1725-xsdb-reference-guide/rst?contentId=1YUa7Fv~hOjL3o7y3s5dng)에 따르면 `-system`은 활성 시스템 리셋이며 `rst`의 기본 동작이다.
6. **적재와 실행**: 리셋 직후 프로세서를 정지시키고 `dow <ELF>`로 올린다. `UPLOAD_VERIFY=1`이면 메모리 내용을 ELF와 대조한다. 마지막으로 `con`으로 실행하고 연결을 끊는다.
7. **보고**: 케이블 일련번호, 디바이스 이름, 타깃, ELF 크기를 출력한다. 실패하면 실패한 단계 이름과 함께 0이 아닌 종료 코드로 끝낸다.

업로드는 FPGA의 비트스트림이 현재 XSA와 같은지까지는 보증하지 못한다. 디바이스 이름만 확인하므로, XSA를 업데이트한 뒤 첫 실행은 `make program`으로 한다.

### 7.8 링커 스크립트와 CMake 설정 관리 (R8)

원칙은 **추적 파일에는 하드웨어 의존 값을 넣지 않고, 하드웨어 의존 값은 BSP와 함께 생성한다**는 것이다.

| 파일 | 추적 | 내용 |
| --- | --- | --- |
| `boards/<BOARD>/<DESIGN>/lscript.ld` | 추적 | Empty Application으로 1회 시드한 뒤 사람이 관리. 스택·힙 크기도 여기서 직접 수정 |
| `boards/<BOARD>/<DESIGN>/CMakeLists.txt` | 추적 | 소스 목록, 앱별 정의와 추가 플래그만 |
| `cmake/mbsdk.cmake` | 추적 | `mbsdk_add_app()`: flags.cmake 로드, `-T lscript.ld`, libxil·libgcc·libc 링크 그룹, 크기 보고 |
| `cmake/toolchain-microblaze.cmake` | 추적 | `$XILINX_VITIS` 기준으로 mb-gcc 지정, `CMAKE_SYSTEM_NAME Generic` |
| `build/<BOARD>/bsp/flags.cmake` | 생성 | BSP 빌드에 쓰인 CPU 플래그(`-mcpu`, `-mxl-*` 등), BSP include·lib 경로 |

```cmake
# boards/m1/m1_app/CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(m1_app C ASM)
include(${MBSDK_ROOT}/cmake/mbsdk.cmake)
mbsdk_add_app(m1_app
  SOURCES src/main.c
  LINKER_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/lscript.ld)
```

CPU 플래그를 추적 파일에 하드코딩하지 않는 이유는, Vivado에서 barrel shifter나 곱셈기 옵션 하나만 바꿔도 libxil.a와 앱의 플래그가 어긋나 조용히 오동작하기 때문이다. `export_bsp.cmake`가 BSP 빌드 결과에서 플래그를 뽑아 `flags.cmake`로 만든다.

## 8. Make 타깃 명세

타깃은 18개이며, `xsct`는 BSP를 다시 만들어야 할 때만, xsdb는 보드에 무언가를 쓸 때만 실행된다. 어떤 타깃도 Python을 실행하지 않는다.

| 타깃 | 인자 | 동작 | 실행하는 Vitis 도구 | 저장소 변경 |
| --- | --- | --- | --- | --- |
| `new-board` | `BOARD`, `XSA`, `CPU`(선택) | 보드 폴더, board.mk, 해시 파일 생성 후 BSP 생성 | `xsct` | 예 |
| `new` | `BOARD`, `DESIGN` | Empty Application으로 앱 폴더 생성, lscript 시드 | `xsct` | 예 |
| `use` | `BOARD`, `DESIGN` | `local.mk`에 현재 선택 기록 | 없음 | 아니오(무시 대상) |
| `list` | 없음 | 보드/앱 목록과 현재 선택 표시 | 없음 | 아니오 |
| `info` | 없음 | 해석된 변수, XSA·stamp 해시, Vitis 버전 출력 | 없음 | 아니오 |
| `build` (기본) | `BUILD_TYPE`(선택) | XSA 해시 검사, 필요 시 BSP 재생성, 앱 증분 빌드 | 필요할 때만 `xsct` | 아니오 |
| `bsp` | `BOARD` | 보드 BSP 강제 재생성 | `xsct` | 아니오 |
| `update-xsa` | `BOARD`, `XSA` | XSA 배치 또는 교체, 해시 갱신, BSP 재생성, 변경 보고 | 교체 시 `xsct` | 예(xsa.sha256) |
| `accept-lscript` | `BOARD`, `DESIGN` | 참조 lscript의 MEMORY 블록을 앱 lscript에 반영 | 없음 | 예(lscript.ld) |
| `bit` | 없음 | ELF·BRAM 검사 후 download.bit 생성 | `updatemem` | 아니오 |
| `program` | `JTAG_CABLE`, `HW_SERVER`(선택) | JTAG 자동 감지 후 download.bit 프로그래밍 | `xsdb` | 아니오 |
| `upload` | `JTAG_CABLE`, `HW_SERVER`(선택) | JTAG 자동 감지, 시스템 리셋, ELF 적재·실행(비트스트림 유지) | `xsdb` | 아니오 |
| `clean` | 없음 | 앱 빌드 트리 삭제 | 없음 | 아니오 |
| `distclean` | 없음 | `build/<BOARD>/<DESIGN>/` 삭제 | 없음 | 아니오 |
| `clean-bsp` | `BOARD` | 보드의 워크스페이스·BSP·추출 hw·ref 삭제 | 없음 | 아니오 |
| `clean-all` | 없음 | `build/` 전체 삭제 | 없음 | 아니오 |
| `check-tree` | 없음 | `build/` 밖 미추적 파일이 있으면 실패 | 없음 | 아니오 |
| `help` | 없음 | 타깃과 인자 요약 출력 | 없음 | 아니오 |

## 9. 구현 작업 목록

작업은 아래 여섯 묶음으로 나누며, 각 묶음은 끝에 적힌 10장의 검증 시나리오를 통과하면 완료로 본다.

### 9.1 사전 확인

Vitis 2025.2 실물로 먼저 확정해야 하는 사항이다. 기준 XSA 하나로 수동 절차를 끝까지 한 번 실행해 확인한다.

- [ ] **가장 먼저**: XSCT 2025.2의 `platform create`(MicroBlaze, standalone)와 `platform generate`로 BSP가 생성·빌드되는지. 안 되면 BSP 생성 방식 전체를 다시 검토한다
- [ ] XSCT가 만든 BSP 산출물(include, libxil.a)과 BSP 컴파일 플래그가 놓이는 경로
- [ ] xsct·xsdb Tcl 스크립트에서 환경 변수(`env` 배열)로 값을 받는 방식
- [ ] mmi의 `InstPath`와 XSCT `-proc` 이름의 대응 규칙
- [ ] XSCT의 Empty Application 템플릿 이름과 생성되는 `lscript.ld`의 위치
- [ ] XSA 안에서 디바이스(파트) 이름을 읽을 파일
- [ ] 클래식 MDM에서 `rst -system` 직후 프로세서 상태와 정지 방법, 메모리 대조 명령 사용 가능 여부
- [ ] xsct, xsdb, updatemem 실행 중 Python 프로세스가 뜨지 않는지
- [ ] Vitis 설치본에 CMake가 들어 있는지와 그 버전. 3.20 이상이면 그것을 쓰고, 아니면 시스템 CMake를 쓴다
- [ ] Vitis 툴체인의 `mb-gcc`, `mb-readelf`, `mb-size` 경로(`$XILINX_VITIS` 기준)

### 9.2 골격

- [ ] `Makefile`과 `mk/config.mk`: `xsct`, `xsdb`, `updatemem`, `cmake`, `XILINX_VITIS` 존재 확인, `VITIS_VERSION`이 2025.2가 아니면 중단, BOARD·DESIGN 해석, board.mk·design.mk 로드
- [ ] `use`, `list`, `info`, `help`, `check-tree`(미추적 파일과 `*.py` 파일 검사) 타깃
- [ ] `.gitignore`(`/build/`, `/local.mk`, `*.xsa`)와 `templates/board/`, `templates/app/`

완료 기준: T14, T15

### 9.3 BSP와 앱 빌드

- [ ] `xsct_bsp.tcl`: 플랫폼 생성, `bsp config`·`bsp setlib` 설정, `platform generate`
- [ ] `mk/bsp.mk`: `build/<BOARD>/.lock` 디렉터리를 `mkdir`로 만드는 방식의 보드 단위 잠금으로 같은 보드의 BSP 동시 생성 차단
- [ ] `export_bsp.cmake`: BSP 산출물 복사, `flags.cmake` 생성, 필수 파일 존재 검사
- [ ] XSA 해시 검사(`cmake -E sha256sum`과 `xsa.sha256` 비교)와 BSP stamp 해시
- [ ] `toolchain-microblaze.cmake`, `mbsdk.cmake`(생성기는 `Unix Makefiles` 고정)
- [ ] `build`, `bsp`, `clean`, `distclean`, `clean-bsp`, `clean-all` 타깃

완료 기준: T1, T3, T4, T5, T8, T16

### 9.4 생성과 XSA 업데이트

- [ ] `xsct_seed.tcl`(Empty Application 시드)와 `new-board`, `new` 타깃
- [ ] `update-xsa`: 배치/교체 분기, 백업, BSP 재생성, xparameters.h·lscript 비교 보고
- [ ] `accept-lscript`: MEMORY 블록만 교체

완료 기준: T2, T6, T7

### 9.5 비트스트림과 JTAG

- [ ] `xsa_extract.cmake`(bit, mmi, 디바이스 이름), `check_elf_bram.cmake`(`mb-readelf` 사용), `bit` 타깃
- [ ] `jtag_upload.tcl`: 케이블 감지, 디바이스 확인, MicroBlaze 타깃 확인, `rst -system`, `dow`, 선택적 메모리 대조, `con`
- [ ] `jtag_program.tcl`과 `program` 타깃(케이블 감지 로직은 upload와 공유)
- [ ] 단계별 오류 메시지와 종료 코드 통일

완료 기준: T9, T10, T11, T12, T13

### 9.6 마무리

- [ ] `m1`, `m1p` 보드와 `m1_app`, `m1p_app` 생성 및 실보드 확인
- [ ] README: 처음 clone한 뒤의 절차, 타깃 요약, 자주 나는 오류와 해결법

완료 기준: T1\~T16 전체

## 10. 검증 계획

검증은 `m1`, `m1p` 실보드와 각 보드의 기준 XSA, 그리고 기준 XSA에서 UART 주소만 바꾼 XSA와 BRAM 크기만 바꾼 XSA로 아래 16개 시나리오를 수행한다. 모든 실패 시나리오는 원인을 밝힌 메시지와 0이 아닌 종료 코드를 함께 확인한다.

| ID | 시나리오 | 기대 결과 | 요구사항 |
| --- | --- | --- | --- |
| T1 | 새로 clone한 뒤 `make update-xsa BOARD=m1 XSA=<기준>`, `make BOARD=m1 DESIGN=m1_app build` | XSA가 배치만 되고(저장소 변경 없음) 빌드 성공 | R3, R5, R9 |
| T2 | `make new-board BOARD=m1 XSA=<기준>`, `make new BOARD=m1 DESIGN=m1_app`, `make build` | 추가 수작업 없이 성공. 앱 폴더에는 design.mk, CMakeLists.txt, lscript.ld, 빈 main.c만 있음 | R1, R2 |
| T3 | .c 파일 하나 수정 후 `make` | 그 파일만 재컴파일, Vitis 프로세스 미실행 | R5 |
| T4 | XSA는 그대로 두고 브랜치 전환 후 복귀 | BSP 재생성 없음 | R5 |
| T5 | `xsa.sha256`과 다른 XSA를 손으로 복사한 뒤 `make` | `make update-xsa` 안내와 함께 중단 | R4, R9 |
| T6 | UART 주소만 다른 XSA로 `update-xsa` | xparameters.h 변경 보고, m1의 모든 앱 재구성, 재빌드 후 보드 정상 동작 | R4 |
| T7 | BRAM 크기만 다른 XSA로 `update-xsa` 후 `accept-lscript` | lscript 경고 출력, 반영 후 MEMORY 블록만 바뀌고 사용자 스택·힙 설정 유지 | R4, R8 |
| T8 | 같은 소스를 XSCT 앱 프로젝트와 SDK로 각각 빌드 | CPU 관련 컴파일 플래그 동일 | R8 |
| T9 | BRAM보다 큰 배열을 넣고 `make bit` | 섹션·주소를 표시하며 실패 | R6 |
| T10 | `make bit`, `make program` | download.bit로 보드가 부팅되고 UART 출력 확인 | R6 |
| T11 | 비트스트림이 올라간 보드에서 소스를 고친 뒤 `make upload` | xsdb 로그에 FPGA 프로그래밍이 없고, 시스템 리셋 후 고친 펌웨어의 출력이 UART에 나타남 | R7 |
| T12 | 펌웨어가 실행 중인 상태에서 `make upload`를 10회 반복 | 매회 UART·타이머·인터럽트가 초기 상태에서 정상 시작 | R7 |
| T13 | 케이블 미연결, 케이블 2개(`JTAG_CABLE` 없음·있음), 빈 FPGA, 다른 디바이스 보드에서 `make upload` | 미연결·빈 FPGA·디바이스 불일치·케이블 미지정은 각각 다른 메시지로 실패, `JTAG_CABLE` 지정 시 성공 | R7 |
| T14 | `DESIGN` 없이 `make`, `BOARD=m1 DESIGN=m1p_app` 지정, m1·m1p 번갈아 빌드 | 앞의 둘은 보드/앱 목록과 함께 실패, 번갈아 빌드 시 산출물 섞임 없음 | R3 |
| T15 | 모든 타깃 실행 후 `make check-tree`와 `git status` | 통과, XSA와 Vitis 메타데이터가 미추적 목록에 나타나지 않고, 저장소에 .py 파일이 없음 | R9 |
| T16 | `clean`, `distclean`, `clean-bsp`, `clean-all` 각각 실행 | 7.5 표의 범위만 삭제 | R5 |

## 참고 자료

- [UG1580 Arguments for updatemem](https://docs.amd.com/r/en-US/ug1580-updatemem/Arguments-for-updatemem)
- [UG1580 Examples](https://docs.amd.com/r/en-US/ug1580-updatemem/Examples)

* [UG1725 rst (xsdb 리셋 옵션)](https://docs.amd.com/r/en-US/ug1725-xsdb-reference-guide/rst?contentId=1YUa7Fv~hOjL3o7y3s5dng)
* [UG1725 jtag targets (케이블 목록)](https://docs.amd.com/r/en-US/ug1725-xsdb-reference-guide/jtag-targets)
* [UG1725 targets (타깃 필터)](https://docs.amd.com/r/en-US/ug1725-xsdb-reference-guide/targets)
* [UG1400 Selecting Target Based on Target Properties (jtag\_cable\_serial 등 속성)](https://docs.amd.com/r/2023.2-English/ug1400-vitis-embedded/Selecting-Target-Based-on-Target-Properties)

- [UG1400 XSCT to Python API Migration (2026.1에서 XSCT 제거)](https://docs.amd.com/r/en-US/ug1400-vitis-embedded/XSCT-to-Python-API-Migration)
- [UG1304 Vitis Tools (XSCT 기능 개요, 2025.2)](https://docs.amd.com/r/en-US/ug1304-versal-acap-ssdg/Vitis-Tools?contentId=kOhFTj0hmVY91VWbGLr5Jw)
- [Porting embeddedsw components to SDT based flow (Lopper·Python 기반 BSP 생성)](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/2743468485/Porting+embeddedsw+components+to+system+device+tree+SDT+based+flow)
