---
name: coupang-review-scraper
description: |
  **쿠팡 상품 리뷰 스크래핑**: Playwright MCP 브라우저 자동화를 사용하여 쿠팡(coupang.com)에서 키워드 검색 후 검색 결과의 모든 상품 페이지를 개별 방문하여 리뷰 수를 수집합니다.
  - 트리거: 쿠팡 리뷰, 쿠팡 상품 분석, 쿠팡 리뷰 수집, 쿠팡 검색 스크래핑, 쿠팡 상품 리뷰 개수, 쿠팡 인기 상품, coupang review, coupang scrape
  - 키워드를 변경하여 다양한 카테고리의 상품 리뷰를 수집할 수 있습니다.
  - 이 스킬은 사용자가 쿠팡에서 상품 데이터를 수집하거나 리뷰 분석을 하고 싶을 때 사용합니다.
compatibility: "Playwright MCP (browser_navigate, browser_click, browser_type, browser_evaluate, browser_run_code, browser_snapshot)"
---

# 쿠팡 상품 리뷰 스크래퍼

쿠팡에서 키워드로 상품을 검색하고, 검색 결과의 각 상품 페이지를 방문하여 리뷰 수를 수집하는 자동화 스킬입니다.

## 왜 각 페이지를 방문하는가?

쿠팡 검색 결과 페이지에는 리뷰 수가 표시되지 않거나 불완전한 경우가 많습니다. 정확한 리뷰 수를 얻으려면 각 상품의 상세 페이지를 직접 방문해야 합니다. 이 스킬은 Playwright MCP의 실제 브라우저 세션을 활용하므로, HTTP 직접 요청 시 발생하는 403 차단 문제를 우회할 수 있습니다.

## 워크플로우

### Step 1: 쿠팡 검색

```
browser_navigate → https://www.coupang.com
browser_snapshot (검색창 ref 확인)
browser_click (검색 입력창 클릭)
browser_type (키워드 입력, submit: true)
```

사용자가 키워드를 지정하지 않으면 확인을 요청합니다. 기본값은 없습니다.

### Step 2: 상품 링크 추출

검색 결과 페이지가 로드되면, `browser_evaluate`로 모든 상품 링크를 추출합니다.

핵심 JavaScript:
```javascript
() => {
  const allLinks = document.querySelectorAll('a[href*="/vp/products/"]');
  const results = [];
  const seen = new Set();
  allLinks.forEach((a) => {
    const href = a.getAttribute('href');
    const match = href.match(/\/vp\/products\/(\d+)/);
    if (match && !seen.has(match[1]) && href.includes('searchRank')) {
      seen.add(match[1]);
      const name = a.textContent.trim().substring(0, 50);
      results.push({ pid: match[1], name: name, url: 'https://www.coupang.com' + href });
    }
  });
  return results;
}
```

`searchRank`가 포함된 링크만 필터링하는 이유는, 광고나 추천 상품이 아닌 실제 검색 결과 상품만 추출하기 위함입니다. `Set`으로 중복 제거합니다.

### Step 3: 배치별 상품 페이지 방문 및 리뷰 수 수집

한 번에 모든 상품을 방문하면 타임아웃(60초)이 발생합니다. **9개씩 배치로 나누어** 처리합니다.

`browser_run_code`를 사용하여 각 배치를 처리합니다:

```javascript
async (page) => {
  const products = [/* 이 배치의 상품 목록 */];
  const results = [];
  for (const p of products) {
    try {
      await page.goto(`https://www.coupang.com/vp/products/${p.pid}`, {
        waitUntil: 'domcontentloaded',
        timeout: 12000
      });
      await page.waitForTimeout(1500);
      const review = await page.evaluate(() => {
        // 방법 1: 직접 셀렉터
        const el = document.querySelector('.prod-buy-header__review-count');
        if (el) {
          const m = el.textContent.match(/[\d,]+/);
          return m ? m[0] : '0';
        }
        // 방법 2: review 클래스 검색
        const els = document.querySelectorAll('[class*="review"]');
        for (const e of els) {
          const m = e.textContent.match(/([\d,]+)\s*개\s*상품평/);
          if (m) return m[1];
        }
        return '0';
      });
      results.push({ rank: p.rank, name: p.name, reviews: review });
    } catch (e) {
      results.push({ rank: p.rank, name: p.name, reviews: 'err' });
    }
  }
  return results;
}
```

**배치 크기를 9로 잡는 이유**: Playwright의 `browser_run_code` 타임아웃이 60초이고, 각 페이지 방문에 약 3-5초가 소요되므로, 9개면 약 30-45초로 안전 마진을 확보할 수 있습니다.

**리뷰 수 추출 전략**: 쿠팡은 페이지 구조가 자주 변경되므로, 1차로 `.prod-buy-header__review-count` 셀렉터를 시도하고, 실패 시 `[class*="review"]` + 정규식 패턴으로 폴백합니다.

### Step 4: 결과 정리 및 출력

`browser_run_code`의 출력이 큰 파일로 저장될 수 있습니다. Python이나 Bash로 JSON을 파싱하여 결과를 정리합니다.

최종 출력 형식 (마크다운 테이블):

```
| 순위 | 상품명 | 리뷰 수 |
|------|--------|---------|
| 1 | 상품A | 50,951 |
| 2 | 상품B | 32,100 |
| ... | ... | ... |
```

리뷰 수 기준으로 내림차순 정렬하여 가장 인기 있는 상품을 상단에 표시합니다.

## 주의사항

- **타임아웃**: 한 번에 너무 많은 상품을 처리하지 않도록 9개씩 배치 처리
- **대기 시간**: 각 페이지 로드 후 1.5초 대기 (`waitForTimeout(1500)`)로 안정성 확보
- **HTTP 직접 요청 불가**: 쿠팡은 봇 차단이 강력하므로 반드시 Playwright 브라우저를 통해 접근
- **셀렉터 변경 가능성**: 쿠팡 DOM 구조가 변경되면 셀렉터 업데이트 필요
- **검색 결과 수**: 보통 한 페이지에 약 27-36개 상품이 표시됨
