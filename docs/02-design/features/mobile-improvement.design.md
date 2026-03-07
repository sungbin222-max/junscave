# Design: 모바일 기능개선 (mobile-improvement)

> **작성일**: 2026-03-07
> **프로젝트**: Juns RPG (Juns Cave)
> **PDCA Phase**: Design
> **참조**: [Plan 문서](../../01-plan/features/mobile-improvement.plan.md)

---

## 1. 설계 개요

Plan 문서의 P0~P2 항목을 구체적인 코드 수준으로 설계한다.
기존 코드 구조(Vanilla JS, 단일 HTML)를 유지하면서 최소 변경으로 최대 효과를 달성한다.

### 설계 원칙
- **비파괴적 추가**: 기존 함수 시그니처 변경 없이, 새 기능은 래퍼 또는 독립 함수로 추가
- **Feature Detection**: 모든 신규 API는 `if ('wakeLock' in navigator)` 패턴 적용
- **CSS-first**: 가능한 것은 JS 없이 CSS만으로 해결
- **Zero Dependency**: 외부 라이브러리 없이 구현

---

## 2. P0: 필수 항목 상세 설계

### 2.1 Wake Lock (화면 꺼짐 방지)

#### 동작 흐름
```
게임 시작 (startNewGame / loadGame)
  └─ requestWakeLock()
       ├─ 성공 → wakeLock 객체 저장
       └─ 실패 → 무시 (graceful degradation)

visibilitychange 이벤트
  ├─ hidden → wakeLock 자동 해제 (브라우저 동작)
  └─ visible → requestWakeLock() 재요청

게임 종료 (goHome / gameOver)
  └─ releaseWakeLock()
```

#### 구현 위치: `script.js`

```javascript
// === Wake Lock 관리 ===
let wakeLockSentinel = null;

async function requestWakeLock() {
    if ('wakeLock' in navigator) {
        try {
            wakeLockSentinel = await navigator.wakeLock.request('screen');
            wakeLockSentinel.addEventListener('release', () => {
                wakeLockSentinel = null;
            });
        } catch (e) {
            // 배터리 부족 등으로 실패 시 무시
        }
    }
}

function releaseWakeLock() {
    if (wakeLockSentinel) {
        wakeLockSentinel.release();
        wakeLockSentinel = null;
    }
}
```

#### 호출 지점
| 함수 | 동작 | 위치 (script.js) |
|------|------|-----------------|
| `startNewGame()` | 함수 시작부에 `requestWakeLock()` 추가 | ~line 1702 |
| `loadGame()` | 함수 시작부에 `requestWakeLock()` 추가 | ~line 1941 |
| `executeGoHome()` | `releaseWakeLock()` 추가 | ui.js |
| `showGameOverModal()` | `releaseWakeLock()` 추가 | ui.js ~line 1316 |

#### visibilitychange 리스너
```javascript
// script.js 하단에 추가
document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'visible' && !isGameOver) {
        requestWakeLock();
    }
});
```

---

### 2.2 모달 UX 개선

#### 2.2.1 상점 모달 (shop-modal) 탭 UI 전환

**현재 문제**: 상점에 물약 4종 + 방어구 + 무기 + 내장비 + 전리품 판매 = 8개 섹션이 세로로 나열되어 모바일에서 과도한 스크롤 발생

**해결**: 탭 네비게이션으로 카테고리 분리

#### HTML 변경 (index.html - shop-modal 내부)

