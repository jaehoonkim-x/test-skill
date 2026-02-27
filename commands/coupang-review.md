---
description: 쿠팡 키워드 검색 후 상품별 리뷰 수 수집
allowed-tools: mcp__playwright__browser_navigate, mcp__playwright__browser_click, mcp__playwright__browser_type, mcp__playwright__browser_evaluate, mcp__playwright__browser_run_code, mcp__playwright__browser_snapshot, Bash
argument-hint: [검색 키워드]
---

쿠팡에서 "$ARGUMENTS" 키워드로 상품을 검색하고 리뷰 수를 수집한다.

만약 키워드가 비어있으면 사용자에게 검색할 키워드를 물어본다.

이 작업을 수행할 때 반드시 coupang-review-scraper 스킬을 참조하여 워크플로우를 따른다.

작업 순서:
1. Playwright MCP로 https://www.coupang.com 에 접속
2. 검색창에 키워드를 입력하고 검색 실행
3. 검색 결과에서 모든 상품 링크를 JavaScript로 추출 (searchRank 포함 링크만 필터링)
4. 9개씩 배치로 나누어 각 상품 페이지를 방문하며 리뷰 수 수집
5. 결과를 리뷰 수 기준 내림차순으로 정렬하여 마크다운 테이블로 출력

결과 테이블에는 순위, 상품명, 리뷰 수를 포함한다.
