# Canvas Queue Visualization Project

## 프로젝트 개요
**목표**: 제니퍼 소프트 X-View 차트의 핵심 개념(실시간 트랜잭션 큐 시각화)을 Canvas로 구현하는 프로토타입 개발

**기간**: 2024.11.13 ~ 진행 중  
**개발자**: Jun (5년 경력 프론트엔드, Three.js 전문가)

---

## 프로젝트 진행 과정

### Phase 1: 제니퍼 X-View 분석
**목적**: APM(Application Performance Monitoring) 차트의 기술적 구현 방법 탐구

**X-View 특징**:
- 실시간 트랜잭션 응답시간을 점으로 표현
- HTML5 Canvas 기반 (10년 전 전환)
- 10년간 개발된 패턴 인식 알고리즘
- 특허 보유 (제10-2499005호)

**리소스 산정**:
```
프로토타입 수준:  6-8주 (1인)
프로덕션 수준:   13-15주 (3-4개월)

핵심 단계:
- 기본 렌더링:        2-3주
- 성능 최적화:        2-3주  
- 실시간 스트리밍:    2주
- 인터랙션:          3주
- 고급 시각화:       2주
```

---

### Phase 2: 첫 번째 프로토타입 (시각 효과 중심)
**파일**: `particle-tunnel-stream.html` (330줄)

**구현 내용**:
- ✅ 파티클 풀 패턴 (프로덕션급 메모리 관리)
- ✅ 터널 효과 + 용수철 진동
- ✅ 충돌 물리 시뮬레이션
- ✅ 동적 터널 길이
- ❌ 인터랙션 없음

**평가**: 
- 시각 효과: ⭐⭐⭐⭐⭐
- 인터랙션: ⭐☆☆☆☆
- 데이터 의미: ❌ 없음

---

### Phase 3: 두 번째 프로토타입 (인터랙션 추가)
**파일**: `interactive-particle-tunnel.html` (600줄, +82%)

**추가 기능**:
- ✅ 마우스 호버 툴팁 (Transaction ID, 응답시간, 상태)
- ✅ 드래그 영역 선택
- ✅ 실시간 통계 패널
- ✅ 데이터 의미 부여 (각 파티클 = 트랜잭션)
- ✅ 에러/정상 색상 구분

**평가**: **7.5/10** (Good - 우수한 프로토타입)
```
기능 완성도:   8/10
시각적 완성도: 9/10
코드 품질:    7/10
성능:        6/10 (1000개 안정, 3000개 불안정)
확장성:      6/10
인터랙션:    8/10
```

**성능 병목**:
- O(n) 선형 검색 (마우스 호버 시 3000개 순회)
- Spatial Index 미구현
- 비활성 파티클도 매 프레임 체크

---

## 🔥 핵심 문제 발견 및 해결

### 문제: 터널 길이 역설
**증상**: `Particles Per Burst`를 줄이면 터널이 오히려 늘어남

**원인 분석**:
```javascript
// ❌ 잘못된 비율 계산
const targetLength = minLength + (particlesInTunnel / burstSize) * (maxLength - minLength);

시나리오:
burstSize = 300, 터널 내 150개 → 비율 0.5
burstSize = 50,  터널 내 30개  → 비율 0.6 (더 높음!)
```

**근본 원인**:
- `burstSize` = "한 번 발사량" (300개)
- `particlesInTunnel` = 터널 내 실제 개수 (동적, 100~1000개 가능)
- **터널 용량 개념 부재** (설계 결함)

**순환 참조 문제**:
```
터널 길이 계산 → particlesInTunnel 카운트 필요
    ↑                           ↓
    └─────── isInTunnel() 호출 ←┘
           (터널 길이 필요)
```

---

### 해결: 큐(Queue) 모델 적용

**Jun의 직관** (정확함!):
```
단계:
1. 데이터가 발사됨 (Burst)
2. 데이터가 쌓임 (Queue)
3. 데이터가 처리되어서 배출됨 (Dequeue)

→ 쌓이는 만큼 터널 길이가 늘어나야 함
→ Burst 줄이면 쌓이는 양 감소 → 터널 짧아짐 (직관적!)
```

