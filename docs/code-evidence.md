# 코드 검토 근거와 표현상 주의점

이 문서는 앞선 대화의 정적 코드 검토 결과를 보존한다. 이번 저장소 문서 준비에서 코드를 재실행하거나 최신 HEAD를 다시 검토하지 않았다.

- 저장소: `moon-in-the-sky/pdps`
- 검토 당시 브랜치: `claude/code-fix-planning-3q7615`
- 고정 커밋: `9a33fd147f01ebb4172f50345d7487ba22760e40`
- 해당 커밋 시각: 2026-09-29T06:38:58Z
- 검토 방식: 관련 함수와 호출부의 정적 확인. 전체 저장소의 완전한 검증이나 성능 실험이 아님.

## 확인한 적용 사례

| 개념 | 코드 | 확인한 범위 |
| --- | --- | --- |
| 그래프·DFS·위상 정렬 | `converters/compute/engine.py`, `_toposort`, `run_specs` | 계산 입력 의존성을 탐색하고 선행 계산부터 실행. 방문 상태로 순환 탐지. API에서 자기 참조 검증도 확인. |
| 해시 인덱스·집합 | `metadata/inetx_loader.py`, `metadata/mil1553_loader.py`, 해당 processors | 채널·스트림 또는 명령 복합 키로 정의 조회. 전체 패킷 디코딩이 상수 시간이라는 뜻은 아님. |
| 이진 탐색 | `engine.py` Lookup, `compute/alignment.py`, `readers/formatter.py` 시간 범위 검색 | 정렬된 기준점·시간 레코드에서 구간 검색. 시간 정렬 전제의 검증 필요. |
| 전처리 | `engine.py`, `_compile_lookup` | 정렬·딕셔너리 구성을 spec별로 한 번 수행하고 샘플 루프에서 재사용. |
| LRU | `core/filepool.py`, `core/dump_writer.py`, `core/worker.py` | OrderedDict로 덤프 출력 파일 핸들을 관리. 모든 출력이나 FPB2 저장에 공통 적용된 것은 아님. |
| FIFO 큐 | `api/concurrency.py`, upload·reprocess routers | 같은 API 프로세스의 queue.Queue와 단일 소비자 스레드. 분산·영속 작업 브로커가 아님. |
| 상태 전이·조건부 갱신 | `api/routers/upload.py`, `_run_job_locked`, `recover_stale_jobs` | waiting인 행만 processing으로 갱신. 취소된 대기 작업 실행 방지. DB 상태와 메모리 큐를 구분. |
| 세마포어 | `readers/formatter.py`, datasets router | Export 슬롯 제한과 종료 시 반환. 프로세스별 제한이며 전 서버 통합·공정성 보장은 아님. |
| 작업 분할·프로세스 풀 | `core/parallel_runner.py`, `core/worker.py` | 청크 처리, spawn Pool, 완료 순서와 청크 순서 구분. 자체 Work Stealing 구현으로 부르지 않음. |
| mmap·memoryview | `core/worker.py` | 입력 파일 매핑과 뷰 사용·정리. 처리 전체의 무복사나 일정 메모리 상한을 뜻하지 않음. |
| 파일 인덱스·지역성 | `writers/param_binary_writer.py`, `readers/fpb2_reader.py`, formatter | 파라미터 블록과 오프셋·레코드 수. 디렉터리 탐색은 별도 비용. B-tree·DB 구현으로 부르지 않음. |
| 진행 중 작업 수 제한 | `readers/formatter.py`, `_iter_zip_parallel` | 병렬 Export의 pending 수 제한. 파라미터 하나의 CSV 전체를 메모리에 구성하므로 바이트 상한과 다름. |
| 관계·제약 | `api/models.py` | 프로젝트–모듈 관계, 복합 외래 키, 유일성 제약 선언. 운영 DB에 적용된 상태까지 확인한 것은 아님. |
| 연결 수명 | `api/db.py`, datasets `_export_prepare` | 세션 정리와 파일 작업 전에 필요한 메타데이터를 확보하고 DB 연결을 반환하는 경로. |
| 요청 번호 | `web/src/lib/requestSequence.js`, `web/src/pages/DataDownload.jsx` | 최신 요청의 응답만 데이터·오류·로딩 상태에 반영. 이전 네트워크 요청 취소와는 다름. |
| 공유 상태·불변 갱신 | `web/src/pages/paramExplorer/ParamExplorer.jsx` | 선택 상태를 상위에서 관리하고 트리·목록에 전달. Set 복사 후 갱신. |
| 비트 연산·부호 확장 | `converters/compute/operators/bitwise.py` | 마스크, 추출, 필드 결합, 선택적 부호 확장. 모든 출력 자료형의 정밀도를 보장하는 것은 아님. |
| 정수 시간·반복 계산 캐시 | `readers/formatter.py` | 정수 기반 ns 구성, 초 단위 IRIG 변환 재사용, 포맷 함수 사전 구성. |

