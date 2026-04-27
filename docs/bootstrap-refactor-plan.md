# Bootstrap 리팩터 + NodeJS 자동화 스크립트 도입 계획

> 작성일: 2026-04-27
> 대상 디렉토리: `bootstrap/`
> 향후 진입점: `Web_Starter.exe` → NodeJS 자동화 스크립트 → Python Phase 1/2

이 문서는 새 fork 환경에서 작업을 이어서 진행하기 위한 **계획 명세서**입니다. 기존 `bootstrap/` 디렉토리의 Python 시드 코드를 정리하고, NodeJS 기반 자동화 스크립트가 분기 호출할 수 있는 구조로 재설계합니다.

---

## 1. 최종 요구사항 (Web_Starter.exe 흐름)

```
[1] Web_Starter.exe 시작
        ↓
[2-1] NodeJS 자동화 스크립트가 DB 검증 시작
        ↓
[2-2] *_seed.py를 기준으로 두 가지 체크
        ├ 테이블 컬럼 검증: seed 파일이 기대하는 테이블/컬럼이 DB에 존재하는가?
        └ 데이터 수 검증: seed 파일의 기대 row 수보다 실제 DB row가 1개라도 적은가?
        ↓
[2-3] 검증 실패 시 → 분기 실행 (아래 "분기 로직" 참고)
        ↓
[3] 자동화 스크립트가 다시 DB 검증 (재검증)
        ↓
[4] Web_Starter.exe가 backend / frontend 실행
```

---

## 2. 분기 로직 (Phase 1 / Phase 2)

| 검증 결과 | 호출되는 Phase | 의미 |
|---|---|---|
| **테이블 자체 없음** 또는 **컬럼 누락** | **Phase 1 → Phase 2 순차 호출** | 빈 테이블부터 만들어야 데이터 INSERT 가능 |
| **테이블·컬럼 OK, 데이터만 부족** | **Phase 2만 호출** | 테이블은 그대로 두고 부족한 row만 보충 |
| **모두 정상** | **둘 다 스킵** | 바로 [3] 재검증으로 |

### Phase 1 — 빈 테이블 생성 (`bootstrap/create_tables.py`)
- `Base.metadata.create_all()` 사용 → `CREATE TABLE IF NOT EXISTS` 의미
- 기존 테이블·데이터 절대 안 건드림 (멱등)
- 데이터 INSERT 0건
- NodeJS 호출: `python bootstrap/create_tables.py` (인자 없음)

### Phase 2 — 더미 데이터 INSERT (`bootstrap/insert_data.py`)
- `ON CONFLICT DO NOTHING/UPDATE`, upsert 패턴 사용
- 기존 row 보존, 누락분만 추가 (멱등)
- 테이블이 존재한다고 가정
- NodeJS 호출: `python bootstrap/insert_data.py` (인자 없음)

### 안전 보장
- **둘 다 가산형(additive) 설계 → 어떤 시나리오에서도 데이터 손실 없음**
- 시나리오 D(테이블·데이터 모두 정상인데 자동화 스크립트가 두 진입점 모두 호출)에서도 안전
- 단, `Base.metadata.create_all()`은 **컬럼 변경(ALTER) 못 함** — 컬럼 drift는 NodeJS가 경고만 표시하고 멈추는 게 안전 (자동 ALTER는 데이터 손실 위험)

---

## 3. 파일별 처리 방침

> ⚠️ **`bootstrap/Old_BootStarpBackup/`는 절대 손대지 않음** (백업 파일, 유지)

### 3.1 신규 작성

| 파일 | 역할 |
|---|---|
| `bootstrap/create_tables.py` | Phase 1 진입점. 모든 모델 import 후 `Base.metadata.create_all()` |
| `bootstrap/insert_data.py` | Phase 2 진입점. 각 도메인의 `seed_xxx()` 함수 순차 호출 |

### 3.2 삭제 대상

| 파일 | 이유 |
|---|---|
| `bootstrap/_bootstrap_common.py` | ORM 0줄, 100% psql/subprocess 오케스트레이션. NodeJS가 대체 |
| `bootstrap/reset_db.py` | ORM 0줄, `DROP SCHEMA public CASCADE`만. NodeJS가 대체 |
| `bootstrap/__pycache__/` | _bootstrap_common 캐시 정리 |

### 3.3 리팩터 (CLI/오케스트레이션 제거 + 단일 함수 노출)

