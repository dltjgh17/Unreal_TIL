# 🎮 인터페이스 기반 아이템 시스템 구현

> SpartaProject - 3강 1차 강의 정리
> 아이템 설계의 공통 인터페이스 정의 및 다양한 아이템 클래스 구현


## 📌 학습 개요

* 인터페이스란?
  → 클래스 간 **공통된 기능을 강제**하는 **계약서** 같은 존재
  → Unreal Engine에서는 `UInterface`를 통해 구현 가능

* 상속 vs 인터페이스

  | 구분    | 상속       | 인터페이스        |
  | ----- | -------- | ------------ |
  | 목적    | 기능 물려받기  | 기능 구현 약속     |
  | 다중 구현 | ❌        | ✅ 가능         |
  | 구현 여부 | 일부 구현 가능 | 반드시 함수 구현 필요 |


## 🧩 인터페이스 설계

```cpp
class IItemInterface
{
    GENERATED_BODY()

public:
    virtual void OnItemOverlap(AActor* OverlapActor) = 0;
    virtual void OnItemEndOverlap(AActor* OverlapActor) = 0;
    virtual void ActivateItem(AActor* Activator) = 0;
    virtual FName GetItemType() const = 0;
};
```

* 공통 함수 정의
* 아이템 클래스는 반드시 이 함수들을 구현해야 함

## 🏗️ 아이템 클래스 구조

```plaintext
ABaseItem (인터페이스 구현)
├── ACoinItem
│   ├── ABigCoinItem
│   └── ASmallCoinItem
├── AHealingItem
└── AMineItem
```

## ⚙️ 공통 베이스 클래스 (`ABaseItem`)

* 공통 구성 요소:

  * `USphereComponent`: 충돌 영역
  * `UStaticMeshComponent`: 외형
* 공통 로직:

  * `OnItemOverlap()` - 오버랩 시 효과 발동
  * `ActivateItem()` - 아이템 고유 동작 수행
  * `DestroyItem()` - 아이템 제거

```cpp
void ABaseItem::OnItemOverlap(...) {
    if (OtherActor && OtherActor->ActorHasTag("Player")) {
        ActivateItem(OtherActor);
    }
}
```


## 🪙 코인 아이템

### `ABigCoinItem`

```cpp
PointValue = 50;
ItemType = "BigCoin";
```

### `ASmallCoinItem`

```cpp
PointValue = 10;
ItemType = "SmallCoin";
```

* ActivateItem: 즉시 사라짐 (`DestroyItem()` 호출)


## ❤️ 힐링 아이템 (`AHealingItem`)

```cpp
HealAmount = 20.0f;
ItemType = "Healing";
```

* ActivateItem: 회복 효과 → 사라짐


## 💣 지뢰 아이템 (`AMineItem`)

```cpp
ExplosionDelay = 5.0f;
ExplosionRadius = 300.0f;
ExplosionDamage = 30.0f;
ItemType = "Mine";
```

* ActivateItem: 일정 시간 후 폭발 예정 (현재는 바로 `DestroyItem()` 호출됨 — 향후 로직 개선 필요)


## 🧲 충돌 처리

* `USphereComponent`를 통해 오버랩 이벤트 수신
* `Collision->OnComponentBeginOverlap.AddDynamic(...)`으로 이벤트 바인딩

```cpp
Collision->SetCollisionProfileName(TEXT("OverlapAllDynamic"));
Collision->OnComponentBeginOverlap.AddDynamic(this, &ABaseItem::OnItemOverlap);
```



## 🧠 정리

* **인터페이스**를 통해 공통된 아이템 시스템 설계 가능
* **지속적인 확장을 고려한 설계** 중요
* **오버랩 이벤트**로 간단한 아이템 상호작용 처리 가능
* 추후 **지뢰 딜레이 폭발 로직**, **코인 획득 UI**, **힐링 효과 적용** 등 확장 예정

