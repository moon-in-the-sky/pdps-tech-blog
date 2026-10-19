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

## 02편(React) 원고의 근거 — 2026-10-19 확인

- 저장소: `moon-in-the-sky/pdps`
- 확인한 브랜치·커밋: `claude/code-fix-planning-3q7615`, `f1894bf8ced8c503074e47b276d2b7c877a3cdf3` (2026-10-19T10:31:26Z)
- 이 브랜치의 원격 최신은 `86deb67`이며, `f1894bf..86deb67`은 `web/`, `api/main.py`, `TECH_STACK.md`, `deploy/`를 바꾸지 않았다. 따라서 아래 근거는 `86deb67`에서도 같다.
- 위 "고정 커밋"(`9a33fd1`)은 이번 확인에 쓴 로컬 복제본에 없어 두 커밋을 직접 비교하지 못했다. 같은 코드(`DataDownload.jsx`)의 줄 번호는 `L33-L75`에서 `L36-L81`로 달라졌으므로, 02편 이후 원고는 `f1894bf`의 링크를 사용한다.
- 확인 방식: 정적 읽기. 빌드·실행·성능 측정은 하지 않았다.

| 원고의 주장 | 코드 | 비고 |
| --- | --- | --- |
| 상위 컴포넌트가 507줄, `useState` 16개 | `ParamExplorer.jsx` (전체) | 사람이 센 값. 코드가 바뀌면 재확인 |
| `ModuleGrid`에 전달하는 props 27개 | `ParamExplorer.jsx` L451-L464 | 사람이 센 값 |
| 선택 상태를 부모가 보관하고 트리·목록에 전달 | `ParamExplorer.jsx` L26, L436-L464 | |
| 계산할 수 있는 값은 상태가 아님(`linkCounts`, `currentModules`) | `ParamExplorer.jsx` L69-L77 | |
| 펼침 상태 기본값을 채워 넣음(주석에 이유) | `ParamExplorer.jsx` L51-L64, L103 | |
| `Set`을 복사해 갱신 | `ParamExplorer.jsx` L129-L133 등 | |
| 폼 초깃값 변경 시 `key`로 재생성 | `ParamFormPanel.jsx` L6-L11, `ParamExplorer.jsx` L477-L486 | 주석에 이유 기록 |
| 요청 순번(`begin`, `isCurrent`)을 데이터·오류·로딩에 적용 | `requestSequence.js` L1-L11, `DataDownload.jsx` L36-L81 | 요청 취소가 아님 |
| 위 훅을 6개 파일에서 사용 | `Notices`, `DataUpload`, `DataDownload`, `LogModal`, `ChangeLogModal`, `ReprocessModal` | |
| 모듈 전환 시 `selIdRef`로 오래된 응답 무시 | `ParamExplorer.jsx` L80-L91, L368 | 훅과 다른 구현 |
| 다운로드는 `AbortController`로 취소 | `Export.jsx` L352-L379, `api.js` L101-L108 | 서버 쪽 정리는 미검증 |
| 라우팅과 권한 확인 컴포넌트 | `App.jsx` L39-L84 | |
| `fetch` 래퍼: 토큰 첨부, 401 처리, 업로드는 XHR | `api.js` L25-L45, L66- | |
| 의존성 6개(실행)·10개(개발), 잠금 파일 327개 | `web/package.json`, `web/package-lock.json`(lockfileVersion 3) | 설치 용량·빌드 시간은 미측정 |
| FastAPI가 `web/dist`를 제공, 캐시 헤더, SPA 대체 응답 | `api/main.py` L79-L122 | |
| 폰트를 CDN 대신 로컬 패키지로 포함(주석) | `index.css` L1-L14 | |
| Vite 개발 서버가 `/api`를 8000 포트로 전달 | `vite.config.js` | |
| 브라우저 지원 정책(Firefox 52+, Edge 16+, Chrome 60+, Safari 제외) | `docs/수정사항/13-frontend-vite.config-2026-09-07.md`, `13-frontend-tailwind.config-2026-09-07.md` | 사용자 확정 정책의 기록 |
| Windows 배포는 설계안이며 스크립트가 없음 | `deploy/windows/README.md`(상태: 설계안), `deploy/windows/`에 README만 있음 | 구현·검증은 72편 |

