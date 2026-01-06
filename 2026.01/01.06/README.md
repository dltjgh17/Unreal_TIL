# [UE5] SeamlessTravel 이후 Pawn 미스폰/미스매치 해결 (PlayerState 기반 스폰)

## 💔 문제 개요

로비 맵에서 **캐릭터 선택 → 준비 완료 → SeamlessTravel로 인게임 전환**했을 때 아래 문제가 발생했습니다.

* 인게임 맵 전환 후 **플레이어 Pawn이 스폰되지 않음**
* 스폰은 됐지만 **선택한 캐릭터와 다른 클래스(DefaultPawn 등)로 생성**
* 일부 클라이언트는 정상인데 **특정 클라이언트만 미스매치/미스폰**
* Travel은 성공 로그가 뜨는데 **PostLogin/RestartPlayer 호출 타이밍이 기대와 다름**

---

## ❤️‍🩹 원인 분석

### 1) SeamlessTravel의 “유지 객체” 특성 오해

SeamlessTravel은 맵이 바뀌어도 일부 객체가 유지됩니다.

* **유지됨**: `PlayerController`, `PlayerState` (대표)
* **유지 안 됨**: `Pawn`, 월드 액터들, `GameMode/GameState`(맵마다 새로 생성)

즉 로비에서 “캐릭터를 스폰해서 들고 가는 방식”은 Travel 이후 깨지기 쉽고,
**결국 인게임 맵에서 Pawn을 새로 스폰하는 구조가 정답**이었습니다.

---

### 2) “선택 정보”가 Pawn에만 남아있던 구조

로비에서 시연용 Pawn을 띄워 테스트하는 과정에서
“선택한 캐릭터 정보”를 Pawn에만 저장하면, Travel 이후 Pawn이 사라지며 **선택 정보도 함께 유실**됩니다.

결과적으로 인게임 맵에서 스폰할 때,

* 어떤 클래스로 스폰해야 할지 근거가 없어 `DefaultPawnClass`로 가거나
* 스폰 실패/Null 상태가 되어 **미스폰**이 발생합니다.

---

### 3) 스폰 시점/권한(서버) 흐름을 벗어난 처리

SeamlessTravel 이후 스폰은 **서버 권한**에서 이루어져야 하고,
엔진 기본 흐름인 아래 루트를 따라야 안정적입니다.

* `HandleStartingNewPlayer()`
* `RestartPlayer()`
* `ChoosePlayerStart()`

로비에서 스폰한 Pawn을 억지로 유지하거나, 클라이언트에서 소유권/스폰을 건드리면
**PIE/Listen/Dedicated** 환경에 따라 결과가 달라지며 재현성이 깨집니다.

---

## ✅ 해결 전략

### 핵심 전략

**선택 정보는 PlayerState에 저장하고, Pawn은 인게임에서 재스폰한다.**

---

### 1) 로비에서 “선택한 캐릭터”를 PlayerState에 저장

캐릭터 버튼 클릭 시 서버 RPC로 다음 값 중 하나를 저장합니다.

* `SelectedCharacterId` (추천: enum/id)
* 또는 `SelectedCharacterClass`

PlayerState는 SeamlessTravel에서 유지되므로 인게임에서도 그대로 사용할 수 있습니다.

> 포인트: 선택 데이터는 **Pawn이 아니라 PlayerState**에 두기

---

### 2) 인게임 GameMode에서 PlayerState 기반으로 스폰 클래스 결정

인게임 GameMode에서 아래 중 하나를 오버라이드합니다.

* `GetDefaultPawnClassForController()`
* 또는 `SpawnDefaultPawnFor()`

그리고 `PlayerState`의 선택 정보를 읽어서 **올바른 Pawn 클래스**를 반환/스폰하도록 구성합니다.
이후 `RestartPlayer()` 흐름에서 자동으로 정확한 캐릭터가 생성됩니다.

---

### 3) 로비 시연용 Pawn은 Travel 전에 정리

로비에서 시연을 위해 띄운 Pawn이 있다면, Travel 전에 정리합니다.

* `UnPossess()`
* `Destroy()`

Travel 이후에는 인게임에서만 일관되게 스폰하도록 만들어
**중복 소유/유령 Pawn/소유권 꼬임** 문제를 근본적으로 제거했습니다.

---

## 🎯 결과

* SeamlessTravel 이후에도 모든 클라이언트가 **자신이 선택한 캐릭터로 정확히 스폰**
* Dedicated/Listen/PIE 환경 차이에도 결과가 안정적으로 동일
* 로비는 **선택/준비 상태 관리**, 인게임은 **전투용 스폰**으로 책임이 분리되어 유지보수가 쉬워짐

---

## 🔍 체크리스트 (재발 방지)

* [ ] 선택 정보는 Pawn이 아니라 **PlayerState에 저장**했는가?
* [ ] 인게임 스폰은 **서버**에서, `RestartPlayer` 흐름에 맞춰 수행되는가?
* [ ] 로비의 시연 Pawn은 Travel 전에 **UnPossess/Destroy**로 정리했는가?
* [ ] `GetDefaultPawnClassForController` 또는 `SpawnDefaultPawnFor`가 선택 값을 반영하는가?

---

## 🧠 한 줄 회고

SeamlessTravel에서 “유지되는 객체/유지되지 않는 객체”를 정확히 이해해야
**선택 정보 유실과 스폰 미스매치**를 근본적으로 막을 수 있다.