| 파일 | 보존 | 제거 | 노출 함수 |
|---|---|---|---|
| `bootstrap/farmos_seed.py` | `USER_SEEDS`, User upsert, 모델 import | argparse, mode 분기, drop/truncate, NCPMS subprocess, pesticide subprocess, summary 출력 | `seed_farmos_users()` |
| `bootstrap/ncpms_seed.py` | `pg_insert + on_conflict`, `_build_payload`, `_load_json_rows` | `_bootstrap_common` import (→ 단순 print로) | `seed_ncpms()` |
| `bootstrap/seed_ai_agent.py` | `seed()`, `_make_decision()`, raw SQL INSERT 라인 | `main()`의 `sys.argv` 처리 | `seed_ai_agent()` |
| `bootstrap/shoppingmall_seed.py` | 모든 데이터 상수, `seed_core_data`, `seed_backoffice_data`, 모델 import | argparse, mode 분기, `truncate_existing_shop_data`, `run_*_pipeline`, `print_db_summary`, `_to_sync_db_url` | `seed_shoppingmall()` |
| `bootstrap/shoppingmall_review_seed.py` | `generate_all_reviews`, raw SQL INSERT | `main()`, `set_log_prefix`, **DELETE 라인** (멱등 INSERT로 변경) | `seed_shoppingmall_reviews()` |

### 3.4 특별 처리 — `bootstrap/pesticide.py`

**결정: Option A — 모델 정의만 남기고 슬림화 (~200줄)**

`pesticide.py`는 1215줄 풀 ETL 스크립트인데, 그중 모델 정의는 113~310줄 약 200줄뿐입니다. JSON raw 파일을 git에 올릴 수 없어서 데이터 적재는 자동화에서 제외합니다.

#### 슬림화 후 남길 내용
- `class Base(DeclarativeBase)`
- 5개 모델 클래스: `Product`, `Crop`, `Target`, `ProductApplication`, `RagDocument`
- 새 함수: `create_pesticide_tables(engine)` — `Base.metadata.create_all(engine)` 한 줄
- 필요한 SQLAlchemy import만

#### 삭제할 내용 (~1000줄)
- argparse PARSER 정의
- 데이터 정제 헬퍼 (clean_text, parse_date, normalize_keyword 등)
- `Stats`, `UpsertCaches` dataclass
- `build_product`, `upsert_product`, `upsert_application`, `upsert_rag_document`
- `populate_database`, `iter_pairs`, `get_or_create_*`
- `build_engine`, `wait_for_postgres_server`, `load_backend_env`, `normalize_db_url`, `first_non_empty`
- `rebuild_tables` (drop_all 포함)
- `collect_source_files`, `load_rows`, `parse_range_from_result_path`, `make_product_id`, `hash_raw_row`
- `build_search_text`, `build_render_text`
- `main()` 함수 + `__main__` 진입
- `_bootstrap_common` import

#### 데이터 적재가 필요할 때 (수동)
풀 ETL 버전은 `bootstrap/Old_BootStarpBackup/pesticide.py`에 그대로 보존되어 있음. JSON raw 파일을 로컬에 두고:
```bash
uv run python bootstrap/Old_BootStarpBackup/pesticide.py --input-dir tools/pesticide-api-crawler/json_raw
```

#### Phase 1에서의 사용
```python
# bootstrap/create_tables.py
from bootstrap.pesticide import Base as PesticideBase, create_pesticide_tables
create_pesticide_tables(engine)  # 빈 RAG 테이블 5개 생성
```

#### Phase 2에서는 호출 안 함
RAG 데이터는 사용자가 별도로 수동 적재.

---

## 4. 파괴적 로직 회피 정책

새 Phase 1/2는 어떤 경우에도 다음을 **하지 않음**:
- `DROP TABLE`
- `DROP SCHEMA`
- `TRUNCATE`
- `DELETE`
- `Base.metadata.drop_all()`
- 컬럼 ALTER

### 기존 코드의 파괴적 로직 (모두 제거 대상)

| 파일 | 파괴적 코드 |
|---|---|
| `farmos_seed.py` | `drop_farmos_tables()`, `truncate_farmos_tables()` |
| `pesticide.py` | `rebuild_tables()`, `LEGACY_TABLE_NAMES` 대상 DROP |
| `shoppingmall_seed.py` | `truncate_existing_shop_data()`, `Base.metadata.drop_all` |
| `shoppingmall_review_seed.py` | `db.execute(text("DELETE FROM shop_reviews"))` ← **가장 위험** |
| `reset_db.py` | `DROP SCHEMA public CASCADE` (파일 자체 삭제) |

