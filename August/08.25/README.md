
# 📆 2025-08-25 TIL – UE5 멀티플레이 채팅 UI(입력 위젯 + 컨트롤러 연결)

## 🎯 오늘 배운 핵심 요약

* UMG `EditableTextBox`로 채팅 입력 위젯 `UCXChatInput` 제작, **Enter**로 커밋 시 컨트롤러에 문자열 전달.
* `ACXPlayerController`가 시작 시 **UI Only 입력 모드**로 전환하고, 채팅 입력 위젯을 생성·표시.
* 컨트롤러는 전달받은 메시지를 화면에 `PrintString`으로 출력(프로토타입 단계의 확인용).

## 📖 개념 풀이

* **위젯에서 입력 이벤트 바인딩**
  `NativeConstruct`에서 `OnTextCommitted` 델리게이트를 등록하고 Enter 입력일 때만 처리.

  ```cpp
  EditableTextBox_ChatInput->OnTextCommitted.AddDynamic(
      this, &UCXChatInput::OnChatTextCommitted);
  ```

* **위젯 → 컨트롤러 메시지 전달**
  `GetOwningPlayer()`로 컨트롤러 가져오기 → `ACXPlayerController`로 캐스팅 → `SetChatMessageString` 호출. 이후 입력창 초기화.

  ```cpp
  MyPC->SetChatMessageString(Text.ToString());
  EditableTextBox_ChatInput->SetText(FText::GetEmpty());
  ```

* **컨트롤러에서 UI 세팅**
  `BeginPlay`에서 `FInputModeUIOnly` 적용 후 `CreateWidget`으로 채팅 위젯을 생성·표시.

  ```cpp
  FInputModeUIOnly InputMode;
  SetInputMode(InputMode);
  bShowMouseCursor = true;

  UCXChatInput* ChatWidget = CreateWidget<UCXChatInput>(this, ChatInputWidgetClass);
  ChatWidget->AddToViewport();
  ```

* **메시지 출력(프로토타입)**
  `SetChatMessageString`으로 문자열 저장 후 `PrintChatMessageString`에서 5초간 출력.

  ```cpp
  UKismetSystemLibrary::PrintString(
      this, ChatMessageString, true, true,
      FLinearColor::Green, 5.0f);
  ```

* **게임모드**
  현재 `ACXGameModeBase`는 골격만 존재, 별도 로직 없음.

## 🔧 실전 예시(오늘 만든 걸 에디터에서 동작시키는 절차)

1. **위젯 BP**: `UCXChatInput` 기반 위젯 생성, `EditableTextBox`의 이름을 `EditableTextBox_ChatInput`으로 지정.
2. **플레이어 컨트롤러 설정**: GameMode에서 `PlayerController Class`를 `ACXPlayerController`로 지정.
3. **위젯 클래스 연결**: `ChatInputWidgetClass`에 위젯 BP 할당.
4. **실행 확인**: PIE 실행 → 입력창 표시 → Enter 입력 시 화면 좌측 상단에 메시지 5초 출력.

## 💻 코드 요약

* 입력 위젯에서 **Enter → 컨트롤러 전달 → 입력창 비우기**
* 컨트롤러에서 **UI Only 모드 적용 → 위젯 생성 → 메시지 출력**

---
