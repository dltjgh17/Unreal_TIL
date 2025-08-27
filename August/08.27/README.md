
# UE5 네트워킹 핵심: NetRole, Actor Ownership, RPC 전송 경로 총정리

> “RPC는 **Invoke(간접 호출)** 이다. **누가 호출했는지(Invoker)**, **누가 소유하는지(Owner)**, **어디서 실행돼야 하는지(Target)** 를 명확히 구분하면 길을 잃지 않는다.”

---

## 0) 한눈에 보는 큰 그림

1. **NetMode vs NetRole**

* **NetMode**: “이 월드는 서버 월드인가? 클라 월드인가?” → `UWorld` 레벨의 정보
* **NetRole**: “이 **액터**는 현재 이 머신에서 어떤 역할인가?” → 실행 지점 판단의 핵심

2. **Actor Ownership**

* RPC가 **어디로 갈 수 있는지**를 결정하는 문지기(Owner = 특정 `PlayerController`)
* `SetOwner(PlayerController)` 를 하지 않으면 **Server-Owned / Unowned** 로 남음

3. **RPC 키워드**

* `Server` : 클라→서버로 보냄 (서버가 실행)
* `Client` : 서버→특정 클라로 보냄 (해당 클라가 실행) — **그 클라가 “소유”한 액터여야 실제 전송**
* `NetMulticast` : 서버→서버 + 모든 클라 (복제 대상에만 도달)

4. **Reliable / Unreliable**

* **Reliable**: 반드시 실행 (데미지, 인벤토리 변경 등 게임성에 영향)
* **Unreliable(기본)**: 누락 가능 (사운드, 파티클 등 코스메틱)

---

## 1) 이미지로 빠르게 개념 잡기

> 아래 그림은 **호출 주체와 실행 위치**를 표로 요약한 것입니다. (빨간 박스는 오늘 포인트)

### A. 우리가 이해해야 하는 두 표

![alt text](image.png)

### B. 용어 포인트 (Invoke / Ownership 라벨)

![alt text](image-1.png)

![alt text](image-2.png)

### C. UFUNCTION 키워드(서버/클라/멀티캐스트) 강조

![alt text](image-3.png)
### D. 자주 쓰는 3가지 케이스

![alt text](image-4.png)

![alt text](image-5.png)

![alt text](image-6.png)

---

## 2) 핵심 개념 정리

### 2-1. Call vs Invoke

* **Call(정적)**: 컴파일 타임에 호출/실행 위치가 고정. 일반 멤버함수 호출.
* **Invoke(동적)**: 런타임에 실행 위치가 결정. **RPC/함수포인터/다형성**.
  → RPC는 **Invoke한다**고 표현하는 게 정확.

### 2-2. Actor Ownership

* **Client-Owned Actor**

  * 서버에서 스폰 + `SetOwner(LocalPlayerController)`
  * 예: **내 플레이어 캐릭터**, 내 무기, 내 인벤토리 컴포넌트
* **Owned by different client**

  * 화면에 보이는 **다른 플레이어의 캐릭터**
* **Server-Owned Actor**

  * 서버에서 스폰되었지만 Owner 없음
  * 예: 상자, 투사체 스포너, 맵 오브젝트
* **Unowned**

  * 소유자 개념이 적용되지 않는 경우

> **중요**: **Client RPC는 “소유자 클라”에게만 날아간다.** Owner가 아니면 **전송이 드랍**되거나 **서버에서만 실행**되어 보낸 의도가 사라진다.

### 2-3. NetRole (이 액터가 이 머신에서 어떤 역할인가?)

* **Authority**

  * 서버의 원본 액터. **게임에 중대한 로직은 여기서만** 처리.
  * `HasAuthority()` 로 분기.
* **AutonomousProxy**

  * **내가 조종 중인** 폰/컨트롤러의 클라 복제본. 입력→서버로 보냄.
* **SimulatedProxy**

  * 남의 캐릭터를 “시뮬”만 하는 클라 복제본. 서버에서 넘어오는 상태만 반영.
* **None**

  * 존재 X

> API 메모: UE5에서는 `GetLocalRole()` / `GetRemoteRole()` 을 사용(과거 `Role/RemoteRole` 대체).

### 2-4. 자주 쓰는 분기 함수

```cpp
if (Actor->HasAuthority()) { /* 서버 전용 로직 */ }
if (Controller && Controller->IsLocalController()) { /* 내 컨트롤러만 */ }
if (Pawn && Pawn->IsLocallyControlled()) { /* 내가 조종 중인 폰만 */ }
```

---

## 3) RPC 라우팅 표 

### 3-1. **서버에서 RPC를 Invoke** 했을 때

| Actor ownership    | Not replicated | **NetMulticast**   | **Server** | **Client**                    |
| ------------------ | -------------- | ------------------ | ---------- | ----------------------------- |
| Client-owned actor | Server         | Server + **모든 클라** | Server     | **소유 클라**(Owning Client)에서 실행 |
| Server-owned actor | Server         | Server + **모든 클라** | Server     | Server(=의도한 전송 아님)            |
| Unowned actor      | Server         | Server + **모든 클라** | Server     | Server(=의도한 전송 아님)            |

 **결론**:
