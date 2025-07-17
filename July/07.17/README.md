
# 📘 TIL - Enhanced Input 기반 캐릭터 입력 처리 (Unreal Engine)

## ✅ 오늘 배운 것

### 1️⃣ **Enhanced Input 시스템에서의 캐릭터 입력 처리 구조**

* **PlayerController**는 키보드/마우스 등의 **입력을 감지**하고, 지정된 `InputMappingContext`(IMC)를 **활성화**함.
* 실제 \*\*행동(이동, 점프 등)\*\*은 캐릭터 클래스(`SpartaCharacter`)의 `SetupPlayerInputComponent()`에서 **바인딩 함수 등록**을 통해 처리함.

### 2️⃣ **입력 액션 바인딩 처리 흐름**

1. `SpartaPlayerController`에서 Enhanced Input용 `UInputAction`들을 보유:

   * `IA_Move`, `IA_Jump`, `IA_Look`, `IA_Sprint` 등
2. `SpartaCharacter`에서 `SetupPlayerInputComponent()`에서 각각의 액션을 바인딩:

   ```cpp
   EnhancedInput->BindAction(PlayerController->MoveAction, ETriggerEvent::Triggered, this, &ASpartaCharacter::Move);
   ```
3. 액션이 **Triggered**, **Completed**될 때 연결된 멤버 함수(`Move`, `StartJump`, `StopJump` 등)가 호출됨.

### 3️⃣ **핵심 기술 요소 정리**

* `FInputActionValue`: Enhanced Input 액션 값 전달용 구조체 (`.Get<float>()` 등으로 값 추출 가능).
* `UFUNCTION()`: 입력 바인딩 대상 함수는 반드시 `UFUNCTION()` 매크로로 선언해야 정상 작동함 (언리얼 리플렉션 필수).
* `ETriggerEvent`: 키 입력 시점 지정 (Pressed, Released가 아닌 **Triggered**, **Completed** 등 사용).
* **스프린트 / 점프**: 누름과 뗌을 구분해 각각 별도 함수로 처리.

---

## 💡 느낀 점

* 기존 `InputComponent->BindAxis()` 방식과 달리 Enhanced Input은 **컨트롤러에서 액션 정의**, **캐릭터에서 처리 함수 등록**으로 역할이 분리되어 있어 훨씬 유연하고 모듈화되어 있음.
* `FInputActionValue`의 구조와 **Value Type 설정**이 바인딩 함수 로직 구현 시 중요하다는 점을 깨달음.
* 앞으로는 **Trigger 이벤트에 따라 함수 분리**하는 습관을 들여야 다양한 입력 동작을 자연스럽게 구현할 수 있을 것 같음.

---

## 📌 메모

* **Trigger 이벤트 정리**

  * `Triggered`: 키를 누르고 있는 동안 반복 호출
  * `Completed`: 키에서 손을 뗐을 때 호출
  * `Started`: 키를 처음 눌렀을 때 호출
* **카메라 회전**은 `AddControllerYawInput`, `AddControllerPitchInput`을 활용하여 구현 예정


