# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

미국 주식시장 리스크 지표 6종을 한 화면에서 모니터링하는 **단일 HTML 파일 대시보드**.
안드로이드 크롬 최적화, GitHub Pages 호스팅.

- **라이브 URL**: https://staeng75-boop.github.io/stocrisk-dashboard/
- **GitHub 저장소**: https://github.com/staeng75-boop/stocrisk-dashboard
- **배포 방법**: `git push origin master` → GitHub Pages 자동 빌드 (1~2분 소요)

## 파일 구조

```
index.html      ← 전체 대시보드 (CSS·JS·HTML 단일 파일)
debug.html      ← API 연결 진단 도구 (각 엔드포인트 성공/실패 확인용)
```

빌드 도구, 패키지 매니저, 테스트 프레임워크 없음. 외부 의존성은 CDN으로만 로드.

## index.html 내부 구조

스크립트 섹션은 아래 순서로 구성되어 있음:

| 섹션 | 위치 | 역할 |
|------|------|------|
| `CFG` | `<script>` 최상단 | API 키·프록시·캐시·**임계값(THR)** 중앙 관리 |
| `LVMETA` | CFG 아래 | 등급별 레이블·이모지·CSS 클래스 매핑 |
| `VERDICT` | LVMETA 아래 | 종합 등급 → 역발상 투자 메시지 |
| `Cache` | - | sessionStorage 30분 캐시 |
| fetch 함수들 | - | `vixFetch`, `tnxFetch`, `yfV7Fetch`, `yfChartFetch`, `fredFetch`, `treasuryTnxFetch` |
| `fgLabel` | RISK ENGINE | F&G 값 → CNN 레이블 변환 |
| `riskLevel` | RISK ENGINE | 지표값 → 등급 문자열 반환 |
| `overallRisk` | RISK ENGINE | 전체 지표 → 종합 등급 산출 |
| `loadAllData` | - | Promise.allSettled로 전체 데이터 병렬 로드 → UI 업데이트 |

## 데이터 소스 및 폴백 체인

| 지표 | 1차 | 2차 | 3차 |
|------|-----|-----|-----|
| F&G | CNN `production.dataviz.cnn.io` (corsproxy → allorigins) | — | — |
| VIX | Yahoo Finance v7 quote | Yahoo Finance v8 chart meta | CBOE CDN |
| 10년물 금리 | FRED `DGS10` | Treasury.gov XML | — |
| MOVE / WTI | Yahoo Finance v7 quote | Yahoo Finance v8 chart meta | — |
| HY 스프레드 | FRED `BAMLH0A0HYM2` | — | — |
| 차트 (SPX·NDX·SOXX) | Yahoo Finance v8 chart (`3mo`) via corsproxy → allorigins → 직접 | — | — |

**CORS 프록시 순서**: `corsproxy.io` → `allorigins.win` → 직접 요청
- stooq.com은 Cloudflare 차단으로 사용 불가 — 절대 사용하지 말 것

## 등급 체계

**F&G (5단계)**:
- 0~25 → `danger` (Extreme Fear / 빨강)
- 26~49 → `caution` (Fear / 주황)
- 50 → `normal` (Neutral / 노랑)
- 51~74 → `greed` (Greed / 연두, 신규 레벨)
- 75~100 → `safe` (Extreme Greed / 초록)

**나머지 지표 (4단계)**: `safe` → `normal` → `caution` → `danger`
`greed` 레벨은 `overallRisk()` 계산 시 `safe`로 취급됨.

## 임계값 수정 방법

`CFG.THR` 블록만 수정하면 됨 (index.html 약 343~350행):

```javascript
THR: {
  vix:     { safe: 15,   normal: 20,   caution: 30   },
  tnxChg:  { safe: 0.05, normal: 0.15, caution: 0.25 }, // %p 일간 변화
  hy:      { safe: 3.0,  normal: 4.5,  caution: 7.0  }, // FRED % 단위
  move:    { safe: 90,   normal: 120,  caution: 160  },
  wtiChg:  { safe: 2,    normal: 4,    caution: 7    }   // $ 일간 변화
}
```

## API 키

- **FRED API Key** (`CFG.FRED_KEY`): `d442daf53a73ea7d562fd671a96674b0`  
  무료 공개 읽기 전용 키 — HTML에 직접 포함 허용.  
  갱신 필요 시: https://fred.stlouisfed.org → 계정 → API Keys

## 캐시 초기화

sessionStorage 키 `stocRisk_v7`. 데이터 구조 변경 시 `CACHE_KEY` 버전 번호 올릴 것 (예: `stocRisk_v8`).

## debug.html 용도

각 API 엔드포인트별 응답 성공/실패를 브라우저에서 직접 확인하는 진단 페이지.
지표가 N/A로 나올 때 원인 파악에 사용. GitHub Pages에서 열어야 CORS 정상 동작.