> * **Client RPC** 를 서버에서 보내려면 **그 함수가 “그 클라가 소유한 액터”** 에 붙어 있어야 한다.
> * **NetMulticast** 는 **서버에서만 Invoke** 해야 의미가 있다.

### 3-2. **클라이언트에서 RPC를 Invoke** 했을 때

| Actor ownership           | Not replicated  | **NetMulticast**(by client) | **Server**  | **Client**      |
| ------------------------- | --------------- | --------------------------- | ----------- | --------------- |
| Owned by invoking client  | Invoking client | Invoking client(=무의미)       | **서버에서 실행** | Invoking client |
| Owned by different client | Invoking client | Invoking client             | **Dropped** | Invoking client |
| Server-owned actor        | Invoking client | Invoking client             | **Dropped** | Invoking client |
| Unowned actor             | Invoking client | Invoking client             | **Dropped** | Invoking client |

**결론**:
> * **Server RPC** 는 **“내가 소유한 액터”** 에서만 성공적으로 서버로 올라간다.
> * 클라가 **NetMulticast를 직접 Invoke** 해도 **본인에게만** 실행되어 의미가 없다.

---

## 4) 실무 케이스 3종 (그림과 함께)

### 케이스 1) **클라가 호출 → 서버에서 실행**해야 하는 로직 (`Server`)

* 예: 공격 입력 → 서버가 데미지 판정 / 탄약 소모 / 보스 HP 반영
* 조건: **Invoker가 소유한 액터**에서 호출해야 드랍되지 않음


**패턴**

```cpp
// 헤더
UFUNCTION(Server, Reliable)
void Server_TryFire();

// 구현: 클라에서 호출 → 서버에서 실행
void AMyCharacter::InputFire()
{
    if (IsLocallyControlled())
    {
        Server_TryFire();     // 클라 → 서버 RPC
        // (코스메틱만) 총구 섬광은 로컬에서도 즉시 보여줄 수 있음
    }
}

void AMyCharacter::Server_TryFire_Implementation()
{
    if (!HasAuthority()) return; // 안전망
    // 탄약 체크, 레이캐스트, 데미지 적용 등 "게임 결과에 영향"
}
```

### 케이스 2) **모든 피어에서 동시에 보여줄** 코스메틱 (`NetMulticast`)

* 예: 폭발 이펙트, SFX, 카메라 흔들림
* 조건: **서버에서 Invoke** 해야 전파가 의미 있음


**패턴**

```cpp
UFUNCTION(NetMulticast, Unreliable)
void Multicast_PlayExplosionFX(FVector Location);

void AMyBomb::Explode()
{
    if (!HasAuthority()) return;
    // 서버에서 폭발 판정/데미지 처리
    Multicast_PlayExplosionFX(GetActorLocation()); // 서버→전원
}

void AMyBomb::Multicast_PlayExplosionFX_Implementation(FVector Loc)
{
    UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ExplosionFX, Loc);
    UGameplayStatics::PlaySoundAtLocation(this, ExplosionSFX, Loc);
}
```

### 케이스 3) **서버가 특정 클라 UI만 갱신**시키고 싶을 때 (`Client`)

* 예: “퀘스트 완료” 팝업을 해당 플레이어에게만 띄우기


**패턴**

```cpp
UFUNCTION(Client, Unreliable)
void Client_ShowQuestComplete(const FText& Msg);

void AQuestSystem::CompleteQuestFor(APlayerController* PC, const FText& Msg)
{
    // PC가 소유한 액터(예: 그 PC의 Controller 또는 그 PC의 Pawn)에 붙은
    // Client RPC를 호출해야 함.
    if (auto* OwnedController = Cast<AMyPlayerController>(PC))
    {
        OwnedController->Client_ShowQuestComplete(Msg);
    }
}

void AMyPlayerController::Client_ShowQuestComplete_Implementation(const FText& Msg)
{
    // 여기서는 UI만 
    HUD->ShowToast(Msg);
}
```

---

## 5) 서버 신뢰 & 검증(Validation)



```cpp
UFUNCTION(Server, Reliable)
void Server_BuyItem(int32 ItemId);

void AMyPlayerController::Server_BuyItem_Implementation(int32 ItemId)
{
    if (!HasAuthority()) return;

    const bool bValid = Shop->IsPurchasable(ItemId, PlayerState->Gold);
    if (!bValid) return;         // 서버에서 파라미터 검증

    Shop->ApplyPurchase(ItemId, PlayerState);  // 신뢰 구간
}
```

---

## 6) 언제 Reliable / 언제 Unreliable?

* **Unreliable(기본)**

  * 코스메틱: 파티클, 사운드, 카메라 셰이크
  * 초당 빈발(틱 수준) 이벤트: “안 와도 큰 문제 없음”
