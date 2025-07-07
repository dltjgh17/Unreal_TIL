
# 📆 2025-07-07 TIL - Unreal Engine C++ 실습 정리

## 🎯 오늘 배운 핵심 내용 요약

오늘은 Unreal Engine 프로젝트에서 **C++ 클래스를 생성하고**, **컴포넌트를 조합하여 액터를 구성하고**, **Tick 및 트랜스폼 제어**를 통해 동적인 동작을 구현하는 과정을 실습했습니다. 또한, 리소스 마이그레이션 및 로그 시스템, 액터의 라이프사이클에 대해서도 배웠습니다.

---

## 🔁 리소스 마이그레이트

- 언리얼 에디터 내부에서 **마이그레이트(Migrate)** 기능을 사용하여 리소스를 다른 프로젝트로 깔끔하게 옮김
- 리소스 의존성을 포함하여 필요한 모든 파일이 자동으로 복사됨

---

## 🏗️ C++ 클래스 생성 및 구조

- 새로운 클래스를 C++로 선언하면 `Source/프로젝트명` 안에 `Private`, `Public` 폴더가 생성됨
- **Private**에는 `.cpp`, **Public**에는 `.h` 파일이 위치
- **Public 폴더의 헤더는 외부 모듈에서 접근 가능**, Private은 내부에서만 사용

---

## ❌ C++ 클래스 삭제 방법

1. 헤더(.h)와 소스(.cpp) 파일 삭제
2. `.uproject`나 빌드 에러 방지를 위해 `.Build.cs`에 등록된 모듈 확인 및 정리
3. 빌드 후 캐시 및 중간 파일 제거

---

## ⚙️ 액터 컴포넌트 구성

```cpp
SceneRoot = CreateDefaultSubobject<USceneComponent>(TEXT("SceneRoot"));
SetRootComponent(SceneRoot);

StaticMeshComp = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("StaticMesh"));
StaticMeshComp->SetupAttachment(SceneRoot);
````

* **SceneComponent**: 여러 컴포넌트를 붙이기 위한 기본 루트
* **StaticMeshComponent**: 고정된 메시 (형태)
* **Material**: 외관 질감 (색상, 반사 등)

---

## 🔄 액터의 라이프사이클 함수

| 순서 | 함수                             | 설명                |
| -- | ------------------------------ | ----------------- |
| 1  | **생성자**                        | 메모리에 올라갈 때 한 번 실행 |
| 2  | **PostInitializeComponents()** | 컴포넌트 초기화 후        |
| 3  | **BeginPlay()**                | 액터가 게임 세계에 스폰된 후  |
| 4  | **Tick()**                     | 매 프레임마다 호출됨       |
| 5  | **Destroy()**                  | 액터가 제거되기 직전       |
| 6  | **EndPlay()**                  | 게임 종료, 레벨 전환 시    |

> 💡 `Destroy()`가 호출되면 내부적으로 `EndPlay()`도 자동 호출됨

---

## 🌀 트랜스폼 제어

```cpp
SetActorLocation(FVector(300.0f, 200.0f, 100.0f));
SetActorRotation(FRotator(0.0f, 90.0f, 0.0f));
SetActorScale3D(FVector(2.0f, 2.0f, 2.0f));
```

* `Location`, `Rotation`, `Scale` 모두 직접 조작 가능
* `FTransform` 클래스를 활용해 한 번에 적용도 가능

---

## ⏱ Tick 함수와 회전

```cpp
if (!FMath::IsNearlyZero(RotationSpeed))
{
    AddActorLocalRotation(FRotator(0.0f, RotationSpeed * DeltaTime, 0.0f));
}
```

* `DeltaTime`을 활용한 회전 구현
* 모든 PC에서 동일한 동작을 보장

---

## 🧩 UPROPERTY & UFUNCTION 실전

* `UPROPERTY`로 컴포넌트와 변수들을 **에디터와 블루프린트에 노출**
* `UFUNCTION`을 사용하여 블루프린트에서 호출 가능한 함수 지정

| 매크로                           | 용도                      |
| ----------------------------- | ----------------------- |
| `VisibleAnywhere`             | 읽기 전용, 에디터에서 보임         |
| `EditAnywhere`                | 에디터에서 수정 가능             |
| `BlueprintCallable`           | 블루프린트에서 호출 가능           |
| `BlueprintPure`               | 상태를 변경하지 않는 함수          |
| `BlueprintImplementableEvent` | C++에서 정의 안하고 BP에서 구현 가능 |

---

## 📋 로그 시스템

```cpp
DECLARE_LOG_CATEGORY_EXTERN(LogSparta, Warning, All);
DEFINE_LOG_CATEGORY(LogSparta);
```

* `UE_LOG(LogSparta, Warning, TEXT("내용"));` 형태로 로그 출력 가능

---

## 🧠 느낀 점

* Tick 함수는 성능에 민감하므로 **꼭 필요한 경우에만 사용해야 한다**는 점을 알게 되었다.
* 액터의 **트랜스폼** 조작은 단순하지만 실시간 제어가 가능해서 활용도가 매우 높음
* `UPROPERTY`와 `UFUNCTION`의 사용 목적과 구분을 익혔고, 특히 `BlueprintImplementableEvent`는 **BP에서 이벤트만 정의할 수 있어서 유연한 설계**가 가능하다는 점이 인상적이었다.
* C++ 클래스를 블루프린트와 연결하려면 **매크로 선언과 접근 범위 지정**이 중요하다.

---

## ✅ 앞으로 해볼 것

* `PostInitializeComponents()` 활용해 컴포넌트 간 상호작용 구현 실습
* 다른 액터와 충돌 이벤트 처리
* Blueprint에서 `OnItemPickedUp()` 함수 구현
* 마테리얼과 메시를 런타임에 변경하는 기능

---

🧾 **한 줄 회고**

> "Unreal에서 C++과 Blueprint를 유기적으로 연결하려면 리플렉션 시스템과 라이프사이클 흐름을 완전히 이해해야 한다."
