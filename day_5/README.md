# 🎮 Unreal Engine TIL - 챕터 1-9 \~ 1-11 학습 정리

## ✅ 학습 목표

* 애니메이션 블루프린트 및 블렌드 스페이스를 활용해 캐릭터에 애니메이션을 적용할 수 있다.
* 블루프린트를 활용하여 기본적인 연산, 조건, 반복 구조를 구성할 수 있다.

---

## 📌 \[챕터 1-9] 애니메이션 블루프린트와 블렌드 스페이스

### 🔸 Animation Blueprint (ABP)란?

* 스켈레탈 메시(Skeletal Mesh)의 애니메이션 로직과 상태 전환을 시각적으로 구성할 수 있는 시스템
* Event Graph: 변수 업데이트용
* Anim Graph: 최종 포즈 결정용

🔗 공식 문서: [애니메이션 블루프린트 in UE5](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/animation-blueprints-in-unreal-engine)

---

### 🔸 애니메이션 에셋 종류

| 종류                 | 설명                                |
| ------------------ | --------------------------------- |
| Animation Sequence | 단일 애니메이션 (걷기, 뛰기 등)               |
| Animation Montage  | 여러 시퀀스를 편집, 합성                    |
| Blend Space        | 여러 시퀀스를 축에 따라 블렌딩 (예: Idle ↔ Run) |

---

### 🔸 ABP 생성 및 적용

1. **애니메이션 블루프린트 생성**: `콘텐츠 > 애니메이션 블루프린트`
2. \*\*SK\_Bot 스켈레톤 선택 → 이름: ABP\_Character\`
3. \*\*Idle 애니메이션(A\_Bot\_Idle)\*\*을 Anim Graph에 드래그
4. **Output Pose와 연결**
5. **BP\_Character 블루프린트**에서 애님 클래스를 ABP\_Character로 설정

---

### 🔸 블렌드 스페이스 적용

1. **Blend Space 생성 → 이름: BS\_NBC\_IdleRun**
2. 축 설정: Y축 = Speed
3. 0일 때 Idle, Max일 때 Run 애니메이션 배치
4. 캐릭터의 Speed를 Event Graph에서 벡터 길이로 계산
5. Blend Space와 Speed를 Anim Graph에서 연결

---

### 🔸 State Machine 구성

* **Idle ↔ Run**: Speed > 0 이면 Run, Speed ≤ 0 이면 Idle
* **Jump/Fall/Land** 상태 구현:

  * IsFalling 값을 Event Graph에서 가져와 IsInAir 변수로 저장
  * Jump → Fall: 자동 전환
  * Fall → Land: !IsInAir
  * Land → Idle/Run: 자동 전환

---

## 🧠 느낀 점

> 애니메이션 시스템은 단순히 모션을 재생하는 것이 아니라, 로직 기반으로 상태에 따라 유연하게 처리할 수 있다는 점이 인상 깊었다. 특히 **Blend Space와 State Machine의 연계**가 매우 직관적이면서도 유연하게 구성된다는 점에서 언리얼의 구조적인 강점이 느껴졌다.

---

## 📌 \[챕터 1-10\~11] 블루프린트 연산 기초

### 🔸 변수 사용법

* 변수 생성 후 컴파일해야 사용 가능
* `Alt + 드래그`: Set 노드
  `Ctrl + 드래그`: Get 노드

---

### 🔸 사칙 연산

* 연산자 검색: `+`, `-`, `*`, `/`
* Integer로 연산하면 소수점은 버림 (실수 필요 시 Float 사용)
* 0으로 나누기: 에러 대신 0 반환

---

### 🔸 비교/논리 연산

| 비교 연산                | 의미     |
| -------------------- | ------ |
| `==`, `!=`           | 같음, 다름 |
| `<`, `>`, `<=`, `>=` | 대소 비교  |

| 논리 연산                     | 의미          |
| ------------------------- | ----------- |
| `AND`, `OR`, `NOT`, `XOR` | 조건 병합, 반전 등 |

---

### 🔸 흐름 제어

#### 🔹 Branch (if)

* 조건을 만족하면 True 핀 실행, 아니면 False

#### 🔹 Sequence

* 실행 흐름을 순차적으로 분기

#### 🔹 Flip Flop

* 실행 핀을 번갈아 실행 (A, B → A → B …)

---

## 🔫 실습: 총알 발사 & 재장전 시스템

* **Bullet = 30**
* 마우스 클릭 시 1발 소모
* `Format Text`로 "남은 총알: {Bullet}" 표시
* R키 입력 시 30으로 재장전

🛠️ 버그 수정 과제:

* Bullet ≤ 0일 때 발사 금지
* Bullet = 30일 때 재장전 X

---

### 🔄 반복문

#### 🔹 While Loop

* 조건이 True일 동안 반복

#### 🔹 For Loop

* 지정된 인덱스 범위(First \~ Last) 동안 반복

#### 🔹 구구단 구현 (중첩 For Loop)

```text
For A in 2~9:
  For B in 1~9:
    Print "{A} x {B} = {A*B}"
```

---

### 🗂️ 열거형 (Enum)

* 관련 상수를 한데 묶어 가독성 향상
* Enum 파일 생성 → Slot\_Head, Slot\_Body, Slot\_Weapon 등 추가
* 변수 타입으로 Enum 사용 가능
* `Switch on Enum`으로 분기 처리 가능

---

## 🧠 느낀 점

> 블루프린트를 통해 프로그래밍 없이도 로직을 시각화할 수 있다는 것이 흥미로웠다. 특히 Print → Branch → Sequence → For Loop 구조를 사용해 **간단한 텍스트 기반 게임 로직**까지 구현해보면서 게임 개발의 전반적인 흐름을 체험할 수 있었다.
> 무엇보다 반복문과 조건, 변수의 흐름을 눈으로 볼 수 있어 초보자에게 매우 친화적인 방식이라 느꼈다.

---

## 📌 추가 학습 계획

* 상태머신 전환 시 애니메이션 전이(Transition Rule)의 조건 추가 로직 연습
* 블루프린트 변수 타입별 활용 예제 실습 (Vector, Float 등)
* 간단한 FSM 상태표 그리기 연습 (Idle → Run → Jump 등)

---


