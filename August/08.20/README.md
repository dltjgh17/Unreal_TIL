# 🧠 Unreal Engine 스마트 포인터, 쉽게 이해하기

## 1. 왜 스마트 포인터가 필요할까?

옛날 방식(C++ 기본 `new`/`delete`)은…

* `new`로 메모리 할당하고 `delete`로 해제해야 했음
* 근데 `delete` 깜빡하면 → **메모리 누수**
* 같은 걸 두 번 `delete` 하면 → **크래시**
* 이미 삭제된 걸 또 쓰면 → **크래시**

즉, **사람이 직접 관리해야 해서 너무 위험**

➡️ 그래서 나온 게 **스마트 포인터** = “자동 메모리 관리자”
👉 내가 일일이 `delete` 할 필요 없이, 조건이 되면 알아서 메모리를 정리해줌

---

## 2. TSharedPtr – 여러 명이 같이 쓰는 포인터

이건 **공동 소유권**
쉽게 말해서, "검 한 자루를 여러 명이 돌려쓰는 상황"이라고 생각하면 됨.

* 누가 쓰면 → “사용자 수 +1”
* 누가 내려놓으면 → “사용자 수 -1”
* 마지막 한 명까지 내려놓으면 → “자동 삭제”

```cpp
TSharedPtr<Sword> player1Sword = MakeShared<Sword>("철검");
TSharedPtr<Sword> player2Sword = player1Sword; // 같이 쓰기 시작

player1Sword = nullptr; // 첫 번째 사람 내려놓음
player2Sword = nullptr; // 두 번째 사람도 내려놓음
// → 이제 진짜로 사라짐!
```

🔑 핵심: 여러 군데에서 같은 데이터를 공유할 때 좋음 (UI, 저장, 사운드 등 동시에 접근하는 경우)

---

## 3. TWeakPtr – 순환 참조 막는 안전 장치

문제:

* 부모가 자식을 `TSharedPtr`로 들고 있고
* 자식도 부모를 `TSharedPtr`로 들고 있으면
  → 서로가 서로를 안 놓음 (메모리 누수)

해결:

* 한쪽을 **약한 참조(Weak)** 로 바꾸면 됨
  👉 약한 참조는 "참고만 함. 네가 사라지면 나도 그냥 빈손됨."

```cpp
class Parent { TSharedPtr<Child> MyChild; };
class Child { TWeakPtr<Parent> MyParent; }; // 약한 참조!
```

사용할 때는 `Pin()` 해서 “아직 살아있니?” 물어보고, 있으면 잠시 빌려 쓰는 것

```cpp
if (TSharedPtr<Parent> p = MyParent.Pin()) {
    p->DoSomething();
}
```

---

## 4. TUniquePtr – 나 혼자만 쓸 때

이건 **독점 소유권**
“내 검은 내가 혼자 쓰고, 아무도 못 가져간다” 느낌.

* 복사 ❌ (한 명만 소유 가능)
* 이동만 가능 (소유권을 넘겨주는 것)

```cpp
TUniquePtr<Sword> mySword = MakeUnique<Sword>("엑스칼리버");
TUniquePtr<Sword> newOwner = MoveTemp(mySword);
// → 이제 newOwner만 검을 갖고 있음
```

🔑 핵심: 오직 한 객체만 관리할 때, 성능이 가장 좋음
예: 파일 핸들, 렌더링 큐, AI 컨트롤러 등 “한 놈만 책임지는” 자원

---

## 5. 어떤 걸 써야 할까?

1. **UObject 계열이면?**
   → `UPROPERTY()` (엔진이 관리해줌, 스마트 포인터 안 씀)

2. **독점 소유(내 거만)?**
   → `TUniquePtr`

3. **여러 곳에서 같이 관리해야 함?**
   → `TSharedPtr`

4. **항상 존재해야 하는 애(절대 null 아님)?**
   → `TSharedRef`

5. **부모-자식처럼 순환 구조 있음?**
   → `TWeakPtr`

---

# 🔑 핵심 정리

* **Raw Pointer** → “검을 줍고, 내가 직접 버려야 함” (깜빡하면 문제남)
* **TSharedPtr** → “여럿이 같이 쓰다가, 마지막 사람이 버리면 사라짐”
* **TWeakPtr** → “빌려만 보고, 주인은 내가 아님” (순환 참조 막기)
* **TUniquePtr** → “나 혼자만 쓴다, 끝까지 내 거임”

---
