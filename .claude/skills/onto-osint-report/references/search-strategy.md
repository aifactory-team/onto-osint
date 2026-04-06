# 검색 전략

## 설정 기반 검색

모든 검색 파라미터는 `config/osint-config.json`에서 읽는다:
- `search.keywords` — 언어별 검색 키워드
- `search.engines` — 검색 엔진 목록
- `search.custom_sites` — 커스텀 사이트 목록
- `search.min_searches` — 최소 검색 횟수

## 키워드 확장

기본 키워드 외에 온톨로지 기반으로 키워드를 확장한다:
1. `ontology/instances.json`에서 최근 7일 내 추가된 주요 엔티티명을 추출
2. 중요도가 높은 엔티티(참조 횟수 3회 이상)를 보조 키워드로 추가
3. 확장 키워드는 기본 키워드와 조합하여 검색 (예: "기본키워드 + 엔티티명")

## 검색 방법

### Method 1: WebSearch (빌트인)
Claude Code의 WebSearch 도구를 직접 호출한다.
각 언어별 키워드로 검색하되, 날짜 범위를 최근 24시간으로 제한한다.

### Method 2: Cheliped Browser (사이트별)
`config/osint-config.json`의 검색 대상 사이트 목록을 확인한다.
`enabled: true`인 사이트만 검색한다.

**검색 엔진** — `search.engines` 배열의 사이트:
```bash
node $CHELIPED_CLI '[{"cmd":"search","args":["검색어","엔진명"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
```

**커스텀 사이트** — `search.custom_sites` 배열의 사이트 (`search_url`의 `{query}`를 키워드로 치환):
```bash
node $CHELIPED_CLI '[{"cmd":"goto","args":["URL"]},{"cmd":"wait","args":["2000"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
```

- 각 검색 후 반드시 `close` 커맨드로 세션을 종료한다
- 사이트 추가/제거는 `config/osint-config.json`만 편집하면 된다

## 출력 스키마: search-results.json

```json
{
  "date": "YYYY-MM-DD",
  "collected_at": "ISO8601",
  "config_topic": "config에서 읽은 주제명",
  "search_count": 12,
  "total_results": 45,
  "expanded_keywords": ["온톨로지에서 확장한 키워드 목록"],
  "searches": [
    {
      "method": "websearch|cheliped",
      "engine": "google|naver|built-in|...",
      "keyword": "검색어",
      "language": "ko|en|ja|zh",
      "timestamp": "ISO8601",
      "result_count": 5,
      "results": [
        { "title": "...", "url": "...", "snippet": "...", "source_name": "..." }
      ]
    }
  ]
}
```

이 파일은 Phase 1에서 쓰고 이후 Phase에서 재읽기 않는다 (write-only, 디버깅/추적 전용).