* **Reliable**

  * 결과를 좌우하는 게임 로직: **데미지/HP/탄약/인벤 변경/점수/웨이브 시작·종료**
  * 희소 이벤트: 문 열림, 보상 지급 등

---

## 7) 실전 체크리스트

* [ ] **중대한 로직은 항상 서버에서**: `HasAuthority()` 로 가드
* [ ] **Client→Server** 요청은 **Owner인 액터에서만** 보내기 (아니면 드랍)
* [ ] **서버→Client** 알림은 **그 클라가 소유한 액터**의 Client RPC로
* [ ] 코스메틱은 `NetMulticast + Unreliable`, 핵심 로직은 `Server + Reliable`
* [ ] 스폰 후 꼭 `SetOwner(OwnerPC)` 확인 (UI/무기/인벤 등 내 소유물)
* [ ] UI/애니/사운드 업데이트는 **클라 전용**으로 처리 (서버 낭비 금지)
* [ ] **테스트**: 전용 서버(Dedicated) + 2클라로 “송신/수신/소유/실행 위치” 눈으로 검증

---

## 8) 디버깅 팁

* **실행 위치 로그**

  ```cpp
  UE_LOG(LogTemp, Log, TEXT("[%s] LocalRole=%d HasAuth=%d IsLocalController=%d"),
         *GetName(), (int32)GetLocalRole(), HasAuthority(), IsLocalController());
  ```
* **소유자 확인**: `GetOwner()` / `GetNetOwningPlayer()` / `GetPlayerController`
* **복제 설정**: `bReplicates=true`, `SetReplicates(true)`, 컴포넌트는 `SetIsReplicated(true)`
* **전용 서버에서만 재현되는 문제** 주의: Listen과 Dedicated는 입력/소유 처리 타이밍이 다를 수 있음

---

## 9) 오늘 배운 것 요약(슬라이드용 5줄)

1. **RPC는 Invoke** — 실행 지점은 런타임에 결정
2. **Actor Ownership** 이 **Client/Server/Multicast 라우팅**의 전제
3. **Server RPC**: **Owner인 액터**에서만 성공 (아니면 드랍)
4. **Client RPC**: **그 클라가 소유한 액터**여야 해당 클라에서 실행
5. **중요 로직=Reliable+Server**, **코스메틱=Unreliable(+Multicast)**

---

## 10) 퀴즈 & 정답


1. **Q. `Server` 키워드 RPC는 어디서 호출되고, 어디서 실행?**
   **A.** 보통 **클라(Owner 액터)** 에서 호출 → **서버**에서 실행됨. Owner가 아니면 **Dropped** 됨.

2. **Q. “내 캐릭터”의 NetRole은 보통 무엇인가?**
   **A.** 서버에서는 **Authority**, 내 클라에서는 **AutonomousProxy** .

3. **Q. Reliable vs Unreliable 차이와 예시는?**
   **A.** Reliable은 **전송 보장**(데미지, 인벤 변경 등), Unreliable은 **누락 가능**(사운드/파티클 등).

4. **Q. 서버가 특정 클라 UI만 띄우고 싶을 때 어떻게 할까?**
   **A.** 그 클라가 **소유한 액터(대개 PlayerController)** 의 **Client RPC** 를 호출.

5. **Q. NetMulticast를 클라이언트가 Invoke하면 어떻게 될까?**
   **A.** **자기 자신(Invoking client)** 에서만 실행(전파되지 않음). 의미 있게 쓰려면 **서버에서 Invoke** 해야 함.

6. **Q. Client RPC를 서버-Owned 액터에서 호출하면 어떻게 될까?**
   **A.** **대상 클라가 없어** 의도한 전송이 되지 않고, 사실상 **서버에서만 실행**되거나 **무효**가 됨.
   → 반드시 **그 클라가 “Owner”인 액터** 에서 호출해야 함.

7. **Q. 왜 로컬롤/리모트롤(Autonomous vs Simulated)을 나눴는가?**
   **A.** 서버의 두 액터가 모두 Authority일 때, **“내가 조종 중인지(Autonomous)”** vs **“남을 시뮬 중인지(Simulated)”** 를 구분해 **입력·예측·보정** 처리와 **RPC 타깃 결정**을 정확히 하기 위해서.

---

## 11) 내 프로젝트 적용 가이드

* **UI/애니/사운드**: 클라에서 처리. 멀티 표시 필요하면 **서버 → Multicast(Unreliable)**
* **HP/EXP/레벨/킬카운트/보스HP**: **서버 소스**만 진실. 클라는 표시만.
* **아이템 획득/퀘스트 보상**: 클라 입력 → **Server RPC(Reliable)** → 처리 후 필요 시 **Client RPC/Multicast**
* **Owner 보장**: 스폰 즉시 `SetOwner(OwnerPC)` — UI/인벤/무기 컴포넌트 전부

---

