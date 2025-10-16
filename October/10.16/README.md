# 아이템 파트 트러블 슈팅
## 1️⃣ 픽업(Overlap)이 작동하지 않을 때

**증상**

- 캐릭터가 아이템에 닿아도 아무 반응이 없음
- 이벤트 함수가 호출되지 않음

**확인 사항**

- `USphereComponent` 또는 `UBoxComponent`의 **Collision Preset**이 `OverlapAllDynamic` 혹은 **Pawn과 Overlap**으로 설정되어 있는지 확인
- 아이템 클래스에 `bReplicates = true` 설정 여부
- BeginPlay에서 `OnComponentBeginOverlap.AddDynamic()`이 정상적으로 연결되었는지 확인
- `PickupDelaySeconds`를 두는 경우, `bCanBePickedUp` 값이 `true`인지 확인

**예시**

```cpp
Sphere->OnComponentBeginOverlap.AddDynamic(this, &AItemPickup::OnOverlap);

```

---

## 2️⃣ 서버 권한(Authority) / 복제(Replication) 문제

**증상**

- 호스트는 정상 작동하지만 클라이언트에서는 아이템을 못줍거나 반응 없음

**원인**

- 클라이언트가 서버 권한이 필요한 함수를 직접 실행함

**해결 방법**

- Overlap 감지는 클라이언트에서도 가능하지만, **실제 아이템 획득 로직은 서버에서만 처리**해야 함
- 클라이언트 → 서버로 요청은 **Server RPC**를 사용

**예시**

```cpp
UFUNCTION(Server, Reliable)
void Server_RequestPickup(AItemPickup* Target);

UFUNCTION(Server, Reliable)
void Server_AddItem(EItemType Item);

```

---

## 3️⃣ 상자 드롭 / 디스폰 타이머 문제

**증상**

- 상자를 부수자마자 아이템이 사라짐
- 또는 일정 시간 후에도 사라지지 않음

**원인**

- 디스폰 타이머를 클라이언트에서 실행하거나, 서버만 Destroy 호출

**해결 방법**

- 스폰과 디스폰 타이머는 **항상 서버에서 실행**
- `Destroy()`는 서버에서 호출하면 자동으로 클라이언트에서도 사라짐

**예시**

```cpp
void AItemPickup::BeginPlay() {
  Super::BeginPlay();
  if (HasAuthority() && AutoDespawnSeconds > 0.f) {
    GetWorld()->GetTimerManager().SetTimer(
      DespawnHandle, this, &ThisClass::ServerDespawn, AutoDespawnSeconds, false
    );
  }
}

void AItemPickup::ServerDespawn() {
  if (HasAuthority()) Destroy();
}

```

---

## 4️⃣ 효과(Heal, SpeedBoost) 적용 및 원복 문제

**증상**

- 힐 효과는 적용되지만 스피드 버프는 안 됨
- 혹은 지속시간 후 원복되지 않음

**원인**

- 타이머 중복으로 인해 마지막 타이머만 남거나 초기화가 안 됨
- 속도 수치가 Replicate되지 않음

**해결 방법**

- 타이머 실행 전 항상 `ClearTimer()` 호출
- 속도나 체력 수치는 `ReplicatedUsing=OnRep_`으로 설정

**예시**

```cpp
void UAttributeComponent::ApplySpeedBuff(float Delta, float Duration) {
  if (!HasAuthority()) return;
  SetCurrentSpeed(CurrentSpeed + Delta);

  GetWorld()->GetTimerManager().ClearTimer(Timer_SpeedBuff);
  GetWorld()->GetTimerManager().SetTimer(
    Timer_SpeedBuff,
    [this]() { SetCurrentSpeed(BaseSpeed); },
    Duration, false
  );
}

```

---