---

### Phase 4: 최종 구현 (큐 기반 시스템)
**파일**: `queue-based-particle-tunnel.html`

**핵심 설계**:
```javascript
class TunnelQueue {
    constructor() {
        this.queue = [];  // 실제 큐 자료구조
    }
    
    enqueue(particle, now) {
        // 터널 입구 도달 → 큐에 추가
        this.queue.push({ particle, enterTime: now });
    }
    
    update(now) {
        // 처리 시간 지나면 배출
        while (this.queue.length > 0) {
            if (now - this.queue[0].enterTime >= processingTime) {
                this.queue.shift(); // FIFO 배출
            } else {
                break;
            }
        }
    }
}

// ✅ 터널 길이 = 큐 크기
function updateTunnelLength() {
    const queueSize = tunnelQueue.getQueueSize();
    const queueRatio = Math.min(queueSize / 300, 1.0);
    const targetLength = minLength + queueRatio * (maxLength - minLength);
}
```

**파티클 생명주기**:
```
1. [발사] → 자유 이동 (파란색)
      ↓
2. [입구 도달] → queue.push() → 큐 대기 (노란색) ⭐
      ↓
3. [처리 시간 경과] → queue.shift() → 배출
      ↓
4. [터널 출구] → 자유 이동 후 소멸
```

**동작 검증**:
```
✅ Burst 증가 → 큐 쌓임 → 터널 길어짐
✅ Processing Time 길게 → 배출 느림 → 큐 쌓임 → 터널 길어짐
✅ Burst 없어도 → 큐에 데이터 있으면 → 계속 배출
✅ Burst 감소 → 적재량 감소 → 터널 짧아짐 (직관적!)
```

---

## 주요 컨트롤 파라미터

### 1. Particles Per Burst
**의미**: 한 번 발사에서 생성할 파티클 개수  
**범위**: 50~500 (기본 200)  
**효과**: 많을수록 큐에 더 많이 쌓임

### 2. Burst Interval
**의미**: 발사 간격 (밀리초)  
**범위**: 50~1500ms (기본 300ms)  
**효과**: 짧을수록 더 자주 발사 → 유입량 증가

### 3. Processing Time ⭐ 핵심
**의미**: 큐에서 처리하는 시간 (밀리초)  
**범위**: 100~3000ms (기본 1000ms)  
**효과**: 
- 짧으면 → 빠른 배출 → 큐 안 쌓임 → 터널 짧음
- 길면 → 느린 배출 → 큐 쌓임 → 터널 길어짐

### 4. Spring Frequency
**의미**: 터널 진동(용수철) 주파수  
**범위**: 0.2~2.0Hz (기본 0.8)  
**효과**: 터널 길이의 사인파 변화 속도

### 5. Spring Amplitude
**의미**: 터널 진동 강도  
**범위**: 0.0~1.0 (기본 0.3)  
**효과**: ±진폭% 만큼 터널 길이 변화

---

## Canvas 학습 커리큘럼 (Jun용)

### Week 1: Canvas 기초 + 애니메이션
```
Day 1-2: 렌더링 API 실습
- 정규화 좌표 ↔ 스크린 좌표 변환
- devicePixelRatio 대응
- 기본 도형, 그라데이션

Day 3-4: 애니메이션 루프 + 물리
- requestAnimationFrame 패턴
- Delta time 기반 업데이트
- 모션 블러 (low alpha)

Day 5: 파티클 시스템
- 간단한 파티클 100개 구현
```

### Week 2: 최적화 + 인터랙션 ⭐ 가장 중요
```
Day 1-3: 파티클 풀, Spatial Index
- Object Pooling (GC 방지)
- Spatial Grid/Quadtree
- O(n²) → O(n) 최적화

Day 4-5: 마우스 이벤트, 툴팁
- hit test (20px threshold)
- 드래그 영역 선택
- DOM + Canvas 혼합 UI
```

