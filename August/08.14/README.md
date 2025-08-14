
## 목표

* 레벨 10에서 보스가 소환되면: **보스 HP 바 표시 + 초기값 반영**
* 전투 중 보스 HP가 변하면: **HUD 자동 갱신**
* 보스가 사망/소멸하면: **보스 HP 바 숨김**

## 왜 이렇게 쓰는가 (설계 이유)

1. **UI 갱신은 보스 생명주기 이벤트에 연결**해야 누락이 없습니다.
   → `BeginPlay()`에서 한 번만 연결해 두면, 데미지/치유/기타 HP 변화 모든 경로를 자동 커버.
2. **UI 접근은 GameMode를 경유**합니다.
   → HUD 위젯 포인터 보유/생성 책임이 GameMode에 있으므로, 보스 → GameMode → HUD 경로가 안전합니다. (이미 `OnBossSpawned`, `UpdateBossHPUI`가 준비되어 있음)&#x20;
3. **소멸 시 정리(EndPlay)** 를 해두면, 레벨 재시작, 보스 사망 등에서 잔상 없이 UI를 닫을 수 있습니다.

## 변경 가이드 (BossMonsterBase만 수정)

### 1) 헤더에 핸들러 선언 (사건 수신 + 정리)

`BossMonsterBase.h` (요지)

```cpp
// 보스 HP 변화를 HUD로 전달할 핸들러
UFUNCTION()
void OnBossHPChanged(float CurrentHP, float MaxHP);

// 소멸/맵 전환 시 UI 닫기
virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
```

> 이유: 델리게이트 바인딩은 `UFUNCTION` 시그니처가 필요합니다. EndPlay는 보스가 사망/맵 종료될 때 HUD를 숨기기 위해.

### 2) BeginPlay에 “표시 + 초기값 갱신 + 델리게이트 연결”

`BossMonsterBase.cpp` (추가/수정 부분)

```cpp
void ABossMonsterBase::BeginPlay()
{
    Super::BeginPlay();

    // ... (기존 오버레이/몽타주/패턴 타이머 코드 그대로) ...

    // 1) 보스 등장 시 HUD 보이기 + 초기값 반영
    if (AMainGameMode* GM = Cast<AMainGameMode>(UGameplayStatics::GetGameMode(this)))
    {
        GM->OnBossSpawned(this);                 // -> HUDWidget->SetBossHPVisible(true) 호출됨 :contentReference[oaicite:5]{index=5}
        GM->UpdateBossHPUI(CurrentHP, MaxHP);    // 초기 HP 표시 갱신 :contentReference[oaicite:6]{index=6}
    }

    // 2) HP 변화 자동 갱신 연결
    // (A) HPComponent가 브로드캐스트를 제공한다면 그걸 구독
    if (UHPComponent* HPComp = FindComponentByClass<UHPComponent>())
    {
        // HPComponent에 OnHPChanged(float,float) 델리게이트가 있다고 가정.
        // 없다면 아래 (B)로 대체하세요.
        HPComp->OnHPChanged.AddDynamic(this, &ABossMonsterBase::OnBossHPChanged);
    }
    else
    {
        // (B) 컴포넌트 델리게이트가 없다면: TakeDamage에서 직접 갱신(아래 3) 참고)
        UE_LOG(LogTemp, Warning, TEXT("Boss has no HPComponent; will update HUD in TakeDamage()"));
    }
}
```

> 이유:
>
> * `OnBossSpawned()`는 이미 GameMode에 구현되어 보스 HP 바를 **보이게** 합니다.&#x20;
> * `UpdateBossHPUI()`는 `HUDWidget->UpdateBossHP(Current, Max)`를 호출하므로 **실제 ProgressBar 값 변경**이 됩니다. &#x20;
> * HPComponent 델리게이트를 구독하면 **모든 HP 변화 경로**(데미지/치유/상태이상 등)를 자동 반영합니다.

### 3) 델리게이트 핸들러 구현 + (대안) TakeDamage fallback

