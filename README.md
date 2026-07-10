# FSS 소비자경보 → Notion 동기화

금융감독원 소비자경보 게시판(`B0000175`)을 크롤링해서 Notion `FSS 소비자경보 DB`에 신규 글을 push하고, 기존 글의 변경(조회수 등)을 patch한다.

- Notion 대상: `Consumer_alert > FSS 소비자경보 DB`
- 참고 저장소: [`../fss_monitor/`](../fss_monitor/) — 동일 아키텍처의 보도자료(B0000188) 파이프라인

## 실행 방식
- **GitHub Actions cron** — 매 4시간마다 자동 실행 (KST 07:30 / 11:30 / 15:30 / 19:30 / 23:30 / 03:30) — [`.github/workflows/crawl.yml`](.github/workflows/crawl.yml)
- **수동 실행** — Actions 탭에서 `Run workflow` (workflow_dispatch)
- **로컬 테스트** — 환경변수 세팅 후 `python run_headless.py`

## 파일 구성
| 파일 | 역할 |
|---|---|
| `config.py` | 게시판 URL(B0000175, menuNo=200204), HTTP 헤더, Notion property 매핑, NEW 배지 기간 |
| `crawler.py` | 목록 페이지 파싱 (`requests` + `BeautifulSoup`, caption 힌트: `소비자경보`) |
| `sync_notion.py` | Notion API wrapper — 스냅샷 로드, diff, create/update, sentinel 갱신 (fss_monitor와 동일) |
| `run_headless.py` | 진입점 — 3-way 분기 (신규 create / 변경 update / 오래된 신규 해제) |
| `.github/workflows/crawl.yml` | Actions 워크플로 (4h cron + workflow_dispatch) |

## Notion DB 스키마

`Consumer_alert > FSS 소비자경보 DB`

| 컬럼 | 타입 | 내용 |
|---|---|---|
| 제목 | Title | 게시글 제목 (예: `(소비자경보 2026-21호) …`) |
| 번호 | Number | 게시판 번호 |
| 등록일 | Date | FSS 게시 등록일 |
| 담당부서 | Text | 담당부서 |
| 조회수 | Number | 조회수 (매 sync 시 갱신) |
| 원문 링크 | URL | FSS 원문 URL |
| 신규 | Checkbox | 최초 수집 이후 24시간 이내면 true |
| 최초 수집 | Date | 크롤러가 처음 발견한 시각 (KST) |
| nttId | Rich text | 중복 방지 키 (FSS 게시글 고유 ID) |

## GitHub Secrets

저장소 `Settings → Secrets and variables → Actions` 에서 등록:

| Name | 필수 | Value |
|---|---|---|
| `NOTION_TOKEN` | ✅ | Notion Integration secret (`ntn_...`) — fss_monitor와 동일 값 재사용 가능 |
| `NOTION_DB_ID` | ✅ | `FSS 소비자경보 DB`의 database ID → `cedd16dc970742b09d2ea7b77de4e539` |
| `NOTION_SUMMARY_PAGE_ID` | ⏸ 선택 | `Consumer_alert` 페이지 ID → `3990f4111b5380c3b732ce8ae933ba62` — 설정 시 매 sync에서 요약 통계 자동 갱신 |
| `NOTION_MENTION_USER_ID` | ⏸ 선택 | Notion 사용자 UUID (CSV 지원 — 콤마 구분으로 다중 대상) — 설정 시 신규 push된 페이지마다 해당 사용자에게 `@mention` 코멘트 자동 작성 |

> Integration이 대상 DB **그리고** 요약 페이지에 각각 연결되어 있어야 한다: 각 페이지 → `···` → `Connections` → Integration 추가.

## 로컬 테스트
```powershell
cd consumer_alert
pip install -r requirements.txt
$env:NOTION_TOKEN = "ntn_..."
$env:NOTION_DB_ID = "cedd16dc970742b09d2ea7b77de4e539"
$env:NOTION_SUMMARY_PAGE_ID = "3990f4111b5380c3b732ce8ae933ba62"   # optional
python run_headless.py
```

## 조정 파라미터 (`config.py`)
- `PAGES_TO_FETCH` — 크롤할 목록 페이지 수 (기본 2)
- `LIST_CAPTION_HINT` — 리스트 테이블 caption 매칭 문자열 (기본 `소비자경보`)
- `NEW_BADGE_HOURS` — 신규 체크 유지 시간 (기본 24)
- `NOTION_RATE_DELAY` — Notion API 호출 간격 초 (기본 0.35)
- `NOTION_PROPS` — 실제 DB 컬럼명과의 매핑