### 문서 사이의 불일치 (원고에서 한쪽을 단정하지 않음)

1. `TECH_STACK.md` §3.4는 상태 관리를 "Context API(`AuthContext`) — 전역 상태가 로그인 사용자 하나뿐"이라고 쓴다. 현재 코드에는 `AuthContext`와 `UploadQueueContext`가 있다.
2. `TECH_STACK.md` §4는 프론트엔드 의존성을 사내 미러나 사전 다운로드 패키지로 오프라인 설치한다고 쓰고, `deploy/windows/README.md`는 빌드 PC에서 빌드한 결과물을 반입하므로 서버에 Node.js가 필요 없다고 쓴다. 전제가 다르다.

### 확인하지 않은 것

- Vite·React Router·Recharts·Tailwind CSS·자체 `fetch` 모듈을 선택할 당시의 후보와 이유. `TECH_STACK.md` §3.4의 선택 이유 표는 작성 시점과 작성자를 확인하지 못했으므로 "문서에 기록된 이유"로만 쓴다.
- `legacy` 플러그인이 만드는 두 벌의 번들(문서 설명). 빌드 결과물은 열어 보지 않았다.
- 다운로드 취소 시 서버 생성기가 정리된다는 `api.js` 주석의 내용.
- 현재 화면을 실제로 배포하는 절차.

## React 도입 이력 — 2026-10-19 확인

`pdps` 저장소의 얕은 복제본(85개 커밋)을 `--depth=2000`으로 늘려 `claude/code-fix-planning-3q7615` 브랜치의 전체 이력(1,150개 커밋)에서 확인했다. 작성자는 모두 Claude 세션이다.

| 시각(UTC) | 커밋 | 내용 |
| --- | --- | --- |
| 2026-06-26 02:07 | `3157666` | CH10 병렬 처리 파이프라인 초기 커밋 |
| 2026-06-26 02:57 | `9ce4535` | React 프론트엔드 전체 구현(페이지 9개, 목업 인증, react-router-dom 7, Recharts). 메시지에 도구 선택 이유 없음 |
| 2026-06-26 02:57 | `66134b7` | Vite 설정. React 19, Vite 8, Tailwind CSS 4(`@tailwindcss/vite`), `/api` → `localhost:8000` 프록시 |
| 2026-06-26 05:50 | `31a9697` | FastAPI 백엔드와 프론트 연동 |
| 2026-06-29 00:02 | `1a9376e` | 폐쇄망 배포를 위해 FastAPI가 React 빌드 결과물을 제공 |
| 2026-06-29 00:39 | `dba96ec` | 구형 브라우저 지원 추가 |
| 2026-06-29 02:25 | `ef86f6a` | 구형 브라우저 지원을 위해 Tailwind CSS 4를 3으로 내림 |
| 2026-06-30 06:04 | `ec67cb3` | `TECH_STACK.md` 추가(선택 이유 표 포함, React 행 없음) |

- 첫 커밋에 Vite React 템플릿 잔재가 있다: `web/src/App.css`의 `.counter`, `web/src/assets/react.svg`, `vite.svg`, `hero.png`.
- 선택 이유나 대안 비교는 커밋 메시지·문서 어디에도 없다. 가장 이른 이유 서술은 4일 뒤의 `TECH_STACK.md`이며 사후 정리다.
- 도입 세션의 대화는 읽을 수 있으나, 한 달 분량이라 초반 대화(디자인 PDF 첨부 포함)는 확인하지 못했다. 세션 식별자는 공개 저장소에 적지 않는다.
- 디자인 PDF는 `pdps` 저장소 전체 이력에 파일로 추가된 적이 없다.

