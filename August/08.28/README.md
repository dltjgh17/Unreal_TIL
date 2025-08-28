# 채팅 숫자야구: 판정/시도관리/승패·리셋 & Replication

## 오늘 배운 핵심 요약

* 판정 로직은 서버(Authority, GameMode)에서. 난수 생성·유효성 검증·판정은 신뢰 가능한 서버에서 처리
* 플레이어 식별은 PlayerState가 담당. 접속 순서로 `Player1/2/...` 부여 → UI/로그용 식별자 일원화
* 핵심 데이터는 Replication. `PlayerNameString`, `CurrentGuessCount`(시도 수), (필요 시) `MaxGuessCount` 를 PlayerState에서 `UPROPERTY(Replicated)` + `DOREPLIFETIME`
* 시도 횟수는 서버에서 증가. 올바른 추측 입력마다 GameMode가 해당 PlayerState의 `CurrentGuessCount++`
* 승리/무승부/리셋 판정은 GameMode. 3S → 승리, 전원 시도 한도 도달 → 무승부, 두 경우 모두 비밀숫자 재생성 + 시도 수 초기화
* 전체 공지(알림)는 PlayerController Replicated 프로퍼티 → UMG 바인딩. `NotificationText`를 복제해 클라 위젯이 자동 반영



## 개념 풀이

### 1) 서버 권위와 판정 위치

* 난수 생성/판정/승패 결정은 클라 조작을 막기 위해 서버에서 단일 소스로 관리해야 한다
* Unreal에선 `AGameModeBase`가 서버 전용이므로 가장 적합

### 2) PlayerState의 역할

* 접속자별 지속/공유 상태(이름, 스코어, 시도 수 등) 저장소
* 모든 클라이언트에 복제되므로 UI 표시에 용이

### 3) Property Replication의 필수 조건

* `UPROPERTY(Replicated)` + `GetLifetimeReplicatedProps(...)` 에서 `DOREPLIFETIME`
* Replicated 설정/등록 누락 시, 클라에서 값이 기본값 유지 → UI에 “None (0/0)”처럼 보일 수 있음

### 4) 승패·무승부·리셋 흐름

1. 채팅 메시지 끝 3자리 → 숫자 후보 판단
2. `JudgeResult(3S/2S1B/…/OUT)` 산출
3. 올바른 입력이면 `IncreaseGuessCount` 실행
4. `JudgeGame`:

   * `StrikeCount == 3` → 승리 공지, 리셋
   * 전원 시도 만료 → 무승부 공지, 리셋

### 5) 공지 표시

* PlayerController에 `NotificationText`(Replicated) 보관 → UMG 바인딩으로 즉시 반영



## 실전 예시

1. 클라: “123” 입력 → `ACXPlayerController::ServerRPCPrintChatMessageString` 호출
2. 서버: GameMode가

   * `IsGuessNumberString("123")` 검사
   * `JudgeResult(Secret, "123")` → “1S1B” 등
   * `IncreaseGuessCount(그 플레이어의 PlayerState)`
   * `JudgeGame(해당 스트라이크 수)` → 승/무/지속 판정
   * 모든 컨트롤러에 결과 메시지 전파(클라이언트 RPC) + 필요 시 `NotificationText` 설정
3. 클라: 채팅창/공지 위젯 업데이트



## 코드 스냅샷

### 1) 판정

```cpp
FString ACXGameModeBase::JudgeResult(const FString& Secret, const FString& Guess)
{
    int32 S=0, B=0;
    for (int32 i=0; i<3; ++i)
    {
        if (Secret[i] == Guess[i]) { ++S; }
        else if (Secret.Contains(FString::Printf(TEXT("%c"), Guess[i]))) { ++B; }
    }
    return (S==0 && B==0) ? TEXT("OUT") : FString::Printf(TEXT("%dS%dB"), S, B);
}
```

### 2) PlayerState Replication

```cpp
UPROPERTY(Replicated) FString PlayerNameString;
UPROPERTY(Replicated) int32 CurrentGuessCount;
UPROPERTY(Replicated) int32 MaxGuessCount;

void ACXPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& Out) const
{
    Super::GetLifetimeReplicatedProps(Out);
    DOREPLIFETIME(ThisClass, PlayerNameString);
    DOREPLIFETIME(ThisClass, CurrentGuessCount);
    DOREPLIFETIME(ThisClass, MaxGuessCount);
}
```

### 3) 시도 증가 & 리셋

```cpp
void ACXGameModeBase::IncreaseGuessCount(ACXPlayerController* PC)
{
    if (auto* PS = PC->GetPlayerState<ACXPlayerState>()) { PS->CurrentGuessCount++; }
}

void ACXGameModeBase::ResetGame()
{
    SecretNumberString = GenerateSecretNumber();
    for (auto* PC : AllPlayerControllers)
        if (auto* PS = PC->GetPlayerState<ACXPlayerState>()) { PS->CurrentGuessCount = 0; }
}
```

### 4) 공지

```cpp
UPROPERTY(Replicated, BlueprintReadOnly)
FText NotificationText;

void ACXPlayerController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& Out) const
{
    Super::GetLifetimeReplicatedProps(Out);
    DOREPLIFETIME(ThisClass, NotificationText);
}
```



## 체크리스트

* [ ] 난수/판정/승패 로직이 GameMode(서버)에만 존재한다
* [ ] PlayerState의 `UPROPERTY(Replicated)`와 `DOREPLIFETIME`을 모두 등록했다
* [ ] OnPostLogin에서 PlayerState 이름 부여가 정상 동작한다
* [ ] 올바른 입력일 때만 시도 수가 증가한다
* [ ] 승리/무승부 후 `ResetGame()`으로 초기화된다
* [ ] `NotificationText` 복제가 되고, UMG 바인딩으로 반영된다
* [ ] Listen/Dedicated + 다중 클라에서 동작 확인했다



## 흔한 이슈

* PIE에서 이름/시도 수가 갱신 안 보임 → Replication 누락 또는 바인딩 미설정
* MaxGuessCount는 모든 플레이어 공통이고 바뀌지 않으면 Replicate 필요 없음
* JudgeResultString 파싱은 안전하게 “OUT” 케이스 분기 추천
* OnLogout에서 AllPlayerControllers 정리 필요



## 퀴즈

1. 판정 로직을 클라이언트에서 하면 안 되는 이유는?
2. Replication이 동작하려면 코드에서 무엇을 반드시 작성해야 하나요?
3. 승리/무승부 이후 즉시 해야 할 공통 동작은?
4. MaxGuessCount를 Replicate 하지 않아도 되는 경우는?

<details>
<summary>정답 보기</summary>

1. 클라 조작/해킹 가능성 때문에 신뢰 불가. 서버 권위 확보 필요
2. `UPROPERTY(Replicated)` 지정 + `GetLifetimeReplicatedProps`에서 `DOREPLIFETIME` 등록
3. `ResetGame()` – 비밀숫자 재생성, 각 PlayerState 시도 수 초기화
4. 전 플레이어 공통 상수이고 런타임 변경이 없다면 Replicate 불필요

</details>  
