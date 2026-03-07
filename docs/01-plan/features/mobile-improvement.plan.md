# Plan: 모바일 기능개선 (mobile-improvement)

> **작성일**: 2026-03-07
> **프로젝트**: Juns RPG (Juns Cave)
> **PDCA Phase**: Plan

---

## 1. 개요

Juns RPG는 PWA + Capacitor 기반의 방치형 던전 RPG 게임이다. 현재 모바일 기기에서 플레이 가능하나, 터치 UX, 화면 활용도, 성능 등에서 개선 여지가 있다. 이 Plan은 모바일 사용자 경험을 전반적으로 향상시키는 것을 목표로 한다.

## 2. 현재 상태 분석

### 2.1 이미 구현된 모바일 지원
- **반응형 CSS**: 1200px, 992px, 768px, 480px, landscape 5단계 미디어쿼리
- **PWA**: manifest.json, Service Worker(stale-while-revalidate), 오프라인 캐싱
- **Capacitor**: `com.junsrpg.app` 네이티브 앱 래핑 설정
- **Safe Area**: `env(safe-area-inset-*)` CSS 변수 적용
- **터치 최적화**: `touch-action: manipulation`, `-webkit-overflow-scrolling: touch`
- **viewport**: `viewport-fit=cover, maximum-scale=1.0, user-scalable=no`

### 2.2 발견된 개선 포인트
| 영역 | 현재 문제 | 영향도 |
|------|-----------|--------|
| **터치 UX** | 전투 로그 스크롤 시 의도치 않은 터치 발생 가능 | 중 |
| **모달 UX** | 상점/장비 모달이 모바일에서 세로 스크롤 과다 | 높음 |
| **가로 모드** | landscape 레이아웃 최소한만 지원 | 중 |
| **성능** | 이모지 기반 렌더링은 가벼우나, 플로팅 텍스트 DOM 생성이 빈번 | 중 |
| **PWA 아이콘** | 192x192, 512x512 두 사이즈만 존재 (maskable 아이콘 없음) | 낮음 |
| **제스처** | 스와이프 등 모바일 네이티브 제스처 미지원 | 중 |
| **화면 꺼짐 방지** | 방치형 게임인데 Wake Lock API 미사용 | 높음 |
| **알림** | 방치 중 게임 오버 시 Push 알림 없음 | 중 |

## 3. 기능 개선 항목

### P0 (필수) - 방치형 핵심 경험

#### 3.1 Wake Lock (화면 꺼짐 방지)
- **목적**: 방치형 게임 특성상 화면이 꺼지면 자동 전투가 중단됨
- **구현**: Screen Wake Lock API 사용
- **영향 파일**: `script.js`
- **대체 방안**: visibility change 이벤트로 복귀 시 자동 재개

#### 3.2 모달 UX 개선 (상점/장비/물약)
- **목적**: 모바일에서 상점 모달이 화면을 벗어나는 문제 해결
- **구현**:
  - 모달 최대 높이를 `max-height: 85vh` + 내부 스크롤로 변경
  - 탭 기반 UI로 상점 카테고리 전환 (세로 스크롤 감소)
  - 닫기 버튼 고정 (sticky bottom)
- **영향 파일**: `style.css`, `ui.js`, `index.html`

#### 3.3 전투 로그 영역 터치 격리
- **목적**: 로그 스크롤 중 하단 버튼 오터치 방지
- **구현**: `log-box`에 `overscroll-behavior: contain` 적용, 터치 이벤트 전파 제한
- **영향 파일**: `style.css`

### P1 (권장) - 모바일 편의성

#### 3.4 스와이프 제스처 지원
- **목적**: 모달 닫기 (아래로 스와이프), 몬스터 타겟 변경 (좌우 스와이프)
- **구현**: touch event 기반 경량 제스처 감지
- **영향 파일**: `ui.js`

#### 3.5 가로 모드 레이아웃 강화
- **목적**: landscape에서 좌우 분할 레이아웃 (전투 / 로그+컨트롤)
- **구현**: landscape 미디어쿼리 개선
- **영향 파일**: `style.css`

#### 3.6 플로팅 텍스트 성능 최적화
- **목적**: 자동 전투 시 빈번한 DOM 생성/삭제 줄이기
- **구현**: 오브젝트 풀링 패턴 적용 (미리 생성된 DOM 재활용)
- **영향 파일**: `ui.js`

### P2 (선택) - 완성도

#### 3.7 PWA 아이콘 보강
- **목적**: 홈 화면 추가 시 더 나은 아이콘 표시
- **구현**: maskable 아이콘 추가, 다양한 사이즈 제공
- **영향 파일**: `manifest.json`, `assets/`

#### 3.8 푸시 알림 (게임 오버 시)
- **목적**: 방치 중 사망 시 사용자에게 알림
- **구현**: Notification API + Service Worker push
- **영향 파일**: `sw.js`, `script.js`

#### 3.9 햅틱 피드백
- **목적**: 전투 시 터치 피드백 (크리티컬, 흑섬 발동 등)
- **구현**: Vibration API (`navigator.vibrate()`)
- **영향 파일**: `script.js`, `audio.js`

## 4. 구현 순서 (권장)

```
Phase 1: P0 필수 항목
  ├─ 3.1 Wake Lock (화면 꺼짐 방지)
  ├─ 3.2 모달 UX 개선
  └─ 3.3 전투 로그 터치 격리

Phase 2: P1 편의성
  ├─ 3.4 스와이프 제스처
  ├─ 3.5 가로 모드 레이아웃
  └─ 3.6 플로팅 텍스트 최적화

Phase 3: P2 완성도
  ├─ 3.7 PWA 아이콘
  ├─ 3.8 푸시 알림
  └─ 3.9 햅틱 피드백
```

## 5. 기술 제약 사항

| 항목 | 제약 | 대응 |
|------|------|------|
| Wake Lock API | Safari 16.4+, Chrome 84+ | 미지원 브라우저는 graceful degradation |
| Vibration API | iOS Safari 미지원 | feature detection으로 분기 |
| Push Notification | 사용자 권한 필요, iOS PWA 제한적 | Notification API로 기본 알림 먼저 |
| Capacitor | 서버 URL이 하드코딩 (`35.199.151.16`) | 환경별 분리 필요 |

## 6. 성공 기준

- [ ] 모바일(iOS/Android) 브라우저에서 화면 꺼짐 없이 방치 플레이 가능
- [ ] 상점/장비 모달이 480px 화면에서 스크롤 없이 탭으로 전환 가능
- [ ] 전투 로그 스크롤 시 하단 버튼 오터치 발생률 0%
- [ ] Lighthouse PWA 점수 90+ 유지
- [ ] 기존 데스크톱 UX 영향 없음

## 7. 영향 범위

### 수정 대상 파일
| 파일 | 변경 내용 |
|------|-----------|
| `frontend/www/script.js` | Wake Lock, 햅틱, 푸시 알림 |
| `frontend/www/ui.js` | 모달 UX, 스와이프, 플로팅 텍스트 최적화 |
| `frontend/www/style.css` | 모달 레이아웃, 터치 격리, landscape |
| `frontend/www/index.html` | 모달 구조 변경 (탭 UI) |
| `frontend/www/sw.js` | 푸시 알림 핸들러 |
| `frontend/www/manifest.json` | 아이콘 추가 |

### 신규 파일 (필요 시)
- `frontend/www/gesture.js` — 스와이프 제스처 유틸리티 (경량)