### Week 3: 실전 프로젝트
```
Day 1-3: 미니 X-View 구현
Day 4:   최적화 + 리팩토링
Day 5:   문서화 + 포트폴리오
```

**Three.js와 차이점**:
- Three.js: 많은 것이 자동화됨
- Canvas: 모든 것을 수동 관리 (더 어려움)
- GC 문제가 더 심각 → 파티클 풀 필수

---

## 기술적 핵심 패턴

### 1. 파티클 풀 패턴 (Object Pooling)
```javascript
// ✅ 프로덕션급 메모리 관리
function initParticles() {
    particles = [];
    for (let i = 0; i < 3000; i++) {
        particles.push(new Particle()); // 미리 생성
    }
}

function fireBurst() {
    for (let particle of particles) {
        if (!particle.active) {
            particle.activate(); // 재활용
        }
    }
}
```

### 2. 정규화 좌표계
```javascript
// 0~1 범위로 정규화 → 해상도 독립적
this.x = 0.3;  // 화면 30% 위치
const screenX = this.x * width;
```

### 3. 모션 블러 (잔상 효과)
```javascript
// 매 프레임 반투명으로 덮어씌우기
ctx.fillStyle = 'rgba(10, 10, 21, 0.15)';
ctx.fillRect(0, 0, width, height);
```

### 4. Delta Time
```javascript
// 프레임 독립적 물리
const dt = (now - lastTime) / 1000;
this.x += this.vx * dt;
```

---

## 성능 분석

### 현재 성능
```
✅ 1000개 파티클: 55-60fps (안정)
⚠️ 2000개 파티클: 50-60fps (간헐적 드롭)
❌ 3000개 파티클: 40-55fps (불안정)
```

### 병목 지점
```javascript
// 1. findNearestParticle: O(n)
for (let particle of particles) {  // 3000번 순회
    const dist = particle.distanceToMouse(mx, my);
}

// 2. 전체 파티클 업데이트: O(n)
for (let particle of particles) {
    particle.update(dt, now);  // 비활성도 순회
}

// 3. 큐 검색: O(n)
const queueIndex = tunnelQueue.queue.findIndex(
    item => item.particle === this
);
```

### 최적화 방안
```javascript
// Spatial Index 적용 시 예상 성능
현재: O(n) = 3000번 순회
최적화: O(1) = 9개 셀만 검색
→ 333배 빠름

// Active Set 관리
현재: 3000개 전체 순회
최적화: 300개 active만 순회
→ 10배 빠름
```

---

## 프로덕션까지의 거리

```
현재 위치:     [========>..................] 40%
              프로토타입         MVP        프로덕션

MVP까지 (+2주):
├─ Spatial Index 구현      4일
├─ 줌/팬 기능              3일
├─ 데이터 필터링           2일
└─ 아키텍처 리팩토링       5일

프로덕션까지 (+6주):
├─ 10K+ 파티클 성능        2주
├─ 테스트 코드            1주
├─ 크로스브라우저         1주
├─ 접근성 (a11y)          3일
├─ 문서화                 4일
└─ 엣지케이스 처리        1주
```

---

## 학습 성과

### Jun의 강점 (활용됨)
- ✅ Three.js 경험 → 애니메이션 루프 이해도
- ✅ 바닐라 JS → 순수 Canvas 구현에 유리
- ✅ 3D 그래픽 → 좌표 변환, 벡터 연산 익숙

### 새로 습득한 개념
- ✅ Canvas 2D API (Three.js와 차이)
- ✅ 파티클 풀 패턴 (GC 최적화)
- ✅ 큐 자료구조를 시각화하는 방법
- ✅ 정규화 좌표계
- ✅ Delta time 기반 물리

### 개선 필요 영역
- ⚠️ Spatial Index (성능 핵심)
- ⚠️ 아키텍처 설계 (전역 변수 → 클래스 캡슐화)
- ⚠️ 테스트 주도 개발

---

## 실무 적용 시 고려사항