### `shoppingmall_review_seed.py` 특별 처리

현재 함수 본문에 무조건 전체 삭제 후 1000건 INSERT. 리팩터 시 다음 중 하나로 변경:
- **(권장)** `INSERT ... ON CONFLICT (id) DO NOTHING` 으로 변경
- `SELECT COUNT(*)`로 1000건 미만일 때만 부족분 INSERT
- 전체 row 수가 0일 때만 1000건 일괄 INSERT

---

## 5. CLI 플래그 시스템 (사용 안 함)

기존 팀원이 만들어둔 `--mode {seed,init,ensure}`, `--rebuild-schema`, `--append`, `--preserve-existing-data`, `--skip-sync` 등의 플래그 분기 시스템은 **새 자동화에서 사용하지 않음**.

### 새 방식의 원칙
- "무엇을 할지" → **NodeJS가 검증 결과 보고 분기**해서 적절한 진입점만 호출
- "파괴적 동작" → **아예 코드에서 제거** (Phase 1/2 둘 다 가산형으로만)
- Python 스크립트는 단일 동작만 (인자 없음, 항상 동일 결과)

---

## 6. NodeJS 자동화 스크립트 책임 (추후 작성)

> 사용자가 별도 가이드 후 진행 예정

```
1. *_seed.py에서 EXPECTED_ROW_COUNTS / 모델 메타정보 추출
   - farmos_seed.py: EXPECTED_ROW_COUNTS = {"users": 2}, POST_PESTICIDE_MIN_ROW_COUNTS
   - shoppingmall_seed.py: EXPECTED_ROW_COUNTS = {16개 테이블}, READY_ROW_COUNTS
   - seed_ai_agent.py: 30건 (인자), shoppingmall_review_seed.py: 1000건

2. DB에 직접 쿼리해서 테이블·컬럼·row 수 검증

3. 결과에 따라 위 분기표대로 Phase 1 / Phase 2 호출
   - python bootstrap/create_tables.py
   - python bootstrap/insert_data.py

4. 재검증 후 backend / frontend 프로세스 실행

5. 컬럼 drift 감지 시 자동 ALTER 금지 (경고 표시 후 중단)
```

### 환경 정보 (참고)
- DB: 단일 `farmos` DB (FarmOS와 ShoppingMall이 같은 DB 공유, 테이블 접두사 `shop_`로 구분)
- 접속: `postgresql+asyncpg://postgres:root@localhost:5432/farmos`
- 인코딩: UTF8, 로케일: Korean_Korea.949
- PostgreSQL: 18.3
- 실시간 테이블 수: 42개 (FarmOS 코어 + 리뷰분석 + NCPMS + RAG pesticide + AI Agent + checkpoint + shop_*)

### `pesticide.py`의 `--input-dir` 처리
JSON raw 파일 경로는 NodeJS가 환경변수(`PESTICIDE_JSON_DIR`) 또는 기본 경로(`tools/pesticide-api-crawler/json_raw`) 사용. 현재는 자동화에서 제외 상태이므로 미해당.

### 루트 `bootstrap.py`
오케스트레이터 역할이 NodeJS로 이전되므로 별도 처리 (현재는 `bootstrap._bootstrap_common` import가 있어 `_bootstrap_common.py` 삭제 시 깨짐). NodeJS 도입 시점에 함께 정리.

---

## 7. 참고: pg_dump 기반 대안 검증 결과

논의 중 "그냥 pg_dump 파일로 대체하면 안 될까?" 옵션도 검토했습니다. 결과:

### 검증 완료 사항
- `pg_dump -F c` (Custom format) 으로 라이브 `farmos` DB 통째 dump 생성 가능
- 한글 round-trip **100% 무손실** 입증 (TEXT, JSONB 모두 SHA256 해시 동일)
- 단일 dump 파일 1개로 42개 테이블 전체 부트스트랩 가능
- 결과물: `scripts/farmos.dump` (185KB, 2026-04-27 생성, 그대로 보존)

### 채택하지 않은 이유
- 기존 Python 코드를 활용하면서 자동화 스크립트가 검증 분기를 책임지는 방향이 사용자 요구사항(2-2 검증 → 2-3 부분 채움 → 3 재검증) 흐름에 더 부합
- 바이너리 dump는 git diff 불가, 매번 재생성 필요

