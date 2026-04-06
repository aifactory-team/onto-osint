# Onto-OSINT — Ontology-Based OSINT Report System

## Project Purpose
설정 파일(주제/목표/범위)을 기반으로 OSINT 검색을 수행하고, 온톨로지를 확장하며, 지식그래프를 구성하여 보고서를 자동 생성하는 범용 시스템.
6단계 파이프라인(수집→추출→온톨로지→그래프→보고서→발행)으로 추적 가능한 일일 보고서를 생성한다.

## Pipeline Architecture

```
Phase 1        Phase 2         Phase 3          Phase 4        Phase 5          Phase 6
수집(Collect) → 추출(Extract) → 온톨로지(Onto) → 그래프(Graph) → 보고서(Report) → 발행(Publish)
    │              │               │                │              │                │
    ▼              ▼               ▼                ▼              ▼                ▼
search-        index.json      ontology/        kg/            YYYY-MM-DD.md    Git + Wiki
results.json   items/          schema.json      YYYY-MM-DD     (KG 시각화 포함)
                               instances.json   .json
```

## Directory Structure

```
config/
└── osint-config.json          # 주제/목표/범위/키워드/온톨로지 시드 (포크 후 이 파일만 수정)

ontology/
├── schema.json                # 클래스 계층 + 관계 유형 (시드 + 진화)
├── instances.json             # 알려진 엔티티 인스턴스
├── kg/
│   ├── YYYY-MM-DD.json        # 일별 KG 스냅샷 (새 트리플)
│   └── cumulative.json        # 누적 KG (전체 트리플)
└── reasoning-log.md           # 추론된 트리플 + 근거 로그

sources/YYYY-MM-DD/
├── search-results.json        # Phase 1 (write-only)
├── index.json                 # Phase 2 (경량 인덱스)
├── items/src-XXX.json         # Phase 2 (개별 소스 상세)
├── entities.json              # Phase 2 (추출된 엔티티/관계)
├── analysis.md                # Phase 3 (온톨로지 확장 근거)
└── report-basis.md            # Phase 4 (보고서 작성 근거)

reports/YYYY/MM/
└── YYYY-MM-DD.md              # Phase 5 (최종 보고서 + KG 시각화)
```

## Agents & Skills

에이전트(역할)와 스킬(상세 규칙)이 분리되어 있다. 로컬과 GHA에서 동일한 파일을 사용한다.

| 에이전트 | 참조 스킬 | Phase |
|----------|----------|-------|
| `.claude/agents/osint-collector.md` | `references/search-strategy.md` | 1 |
| `.claude/agents/osint-extractor.md` | `references/extraction-rules.md` | 2 |
| `.claude/agents/osint-reasoner.md` | `references/ontology-reasoning.md` | 3-4 |
| `.claude/agents/osint-reporter.md` | `references/report-format.md` | 5 |

오케스트레이터: `.claude/skills/onto-osint-report/skill.md`

## Configuration

모든 도메인 특화 설정은 `config/osint-config.json`에 집중되어 있다.
이 프로젝트를 포크한 후 이 파일만 수정하면 새로운 주제의 OSINT 시스템이 된다.

## Commit Convention
- 보고서 커밋: `report: daily OSINT update (YYYY-MM-DD)`
- 온톨로지 변경: `ontology: expand schema/instances (YYYY-MM-DD)`
- 구조/설정 변경: `chore: 설명`
- `git add sources/ reports/ ontology/` — 온톨로지 디렉토리도 함께 커밋한다

## Rules
- 출처 URL 없는 정보는 보고서에 포함하지 않는다
- 보고서 언어는 `config/osint-config.json`의 `report_language` 설정을 따른다
- 동일 뉴스가 여러 매체에서 보도된 경우 대표 1개만 포함하되 출처 목록에 모두 기재한다
- 보고서가 비어있더라도 파일은 생성한다 (특이사항 없음 보고서)
- 파이프라인 중간 산출물은 항상 생성한다
- 온톨로지 변경은 반드시 근거(reasoning-log)를 남긴다
- 지식그래프 시각화는 Mermaid 다이어그램으로 보고서에 포함한다