`BossMonsterBase.cpp`

```cpp
void ABossMonsterBase::OnBossHPChanged(float NewHP, float InMaxHP)
{
    if (AMainGameMode* GM = Cast<AMainGameMode>(UGameplayStatics::GetGameMode(this)))
    {
        GM->UpdateBossHPUI(NewHP, InMaxHP); // HUD 갱신 (GameMode 경유) :contentReference[oaicite:10]{index=10}
    }
}

// ── 델리게이트가 없는 프로젝트라면, TakeDamage 끝에서 직접 갱신해도 됩니다.
float ABossMonsterBase::TakeDamage(float DamageAmount, FDamageEvent const& DamageEvent,
                                   AController* EventInstigator, AActor* DamageCauser)
{
    float Actual = Super::TakeDamage(DamageAmount * DefenceModifier, DamageEvent, EventInstigator, DamageCauser);

    // 여기서 CurrentHP가 감소되도록 되어 있다고 가정 (당신 코드에 이미 존재)
    // 감소 처리 이후 HUD 동기화:
    if (AMainGameMode* GM = Cast<AMainGameMode>(UGameplayStatics::GetGameMode(this)))
    {
        GM->UpdateBossHPUI(CurrentHP, MaxHP);
    }

    // (기존 Phase2 전환/로그 로직 유지)
    if ((CurrentHP / MaxHP) < 0.5f)
    {
        UE_LOG(LogTemp, Warning, TEXT("HP가 50 이하로 떨어졌습니다. Current HP: %f, Max HP: %f"), CurrentHP, MaxHP);
    }
    if (!bHasEnteredPhase2 && !bIsPhaseChanging && (CurrentHP / MaxHP) < 0.5f)
    {
        UE_LOG(LogTemp, Warning, TEXT("HP가 50 이하로 떨어졌습니다. Phase2로 전환합니다."));
        StartPhase2Transition();
    }

    return Actual;
}
```

> 이유:
>
> * **우선순위는 델리게이트 바인딩**입니다(가장 견고).
> * 델리게이트 지원이 없다면 **fallback**으로 `TakeDamage`에서 즉시 반영해도(가장 쉬움) UI는 동작합니다.

### 4) 보스 종료 시 UI 정리

`BossMonsterBase.cpp`

```cpp
void ABossMonsterBase::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (AMainGameMode* GM = Cast<AMainGameMode>(UGameplayStatics::GetGameMode(this)))
    {
        GM->UpdateBossHPUI(0.f, 1.f);     // 0%로 정리(선택)
        GM->OnBossSpawned(nullptr);       // 굳이 전달할 필요는 없지만 의미상 호출 가능
        GM->GetHUDWidget() ? GM->GetHUDWidget()->SetBossHPVisible(false) : void(); // 바로 숨김도 가능
        // 또는 그냥: GM->UpdateBossHPUI(0,1); GM->ShowBossHP(false);
    }
    Super::EndPlay(EndPlayReason);
}
```

> 이유: 보스가 죽거나 맵이 닫히면 **보스 HP UI를 숨겨** 다음 라운드에 잔상이 남지 않습니다.

---

## 요약

* **UWBP\_HUD / MainGameMode.cpp는 로직상 그대로 OK.** 이미 필요한 함수가 정확히 구현되어 있음. &#x20;
* **MainGameMode.h는 선언 문법만 살짝 정정**(접두 지우고 세미콜론 추가).&#x20;
* **BossMonsterBase만 보강**:

  * `BeginPlay()`에서: 보스 등장 통지 + 초기 HP 반영 + 델리게이트 구독
  * `OnBossHPChanged()`에서: HUD 갱신
  * (옵션) `TakeDamage()`에서 fallback 갱신
  * `EndPlay()`에서: UI 숨김 정리

위 수정만 반영하시면, **레벨 10에서 보스가 나오면 HP바가 켜지고, 전투 중 실시간으로 연동**되며, 종료 시 깔끔히 사라집니다.
