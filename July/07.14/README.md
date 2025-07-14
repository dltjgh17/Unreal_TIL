# 📘 2주차 강의 요약 - 캐릭터 구현 & 입력 시스템

## 🎮 캐릭터 관련 클래스 구성

### ✅ Game Mode 클래스

* **게임 전반의 총괄 관리자**

  * 플레이어 생성
  * 플레이어 컨트롤러 할당
  * 게임 규칙 관리

### ✅ 주요 클래스 역할

| 클래스명                 | 역할 설명                             |
| -------------------- | --------------------------------- |
| **Pawn**             | 플레이어가 조작할 수 있는 가장 기본 단위 (상위 클래스)  |
| **Character**        | `Pawn` 상속, 2족 보행 캐릭터를 위한 기본 기능 탑재 |
| **PlayerController** | 캐릭터에 ‘빙의’되어 입력을 처리                |
| **GameState**        | 게임 전역 상태 관리 (점수 등)                |
| **PlayerState**      | 개별 플레이어 상태 관리 (닉네임, 점수 등)         |

### 🧠 캐릭터 vs 폰

* **Pawn**: 범용적, 자유도 높음
* **Character**: 사람형 캐릭터에 특화, 기본 이동 로직 내장

## 🧱 Character 구성 요소

| 컴포넌트명              | 설명                |
| ------------------ | ----------------- |
| Capsule Component  | 충돌 범위 정의          |
| Arrow Component    | 캐릭터 방향 시각화        |
| Skeletal Mesh      | 뼈대 기반 애니메이션 처리    |
| Character Movement | 이동, 점프 등 내장 이동 기능 |
| Spring Arm         | 카메라 거리 및 회전 조절    |
| Camera             | 시점 처리용 카메라        |

---

## 🎥 카메라 구성 예시 코드

### 🔹 Header (`SpartaCharacter.h`)

```cpp
class USpringArmComponent;
class UCameraComponent;

UCLASS()
class SPARTAPROJECT_API ASpartaCharacter : public ACharacter
{
	GENERATED_BODY()

public:
	ASpartaCharacter();

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Camera")
	USpringArmComponent* SpringArmComp;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Camera")
	UCameraComponent* CameraComp;

protected:
	virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
};
```

### 🔹 Source (`SpartaCharacter.cpp`)

```cpp
#include "SpartaCharacter.h"
#include "Camera/CameraComponent.h"
#include "GameFramework/SpringArmComponent.h"

ASpartaCharacter::ASpartaCharacter()
{
	PrimaryActorTick.bCanEverTick = false;

	SpringArmComp = CreateDefaultSubobject<USpringArmComponent>(TEXT("SpringArm"));
	SpringArmComp->SetupAttachment(RootComponent);
	SpringArmComp->TargetArmLength = 300.f;
	SpringArmComp->bUsePawnControlRotation = true;

	CameraComp = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
	CameraComp->SetupAttachment(SpringArmComp, USpringArmComponent::SocketName);
	CameraComp->bUsePawnControlRotation = false;
}

void ASpartaCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);
}
```


## 🎮 PlayerController 핵심 개념

| 기능                  | 설명                       |
| ------------------- | ------------------------ |
| 입력 처리               | Enhanced Input System 활용 |
| 카메라 제어              | 마우스 회전 등                 |
| UI 상호작용             | 예: 클릭, 선택 등              |
| Possess / Unpossess | 폰 또는 캐릭터에 빙의하거나 해제       |



## 🎮 Enhanced Input System

### 구성 요소

* **Input Mapping Context (IMC)**: IA들을 모아놓는 묶음 (스위치 역할)
* **Input Action (IA)**: 추상적인 행동 정의

| IA 예시       | 설명        |
| ----------- | --------- |
| `IA_Jump`   | 점프        |
| `IA_Move`   | 이동 (WASD) |
| `IA_Look`   | 마우스 회전    |
| `IA_Sprint` | Shift 달리기 |

> 📌 **Tip**: 마우스 회전 시 Y축은 `Negate` 옵션을 줘야 자연스럽게 동작함


## 📝 오늘 구현 예정 IA 목록

1. `IA_Move` - WASD 이동
2. `IA_Look` - 마우스 회전
3. `IA_Jump` - 스페이스 점프
4. `IA_Sprint` - 쉬프트 달리기



## ✅ 정리 요약

* `Character`는 `Pawn`의 특화 버전으로, 빠르게 구현 가능하지만 사람형에 한정됨
* `PlayerController`는 입력과 UI, 카메라 등 대부분의 상호작용을 담당
* `Enhanced Input System`은 IMC/IA 구조로 유연하고 깔끔한 입력 처리 제공
* 카메라 시스템은 SpringArm과 Camera 컴포넌트를 통해 구성됨