## 중요한 고정 코드 링크

- [위상 정렬](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/converters/compute/engine.py#L264-L286)
- [Lookup 사전 구성](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/converters/compute/engine.py#L133-L172)
- [해시 인덱스](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/metadata/inetx_loader.py#L111-L121)
- [파일 기반 이진 탐색](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/readers/formatter.py#L673-L738)
- [LRU](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/core/filepool.py#L85-L118)
- [작업 큐](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/api/concurrency.py)
- [작업 시작 상태 갱신](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/api/routers/upload.py#L698-L711)
- [Export 실행 제한](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/readers/formatter.py#L343-L387)
- [병렬 Export](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/readers/formatter.py#L396-L481)
- [프로세스 풀](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/core/parallel_runner.py#L257-L310)
- [모델과 제약](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/api/models.py#L155-L205)
- [React 응답 순서 제어](https://github.com/moon-in-the-sky/pdps/blob/9a33fd147f01ebb4172f50345d7487ba22760e40/web/src/pages/DataDownload.jsx#L33-L75)

## 원고 작성 전에 다시 확인할 사항

1. 시간 정렬: 이 커밋의 `engine.py:361`은 `align = hold if kind == "int" else interp`이며 실제 샘플 루프에서 호출한다. 사용자가 정한 derived ZOH 정책과 구현 상태를 재확인한다. Lookup 기준점의 값 보간과 혼동하지 않는다.
2. 메모리 사용: FPB2 writer는 bytearray 버퍼를 누적한다. 계산 엔진도 일부 시계열을 list로 구성한다. 입력에 mmap을 사용한다고 전체 파이프라인이 일정 메모리를 쓰는 것은 아니다.
3. 스트리밍: ZIP의 외부 전달과 파라미터 CSV의 내부 생성 방식을 구분한다. 병렬 경로는 파라미터 하나 전체를 bytes로 만들어 전달한다.
4. 시간 검색: 이진 탐색의 정렬 전제를 확인한다. 시간 역행이 있는 입력에 대한 정확성을 정적 코드만으로 확정하지 않는다.
5. DB: 모델 선언과 실제 마이그레이션·운영 DB 제약 적용은 별도로 확인한다. 전체 3NF나 ACID 엔진 구현을 주장하지 않는다.
6. 성능: 정적 코드 구조를 성능 개선의 실측 근거로 쓰지 않는다. 개별 수치는 입력·환경·커밋·동시성 조건과 재확인한다.
7. 동시성: 프로세스별 제한을 서버 전체의 분산 제어로 설명하지 않는다. 모든 경합·취소·복구가 검증됐다고 단정하지 않는다.

## 확장 학습의 표현

현재 코드를 소재로 원리를 학습할 수 있지만, 힙 기반 병합·MVCC·리틀의 법칙·정규형 검증·정식 상태 머신 프레임워크·자원 샌드박스 등을 직접 구현했다고 표현하지 않는다. 실제 적용·실험 결과가 확보되면 근거와 함께 문서를 갱신한다.
