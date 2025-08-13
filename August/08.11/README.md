# 📆 2025-08-13 TIL - Unreal Engine **UI/BGM & Git 병합**

## 🎯 오늘 배운 핵심 내용 요약

* UMG로 **Help 오버레이(아코디언)** 구성: Controls / Items / Objectives, 한 번에 하나만 펼치기
* HUD에 **탄약 텍스트 `현재탄/∞`** 고정 포맷, **무기 아이콘** 실시간 갱신
* GameMode에 **배경음악(BGM)** 재생 유틸 추가(메뉴/인게임/게임오버 전환 시 페이드)
* Git에서 **.uasset/.umap 바이너리 충돌**을 `--theirs`(develop)로 해결 + LFS 점검


## 🧩 UMG — Help 오버레이(아코디언)

* 레이아웃
  `Canvas → Border → (상단) HorizontalBox(Help, Spacer, ESC) + (하단) ScrollBox(SB_Content)`
  `SB_Content` 내부: **VerticalBox → ExpandableArea 3개**(`EA_Controls`, `EA_Items`, `EA_Objectives`)
* 겹침 이슈 해결 포인트

  * EA들이 **Canvas Slot**이 아니고 **Vertical Box Slot**에 붙어 있어야 함
  * `SB_Content.Orientation = Vertical`, EA Slot에 **Padding**(아래 6\~8) 부여
  * EA 설정: `HeaderPadding=8,2`, `AreaPadding=4`, `MaxHeight=0`, 미리보기에선 하나만 `IsExpanded=true`
* C++ 위젯: `UWBP_HelpAccordion`

  * `OnExpansionChanged`로 **한 번에 하나만 펼침**
  * ESC 버튼/키 → `CloseHelp()` → `AMainGameMode::HideHelp()`
* Help 본문(요약)

  * **Controls**: WASD, Space, Shift, Ctrl, R, M1 사격, M2 조준, 휠/숫자 무기교체, (키 지정) 근접/상호작용
  * **Items**: 회복/피해증가/이속증가/무적/경험치 (인벤토리: 구현 시)
  * **Objectives**: 레벨업=경험치 100, 특정 레벨 보스 소환, 사망=오버/보스 처치=클리어
* 배경 이미지: Canvas 최하단 `Image`(전체 앵커, **ZOrder=0**)


## 🎯 HUD & 크로스헤어

* 탄약 텍스트: `UUWBP_HUD::UpdateAmmoText(Current, Reserve, /*bInfinite*/true)` → **`"%d/∞"`**
* 무기 아이콘: `UUWBP_HUD::SetWeaponIcon(UTexture2D* Icon)`로 즉시 갱신
* 크로스헤어 표시 제어

  * `UPlayerUIComponent`에서 BeginPlay의 `AddToViewport` 제거
  * `ShowCrosshair() / HideCrosshair()` 제공 → **HUD 화면에서만** 보이도록 GameMode에서 호출


## 🔀 Git — develop 병합 & 바이너리 충돌 처리

* 병합 흐름

```bash
git fetch --all
git checkout develop
git pull origin develop

git checkout feature/UI
git merge develop  # 충돌 발생
```

* **.uasset/.umap** 충돌은 텍스트 머지가 불가 → 이번 머지는 **develop 쪽 선택**

```bash
git checkout --theirs "Content/Blueprints/Character/BP_MainCharacter.uasset"
git add "Content/Blueprints/Character/BP_MainCharacter.uasset"
git commit -m "Merge develop into feature/UI (take theirs for binary assets)"
```

* Git LFS 경고 대처(한 번만)

```bash
git lfs install
git lfs fetch origin
git lfs checkout
```

* 테스트 후 develop 반영

```bash
git checkout develop
git merge feature/UI
git push origin develop
```

## 📎 오늘 바꾼(추가/수정) 주요 파일

| 구분       | 파일/클래스                      | 포인트                                                       |
| -------- | --------------------------- | --------------------------------------------------------- |
| UI(C++)  | `UWBP_HelpAccordion.h/.cpp` | 단일 펼침, ESC 닫기                                             |
| UI(C++)  | `UWBP_HUD.h/.cpp`           | 탄약 `"%d/∞"`, 무기 아이콘                                       |
| 컴포넌트     | `PlayerUIComponent.h/.cpp`  | `Show/HideCrosshair()`                                    |
| GameMode | `MainGameMode.h/.cpp`       | `ShowHelp/HideHelp`, `PlayMusic/StopMusic/SetMusicVolume` |
| UMG      | `WBP_Help.uasset`           | ScrollBox+VerticalBox+EA 3개, 배경 이미지                       |
| UMG      | `WBP_HUD.uasset`            | 텍스트/이미지 바인딩                                               |


## 🧠 느낀 점

* UMG 겹침은 **Slot 타입**이 90%의 원인. 레이아웃 꼬이면 *부모 컨테이너와 Slot*부터 본다.
* 탄약 포맷을 단순화하면(∞) **갱신 시점**이 더 중요해진다(사격/장전 이벤트 연동).
* UE 자산 충돌은 대화가 답이다. 동시에 같은 블루프린트 만지지 않게 **역할 분리/조기 병합** 필요.


## ✅ 앞으로 해볼 것

* 옵션 메뉴: **BGM/SFX 볼륨 슬라이더**(SoundClass+SoundMix) 연동
* 무기 아이콘 매핑 **DataTable/Map**로 일원화
* 특수 이동(대시/더블점프/구르기) **쿨타임 UI** 연동
* `.gitattributes` LFS 추적 재확인(`*.uasset *.umap`)


