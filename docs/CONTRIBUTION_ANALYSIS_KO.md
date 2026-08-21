# FEMU Priority Queue 기여 분석 리포트

## 1. 기여 배경

FEMU의 BlackBox FTL, FDP reclaim-unit 관리, NVMe request scheduling은 공통 priority queue 구현을 사용한다. Heap invariant가 깨지면 잘못된 GC victim 선택, queue corruption, NULL dereference처럼 storage control path 전체에 영향을 줄 수 있다.

최근 upstream에는 두 가지 heap repair 수정이 반영됐다.

1. commit `66067fabc`: `pqueue_change_priority()`가 old/new 값 비교 대신 현재 parent relation으로 repair 방향을 선택하도록 수정
2. commit `a4ed1d8c9`: `pqueue_randpop()`의 replacement가 위로 이동해야 하는 경우를 처리하도록 수정

두 수정 모두 중요하지만 `hw/femu/lib/pqueue.c`를 직접 실행하는 unit test가 없어 이후 refactoring에서 같은 문제가 재발해도 full-system 증상으로만 드러날 가능성이 있었다.

## 2. 구현 내용

### Meson 연결

`tests/unit/meson.build`에 `test-femu-pqueue` target을 추가하고 실제 `hw/femu/lib/pqueue.c`를 test executable에 함께 compile한다. Mock implementation을 사용하지 않기 때문에 production priority queue code 자체가 검증 대상이다.

### Test fixture

`TestNode`는 priority와 heap position만 가진 최소 node다. Comparator는 FEMU의 FDP·line GC·NVMe request queue와 동일하게 `next > current`를 사용해 작은 값이 root에 오는 min-heap을 구성한다.

### Subtest 1: 기본 순서

`40, 10, 50, 20, 30`을 삽입한 뒤 pop 결과가 `10, 20, 30, 40, 50`인지 확인한다. 각 pop 뒤 `pqueue_is_valid()`를 호출해 중간 단계에서도 invariant가 유지되는지 검증한다.

### Subtest 2: 이미 갱신된 priority

`10, 20, 30, 40, 50` heap의 마지막 node priority를 5로 먼저 바꾼 뒤 `pqueue_change_priority()`를 호출한다. 수정 전 logic은 `old_pri == new_pri == 5`를 비교해 percolate-down을 선택하고 parent 20 아래에 priority 5를 남긴다. 현재 logic은 parent와 node를 비교해 bubble-up을 선택한다.

Acceptance condition:

- `pqueue_is_valid(queue) == true`
- 변경 node가 `pqueue_peek()` 결과와 동일

### Subtest 3: Random removal replacement

Heap array가 `1, 4, 2, 5, 6, 3`이 되도록 삽입한 뒤 position 4의 priority 5 node를 random removal 대상으로 선택한다. 마지막 priority 3 node가 position 4를 대체하면 parent priority 4보다 위로 이동해야 한다. Test는 platform별 `rand()` sequence를 가정하지 않고 원하는 position이 나오는 seed를 먼저 찾은 뒤 동일 seed로 실제 함수를 호출한다.

Acceptance condition:

- 제거된 node가 예상 node와 동일
- replacement repair 후 `pqueue_is_valid(queue) == true`

## 3. 실제 실행 결과

| 항목 | 값 |
|---|---|
| 실행일 | 2026-08-21 |
| Host | macOS 26.5.2, arm64 |
| Compiler | Apple clang 17.0.0 |
| QEMU/FEMU base | `39664d242` |
| Test commit | `c268210a4a13c0f0d569c0c315bf2b887621e155` |
| Meson suite | `qemu:unit / test-femu-pqueue` |
| Subtest | 3 |
| PASS | 3 |
| FAIL | 0 |
| 실행 시간 | 0.02초 |
| checkpatch | 오류 0, 경고 0 |

Source SHA-256:

- `tests/unit/test-femu-pqueue.c`: `19e19a1b537e9882da12b541f2ee547c1589cefb6afbac755fbd6ddaa89ddd07`
- `hw/femu/lib/pqueue.c`: `4d607a369fdbd1badb6e5fe1ff474a188af3244f64470869825aad9e650681b7`

## 4. Storage system 관점의 의미

Priority queue는 FTL 정책 자체보다 낮은 계층의 자료구조지만 GC victim ordering과 request expiration ordering의 결정성을 보장한다. 이 계층을 unit test로 분리하면 full FEMU VM을 띄우지 않고도 다음 regression을 빠르게 발견할 수 있다.

- valid-page count가 줄어든 victim이 root로 올라오지 않는 문제
- random victim 선택 뒤 heap 구조가 손상되는 문제
- 동일 node의 position callback이 repair 중 잘못 갱신되는 문제
- pop 결과가 comparator contract와 달라지는 문제

Full-system fio test는 결과 증상을 검출하는 데 유용하지만 원인을 priority queue까지 좁히기 어렵다. 이번 unit test는 source-level invariant를 0.02초에 직접 검사해 디버깅 범위를 줄인다.

## 5. Upstream 제출 상태

- Fork: `shining-b-02/FEMU`
- Branch: `test/pqueue-priority-change`
- Upstream PR: [MoatLab/FEMU #196](https://github.com/MoatLab/FEMU/pull/196)
- 상태: Open, Ready for review, merge conflict 없음
- 로컬 검증: PASS
- Upstream review: 아직 없음
- Upstream check: 아직 보고되지 않음

Fork 전용 한국어 문서 commit은 upstream PR branch에 포함하지 않아 PR 범위를 unit test 두 파일로 유지했다.

## 6. 한계와 다음 검증

현재 test는 자료구조 단위의 correctness를 다룬다. 다음 단계에서는 x86_64 Linux/KVM에서 FDP 또는 BlackBox workload를 실행해 queue repair가 실제 GC victim sequence와 tail latency에 미치는 영향을 trace로 확인할 수 있다. 그 결과도 unit test와 분리해 raw fio JSON, FEMU log, source revision과 함께 보존해야 한다.
