# Starizon Light — 코드 위키

> 팀원 누구나 "이 변수/함수가 뭘 하는 거지?" 궁금할 때 찾아보는 문서. 기획서(GDD) 6장의 클래스 설계를 실제 구현 기준으로 풀어서 설명함.

## 보는 법
- 클래스 단위로 나뉘어 있어요. 목차에서 원하는 클래스로 이동하세요.
- 각 항목은 **역할 → 멤버 변수 → 함수 → 사용 예시** 순서예요.
- 코드를 몰라도 "이게 왜 필요한지"는 이해할 수 있게 썼어요.

## 상태 표시
- ✅ 구현 완료 — 실제로 만들어져서 빌드까지 확인됨
- 🚧 예정 — 기획서엔 있지만 아직 코드 없음

## 목차
1. [StarizonTypes](#1-starizontypes)
2. [UInventorySubsystem](#2-uinventorysubsystem)
3. [UStarizonGameInstance](#3-ustarizongameinstance)
4. [UWorldTimeSubsystem](#4-uworldtimesubsystem)
5. [USaveGameStarizon](#5-usavegamestarizon)
6. [전체 클래스 관계도](#6-전체-클래스-관계도)
7. [아직 안 만든 클래스](#7-아직-안-만든-클래스-)

---

## 1. StarizonTypes

> 파일 위치: `Source/GameCapstone/Public/Player/StarizonTypes.h`
> 상태: ✅ 구현 완료

### 역할
여러 클래스가 공통으로 참조하는 **데이터 타입 모음**. 클래스가 아니라, 그 안에 담기는 "값 덩어리"들이 정의된 파일이에요. 새 클래스에서 아이템이나 퀘스트를 다뤄야 하면 이 파일을 include하면 됩니다.

### EQuestType — 퀘스트 종류

| 값 | 표시 이름 | 의미 |
|---|---|---|
| `Sell` | 처분형 | 상인 재고가 많을 때 — "이거 좀 사가 달라" |
| `Fetch` | 조달형 | 상인 재고가 부족할 때 — "이거 좀 구해와 달라" |

### EQuestState — 퀘스트 상태 (수락/잠금 흐름)

| 값           | 표시 이름  | 의미                                           |
| ----------- | ------ | -------------------------------------------- |
| `Proposed`  | 제안됨    | 상인이 방금 이 퀘스트를 제안한 상태, 아직 수락 안 함              |
| `Accepted`  | 수락됨,잠금 | 플레이어가 수락함. 이 상태인 동안 그 상인은 새 퀘스트를 다시 안 만듦(잠금) |
| `Completed` | 완료됨    | 요구 조건을 채워서 끝남                                |
| `Expired`   | 기간 초과  | 제한 시간 안에 못 끝내서 자동으로 풀림                       |

**왜 필요한가**: 상인한테 퀘스트를 받으면, 그 퀘스트가 끝나거나 시간이 지나기 전까지는 같은 상인이 새 퀘스트를 안 만들게 하기 위한 상태값이에요. 이게 없으면 매번 대화할 때마다 다른 퀘스트가 튀어나와서 이상해져요.

### FItemSlot — 인벤토리 한 칸

"이 아이템이 몇 개 있다"는 정보 하나를 표현하는 구조체예요. 골드도, 식량도, 교역품도 전부 이 틀 하나로 표현해요.

| 멤버             | 타입              | 의미                                                               |
| -------------- | --------------- | ---------------------------------------------------------------- |
| `ItemId`       | FName           | 아이템 이름표. 예: `"food"`, `"gold"`, `"cannonball"`                   |
| `Quantity`     | int32           | 몇 개 있는지                                                          |
| `bIsTradeGood` | bool (기본값 true) | 거래 UI에 보여도 되는 아이템인지. **`gold`만 예외적으로 `false`**로 둬서 거래 목록에 안 뜨게 함 |

**실사용 예시**
```
FItemSlot{ ItemId: "food", Quantity: 10, bIsTradeGood: true }   // 식량 10개, 거래 가능
FItemSlot{ ItemId: "gold", Quantity: 500, bIsTradeGood: false } // 골드 500, 거래 UI엔 안 보임
```

**골드도 아이템인 이유**: 원래는 골드를 별도 변수(`int32 Gold`)로 관리하려 했는데, 그러면 "골드 증감"과 "아이템 증감"을 각각 다른 함수로 처리해야 해서 코드가 두 배로 늘어나요. 그냥 골드도 `FItemSlot` 하나로 취급하고 `bIsTradeGood`만 꺼두면, `UInventorySubsystem`의 `AddItem`/`RemoveItem` 함수 하나로 골드든 아이템이든 다 처리할 수 있어요.

### FQuestData — LLM이 만든 퀘스트 한 개

| 멤버                | 타입                  | 의미                                          |
| ----------------- | ------------------- | ------------------------------------------- |
| `QuestId`         | FName               | 이 퀘스트를 구분하는 이름표                             |
| `QuestText`       | FString             | LLM이 만든 실제 문장. 예: "식량이 다 떨어져서 큰일이오..."      |
| `QuestType`       | EQuestType          | 처분형/조달형                                     |
| `State`           | EQuestState         | 지금 이 퀘스트가 어떤 상태인지                           |
| `TargetItemID`    | FName               | 뭘 요구하는지 (아이템 이름표)                           |
| `TargetAmount`    | int32               | 얼마나 요구하는지                                   |
| `TimeLimit`       | float               | 이 시간 안에 안 끝내면 자동으로 잠금 풀림(`Expired`)         |
| `RewardItems`     | TArray\<FItemSlot\> | 보상. 골드만이 아니라 다른 아이템도 줄 수 있게 배열로 만듦          |
| `GiverMerchantID` | FName               | 이 퀘스트를 준 상인이 누구인지 (나중에 "이 퀘스트 누가 줬더라" 역추적용) |

**주의**: `QuestText`는 LLM이 만들지만, `RewardItems`(보상)는 **LLM이 절대 정하지 않아요.** 게임 로직이 공식(요구수량 × 시장가 × 난이도계수)으로 계산해서 여기 채워 넣어요. LLM은 "문장"만 책임지고, "숫자"는 게임이 책임진다는 원칙이에요.

---

## 2. UInventorySubsystem

> 파일 위치: `Source/GameCapstone/Public/Player/InventorySubsystem.h/.cpp`
> 상속: `UGameInstanceSubsystem`
> 상태: ✅ 구현 완료

### 역할
**게임 안의 모든 아이템(골드 포함)을 관리하는 유일한 곳.** 인벤토리와 관련된 건 다른 어디에도 안 두고, 전부 여기서 처리해요.

**왜 컴포넌트가 아니라 서브시스템인가**: 처음엔 플레이어 캐릭터에 붙는 컴포넌트로 만들려고 했는데, 이러면 씬(레벨)이 전환될 때 캐릭터 액터가 사라지면서 데이터도 같이 날아갈 위험이 있어요. `UGameInstanceSubsystem`은 게임이 켜져있는 동안 절대 사라지지 않는 객체라서, 항구든 바다든 씬이 바뀌어도 인벤토리가 안전하게 유지돼요. 대규모 게임 스튜디오들도 싱글플레이어 게임의 인벤토리는 보통 이런 방식(전용 매니저)으로 관리해요.

**꺼내 쓰는 법**: 다른 클래스에서 인벤토리가 필요하면 이렇게 접근해요.
```cpp
UInventorySubsystem* Inv = GetGameInstance()->GetSubsystem<UInventorySubsystem>();
Inv->AddItem("food", 10);
```

### 멤버 변수 (private — 외부에서 직접 못 건드림)

| 변수 | 타입 | 의미 |
|---|---|---|
| `Items` | TArray\<FItemSlot\> | 현재 보유한 모든 아이템 목록. 이걸 직접 건드리면 안 되고, 아래 함수들을 통해서만 접근해야 함 |

### 함수

| 함수                                        | 반환                         | 하는 일                                                                               |
| ----------------------------------------- | -------------------------- | ---------------------------------------------------------------------------------- |
| `AddItem(ItemID, Qty, bIsTradeGood=true)` | void                       | 아이템 추가. 이미 있으면 수량만 더하고, 없으면 새로 만듦                                                  |
| `RemoveItem(ItemID, Qty)`                 | bool                       | 아이템 제거. **수량이 부족하면 아무것도 안 건드리고 `false` 반환** — 이 방어 로직이 중요함 (예: 돈 없는데 물건 사려는 걸 막아줌) |
| `GetItemCount(ItemID)`                    | int32                      | 지금 몇 개 있는지 조회만. 없으면 0                                                              |
| `GetTradeGoods()`                         | TArray\<FItemSlot\>        | 거래 UI에 보여줄 목록만 골라서 반환 (`bIsTradeGood=true`인 것만, gold는 자동으로 빠짐)                     |
| `GetAllItems()`                           | const TArray\<FItemSlot\>& | 전체 목록 조회 (세이브할 때 씀)                                                                |
| `SetAllItems(InItems)`                    | void                       | 전체 목록을 통째로 교체 (세이브 파일 불러올 때 씀)                                                     |

### 델리게이트 (이벤트 알림)

| 이름 | 설명 |
|---|---|
| `OnInventoryChanged` | 아이템이 추가/제거될 때마다 자동으로 알림이 감. UI 쪽(인벤토리 화면, 골드 표시 등)이 이걸 구독해두면, 인벤토리가 바뀔 때마다 화면이 자동으로 새로고침됨. 매 프레임 확인할 필요 없이 "바뀔 때만" 반응하니까 효율적 |

### 사용 예시

**거래 성사 시** (상인에게 물건 살 때)
```cpp
Inv->RemoveItem("gold", Price);      // 골드 차감
Inv->AddItem("cloth", 10);            // 산 물건 추가
```

**침몰 패널티 시**
```cpp
int32 CurrentGold = Inv->GetItemCount("gold");
Inv->RemoveItem("gold", CurrentGold * 0.2f);   // 골드 20% 차감
Inv->RemoveItem("food", Inv->GetItemCount("food"));  // 식량 전량 손실
```

---

## 3. UStarizonGameInstance

> 파일 위치: `Source/GameCapstone/Public/SystemManager/StarizonGameInstance.h/.cpp`
> 상속: `UGameInstance`
> 상태: ✅ 구현 완료

### 역할
**씬(레벨)이 바뀌어도 사라지면 안 되는 전역 데이터**를 보관하는 곳. 항구에서 바다로, 바다에서 다른 항구로 이동해도 여기 있는 값들은 계속 유지돼요.

**여기 없는 것에 주의**: 인벤토리(아이템/골드)는 여기 없어요! 그건 `UInventorySubsystem`이 전담해요. `GameInstance`는 **레벨/경험치, 퀘스트 목록, 마지막 입항 항구, 세이브·로드**만 담당해요.

### 멤버 변수 (private)

| 변수                              | 타입                   | 의미                            |
| ------------------------------- | -------------------- | ----------------------------- |
| `MerchantLevel` / `MerchantExp` | int32                | 상인 레벨/경험치                     |
| `CombatLevel` / `CombatExp`     | int32                | 전투 레벨/경험치                     |
| `ActiveQuests`                  | TArray\<FQuestData\> | 지금 진행 중인 퀘스트 목록               |
| `LastPortID`                    | FName                | 마지막으로 입항했던 항구. 침몰했을 때 여기로 돌아감 |

### 함수 — 성장 (레벨/경험치)

| 함수 | 반환 | 하는 일 |
|---|---|---|
| `AddMerchantExp(Amount)` | void | 상인 경험치 지급. 레벨업 조건(100×현재레벨) 넘으면 자동으로 레벨업까지 처리 |
| `AddCombatExp(Amount)` | void | 전투 경험치 지급. 위와 동일한 방식 |
| `GetMerchantLevel()` / `GetCombatLevel()` | int32 | 조회만 |

### 함수 — 항구 (침몰 복귀 지점)

| 함수 | 반환 | 하는 일 |
|---|---|---|
| `SetLastPort(PortID)` | void | 입항할 때 호출. **이 함수 안에서 자동저장(`SaveToSlot`)까지 같이 실행됨** — 별도 자동저장 타이머를 안 만들고 입항 이벤트에 저장을 끼워 넣은 것 |
| `GetLastPortID()` | FName | 조회만 |

### 함수 — 세이브/로드 (자동저장 전용, 수동 슬롯 없음)

| 함수 | 반환 | 하는 일 |
|---|---|---|
| `SaveToSlot()` | void | 지금 상태(레벨/퀘스트/항구 + UInventorySubsystem의 인벤토리 + UWorldTimeSubsystem의 날짜)를 세이브 파일로 저장 |
| `LoadFromSlot()` | bool | 세이브 파일을 불러와서 복원. 파일이 없으면 `false` |
| `DoesSaveExist()` | bool | 세이브 파일이 있는지만 확인 (로비 화면에서 씀) |
| `ResetSaveData()` | void | 세이브 파일 삭제 (설정 화면의 "초기화" 버튼용) |

### 델리게이트

| 이름 | 설명 |
|---|---|
| `OnPlayerLevelChanged` | 레벨업하거나 세이브를 로드해서 레벨이 바뀔 때 UI에 알림 |

### Save/Load가 실제로 하는 일

```
SaveToSlot() 호출 시
  GameInstance 멤버(레벨/경험치/퀘스트/항구) → 세이브 파일로 복사
  + GetSubsystem<UInventorySubsystem>()에서 인벤토리 꺼내서 → 세이브 파일로 복사
  + GetSubsystem<UWorldTimeSubsystem>()에서 날짜 꺼내서 → 세이브 파일로 복사

LoadFromSlot() 호출 시 (반대 방향)
  세이브 파일 → GameInstance 멤버로 복사
  세이브 파일 → UInventorySubsystem::SetAllItems()로 복사
  세이브 파일 → UWorldTimeSubsystem::SetCurrentDay()로 복사
```

**왜 GameInstance가 서브시스템 두 개(인벤토리, 시간)를 대신 꺼내오는가**: 서브시스템은 각자 자기 데이터만 관리하고, 세이브 파일과 직접 통신하지 않아요. `GameInstance`가 "세이브 담당자" 역할을 맡아서, 필요한 데이터를 여기저기서 모아 세이브 파일 하나로 합치고, 반대로 불러올 때도 각 서브시스템에 값을 나눠주는 역할을 해요.

---

## 4. UWorldTimeSubsystem

> 파일 위치: `Source/GameCapstone/Public/SystemManager/WorldTimeSubsystem.h/.cpp`
> 상속: `UGameInstanceSubsystem`, `FTickableGameObject`
> 상태: ✅ 구현 완료

### 역할
**게임 안의 시간(날짜)이 흘러가게 하는 곳.** 기획서 설계(하이브리드 시간배속: 실시간 1초 = 게임 내 N분)를 그대로 구현한 거예요.

**`FTickableGameObject`가 왜 필요한가**: 이걸 상속하면 매 프레임(`Tick`)마다 자동으로 함수가 호출돼요. 이 특성을 이용해서 "시간이 계속 흘러간다"는 걸 구현해요. 다른 클래스들은 이걸 상속 안 해도 되는데, 시간 시스템은 "매 프레임 조금씩 진행"이 핵심이라 필요해요.

### 멤버 변수 (private)

| 변수                         | 타입             | 의미                                   |
| -------------------------- | -------------- | ------------------------------------ |
| `GameMinutesPerRealSecond` | float (기본값 10) | 배속. 실시간 1초 = 게임 내 몇 분인지. 기획 확정 전 임시값 |
| `CurrentDay`               | int32          | 지금 게임 내 며칠째인지                        |
| `AccumulatedGameMinutes`   | float          | 오늘 하루 중 몇 분이 지났는지 (누적)               |

### 함수

| 함수 | 반환 | 하는 일 |
|---|---|---|
| `Tick(DeltaTime)` | void | 매 프레임 자동 호출됨. 시간을 누적하고, 하루(1440분) 넘으면 `CurrentDay`를 올림 |
| `GetCurrentDay()` | int32 | 지금 게임 내 날짜 조회 |
| `GetDayProgress()` | float (0.0~1.0) | 오늘 하루 중 얼마나 지났는지 (HUD 시계 표시용) |
| `SetCurrentDay(Day)` | void | 세이브 로드할 때 날짜를 강제로 맞추는 용도 |

### 델리게이트

| 이름 | 설명 |
|---|---|
| `OnGameDayChanged` | 날짜가 하루 넘어갈 때마다 알림. **NPC 상인의 재고 리필 로직(기획서 3.4)이 나중에 이걸 구독해서 "하루 지났으니 재고 채우기"를 실행할 예정** |

### 사용 예시

```cpp
UWorldTimeSubsystem* Time = GetGameInstance()->GetSubsystem<UWorldTimeSubsystem>();
int32 Today = Time->GetCurrentDay();
```

### 참고 — 배속 계산
```
GameMinutesPerRealSecond = 10  (임시값)
→ 게임 내 1일(1440분)이 지나려면 실시간 144초(2.4분) 걸림
```
이 값은 아직 확정이 아니라서, 플레이테스트하면서 바뀔 수 있어요.

---

## 5. USaveGameStarizon

> 파일 위치: `Source/GameCapstone/Public/SystemManager/SaveGameStarizon.h`
> 상속: `USaveGame`
> 상태: ✅ 구현 완료

### 역할
**자동저장 파일 안에 실제로 뭐가 담기는지**를 정의한 클래스. 이 클래스 자체엔 로직(함수)이 거의 없어요 — 그냥 "저장할 값들을 모아두는 상자"예요. 실제 저장/불러오기 로직은 `UStarizonGameInstance`의 `SaveToSlot()`/`LoadFromSlot()`이 담당해요.

**수동 저장 슬롯 없음**: 기획서 설계대로, 플레이어가 직접 "저장하기" 버튼을 누르는 기능은 없어요. 입항할 때마다 자동으로 저장돼요 (`UStarizonGameInstance`의 `SetLastPort()` 참고).

### 저장되는 값들

| 필드                              | 타입                   | 어디서 가져오는지                                   |
| ------------------------------- | -------------------- | ------------------------------------------- |
| `Inventory`                     | TArray\<FItemSlot\>  | UInventorySubsystem의 `GetAllItems()`에서 복사   |
| `ActiveQuests`                  | TArray\<FQuestData\> | UStarizonGameInstance 자체 멤버에서 복사            |
| `MerchantLevel` / `MerchantExp` | int32                | UStarizonGameInstance                       |
| `CombatLevel` / `CombatExp`     | int32                | UStarizonGameInstance                       |
| `LastPortID`                    | FName                | UStarizonGameInstance — 로드 시 이 위치로 스폰됨      |
| `CurrentGameDay`                | int32                | UWorldTimeSubsystem의 `GetCurrentDay()`에서 복사 |

### 완료 기준
기획서 6.7의 기준: **"저장 후 재실행 시 동일한 위치/골드/인벤토리로 정확히 복원됨"**

확인하는 법: 게임에서 아이템 좀 사고, 어디 항구에 입항한 다음 게임 껐다가 다시 켜보기 → 로비에서 이어하기 눌렀을 때 아까 그 항구에서 시작하고, 인벤토리도 그대로면 성공.

---

## 6. 전체 클래스 관계도

```
UStarizonGameInstance (전역 데이터, 세이브/로드 총괄)
 ├─ TArray<FQuestData> ActiveQuests        ← StarizonTypes
 ├─ MerchantLevel / CombatLevel 등
 │
 ├─ GetSubsystem<UInventorySubsystem>()    ← 인벤토리는 여기서 꺼내옴
 │    └─ TArray<FItemSlot> Items           ← StarizonTypes
 │
 └─ GetSubsystem<UWorldTimeSubsystem>()    ← 게임 날짜는 여기서 꺼내옴
      └─ CurrentDay

USaveGameStarizon (저장 파일 형태)
 ├─ Inventory (UInventorySubsystem에서 복사)
 ├─ ActiveQuests (GameInstance에서 복사)
 └─ CurrentGameDay (WorldTimeSubsystem에서 복사)
```

**핵심 원칙**: 인벤토리는 `UInventorySubsystem` 하나가 전담해요. GameInstance나 플레이어 캐릭터 어디에도 인벤토리 데이터를 따로 들고 있지 않아요 — 필요하면 항상 `GetSubsystem<UInventorySubsystem>()`으로 꺼내서 씁니다.

---

## 7. 아직 안 만든 클래스 🚧

| 클래스                                         | 기획서 위치               |
| ------------------------------------------- | -------------------- |
| `APlayerCharacter`, `IInteractable`         | 6.1 플레이어             |
| `AShipPawn` (이동/전투)                         | 6.2 함선               |
| `ABaseNPC`, `AMerchantNPC`, `AMainQuestNPC` | 6.3 NPC/상인           |
| `UTradeComponent`, `FTradeGoodData`         | 6.4 교역 경제            |
| `ULlamaClientSubsystem`, GBNF 연동            | 6.5 LLM 연동 (최우선 리스크) |
| UI 위젯 일체                                    | 6.6 UI/UX            |

다음에 이 클래스들을 만들면, 위 형식(역할 → 멤버 변수 → 함수 → 사용 예시)에 맞춰서 이 문서에 이어서 추가하면 됩니다.