### APM 관점에서의 의미
```
파티클 = 트랜잭션
큐 = 처리 대기열
터널 길이 = Queue Depth (부하 지표)
처리 시간 = 시스템 처리 용량

유입 > 처리 → 큐 쌓임 → 시스템 부하 증가 (시각화!)
유입 < 처리 → 큐 비움 → 시스템 여유
```

### 제니퍼 X-View와 차이
```
제니퍼 (10년 개발):
- 패턴 인식 알고리즘
- 머신러닝 이상탐지
- 엔터프라이즈 통합
- 다중 데이터 소스
- 보안/권한 관리

이 프로토타입 (3-4일):
- 핵심 개념 증명 (PoC)
- 시각적 완성도 높음
- 기본 인터랙션
- 단일 데이터 소스
```

**달성률**: 약 28% (핵심 개념은 100%)

---

## 다음 단계

### 즉시 적용 가능한 개선
1. **Spatial Grid 구현** (우선순위 1)
   - 마우스 호버 최적화
   - 3000개 파티클에서 60fps 보장

2. **큐 검색 최적화**
   - `findIndex` 제거
   - Map 기반 인덱싱

3. **Offscreen Canvas 캐싱**
   - 터널 미리 렌더링
   - 50% 성능 향상 예상

### 장기 로드맵
- 시간축 네비게이션 (줌/팬)
- 데이터 필터링 (에러만 보기)
- WebGL 마이그레이션 (10K+ 파티클)
- 실시간 데이터 소스 연동

---

## 참고 자료

### 필수 문서
- MDN Canvas Tutorial: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial
- Game Programming Patterns: https://gameprogrammingpatterns.com
- HTML5 Canvas Deep Dive (책): Josh Marinacci

### 핵심 파일
```
1. particle-tunnel-stream.html (330줄)
   - 시각 효과 프로토타입
   
2. interactive-particle-tunnel.html (600줄)
   - 인터랙션 추가 버전
   
3. queue-based-particle-tunnel.html (최종)
   - 큐 시스템 구현 ⭐
```

### 코드 하이라이트
```javascript
// 1. TunnelQueue 클래스 (lines 80-120)
// 2. Particle 생명주기 (lines 180-250)
// 3. 터널 길이 계산 (lines 140-155)
```

---

## 결론

**이 프로젝트의 핵심 교훈**:
1. **비율 vs 절대량**: 직관적 UX를 위해서는 절대량 기반 계산이 중요
2. **큐 모델의 힘**: 자료구조를 시각화하면 복잡한 시스템을 이해하기 쉬움
3. **순환 참조 주의**: 터널 길이 ↔ 터널 내 파티클 수의 순환 참조 문제
4. **프로토타입 vs 프로덕션**: 3일 vs 3개월, 40배 차이의 실체

**Jun의 성과**:
- 3-4일만에 동작하는 큐 시각화 프로토타입 완성
- 제니퍼 X-View의 핵심 개념 증명 성공
- 파티클 풀 등 프로덕션급 패턴 적용
- **직관적 문제 인식 능력** (burstSize ≠ 터널 용량)

**점수**: 7.5/10 (Good - 우수한 프로토타입)  
**투자 대비 효과**: 매우 높음 (3일 → 포트폴리오급 결과물)


파라미터 간단 정리
Particles Per Burst: 160
한 번에 발사되는 파티클 개수

160개씩 발사
많을수록 → 유입량 증가 → 큐 쌓임


Burst Interval (ms): 1500
발사 주기

1.5초마다 한 번씩 발사
짧을수록 → 자주 발사 → 유입량 증가


Processing Time (ms): 100
큐에서 처리하는 시간

파티클이 터널(큐)에 머무는 시간
길면 → 천천히 배출 → 큐 쌓임 → 터널 길어짐
짧으면 → 빨리 배출 → 큐 비움 → 터널 짧아짐


Spring Frequency: 1.9
터널 진동 속도

1.9Hz = 초당 1.9번 팽창/수축
높을수록 → 빠른 진동


