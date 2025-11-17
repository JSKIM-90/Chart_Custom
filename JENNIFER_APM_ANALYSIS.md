# 제니퍼 소프트 APM 차트 구현 분석

## 기술 스택 추정

### 1. 렌더링 기술: Canvas API
- **이유**: SVG는 대량의 실시간 데이터 처리에 부적합
- SVG는 각 요소가 DOM 노드로 생성되어 수천~수만 개의 데이터 포인트에서 메모리/성능 병목 발생
- Canvas는 픽셀 기반 렌더링으로 데이터 양과 무관하게 일정한 성능 유지 가능
- 제니퍼 소프트 APM은 자체 개발한 Canvas 기반 차트 엔진 사용으로 추정

### 2. 백그라운드 데이터 처리: Web Worker + WebAssembly
- **핵심 특허 기술**: 실시간 데이터 스트리밍 및 처리 알고리즘
- **Web Worker**: 메인 스레드 블로킹 방지를 위한 필수 구성요소
  - 데이터 집계, 필터링, 변환을 백그라운드에서 수행
  - UI 렌더링 60fps 유지
- **WebAssembly**: JavaScript 대비 5-10배 빠른 연산 성능
  - 복잡한 통계 계산 (평균, 백분위수, 집계)
  - 대용량 데이터 파싱 및 변환
  - 시간 윈도우 기반 데이터 슬라이싱

### 3. 실시간 데이터 처리 - 진짜 핵심 난제

시각화 자체보다 **훨씬 어려운 부분**:

#### 3.1 데이터 스트리밍
- WebSocket 또는 Server-Sent Events를 통한 실시간 데이터 수신
- 네트워크 지연 및 패킷 손실 처리
- 재연결 및 데이터 무결성 보장

#### 3.2 시계열 데이터 집계
- 시간 윈도우 기반 데이터 버킷팅 (1초, 5초, 1분 등)
- 롤링 윈도우 계산 (Moving Average, Percentile)
- 다중 해상도 데이터 유지 (1분, 1시간, 1일)

#### 3.3 메모리 효율성
- **순환 버퍼(Circular Buffer)**: 고정 크기 메모리에서 오래된 데이터 자동 삭제
- **데이터 다운샘플링**: 화면 해상도에 맞춰 데이터 포인트 감소
  - 예: 1920px 너비 화면에 10만 개 데이터 포인트는 불필요 → 최대 1920개로 다운샘플링
- **Virtual Windowing**: 현재 보이는 시간 범위의 데이터만 메모리에 유지

#### 3.4 60fps 렌더링 유지
- RequestAnimationFrame 기반 렌더링 루프
- 더티 체킹(Dirty Checking)으로 변경된 영역만 다시 그리기
- OffscreenCanvas로 렌더링을 Worker로 완전히 이동

## 개발 리소스 추정

### 필요 인력
- **Canvas API 전문 개발자 2-3명**
  - 복잡한 차트 렌더링 (라인, 바, 히트맵 등)
  - 인터랙션 처리 (줌, 패닝, 툴팁, 범례)
  - 애니메이션 및 전환 효과

### 개발 기간 추정
**최소 4-6개월** (2개월은 과소평가)

#### Phase 1: 기본 차트 엔진 (2-3개월)
- Canvas 렌더링 파이프라인 구축
- 기본 차트 타입 구현 (라인, 바, 영역)
- 축, 그리드, 범례 시스템
- 기본 인터랙션 (호버, 클릭)

#### Phase 2: 실시간 데이터 파이프라인 (2-3개월)
- Web Worker 아키텍처 설계
- WebAssembly 데이터 처리 모듈 개발
- 스트리밍 데이터 버퍼 관리
- 다운샘플링 알고리즘 구현

#### Phase 3: 고급 기능 및 최적화 (1-2개월)
- 다양한 차트 타입 확장
- 복잡한 인터랙션 (줌, 패닝, 범위 선택)
- 성능 프로파일링 및 튜닝
- 크로스 브라우저 테스트
- 엣지 케이스 처리

## 핵심 최적화 기법

### 1. OffscreenCanvas
```javascript
// Worker에서 렌더링 수행
const offscreen = canvas.transferControlToOffscreen();
worker.postMessage({ canvas: offscreen }, [offscreen]);
```

### 2. 데이터 다운샘플링
```javascript
// LTTB (Largest Triangle Three Buckets) 알고리즘
function downsample(data, threshold) {
  // 시각적으로 중요한 포인트만 유지
}
```

### 3. 더티 영역 렌더링
```javascript
// 전체가 아닌 변경된 영역만 다시 그리기
ctx.clearRect(dirtyX, dirtyY, dirtyWidth, dirtyHeight);
```

### 4. 메모리 효율적 버퍼
```javascript
// 순환 버퍼로 오래된 데이터 자동 삭제
class CircularBuffer {
  constructor(size) {
    this.buffer = new Float64Array(size);
    this.head = 0;
  }
  push(value) {
    this.buffer[this.head] = value;
    this.head = (this.head + 1) % this.buffer.length;
  }
}
```

## 결론

제니퍼 소프트 APM 수준의 실시간 차트 시스템 개발은:
- **기술적 난이도**: 매우 높음
- **핵심 차트**: Canvas 렌더링보다 **데이터 처리 파이프라인**
- **개발 기간**: 최소 4-6개월 (숙련된 팀 기준)
- **필수 기술**: Canvas, Web Worker, WebAssembly, 실시간 데이터 처리 알고리즘

단순 시각화가 아닌, **성능 최적화된 실시간 모니터링 플랫폼** 개발 프로젝트로 접근해야 함.

## 참고 자료
- Jennifer_Frame 폴더: 제니퍼 소프트 APM UI 레퍼런스 이미지 (60개)
- 현재 프로젝트: Canvas 기반 다양한 차트 시각화 구현 중
