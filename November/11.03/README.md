# TIL – Phase 시스템 구축 & HUD 연동

## 오늘 배운 핵심

* GameMode는 서버 전용이며 게임 흐름(Phase, Timer)을 관리한다.
* GameState는 클라이언트에 복제되는 상태 저장소 역할을 하며 UI가 읽는 데이터의 근원이다.
* UI는 GameState 변수 변경을 이벤트 바인딩 방식으로 처리한다. (Tick 불필요)
* Phase 진행 순서: Waiting → Build → Combat → Reward → Victory/GameOver
* 타이머는 GameMode에서 매초 감소시키고 GameState에 반영한다.
* 멀티플레이 PIE 테스트 시 포커스 전환은 Shift + F1로 한다.

---

## 개념 정리

| 개념               | 내용                                 |
| ---------------- | ---------------------------------- |
| GameMode         | 서버에서만 실행. Phase, Wave, Timer 로직 담당 |
| GameState        | Phase/Timer 등 상태 변수 유지 및 클라이언트에 복제 |
| PlayerController | HUD 생성 및 UI 연결                     |
| HUD Widget       | GameState 이벤트 수신 후 화면 갱신           |
| OnRep            | 복제된 변수 변경 감지 함수                    |
| Delegate Bind    | 값 변경 이벤트를 UI에 전달하기 위한 바인딩 구조       |

핵심 흐름

```
GameMode → Phase/Timer 변경
→ GameState 변수 업데이트 및 OnRep 호출
→ HUD가 이벤트를 받아 UI 갱신
```

---

## 실전 정리

* Phase Enum 정의
* GameMode에서 Phase 진행 및 타이머 처리
* GameState에 Phase/RemainingTime 저장 후 RepNotify
* Widget Construct에서 GameState 캐스트 후 이벤트 바인딩
* 초기 UI 세팅을 위해 Construct 직후 GameState 값 직접 수동 반영
* 시간 표시 시 Format Text 사용 (`{Minute}:{Second|00}`)

---

## Trouble Shooting

문제
HUD 위젯이 PlayerController에서 선택되지 않음

원인
위젯 에셋 위치 이동으로 인한 Redirector 문제

해결
Content Browser → View Options → Show Redirectors
→ 해당 폴더 우클릭 → Fix Up Redirectors 실행

---

## 체크리스트

* [x] Phase Enum 생성
* [x] GameMode Phase 전환 로직 구현
* [x] GameState RepNotify 설정
* [x] Widget 이벤트 바인딩 및 초기값 처리
* [x] FormatText로 타이머 표시
* [x] 멀티 플레이 테스트 완료

---

## 다음 할 일

* Phase별 실제 기능 연결 (전투, 건설 등)
* Wave/몬스터 스폰 매니저 생성
* 보상 Phase에서 보상 처리 로직 추가
* UI팀 공유용 Phase 시스템 문서 작성