### 보존 결정
- `scripts/farmos.dump` (Apr 27, 신규, 42 테이블 최신, 185KB) — 보존
- `backend/data/farmos.dump` (Apr 7, 4 테이블 구버전, 1.74MB) — 보존 (사용자 결정)

향후 dump 방식이 다시 매력적으로 보이면 이 자료 참고해서 전환 가능.

---

## 8. 다음 세션에서 바로 진행할 수 있는 순서

새 fork에서 작업 재개 시 다음 순서로 진행:

### Step 1 — 삭제
```bash
rm bootstrap/_bootstrap_common.py
rm bootstrap/reset_db.py
rm -rf bootstrap/__pycache__
```

### Step 2 — `pesticide.py` 슬림화
- 113~310줄 (Base + 5 모델 클래스) 보존
- 1~66줄 중 SQLAlchemy import만 보존
- 나머지 모두 삭제
- 새 함수 추가: `create_pesticide_tables(engine)` → `Base.metadata.create_all(engine)`

### Step 3 — 5개 시드 파일 리팩터
각각 단일 `seed_xxx()` 함수만 노출. argparse, main(), 파괴적 로직, `_bootstrap_common` 의존 모두 제거.

특히 `shoppingmall_review_seed.py`는 DELETE 라인을 `ON CONFLICT DO NOTHING`으로 변경.

### Step 4 — 신규 진입점 작성
- `bootstrap/create_tables.py` (Phase 1)
- `bootstrap/insert_data.py` (Phase 2)

### Step 5 — 동작 검증
- `python bootstrap/create_tables.py` → 테이블 생성 확인
- `python bootstrap/insert_data.py` → 데이터 삽입 확인
- 멱등성 확인: 둘 다 두 번 호출해도 안전한지

### Step 6 — NodeJS 자동화 스크립트 작성
사용자가 별도 가이드 후 진행.

### Step 7 — 루트 `bootstrap.py` 처리
NodeJS가 오케스트레이션 책임을 가져갈 시점에 함께 처리.

---

## 9. 주의사항 체크리스트

- [ ] `bootstrap/Old_BootStarpBackup/` 절대 손대지 않음
- [ ] `bootstrap/pesticide.py` 슬림화 시 모델 클래스 5개 + `__tablename__` 정확히 보존
- [ ] 모든 시드 함수에서 DELETE/TRUNCATE/DROP 0개 보장
- [ ] `INSERT` 구문은 모두 `ON CONFLICT DO NOTHING/UPDATE` 멱등 보장
- [ ] `Base.metadata.create_all()`은 컬럼 ALTER 안 함 — drift는 NodeJS가 경고
- [ ] `_bootstrap_common.py` 삭제 시 루트 `bootstrap.py`가 import 실패 (예상된 broken state)
- [ ] 환경변수 `DATABASE_URL` 일관성 확인 (FarmOS는 asyncpg, ShoppingMall은 psycopg2)

---

## 10. 컨텍스트 (왜 이 작업을 하는가)

### 배경
- 기존 `bootstrap/` 디렉토리는 Python 기반 풀 오케스트레이션 (~3000줄)
- `bootstrap.py` (root) + `_bootstrap_common.py` + 6개 시드 파일 + `reset_db.py` 구성
- CLI 플래그 분기 시스템 (`--mode`, `--rebuild-schema` 등)으로 동작 결정
- `Web_Starter.exe`가 이를 호출하는 구조

### 변경 동기
- Web_Starter.exe → NodeJS 자동화 → 검증 기반 분기 호출 구조로 단순화
- Python은 "무엇을 할지" 책임만, "언제·어떻게 할지"는 NodeJS가 책임
- 파괴적 로직 제거로 데이터 안전성 향상
- pesticide JSON raw 파일은 git에 못 올림 → pesticide 데이터는 수동 적재로 분리

### 영향 범위
- `bootstrap/` 디렉토리 (Old_BootStarpBackup 제외)
- 루트 `bootstrap.py` (추후 처리)
- Web_Starter.exe (NodeJS 도입 시점에 변경)

### 영향 없는 영역
- `backend/` Python 코드 (모델 정의, API 등)
- `frontend/`
- `shopping_mall/` (DB는 공유하지만 시드 코드만 정리)
- `tools/pesticide-api-crawler/`, `tools/ncpms-api-crawler/`