```html
<!-- shop-modal 기존 구조에 탭 추가 -->
<div id="shop-modal">
    <div class="shop-title">⛺ 떠돌이 상인</div>
    <div class="floor-badge shop-coins-badge">🪙 <span id="shop-coins">0</span></div>

    <!-- 신규: 탭 네비게이션 -->
    <div class="shop-tabs">
        <button class="shop-tab active" data-tab="potions" onclick="switchShopTab('potions')">💊 물약</button>
        <button class="shop-tab" data-tab="equipment" onclick="switchShopTab('equipment')">🗡️ 장비</button>
        <button class="shop-tab" data-tab="my-items" onclick="switchShopTab('my-items')">🎒 내 장비</button>
    </div>

    <div class="shop-container">
        <!-- 탭별 컨텐츠는 기존 shop-row들을 그룹핑 -->
        <div class="shop-tab-content" id="shop-tab-potions">
            <!-- 기존 물약 shop-row 유지 -->
        </div>
        <div class="shop-tab-content" id="shop-tab-equipment" style="display:none;">
            <!-- 기존 방어구/무기 shop-row 유지 -->
        </div>
        <div class="shop-tab-content" id="shop-tab-my-items" style="display:none;">
            <!-- 기존 내 장비/전리품 판매 shop-row 유지 -->
        </div>
    </div>
    <button class="btn-close-red" onclick="closeShop()">상점 나가기</button>
</div>
```

#### JS 함수 (ui.js에 추가)

```javascript
function switchShopTab(tabName) {
    playSound('click');
    // 모든 탭 비활성화
    document.querySelectorAll('.shop-tab').forEach(t => t.classList.remove('active'));
    document.querySelectorAll('.shop-tab-content').forEach(c => c.style.display = 'none');
    // 선택 탭 활성화
    document.querySelector(`.shop-tab[data-tab="${tabName}"]`).classList.add('active');
    document.getElementById(`shop-tab-${tabName}`).style.display = 'block';
}
```

#### CSS 추가 (style.css)

```css
/* 상점 탭 네비게이션 */
.shop-tabs {
    display: flex;
    gap: 4px;
    padding: 0 8px;
    margin-bottom: 8px;
}

.shop-tab {
    flex: 1;
    padding: 8px 4px;
    border: 1px solid var(--c-border);
    border-radius: var(--radius-sm);
    background: var(--c-surface-light);
    color: var(--c-text-dim);
    font-size: 13px;
    cursor: pointer;
    transition: all 0.2s;
}

.shop-tab.active {
    background: var(--c-primary-dim);
    color: var(--c-text);
    border-color: var(--c-primary);
}
```

#### 2.2.2 모달 공통 스크롤 개선

**CSS 변경** (`style.css` - 768px 미디어쿼리 내)

```css
@media (max-width: 768px) {
    /* 상점 모달 */
    #shop-modal {
        max-height: 100vh;
        max-height: 100dvh; /* dynamic viewport height */
        overflow-y: auto;
        padding-bottom: max(16px, var(--sab));
    }

    #shop-modal .shop-container {
        max-height: 60vh;
        max-height: 60dvh;
        overflow-y: auto;
        -webkit-overflow-scrolling: touch;
    }

    /* 닫기 버튼 고정 */
    #shop-modal .btn-close-red {
        position: sticky;
        bottom: 0;
        z-index: 10;
    }

    /* 장비 모달 */
    #equipment-modal .management-container {
        max-height: 60dvh;
        overflow-y: auto;
        -webkit-overflow-scrolling: touch;
    }

    /* 물약 모달 */
    #item-select-modal .potion-container {
        max-height: 55dvh;
        overflow-y: auto;
    }
}
```

#### 2.2.3 장비 모달 - 이미 탭 기반

현재 `openInventoryModal(activeTab)` 함수가 `equipment`, `loot`, `stats` 탭을 지원하고 있으므로, 하단 컨트롤에서 이미 분리하여 호출 중이다. **추가 변경 불필요**.

---

### 2.3 전투 로그 터치 격리

#### CSS 변경 (style.css)

```css
.log-box {
    /* 기존 속성 유지 + 추가 */
    overscroll-behavior: contain;
    touch-action: pan-y;
    -webkit-overflow-scrolling: touch;
}
```

**변경량**: CSS 3줄 추가. JS 변경 없음.

---

## 3. P1: 편의성 항목 상세 설계

### 3.1 스와이프 제스처

#### 설계: 인라인 구현 (별도 파일 불필요)

`ui.js` 하단에 경량 제스처 감지기를 추가한다. 별도 `gesture.js` 파일은 만들지 않는다 (YAGNI 원칙).