Spring Amplitude
터널 진동 크기

터널 길이가 얼마나 크게 변하는지
0.0 = 진동 없음
1.0 = 기본 길이의 ±100% 변화


요약
유입: Burst Size × (1000 / Burst Interval)
     = 160 × (1000 / 1500) 
     = 초당 약 107개

배출: 터널에 있는 파티클들이 100ms 후 배출

→ 유입 > 배출 → 큐 쌓임 → 터널 길어짐재시도코드를 실행하려면 설정 > 기능에서 코드 실행 및 파일 생성을 활성화하세요.J진동속도와 진동 크기가 의미하는바가 뭐지?사고 과정사용자가 Spring Frequency(진동 속도)와 Spring Amplitude(진동 크기)가 실제로 무엇을 의미하는지 궁금해하고 있다.
코드를 보면:
javascriptfunction getTunnelLength() {
    return dynamicTunnelLength * (1 + Math.sin(time * springFreq * Math.PI * 2) * springAmp);
}
이것은 터널 길이를 사인파로 변화시키는 것이다.

dynamicTunnelLength: 기본 터널 길이 (큐 크기에 따라 결정)
Math.sin(time * springFreq * Math.PI * 2): 시간에 따른 사인파
springAmp: 진폭 (얼마나 크게 변하는지)

실제 의미:

이것은 시각적 효과일 뿐
실제 큐 동작과는 무관
터널이 호흡하듯 늘었다 줄었다 하는 연출

사용자가 "의미하는 바"를 물어본 것은 - 이게 실제 큐 시스템과 어떤 관련이 있는지, 아니면 단순한 시각 효과인지 궁금한 것 같다.Spring 파라미터의 실제 의미
이것은 순수 시각 효과입니다! 실제 큐 동작과는 무관합니다.

코드 동작
javascriptfunction getTunnelLength() {
    // 기본 터널 길이 (큐 크기로 결정됨)
    const baseLength = dynamicTunnelLength; 
    
    // ✨ 시각 효과: 사인파로 진동 추가
    const oscillation = Math.sin(time * springFreq * Math.PI * 2) * springAmp;
    
    return baseLength * (1 + oscillation);
}
```

---

### 실제 의미

**Spring Frequency (진동 속도)**
```
터널이 "숨쉬듯" 팽창/수축하는 속도

1.9Hz = 1초에 1.9번 늘었다 줄었다
        ~~~~ ~~~~ (빠른 호흡)
```

**Spring Amplitude (진동 크기)**
```
얼마나 크게 호흡하는지

0.3 = 기본 길이의 ±30% 변화
      |====| → |======| → |====|
      짧음      길어짐      다시 짧음
```

---

## 왜 이런 효과를 넣었나?

### 1. 시각적 생동감
```
진동 없음: ========  (정적, 지루함)
진동 있음: ~~~====~~~ (살아있는 느낌)
```

### 2. 실시간 시스템 표현
```
APM 차트는 "살아있는 시스템"을 보여줌
→ 호흡하는 듯한 진동으로 "지금 동작 중"임을 표현
```

### 3. 데이터 밀도 강조
```
큐가 쌓이면:
- 터널 기본 길이 증가 (실제 데이터)
- 진동도 더 크게 보임 (시각적 강조)

정리
요소실제 동작시각 효과큐 크기✅ 터널 기본 길이 결정-Spring Freq-✨ 호흡 속도Spring Amp-✨ 호흡 크기
결론: Spring 파라미터는 장식이고, 실제 큐 동작은 dynamicTunnelLength가 결정합니다.
javascript// 실제 중요한 것
dynamicTunnelLength = f(큐 크기)

// 시각 효과
최종 터널 길이 = dynamicTunnelLength × (1 ± 진동)
진동을 끄고 싶다면 Spring Amplitude: 0으로 설정하면 됩니다!

---

*Last Updated: 2024.11.13*  
*Developer: Jun (Frontend, 5yr exp.)*  
*Project Status: Phase 4 Complete - Queue System*