---
name: onto-osint-report
description: "온톨로지 기반 OSINT 일일 보고서를 자동 생성하는 파이프라인 오케스트레이터. 6단계(수집→추출→온톨로지→그래프→보고서→발행)로 구성되며, 지식그래프 시각화와 온톨로지 추론을 포함한다. 'OSINT 보고서', 'daily report', '일일 보고서 생성', '지식그래프 보고서', 'ontology report' 요청 시 사용."
---

# Onto-OSINT Report — 파이프라인 오케스트레이터

이 스킬은 4개 에이전트를 순차적으로 연결하여 온톨로지 기반 OSINT 일일 보고서를 생성한다.
각 에이전트는 자신의 역할(`.claude/agents/`)과 참조 스킬(`references/`)을 읽고 작업한다.
모든 도메인 특화 설정은 `config/osint-config.json`에서 읽는다.

## 실행 모드
서브 에이전트 파이프라인 — 6개 Phase가 순차 의존 관계이므로 직렬 실행한다.

## 실행 흐름

### Phase 0: 준비
1. `config/osint-config.json`을 읽어 프로젝트 설정을 파악한다
2. 대상 날짜를 결정한다
3. 디렉토리를 생성한다:
   ```bash
   mkdir -p sources/YYYY-MM-DD/items
   mkdir -p reports/YYYY/MM
   mkdir -p ontology/kg
   ```
4. 이전 N일(`project.lookback_days`)의 `sources/*/index.json` 파일 목록을 조회한다
5. 이전 N일의 `reports/YYYY/MM/*.md` 파일 목록을 조회한다
6. `ontology/schema.json`이 비어있으면 config의 `ontology.seed_classes`와 `ontology.seed_relations`로 초기화한다

### Phase 1: 수집 (Collect)
**에이전트:** `.claude/agents/osint-collector.md`를 읽고 역할을 따른다
**참조 스킬:** `references/search-strategy.md`를 읽고 검색 전략·방법·스키마를 따른다
**입력:** `config/osint-config.json`, `ontology/instances.json` (엔티티명 기반 확장 검색)
**산출물:** `sources/YYYY-MM-DD/search-results.json`

### Phase 2: 추출 (Extract)
**에이전트:** `.claude/agents/osint-extractor.md`를 읽고 역할을 따른다
**참조 스킬:** `references/extraction-rules.md`를 읽고 추출 규칙·태깅 기준·스키마를 따른다
**입력:** `search-results.json`, 이전 N일 `sources/*/index.json`, `ontology/instances.json`
**산출물:** `sources/YYYY-MM-DD/index.json` + `items/src-XXX.json` + `entities.json`

### Phase 3: 온톨로지 확장 + 지식그래프 구성 (Ontology & Graph)
**에이전트:** `.claude/agents/osint-reasoner.md`를 읽고 역할을 따른다
**참조 스킬:** `references/ontology-reasoning.md`를 읽고 확장 원칙·추론 규칙·KG 스키마를 따른다
**입력:** `entities.json`, `ontology/schema.json`, `ontology/instances.json`, `ontology/kg/cumulative.json`, 이전 N일 `reports/**/*.md`
**산출물:**
- `ontology/schema.json` (업데이트)
- `ontology/instances.json` (업데이트)
- `ontology/kg/YYYY-MM-DD.json` (일별 스냅샷)
- `ontology/kg/cumulative.json` (업데이트)
- `ontology/reasoning-log.md` (추론 로그 추가)
- `sources/YYYY-MM-DD/analysis.md`
- `sources/YYYY-MM-DD/report-basis.md`

### Phase 4: 보고서 (Report)
**에이전트:** `.claude/agents/osint-reporter.md`를 읽고 역할을 따른다
**참조 스킬:** `references/report-format.md`를 읽고 보고서 구조·KG 시각화·Wiki 규칙을 따른다
**입력:** `report-basis.md`, `analysis.md`, 포함 결정된 `items/`, `ontology/kg/YYYY-MM-DD.json`
**산출물:** `reports/YYYY/MM/YYYY-MM-DD.md`

### Phase 5: 커밋 & 발행
```bash
git add sources/ reports/ ontology/
git commit -m "report: daily OSINT update (YYYY-MM-DD)"
git push
```

Wiki 발행 (config의 `report.wiki_publish`가 true일 때):
- wiki.git clone → 보고서 복사 (frontmatter 제거) → Home.md, _Sidebar.md 업데이트 → push

## 데이터 전달

| Phase | 입력 | 산출물 | 비고 |
|-------|------|--------|------|
| 0 | config | 디렉토리 | 초기화 |
| 1 | config, instances | search-results.json | write-only |
| 2 | search-results, prev index, instances | index + items + entities | 태깅 + 추출 |
| 3 | entities, schema, instances, cumulative, prev reports | schema, instances, KG, analysis, report-basis | 핵심 |
| 4 | report-basis, analysis, items, KG | report.md | 최종 |
| 5 | all files | git commit | 발행 |

## 에러 핸들링

| Phase | 에러 | 전략 |
|-------|------|------|
| 0 | config 파일 없음 | 파이프라인 중단, 에러 메시지 |
| 1 | 검색 전체 실패 | 빈 search-results.json, 다음 Phase 계속 |
| 2 | 이전 index.json 없음 | 전체 `new`로 태깅 |
| 3 | 온톨로지 파일 없음 | config seed로 초기화 후 진행 |
| 3 | entities.json 비어있음 | 추론 건너뛰고 빈 KG 스냅샷 |
| 4 | 포함 항목 0건 | "특이사항 없음" 보고서 |
| 5 | Wiki clone 실패 | 메인 리포 커밋 유지, Wiki 스킵 |

## 테스트 시나리오

### 정상 흐름
1. config에 주제/키워드 설정
2. 6단계 파이프라인 실행
3. sources/, ontology/, reports/ 모두 생성 확인
4. 보고서에 Mermaid KG 시각화 포함 확인
5. Git commit + Wiki 발행 확인

### 첫 실행 (빈 온톨로지)
1. ontology/ 파일이 비어있는 상태에서 실행
2. Phase 0에서 seed로 초기화되는지 확인
3. Phase 2에서 이전 index 없이 전체 new 태깅 확인
4. Phase 3에서 첫 KG 구성 확인

### 에러 흐름
1. 검색 전체 실패 시 빈 보고서 생성 확인
2. 온톨로지 파일 손상 시 seed 재초기화 확인
