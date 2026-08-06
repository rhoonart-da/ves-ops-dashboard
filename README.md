# VES 운영 대시보드 — 시스템 설명서

맥미니 6대가 밤새 만든 쇼츠를 **사람이 검수하고, 합격분이 자동 발행되기까지**를 한 화면에서
보고 조작하는 팀 공용 대시보드. 2026-08-03 목업 → 08-06 현재 운영 중.

> **이 문서의 목적**: 이 시스템은 [VES 통합 아키텍처](#8-통합-아키텍처로의-승계)의
> *Phase 2 「검수 대시보드 v1」* 에 해당한다. 통합 작업 때 **무엇이 무엇의 전신인지**,
> **옮기면서 깨뜨리면 안 되는 규칙이 무엇인지**를 남긴다.
> 화면 코드 원본은 brain 레포 `deploy/edge-dashboard/` — 이 레포는 배포본만 담는다.

---

## 1. 구성

```
 브라우저 ──── GitHub Pages (이 레포)          Supabase Edge Function `dashboard`
              index.html 단일 파일       ──▶   API 전용 (JSON) · CORS 허용
              React 18 + htm (esm.sh)          x-ves-key 헤더 인증
              빌드 없음 · 드래그 배포 없음
                                                  │  읽기          │ 쓰기 3곳
                                                  ▼                ▼
                          fdidiqd (파이프라인 DB) · laeebly (원천, 읽기전용 세션 강제)
                          YouTube Data API v3 · GitHub API(선택)
```

**왜 화면과 API 가 갈라져 있나** — Supabase 기본 도메인은 `text/html` 을 `text/plain` 으로
강제한다(피싱 방지). 그래서 HTML 서빙이 불가능해 화면은 GitHub Pages, 함수는 API 전용으로 나눴다.
합칠 때 커스텀 도메인을 붙이면 한 곳으로 모을 수 있다.

**인증** — 접속 코드 하나(`DASHBOARD_PASSWORD` 시크릿)를 `x-ves-key` 헤더로. 팀 공용이라
계정 개념이 없다. 검수자 이름은 사람이 직접 입력해 결정 행에 기록한다(신원 증명 아님).
브라우저는 헤더에 non-ASCII 를 못 담으므로 화면이 `encodeURIComponent` 로 보내고 서버가 되돌린다.

---

## 2. 데이터 원천과 신뢰 순서

| 원천 | 쓰임 | 성질 |
|---|---|---|
| `clips` · `clip_metadata` (fdidiqd) | 영상 피드 · 검수 목록 · 회차 | **영구 기록. 가장 신뢰** |
| `review_decisions` (fdidiqd) | 합격/반려 · 사유 · 검수자 | 사람 결정의 정본 |
| `machine_heartbeats` (fdidiqd) | 머신 상태 · 실행 결과 · 상태 스냅샷 | **휘발적** — 머신당 최신 1건만 유효 |
| `youtube_studio` (laeebly) | 채널 성과(조회수·apv·kwr) | 읽기 전용, ETL 지연 있음 |
| YouTube Data API | 공개 여부 · 실제 공개 시각 · 채널 아바타 | 실시간, 쿼터 있음 |
| GitHub API | 커밋 목록 (선택) | 토큰 없으면 비활성 |

> ⚠ **하트비트는 사실의 원본이 아니다.** 같은 사실이 DB 에도 있으면 DB 를 믿는다.
> 실측 사고 2건: ① 발행 이력을 하트비트 스냅샷에서 읽었더니 다음 실행이 스냅샷을 덮어
> 어제 기록이 사라졌다 → 피드(`clips.published_at`)로 교체. ② 실패를 "머신당 최신 비트"에서만
> 읽었더니 재생성 실행이 새 비트를 남기는 순간 새벽 실패가 통째로 사라졌다 → **오늘 전 비트**에서 수집.

---

## 3. 화면

| 탭 | 하는 일 |
|---|---|
| **홈** | 6대 상태 보드(채널별 신호등 2점) · 드릴다운 · 검수 대기 KPI · 날짜 선택(과거 마감 기록) |
| **검수함** | Storage 사본 재생 → 합격/반려 클릭. judge 점수·사유 표시(참고용), 맥 탭, 사유 수정 |
| **영상 피드** | 전체 탭 = 최신순 격자 / 맥 탭 = 채널별 가로 스트립(오른쪽이 최신) |
| **채널 성과** | laeebly 기준 채널 리그 + 채널별 영상 드릴다운 |
| **머신 로그** | 최근 실행의 채널별 결과 · 실패 상세 · 무보고 경보 · 상태파일 이상 |
| **커밋·배포** | 두 레포 최신 커밋 + 6대 pull 매트릭스(SHA 대조) |

---

## 4. 신호등 규칙 (홈 보드의 핵심 로직)

채널마다 점 **2개**: 왼쪽 = **생성**(공장이 도는가), 오른쪽 = **공개**(오늘 몫이 나갔는가).
색은 두 열에서 **같은 뜻**이고, 어느 단계인지는 카드의 열 머리글이 말한다.

```
● 초록 완료   ● 노랑 대기   ◉ 빨강링 반려   ● 빨강 실패   ● 회색 없음
○ 링(도넛) = 재생성 사이클을 거친 뒤의 상태
```

**판정은 언제나 "오늘(KST)" 기준이다.** 어제의 초록이 오늘의 문제를 가리면 안 된다.
지난 이력은 드릴다운(시각 표기)이 담당한다.

생성 점 결정 순서:

1. 오늘 **부정 이벤트**(실패 `failed` · 중복 보류 `dup_hold`)가 있었나?
   - 없으면 → 검수 대기 > 반려 > 합격/발행 > 없음
   - 있으면 → 그 **이후에** 온 신호를 찾는다
     - 재생성분이 검수 대기 → **노랑 링**
     - 재생성분이 합격/발행 → **초록 링**
     - 아무 신호 없음 → 실패면 **빨강**, 중복 보류면 **회색 링**

`dup_hold` = 3회 재생성이 전부 중복이라 아무것도 못 건진 상태. 그냥 넘어간 채널(`skipped`)과
같은 회색으로 묻혀 있던 것을 분리했다(2026-08-06). **소재 소진의 신호**라 사람이 봐야 한다.

---

## 5. 쓰기 경로는 3개뿐

이 시스템은 **거의 전부 읽기**다. 쓰는 곳은 셋:

| 경로 | 쓰는 것 | 성격 |
|---|---|---|
| `POST /api/decision` | `review_decisions` upsert (합격/반려 · 유형 · 사유 · 검수자 · 아이콘) | 사람 결정 |
| `POST /api/note` | 같은 행의 `note` 만 | 결정 불변, 사유만 수정 |
| `POST /api/snapshot-daily` | `dashboard_daily_snapshots` upsert | 23:55 KST 크론, 멱등 |

`/api/snapshot-daily` 는 **무인증**이다 — pg_cron(pg_net)이 호출하며 시크릿을 알 수 없고,
오늘 날짜의 마감 기록을 라이브 데이터로 재계산해 덮어쓸 뿐이라 멱등·무해하다.

**대시보드는 유튜브를 건드리지 않는다.** 발행도, 공개 전환도, 삭제도 하지 않는다.
결정만 DB 에 남기고 실제 발행은 각 맥의 픽업이 한다.

---

## 6. 검수 → 발행 파이프라인 (대시보드가 끼어 있는 자리)

```
밤 생성(scene_loop)
   └ 장면 확정 즉시 → upload_review_clips
        ├ ingest_aivideo_run (provenance 적재)
        ├ Storage `review-clips/<machine_id>/<clip_id>.mp4` 업로드 + clips.storage_path 기록
        └ judge 선실행 (표시용)
   ▼
검수함에 등장  ──▶  ⚑ 사람: 합격 / 반려(장면·제작) + 사유
   ▼
scene_publish_loop (launchd 10분 픽업, 6대 전부)
   ├ 합격 → publish_youtube: private + publishAt 예약 업로드
   │        · 오늘 몫이 비었으면 지금+5분, 아니면 다음 19:00
   │        · 발행 확인되면 Storage 사본 자동 정리(자기 머신 것만)
   └ 반려 → 상태파일에 반려 스탬프 → 슬롯 즉시 해제 → 그 밤 대체 장면 생성
            · 장면 반려: 그 구간 재사용 금지  · 제작 반려: 같은 구간 재시도 허용
```

지켜야 할 것:

- **공개 전환은 예약(publishAt)으로만 자동화된다.** 이미 올라간 영상의 공개 상태를 코드로
  바꾸는 것(`videos.update`)은 토큰 권한이 없어 실패한다 — 업로드 시점 예약이 유일한 우회로다.
- **judge 는 표시·안전게이트 전용.** 성과 예측이 아니며 자동 합격/반려·정렬 강제에 쓰지 않는다.
- **결정은 100% 사람.** 자동 판정 경로가 없다.

---

## 7. 스키마 · 배포

**마이그레이션** (brain `docs/migrations/`, 전부 가산적):

| 번호 | 내용 |
|---|---|
| 0007 | `machine_heartbeats` — 실행 시작/종료 비트, 채널별 결과, 상태·발행 스냅샷 |
| 0008 | `review_decisions` + `clips.storage_path` |
| 0009 | `review_decisions.reject_type` (`scene` \| `production`) |
| 0010 | `clips.source_episode` — 원작 회차(`clips.episode` 는 쇼츠 라벨이라 회차가 아님) |
| 0011 | `dashboard_daily_snapshots` + pg_cron 23:55 KST |
| 0012 | `review_decisions.reviewer_icon` — 검수자 프로필(표시 전용) |

**배포**

- 화면: 이 레포 `index.html` 을 푸시 → GitHub Pages 자동 반영(1~2분). 빌드 없음.
- API: brain `deploy/edge-dashboard/index.ts` 를 Supabase Edge Function `dashboard` 로 배포
  (`verify_jwt=false`).
- 시크릿: `DASHBOARD_PASSWORD` · `LAEEBLY_DB_URL` · `YOUTUBE_API_KEY` · `GITHUB_TOKEN`(선택).
  값 끝의 개행이 딸려오기 쉬워 서버가 전부 `trim()` 한다.

---

## 8. 통합 아키텍처로의 승계

지금 조각들은 대부분 **통합 아키텍처의 전신**이다. 옮길 때 대응:

| 지금 (이 시스템) | 통합 아키텍처 | 옮길 때 |
|---|---|---|
| `machine_heartbeats` | `node_registry` + `job_events` | 심박·capability 는 그대로, 채널별 결과는 잡 이벤트로 분해 |
| `review_decisions` | `review_queue` (publish_gate 결과) | **테이블을 유지하고 참조만 바꾸는 쪽을 권한다** — 반려 유형·사유가 이미 dedup·재생성 정책에 물려 있다 |
| Storage `review-clips` | `artifacts` 카탈로그 | 키 규칙(`<machine>/<clip_id>.mp4`)은 유지 — 한글 키는 Storage 가 거부한다 |
| `scene_publish_loop` 10분 픽업 | `job_queue` claim (publish 잡) | 픽업 주기 10분은 폴링 3초로 흡수. 락(mkdir)은 `FOR UPDATE SKIP LOCKED` 로 대체 |
| `clips.source_episode` | `work_orders` 의 회차 | 회차는 지금 상태파일이 유일한 출처 — 지시서가 생기면 그쪽이 정본 |
| `dashboard_daily_snapshots` | `job_events` 기반 집계 | 이벤트 로그가 생기면 스냅샷은 캐시로 격하 가능 |
| 신호등 규칙(§4) | `job_queue.status` + `review_queue` | **규칙 자체는 살려라** — 오늘 기준·재생성 링·중복 보류는 실운영에서 하나씩 벌어 얻은 것 |

**옮겨도 유지해야 할 불변**

1. 공개 전환은 사람(지오블락이 업로드 UI 에서만 설정 가능).
2. judge 는 안전게이트·표시 전용 — 승격·자동 결정 금지.
3. 검수 결정은 사람 100%, 결정 데이터가 발행 게이트의 유일한 입력.
4. 발행이 안 되는 방향이 안전한 방향 — 조회 실패 시 발행하지 않고 멈춘다.
5. 신호등은 "오늘" 기준. 과거는 마감 스냅샷으로 본다.

---

## 9. 함정 노트 (실측으로 얻은 것 — 재발 방지)

- **시각**: DB 는 UTC, 맥 상태파일은 오프셋 없는 로컬(KST) 문자열이 섞여 온다. 화면은 전부
  브라우저 로컬로 변환해 표시하고, 서버의 "오늘" 판정은 `kstDayOf()` 하나로 통일했다.
  문자열을 그냥 `slice` 하면 9시간 어긋난다.
- **`dashboard_daily_snapshots.created_at` 은 행이 처음 생긴 시각**이다. upsert 는 payload 만
  갱신하므로 "어젯밤 크론이 돌았나"는 `payload->>'generated_at'` 으로 봐야 한다.
- **검수 목록의 기준은 "사본이 있는 클립"** 인데, 발행된 합격작은 사본이 정리돼 목록에서 빠진다.
  최근 결정은 반드시 `review_decisions` 를 기준으로 뽑고, 조회 창(현재 200건)이 곧 표시 한계다.
- **다작품 슬롯**(`재미쇼츠·유미의 세포들 시즌3`)은 채널명이 아니다. 슬롯의 `channel` 필드가
  정본이고, 없으면 `·` 앞부분을 정본 채널명과 대조한다. 이걸 빼면 그 채널이 통째로 누락된다.
- **CSS**: `box-sizing: border-box` 상태에서 고정폭 타일에 구분선 여백을 주면 그만큼 썸네일이
  좁아진다. 폭을 보정하거나 여백을 바깥에 둘 것.

---

## 10. 미결

운영 중 결정 대기 항목은 brain 레포 `docs/PENDING_DISCUSSIONS.md` 에 모여 있다.
이 시스템과 직접 얽힌 것: 중복 판정에 의미 단위 추가(#8) · 소재 소진 회차 자동 스킵(#9) ·
사람–LLM 판단 비교 뷰(#7) · Vite 빌드 전환(#6).

---

*최종 갱신 2026-08-06 · 원본 코드: `rhoonart-dev/ai-improvement-edit-video` `deploy/edge-dashboard/`*