#### 지원 제스처
| 제스처 | 대상 | 동작 |
|--------|------|------|
| 아래로 스와이프 (>80px) | 모달 오버레이 | 모달 닫기 |
| 좌/우 스와이프 (>60px) | `#monster-area` | 타겟 몬스터 변경 |

#### 구현 (ui.js 하단에 추가)

```javascript
// === 모바일 스와이프 제스처 ===
(function initGestures() {
    let touchStartY = 0;
    let touchStartX = 0;

    // 모달 닫기 제스처 (아래로 스와이프)
    const modalIds = [
        'shop-modal', 'equipment-modal', 'item-select-modal',
        'scoreboard-modal', 'notice-modal'
    ];
    const closeMap = {
        'shop-modal': closeShop,
        'equipment-modal': closeEquipment,
        'item-select-modal': closeItemSelect,
        'scoreboard-modal': closeScoreboardModal,
        'notice-modal': closeNoticeModal
    };

    document.addEventListener('touchstart', (e) => {
        touchStartY = e.touches[0].clientY;
        touchStartX = e.touches[0].clientX;
    }, { passive: true });

    document.addEventListener('touchend', (e) => {
        const deltaY = e.changedTouches[0].clientY - touchStartY;
        const deltaX = e.changedTouches[0].clientX - touchStartX;

        // 아래로 스와이프 → 모달 닫기
        if (deltaY > 80 && Math.abs(deltaX) < 50) {
            for (const id of modalIds) {
                const modal = document.getElementById(id);
                if (modal && modal.style.display !== 'none') {
                    closeMap[id]();
                    break;
                }
            }
        }
    }, { passive: true });

    // 몬스터 타겟 변경 제스처 (좌우 스와이프)
    const monsterArea = document.getElementById('monster-area');
    if (monsterArea) {
        let mTouchStartX = 0;
        monsterArea.addEventListener('touchstart', (e) => {
            mTouchStartX = e.touches[0].clientX;
        }, { passive: true });

        monsterArea.addEventListener('touchend', (e) => {
            const deltaX = e.changedTouches[0].clientX - mTouchStartX;
            if (Math.abs(deltaX) > 60) {
                const aliveMonsters = monsters.filter(m => m.hp > 0);
                if (aliveMonsters.length <= 1) return;
                const dir = deltaX > 0 ? -1 : 1;
                let newIndex = player.targetIndex + dir;
                // 살아있는 몬스터 인덱스 범위 내에서 순환
                if (newIndex < 0) newIndex = monsters.length - 1;
                if (newIndex >= monsters.length) newIndex = 0;
                while (monsters[newIndex].hp <= 0) {
                    newIndex = (newIndex + dir + monsters.length) % monsters.length;
                }
                player.targetIndex = newIndex;
                updateMonsterUI();
            }
        }, { passive: true });
    }
})();
```

---

### 3.2 가로 모드 레이아웃 강화

#### CSS 변경 (style.css - landscape 미디어쿼리 교체)

```css
@media (max-height: 500px) and (orientation: landscape) {
    .game-container {
        display: grid;
        grid-template-columns: 1fr 1fr;
        grid-template-rows: auto 1fr auto;
        gap: 4px;
        padding: 4px;
    }

    /* 헤더: 전체 폭 */
    .header {
        grid-column: 1 / -1;
    }

    /* 왼쪽: 전투 필드 */
    .battle-field {
        grid-column: 1;
        grid-row: 2;
        flex-direction: row;
        gap: 10px;
        padding: 4px;
    }

    /* 오른쪽: 로그 + 컨트롤 */
    .log-box {
        grid-column: 2;
        grid-row: 2;
        height: auto;
        max-height: 30vh;
    }

    /* 하단 바: 전체 폭 */
    .mp-bar-container, .xp-bar-container {
        grid-column: 1 / -1;
        padding: 2px 8px;
    }

    .controls {
        grid-column: 1 / -1;
        grid-template-columns: repeat(4, 1fr);
    }

    .emoji { font-size: 40px; }
    .xp-bar-bg { height: 8px; }
}
```

---

### 3.3 플로팅 텍스트 성능 최적화

