# 디버그 계약과 자동 플레이

## 프로젝트에 맞춘 최소 계약

다음은 설계 예시다. 실제 게임의 상태와 생명주기에 맞게 구현하고 메서드가 동작하는지 확인한다. 빈 메서드나 상수 상태를 구현으로 제출하지 않는다.

```ts
interface GameDebug {
  version: 1;
  snapshot(): {
    scene: string;
    seed: number;
    tick: number;
    ready: boolean;
    pendingJobs: number;
    player: Record<string, unknown>;
    progress: Record<string, unknown>;
    errors: string[];
  };
  scenarios(): string[];
  loadScenario(name: string, seed: number): Promise<void>;
  metrics(): {
    sampleCount: number;
    frameMsP50: number | null;
    frameMsP95: number | null;
    counters: Record<string, number>;
  };
}
```

- snapshot은 라이브 객체 참조를 노출하지 않는 직렬화 가능한 복사본으로 만든다. 수치 단위를 정하고 NaN/Infinity를 검증한다.
- loadScenario는 이름을 검증하고, 엔진이 실제 플레이 가능한 상태가 된 후 완료한다. 실패 시 구체적인 오류를 반환한다.
- 별도 `step(ticks)`가 필요하면 실시간 루프를 정지한 상태에서만 허용해 이중 진행을 방지한다.
- 테스트 명령을 추가한다면 허용된 명령만 받고 유효성을 검사한다. 가능하면 일반 게임과 같은 규칙 함수를 사용한다.
- 지형 스트리밍처럼 작업이 계속 생기면 모든 pendingJobs=0을 무조건 기다리지 말고 해당 카메라 영역의 준비 조건을 정의한다.
- 개발 빌드 플래그를 기본 게이트로 사용한다. E2E용 빌드는 별도 설정하며 배포용 빌드에는 조작 코드를 포함하지 않는다.

## 시나리오 정의

각 장면에 이름, seed, 초기 상태, 준비 조건, 입력 순서, 기대 상태, 타임아웃을 둔다. 시뮬레이션 결과에 날짜가 관여하면 시계도 고정한다.

| 게임 | 정상 흐름 | 경계/실패 | 복원/장시간 |
|---|---|---|---|
| 편의점 | morningRush | emptyShelf, checkoutQueue | dayEnd, saveReload |
| 방치형 RPG | firstBattle | defeat, inventoryFull | offlineReturn, saveReload |
| 가족 캠프 | prepareDinner | missingIngredient | stageComplete, restart |
| 우주 탐험 | orbit, descent | terrainPending, collision | landing, saveReload |

이는 선택 가능한 예시이며 모든 이름을 구현할 의무는 없다. 멀티플레이는 실제 여러 클라이언트와 권한 있는 서버 상태로 확인해야 한다. 한 클라이언트의 상태 주입은 온라인 협동 검증을 대신하지 않는다.

## Playwright 검증 패턴

프로젝트에서 사용하는 브라우저 실행 도구와 Playwright 버전에 맞춰 작성한다. 이 예시는 테스트 설계용이며 복사만으로 완성된 테스트가 아니다.

1. 새 브라우저 컨텍스트와 전용 저장 네임스페이스로 시작한다.
2. 탐색 전에 pageerror, console.error를 수집한다.
3. 게임을 열고 진단 API를 기다린 뒤 시나리오를 불러온다.
4. 준비 상태를 조건 기반으로 기다린다. 임의의 긴 sleep으로 대신하지 않는다.
5. 전 상태를 캡처하고 실제 입력을 보낸다. 캔버스 포인터는 크기/좌표 변환을 고려하고 키는 누름/해제를 모두 실행한다.
6. 기대 상태를 제한된 시간 동안 기다리고 스크린샷과 후 상태를 저장한다.
7. 실제 규칙의 불변식과 의도한 결과를 검증하고 처리되지 않은 오류를 실패로 취급한다.
8. 실패하면 trace, 상태, 오류, 스크린샷을 남긴다. 첫 입력이 먹지 않는 경우 포커스·오버레이·좌표계를 확인한다.

검증은 두 경로를 구분한다.
- 상태 준비/격리 검증: 진단 API로 어려운 상황을 빠르게 구성한다.
- 플레이 검증: 입력→실제 처리→관찰 가능한 결과를 확인한다. completeStage/setBalance 같은 명령으로 결과를 만들어놓고 플레이 성공이라고 하지 않는다.

## 측정과 증거

실행 기록에 빌드 식별자, 시나리오/seed, 브라우저와 headless 여부, 렌더링 백엔드, 뷰포트/DPR, 워밍업, 측정 시간, 샘플 수를 남긴다.

프레임 시간은 P50/P95(ms)와 샘플 수를 함께 기록한다. 표본이 없으면 null로 보고한다. 장기 누적 대신 제한된 샘플 버퍼를 사용한다. 수집 불가 항목은 추정값으로 채우지 않는다.

2D는 활성 객체/물리 바디/풀 크기/입력 반응, 3D는 draw call/triangle/texture/geometry/worker 대기/폐기 작업/전송 바이트 등 문제와 관련된 것만 수집한다. 브라우저에서 GPU 시간이나 메모리를 얻을 수 없으면 그 한계를 명시한다.

전후 비교는 동일 조건으로 수행한다. 실제 기기 FPS, headless 프레임 간격, 알고리즘 시뮬레이션 결과를 구분하고 성능 향상과 기능 회귀를 함께 판단한다. 경계 근처이거나 변동이 크면 반복 측정하되 충분한 근거가 생기면 불필요한 재실행을 멈춘다.
