# FEMU 한국어 기여 요약

이 저장소는 QEMU/KVM 기반 NVMe SSD emulator인 [MoatLab/FEMU](https://github.com/MoatLab/FEMU)의 fork입니다. Upstream FEMU는 application, guest OS, NVMe interface, SSD model을 포함하는 full-system storage 연구 환경이며 BlackBox SSD, OpenChannel SSD, ZNS, NoSSD, CSD mode를 제공합니다.

Upstream의 설치·실행·configuration 전체 문서는 [README.md](README.md)를 기준으로 합니다. 이 문서는 `shining-b-02` fork에서 수행한 priority queue regression test 기여와 검증 결과를 설명합니다.

## 기여 내용

FEMU FTL과 NVMe request path가 사용하는 `hw/femu/lib/pqueue.c`에는 최근 heap repair 관련 수정이 두 차례 들어갔지만 이를 직접 고정하는 unit test가 없었습니다.

- `pqueue_change_priority()`가 이미 갱신된 priority를 받는 경우 parent relation으로 repair 방향을 선택
- `pqueue_randpop()`이 임의 위치를 마지막 원소로 치환한 뒤 replacement가 parent보다 높은 우선순위라면 위로 이동

이 fork는 `tests/unit/test-femu-pqueue.c`와 Meson target을 추가해 두 동작을 독립적으로 검증합니다. Upstream 제출본은 [MoatLab/FEMU PR #196](https://github.com/MoatLab/FEMU/pull/196)에서 review 가능한 상태입니다.

## 실제 테스트 결과

2026-08-21에 macOS ARM64의 최소 QEMU system build에서 실행했습니다.

| 항목 | 결과 |
|---|---:|
| Meson test suite | PASS |
| GLib subtest | 3/3 PASS |
| 실행 시간 | 0.02초 |
| checkpatch | 오류 0, 경고 0 |
| Compiler | Apple clang 17.0.0 |
| 테스트 대상 commit | `c268210a4a13c0f0d569c0c315bf2b887621e155` |

검증한 subtest:

1. `/femu/pqueue/pop-order`
2. `/femu/pqueue/change-preupdated-priority`
3. `/femu/pqueue/randpop-bubbles-replacement`

기계 판독 가능한 결과와 source hash는 [`evidence/pqueue-unit-macos-arm64-2026-08-21.json`](evidence/pqueue-unit-macos-arm64-2026-08-21.json)에 있습니다.

## 재현 방법

아래 구성은 KVM 없이 unit target만 검증하는 최소 build입니다.

```bash
./configure \
  --target-list=aarch64-softmmu \
  --without-default-features \
  --disable-docs \
  --disable-werror

ninja -C build tests/unit/test-femu-pqueue
build/pyvenv/bin/meson test \
  -C build \
  --suite unit \
  test-femu-pqueue \
  --print-errorlogs

./scripts/checkpatch.pl \
  --strict \
  --no-tree \
  -f tests/unit/test-femu-pqueue.c
```

## 테스트가 검출하는 회귀

`change-preupdated-priority`는 FEMU FTL 호출부처럼 node의 priority를 먼저 갱신하고 queue에 통지합니다. 과거 구현처럼 old priority와 new priority만 비교하면 두 값이 같아져 아래 방향 repair를 선택하고 min-heap invariant가 깨집니다. 테스트는 변경된 node가 root로 이동하고 전체 heap이 유효한지 확인합니다.

`randpop-bubbles-replacement`는 random removal 위치에 마지막 node가 들어왔을 때 그 node가 새 parent보다 높은 priority를 갖도록 heap을 구성합니다. 아래 방향 repair만 수행하면 invalid heap이 되며, 현재 구현은 replacement를 위로 이동시켜 invariant를 복구합니다.

## 검증 범위

이번 결과는 FEMU 내부 자료구조의 unit test입니다. Apple Silicon macOS에서 실행했으므로 x86_64 KVM, guest NVMe I/O, fio throughput, NAND timing을 검증한 결과로 해석하지 않습니다. Full-system FEMU 성능 실험은 별도 [FEMU SSD I/O Path Lab](https://github.com/shining-b-02/femu-ssd-io-path-lab)에서 수행합니다.

## 분석 리포트

버그 원인, 테스트 설계, source-level 연결, 검증 범위, upstream 상태는 [docs/CONTRIBUTION_ANALYSIS_KO.md](docs/CONTRIBUTION_ANALYSIS_KO.md)에 정리했습니다.