#### 현재 문제
- `showFloatingText()` 호출 시 매번 `createElement` + `appendChild` + `setTimeout(remove)`
- `script.js`에서 36회 호출 → 자동 전투 시 초당 2~5회 DOM 생성/삭제

#### 해결: Object Pool 패턴

```javascript
// ui.js - showFloatingText 교체

const FLOAT_POOL_SIZE = 15;
const floatingPool = [];

// 풀 초기화 (DOMContentLoaded 후)
function initFloatingPool() {
    const battleField = document.getElementById('battle-field');
    for (let i = 0; i < FLOAT_POOL_SIZE; i++) {
        const el = document.createElement('div');
        el.className = 'floating-text';
        el.style.display = 'none';
        battleField.appendChild(el);
        floatingPool.push({ el, inUse: false, timer: null });
    }
}

function showFloatingText(text, targetElement, type) {
    if (!targetElement) return;

    // 풀에서 사용 가능한 요소 찾기
    let poolItem = floatingPool.find(p => !p.inUse);
    if (!poolItem) {
        // 풀이 가득 차면 가장 오래된 것 재사용
        poolItem = floatingPool[0];
        clearTimeout(poolItem.timer);
    }

    const textEl = poolItem.el;
    poolItem.inUse = true;

    // 요소 재설정
    textEl.className = `floating-text ${type}`;
    textEl.innerText = text;
    textEl.style.display = '';

    const battleField = document.getElementById('battle-field');
    const targetRect = targetElement.getBoundingClientRect();
    const battleFieldRect = battleField.getBoundingClientRect();

    const x = targetRect.left - battleFieldRect.left + (targetRect.width / 2)
              - (textEl.offsetWidth / 2) + (Math.random() * 20 - 10);
    const y = targetRect.top - battleFieldRect.top - 30 + (Math.random() * 10 - 5);

    textEl.style.left = `${x}px`;
    textEl.style.top = `${y}px`;

    // CSS 애니메이션 재시작
    textEl.style.animation = 'none';
    textEl.offsetHeight; // reflow 트리거
    textEl.style.animation = '';

    poolItem.timer = setTimeout(() => {
        textEl.style.display = 'none';
        poolItem.inUse = false;
    }, 1200);
}
```

#### 호출 지점
- `initFloatingPool()`을 `startNewGame()` 시작부에서 1회 호출

---

## 4. P2: 완성도 항목 상세 설계

### 4.1 PWA 아이콘 보강

#### manifest.json 변경

```json
{
  "icons": [
    { "src": "assets/icon.png", "sizes": "192x192", "type": "image/png" },
    { "src": "assets/icon.png", "sizes": "512x512", "type": "image/png" },
    { "src": "assets/icon-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

> **Note**: `icon-maskable.png`는 기존 `icon.png` 기반으로 safe zone(중앙 80%) 내에 주요 콘텐츠가 있는 버전으로 별도 제작 필요.

---

### 4.2 게임 오버 알림 (Notification API)

#### 설계: Service Worker push 대신 간단한 Notification API 사용

```javascript
// script.js - 게임 오버 시 알림

function notifyGameOver(finalFloor) {
    if (!('Notification' in window)) return;
    if (Notification.permission === 'granted') {
        new Notification('Juns RPG', {
            body: `용사가 B${finalFloor}F에서 쓰러졌습니다!`,
            icon: 'assets/icon.png',
            tag: 'game-over' // 중복 알림 방지
        });
    }
}
```

#### 권한 요청 시점
```javascript
// 게임 시작 시 1회 요청 (startNewGame 함수 내)
if ('Notification' in window && Notification.permission === 'default') {
    Notification.requestPermission();
}
```

#### 호출 지점
- `showGameOverModal(score)` 함수 내부에서 `notifyGameOver(score)` 호출

---

### 4.3 햅틱 피드백

#### 구현 (script.js에 유틸리티 함수 추가)

```javascript
function vibrate(pattern) {
    if (navigator.vibrate) {
        navigator.vibrate(pattern);
    }
}
```

#### 적용 포인트
| 이벤트 | 진동 패턴 | 호출 위치 |
|--------|-----------|-----------|
| 치명타 적중 | `vibrate(50)` | 공격 함수 내 crit 분기 |
| 흑섬 발동 | `vibrate([100, 50, 100])` | blackFlash 발동 분기 |
| 게임 오버 | `vibrate([200, 100, 200])` | `showGameOverModal()` |
| 레벨업 | `vibrate(30)` | 레벨업 처리 분기 |

---

## 5. 구현 순서 (Implementation Order)

```
Step 1: CSS 변경 (비파괴적, 즉시 적용)
  ├─ 2.3 log-box 터치 격리 (3줄)
  ├─ 2.2.2 모달 스크롤 개선 (~20줄)
  ├─ 2.2.1 상점 탭 CSS (~20줄)
  └─ 3.2 landscape 레이아웃 (~30줄)

Step 2: HTML 변경
  └─ 2.2.1 상점 탭 구조 (기존 shop-row를 tab-content로 래핑)

Step 3: JS - Core (script.js)
  ├─ 2.1 Wake Lock 함수 + visibilitychange 리스너
  ├─ 4.3 vibrate 유틸리티
  ├─ 4.2 Notification 권한 요청 + notifyGameOver
  └─ 호출 지점 연결 (startNewGame, loadGame, gameOver)

Step 4: JS - UI (ui.js)
  ├─ 2.2.1 switchShopTab 함수
  ├─ 3.3 플로팅 텍스트 Object Pool (showFloatingText 교체)
  ├─ 3.1 스와이프 제스처 IIFE
  └─ openShop() 수정 (탭 초기화)

Step 5: PWA
  └─ 4.1 manifest.json 아이콘 추가
```

## 6. 파일별 변경 요약

| 파일 | 변경 유형 | 변경량 (추정) |
|------|-----------|---------------|
| `style.css` | 수정 | +70줄 (미디어쿼리 추가/수정) |
| `index.html` | 수정 | +15줄 (상점 탭 구조) |
| `script.js` | 수정 | +50줄 (Wake Lock, vibrate, notify) |
| `ui.js` | 수정 | +100줄 (탭, Pool, 스와이프) |
| `manifest.json` | 수정 | +3줄 (maskable 아이콘) |
| **합계** | | **~238줄 추가** |

## 7. 테스트 체크리스트

### P0 테스트
- [ ] **Wake Lock**: 모바일 Chrome/Safari에서 게임 중 화면 자동 꺼짐 없음
- [ ] **Wake Lock 해제**: 홈으로 이동 또는 게임 오버 시 정상 해제
- [ ] **Wake Lock 복구**: 탭 전환 후 복귀 시 재획득
- [ ] **상점 탭**: 물약/장비/내장비 탭 전환 정상 동작
- [ ] **상점 탭**: 탭 전환 시 구매/판매 기능 유지
- [ ] **모달 스크롤**: 480px 화면에서 모달이 화면 밖으로 나가지 않음
- [ ] **닫기 버튼**: 모바일에서 상점 닫기 버튼 항상 보임 (sticky)
- [ ] **로그 터치 격리**: 로그 스크롤 시 페이지 바운스 없음

### P1 테스트
- [ ] **스와이프 닫기**: 상점/장비/물약 모달에서 아래 스와이프로 닫기
- [ ] **스와이프 타겟**: 몬스터 영역에서 좌우 스와이프로 타겟 변경
- [ ] **가로 모드**: landscape에서 좌우 분할 레이아웃 표시
- [ ] **플로팅 텍스트**: 자동 전투 5분 이상 시 DOM 누적 없음

### P2 테스트
- [ ] **알림**: 게임 오버 시 브라우저 알림 표시 (권한 허용 시)
- [ ] **햅틱**: 크리티컬/흑섬 시 진동 (Android Chrome)
- [ ] **Maskable 아이콘**: 홈 화면 추가 시 원형 아이콘 정상 표시

### 회귀 테스트
- [ ] **데스크톱**: 1920px에서 기존 레이아웃 깨짐 없음
- [ ] **기존 기능**: 자동 전투, 상점 구매/판매, 스탯 분배 정상
- [ ] **오프라인**: PWA 오프라인 모드 정상 동작
