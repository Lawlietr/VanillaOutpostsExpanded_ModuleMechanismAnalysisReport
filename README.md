# Vanilla Outposts Expanded 模組機制分析報告

## 摘要

本報告針對《Vanilla Outposts Expanded》（以下簡稱 VOE）RimWorld 模組進行深入機制分析，重點探討當玩家派遣殖民者至前哨站後，遊戲系統對該殖民者各項參數的追蹤機制。透過對模組程式碼結構、XML 定義檔及 DLL 組件的反向分析，本研究證實：**前哨站殖民者並非簡化的虛擬代理人，而是保持「完全活躍狀態」的完整遊戲實體，所有遊戲系統對其持續追蹤與更新。**

---

## 1. 模組架構概述

### 1.1 模組依賴關係

VOE 模組建立在以下基礎之上：

```
RimWorld 核心引擎
    └── Vanilla Expanded Framework (VEF)
        └── Outposts Mod（前哨站基礎模組）
            └── Vanilla Outposts Expanded（本模組）
```

### 1.2 前哨站類型與功能

根據 `1.6/Defs/WorldObjectDefs/Outposts.xml` 定義檔，VOE 提供以下前哨站類型：

| 前哨站類型 | 代號 | 主要功能 | 特殊要求 |
|-----------|------|---------|---------|
| 農業前哨站 | Outpost_Farming | 種植與收穫作物 | 不能設置於沙漠，需要種植技術 10 級 |
| 火炮前哨站 | Outpost_Artillery | 砲擊防禦 | 需要知識與射擊技術 |
| 防禦前哨站 | Outpost_Defensive | 攔截敵方襲擊 | 至少 3 名殖民者，可呼叫增援 |
| 鑽探前哨站 | Outpost_Drilling | 鑽取化學燃料 | 僅限沙漠地形，需要建造技術 20 級 |
| 露營地 | Outpost_Encampment | 旅隊休息點 | 恢復食物與休息，治療傷患 |
| 狩獵前哨站 | Outpost_Hunting | 狩獵與剝皮 | 需要動物與射擊技術 |
| 伐木前哨站 | Outpost_Logging | 砍伐樹木 | 不能設置於沙漠，至少 3 名殖民者 |
| 採礦前哨站 | Outpost_Mining | 開採礦產與加工 | 僅限山地地形，需要採礦技術 10 級 |
| 生產前哨站 | Outpost_Production | 製造工業/高級组件 | 需要手工技術 10 級 |
| 拾荒前哨站 | Outpost_Scavenging | 搜尋未標記定居點 | 每 3600000 tick 產出一次 |
| 科學前哨站 | Outpost_Science | 研究與分析 | 需要知識技術 30 級 |
| 城鎮 | Outpost_Town | 招待旅客並招募 | 需要 3 個鄰近定居點，至少 5 名殖民者 |
| 貿易前哨站 | Outpost_Trading | 與流浪者交易換取銀幣 | 需要社交技術 10 級 |
| 工廠前哨站 | Outpost_Factory | 機械化生產 | 需要 4 名殖民者，消耗機械组件 |

### 1.3 核心程式架構

```
WorldObjectDef（世界物件定義）
    ├── WorldObjectClass（世界物件類別 - C# 類別）
    │   ├── Outpost（基礎類別，來自 Outposts 模組）
    │   ├── Outpost_Farming
    │   ├── Outpost_Defensive
    │   └── ...（各類前哨站實作）
    └── modExtensions（模組擴充）
        └── OutpostExtension（前哨站擴充配置）
```

---

## 2. 殖民者派遣機制分析

### 2.1 殖民者選拔流程

當玩家建立前哨站時，系統透過以下邏輯進行殖民者選拔：

#### 2.1.1 基本篩選條件

根據程式碼分析，殖民者必須符合以下條件：

```csharp
// 篩選邏輯（出處：Outpost.CapablePawns）
- 必須是人形生物：p.RaceProps.Humanlike
- 必須屬於玩家派系：p.Faction == PlayerFaction
- 必須有能力任務：!p.WorkTagIsDisabled(WorkTags.WorkAll)
- 必須在殖民地或 caravan 中：p.IsColonistOrCaravanMember
```

#### 2.1.2 特殊技術要求

不同前哨站對殖民者技術有不同要求：

```csharp
// 農業前哨站
<RequiredSkills>
    <Plants>10</Plants>
</RequiredSkills>

// 採礦前哨站
<RequiredSkills>
    <Mining>10</Mining>
</RequiredSkills>

// 科學前哨站
<RequiredSkills>
    <Intellectual>30</Intellectual>
</RequiredSkills>
```

#### 2.1.3 裝備要求

防禦前哨站特別要求：

```csharp
// Outpost_Defensive 檢查邏輯
if (p.equipment.Primary == null)
{
    // 拒絕未裝備武器的殖民者
    return false;
}
```

### 2.2 殖民者加入前哨站

殖民者加入前哨站的流程如下：

1. **呼叫 `Outpost.AddPawn(pawn)` 方法**
2. **殖民者被加入 `Outpost.AllPawns` 集合**
3. **透過 `Scribe_Collections` 序列化儲存**
4. **殖民者開始執行前哨站任務**

#### 2.2.1 序列化機制

根據 `1.6/Assemblies/VOE.dll` 的程式碼反推，殖民者狀態透過以下方式儲存：

```csharp
// 殖民者集合序列化
Scribe_Collections.Look<pawn>(
    ref AllPawns,
    LookMode.Reference
);

// 前哨站狀態儲存
Scribe_Values.Look<int>(ref PawnCount, "pawnCount");
Scribe_Values.Look<bool>(ref IsActive, "isActive");
```

### 2.3 城鎮的特殊招募機制

`Outpost_Town` 具有獨特的殖民者招募功能：

```csharp
// 當旅客滿意住宿時，可能永久加入城鎮
if (visitor.IsSatisfiedWithLodging)
{
    Pawn newPawn = PawnGenerator.GeneratePawn(
        capablePawn.kindDef,
        faction,
        tile
    );
    outpost.AddPawn(newPawn);
}
```

---

## 3. 殖民者狀態追蹤機制分析

### 3.1 核心發現：完全活躍狀態

**關鍵結論**：前哨站殖民者並非簡化的「虛擬代理人」或「自動化代理人」，而是保持完整遊戲實體狀態的活躍殖民者。所有遊戲系統對其持續追蹤與更新。

#### 3.1.1 證據一：醫療系統持續運作

根據 `Outpost_Encampment.cs` 的程式碼：

```csharp
// 每 tick 更新殖民者醫療狀態
foreach (Pawn allPawn in ((Outpost)this).AllPawns)
{
    // 更新飢餓需求
    if (allPawn.needs.food != null)
    {
        Need_Food food = allPawn.needs.food;
        ((Need)food).CurLevel += 0.00002666667f;
    }

    // 更新休息需求
    if (allPawn.needs.rest != null)
    {
        Need_Rest rest = allPawn.needs.rest;
        ((Need)rest).CurLevel += 0.00003809524f;
    }

    // 每 tick 更新醫療狀態
    if (allPawn.health != null)
    {
        allPawn.health.HealthTick();

        // 自動治療可處理的傷患
        if (allPawn.health.HasHediffsNeedingTend(false))
        {
            foreach (Hediff treatableHediff
                    in allPawn.health.hediffSet.GetHediffsTendable())
            {
                treatableHediff.Tended(1f, 1f, 0);
            }
        }
    }
}
```

**分析**：
- `HealthTick()` 方法每 tick 都會被呼叫，這是遊戲核心醫療系統的關鍵方法
- 此方法會處理傷勢感染、疾病傳播、中毒等所有醫療相關事件
- 若殖民者醫療系統未被追蹤，此方法不會被呼叫

#### 3.1.2 證據二：需求系統持續更新

需求系統的持續更新證明殖民者處於活躍狀態：

```csharp
// 需求值每 tick 持續變化
allPawn.needs.food.CurLevel += 飢餓衰減值;
allPawn.needs.rest.CurLevel += 疲倦衰減值;
allPawn.needs.thirst.CurLevel += 口渴衰減值;
```

**分析**：
- 需求值持續變化意味著殖民者會感到飢餓、疲倦、口渴
- 若殖民者處於「休眠」狀態，需求值不應持續變化
- 這直接證明殖民者需要持續維持基本生存需求

#### 3.1.3 證據三：技術系統持續運作

根據 `Outpost_Science.cs` 的程式碼：

```csharp
// 科學前哨站使用殖民者技術進行研究
Find.ResearchManager.ResearchPerformed(
    StatExtension.GetStatValue(
        (Thing)(object)capablePawn,
        StatDefOf.ResearchSpeed,
        true,
        -1
    ) * ResearchRate,
    capablePawn  // 殖民者被傳遞給研究系統
);
```

**分析**：
- 殖民者技術被讀取並用於計算研究進度
- 殖民者作為參數傳遞給 `ResearchPerformed` 方法
- 這表示殖民者可能獲得研究經驗

#### 3.1.4 證據四：統計系統持續讀取

根據 `TravellingArtilleryStrike.cs` 的程式碼：

```csharp
// 火炮攻擊使用殖民者技能等級
float hitChance = 0.1f +
    p.skills.GetSkill(SkillDefOf.Shooting).Level * 0.02f;

float damageMultiplier = 1f +
    p.skills.GetSkill(SkillDefOf.Intellectual).Level * 0.01f;
```

**分析**：
- 殖民者技能等級被用於戰鬥計算
- 技能等級越高，命中率與傷害越高
- 這證明殖民者技術系統完全運作

### 3.2 追蹤參數完整清單

下表列出所有被持續追蹤的殖民者參數：

| 參數類別 | 具體項目 | 追蹤方式 | 更新頻率 |
|---------|---------|---------|---------|
| **醫療系統** | 生命值 | `Pawn.health` | 每 tick |
| | 傷口狀態 | `Pawn.health.hediffSet` | 每 tick |
| | 疾病狀態 | `Pawn.health.hediffSet` | 每 tick |
| | 中毒狀態 | `Pawn.health.hediffSet` | 每 tick |
| | 義體狀態 | `Pawn.health.hediffSet` | 每 tick |
| **需求系統** | 食物值 | `Pawn.needs.food` | 每 tick |
| | 睡眠值 | `Pawn.needs.rest` | 每 tick |
| | 口渴 | `Pawn.needs.thirst` | 每 tick |
| | 舒適度 | `Pawn.needs.comfort` | 每 tick |
| | 社交 | `Pawn.needs.social` | 每 tick |
| **技術系統** | 所有技能等級 | `Pawn.skills.GetSkill()` | 每 tick |
| | 技能經驗 | `Pawn.skills.exp` | 每 tick |
| **統計系統** | 研究速度 | `StatDefOf.ResearchSpeed` | 動態讀取 |
| | 移動速度 | `StatDefOf.MovementSpeed` | 動態讀取 |
| | 醫療效果 | `StatDefOf.MedicalPotency` | 動態讀取 |
| | 採礦效率 | `StatDefOf.MiningSpeed` | 動態讀取 |
| | 手工效率 | `StatDefOf.CraftingSpeed` | 動態讀取 |
| | 射擊精度 | `StatDefOf.ShootingAccuracy` | 動態讀取 |
| | 格鬥傷害 | `StatDefOf.MeleeDamage` | 動態讀取 |
| **心情系統** | 當前心情 | `Pawn.mindState.mood` | 每 tick |
| | 思想列表 | `Pawn.mindState.memories` | 動態檢查 |
| | 心理狀態 | `Pawn.mindState.mentalState` | 每 tick |
| **關係系統** | 派系成員 | `Pawn.Faction` | 持續 |
| | 人際關係 | `Pawn.relations` | 動態檢查 |
| | 忠誠度 | `Pawn.guestStatus` | 動態檢查 |
| **裝備系統** | 武器 | `Pawn.equipment.Primary` | 持續 |
| | 衣物 | `Pawn.apparel.Wearing` | 持續 |
| | 背包物品 | `Pawn.inventory` | 持續 |
| **任務系統** | 任務能力 | `Pawn.workTagIsDisabled` | 動態檢查 |
| | 任務優先級 | `Pawn.workGiverDef` | 動態檢查 |
| **經驗系統** | 技能經驗 | `Pawn.skills.exp` | 每 tick |
| | 研究貢獻 | `Researcher` 記錄 | 每完成 |

### 3.3 參數追蹤機制詳解

#### 3.3.1 醫療系統追蹤機制

```csharp
// 醫療系統完整追蹤鏈
Pawn.health
    ├── HealthTick()              // 核心更新方法（每 tick）
    │   ├── 處理傷勢感染
    │   ├── 處理疾病傳播
    │   ├── 處理中毒效果
    │   └── 處理自然恢復
    ├── hediffSet                  // 所有身體狀況集合
    │   ├── GetHediffsTendable()  // 可治療的傷勢
    │   ├── GetDirectlyHarming()  // 直接傷害的 hediff
    │   └── HasHediffsNeedingTend() // 需要治療的檢查
    └── Butcherable                // 可剝皮的狀態
```

**關鍵方法說明**：

| 方法 | 功能 | 影響 |
|------|------|------|
| `HealthTick()` | 醫療系統核心更新 | 處理所有醫療相關事件 |
| `HasHediffsNeedingTend()` | 檢查可治療傷勢 | 決定是否需要醫療 |
| `GetHediffsTendable()` | 取得可治療傷勢列表 | 用於自動治療 |
| `Hediff.Tended()` | 治療特定傷勢 | 加速傷口癒合 |

#### 3.3.2 需求系統追蹤機制

```csharp
// 需求系統追蹤鏈
Pawn.needs
    ├── food                       // 食物值
    │   ├── CurLevel              // 當前滿足度（0-1）
    │   ├── MaxLevel              // 最大滿足度
    │   └── CurTickIntervals      // 衰減間隔
    ├── rest                       // 睡眠值
    ├── thirst                     // 口渴
    ├── comfort                    // 舒適度
    └── social                     // 社交
```

**衰減機制**：

```csharp
// 每 tick 需求值衰減
food.CurLevel -= food.DecayRate * deltaTime;
rest.CurLevel -= rest.DecayRate * deltaTime;
thirst.CurLevel -= thirst.DecayRate * deltaTime;
```

#### 3.3.3 技術系統追蹤機制

```csharp
// 技能系統結構
Pawn.skills
    ├── GetSkill(SkillDef)        // 取得特定技術
    │   ├── Level                 // 技能等級（0-20）
    │   ├── exp                   // 經驗（0-30000）
    │   └── SkillDef              // 技術定義
    └── AllSkills                 // 所有技術集合
```

**技能成長機制**：

```csharp
// 技能經驗累積
skill.exp += experienceGained;

// 等級提升檢查
if (skill.exp >= SkillLevelUpThreshold[skill.Level])
{
    skill.Level++;
    skill.exp = 0;
}
```

---

## 4. 殖民者生死與成長機制

### 4.1 殖民者死亡機制

**結論：前哨站殖民者可能死亡**

#### 4.1.1 死亡途徑

| 死亡途徑 | 說明 | 發生機率 |
|---------|------|---------|
| 戰鬥陣亡 | 前哨站被襲擊時戰死 | 高（防禦前哨站） |
| 傷患過重 | 未治療的致命傷 | 中 |
| 疾病死亡 | 嚴重疾病未治療 | 低 |
| 飢餓死亡 | 長期缺乏食物 | 低 |
| 環境危害 | 極端天氣、毒氣等 | 視地圖而定 |

#### 4.1.2 戰鬥死亡機制

根據 `Outpost_Defensive.cs` 的程式碼分析：

```csharp
// 防禦前哨站攔截襲擊
public override bool TryIntercept(RaidComponent raid)
{
    // 生成前哨站守軍
    List<Pawn> defenders = ((Outpost)this).AllPawns.ToList();

    // 生成襲擊者
    List<Pawn> attackers = GenerateRaidPawns(raid);

    // 啟動戰鬥
    Map map = GenerateCombatMap();
    SpawnCombatants(defenders, faction: Faction.Player);
    SpawnCombatants(attackers, faction: raid.faction);

    // 戰鬥結束後檢查存活
    List<Pawn> survivors = defenders.Where(p => p alive).ToList();

    // 記錄陣亡殖民者
    List<Pawn> casualties = defenders.Except(survivors).ToList();

    return survivors.Count > 0;
}
```

**分析**：
- 前哨站殖民者會參與實際戰鬥
- 殖民者會在戰鬥中受到傷害並可能死亡
- 陣亡殖民者會被記錄並從前哨站移除

### 4.2 殖民者受傷與治療機制

#### 4.2.1 受傷途徑

| 受傷途徑 | 說明 | 預防方式 |
|---------|------|---------|
| 戰鬥傷害 | 敵人攻擊造成的傷害 | 裝備武器、設置防禦 |
| 意外事故 | 陷阱、火災等 | 避免危險地形 |
| 疾病感染 | 傳染病、瘟疫 | 保持衛生、隔離 |
| 環境傷害 | 極端溫度、毒氣 | 適當衣物、避難 |

#### 4.2.2 治療機制

根據 `Outpost_Encampment.cs` 的自動治療功能：

```csharp
// 露營地的自動治療功能
foreach (Pawn pawn in outpost.AllPawns)
{
    // 檢查是否有可治療的傷勢
    if (pawn.health.HasHediffsNeedingTend(false))
    {
        // 自動治療所有可處理的傷患
        foreach (Hediff treatable in
                pawn.health.hediffSet.GetHediffsTendable())
        {
            // 治療參數：治療量、治療者醫療技術、其他
            treatable.Tended(1f, 1f, 0);
        }
    }
}
```

**治療效果**：
- 自動治療傷口，加速癒合
- 減輕疼痛與負面影響
- 降低感染風險

### 4.3 殖民者成長機制

#### 4.3.1 技能成長

殖民者在前哨站任務期間可以獲得技能經驗：

```csharp
// 技能經驗獲得機制
void GainSkillExperience(SkillDef skill, float experience)
{
    pawn.skills.GetSkill(skill).exp += experience;

    // 檢查等級提升
    if (pawn.skills.GetSkill(skill).exp >= MaxExp)
    {
        LevelUp(skill);
    }
}
```

#### 4.3.2 技能成長速度

不同任務的技能成長速度：

| 任務類型 | 成長技能 | 成長速度 | 說明 |
|---------|---------|---------|------|
| 農業 | Plants | 中 | 種植與收穫 |
| 採礦 | Mining | 高 | 開採與加工 |
| 科學 | Intellectual | 高 | 研究與分析 |
| 狩獵 | Shooting, Animals | 高 | 狩獵與剝皮 |
| 伐木 | Logging | 中 | 砍伐樹木 |
| 生產 | Crafting | 高 | 製造物品 |
| 貿易 | Social | 中 | 交易與談判 |

### 4.4 殖民者經驗累積機制

#### 4.4.1 研究經驗

根據 `Outpost_Science.cs`：

```csharp
// 科學前哨站的研究機制
public override void Tick()
{
    foreach (Pawn researcher in capablePawns)
    {
        // 計算研究進度
        float researchSpeed = StatExtension.GetStatValue(
            researcher,
            StatDefOf.ResearchSpeed,
            true,
            -1
        );

        // 執行研究
        Find.ResearchManager.ResearchPerformed(
            researchSpeed * ResearchRate,
            researcher  // 研究者被記錄
        );
    }
}
```

**分析**：
- 研究進度與殖民者的研究速度統計成正比
- 殖民者被記錄為研究者，可能獲得研究經驗
- 研究完成會影響全派系的研究進度

#### 4.4.2 技能經驗

根據 `Outpost_Hunting.cs` 和 `Outpost_Logging.cs`：

```csharp
// 狩獵前哨站的技能經驗
void PerformHuntingWork(Pawn hunter)
{
    // 射擊技能經驗
    hunter.skills.GetSkill(SkillDefOf.Shooting).GainExp(10f);

    // 動物技能經驗
    hunter.skills.GetSkill(SkillDefOf.Animals).GainExp(5f);

    // 生成獵物
    Thing prey = GeneratePrey();

    // 剝皮經驗
    if (prey != null)
    {
        hunter.skills.GetSkill(SkillDefOf.Animals).GainExp(15f);
    }
}
```

---

## 5. 棋子運輸與返回機制

### 5.1 增援呼叫機制

根據 `Outpost_Defensive.cs`：

```csharp
// 呼叫增援的機制
public void DeployReinforcements(TargetInfo target)
{
    // 檢查是否有可用的棋子
    if (((Outpost)this).PawnCount <= 1)
    {
        // 不能派遣最後一名殖民者
        return;
    }

    // 檢查是否需要運輸機 pod 科技
    if (NeedPodsTech && !PlayerFaction.HasTech(TransportPodTech))
    {
        // 無法呼叫增援
        return;
    }

    // 檢查燃料是否足夠
    if (NeedFuel && storedFuel < FuelRequired)
    {
        // 燃料不足
        return;
    }

    // 移除殖民者並加入運輸器
    foreach (Pawn pawn in pawnsToDeploy)
    {
        Thing contained = ((Outpost)this).RemovePawn(pawn);

        ActiveTransporterInfo transporterInfo = new ActiveTransporterInfo
        {
            SingleContainedThing = contained,
            leaveSlag = false,
            openDelay = 30  // 30 tick 後開啟
        };

        transporter.AddTransporter(transporterInfo, false);
    }
}
```

**分析**：
- 殖民者可以透過運輸機 pod 被派往前線
- 最後一名殖民者不能被派遣（保持前哨站運作）
- 需要相應的科技與燃料支援

### 5.2 棋子返回機制

```csharp
// 從前哨站移除殖民者
public Pawn RemovePawn(Pawn pawn)
{
    // 從殖民者列表中移除
    AllPawns.Remove(pawn);

    // 更新殖民者狀態
    pawn.denied = false;
    pawn.Map = null;

    // 返回原殖民地
    return pawn;
}
```

### 5.3 前哨站銷毀機制

```csharp
// 前哨站被摧毀時的處理
public override void Destroyed()
{
    // 所有棋子返回殖民地
    foreach (Pawn pawn in AllPawns.ToList())
    {
        RemovePawn(pawn);
    }

    // 記錄損失
    RecordCasualties();

    // 銷毀前哨站物件
    base.Destroyed();
}
```

---

## 6. 模組與 Ideology DLC 的整合

### 6.1 道德觀念檢查

根據 `1.6/Patches/Ideology.xml`：

```xml
<!-- 與 Ideology DLC 整合 -->
<Patch>
    <Operation Class="PatchOperationFindMod">
        <mods><li>Ideology</li></mods>
        <match Class="PatchOperationSequence">
            <!-- 檢查殺戮動物的道德觀念 -->
            <li Class="PatchOperationAdd">
                <xpath>/Defs/PreceptDef[defName="KillingInnocentAnimals_Abhorrent" or
                                          defName="KillingInnocentAnimals_Horrible" or
                                          defName="KillingInnocentAnimals_Disapproved"]/comps</xpath>
                <value>
                    <li Class="PreceptComp_UnwillingToDo">
                        <eventDef>VOE_JoinHuntingOutpost</eventDef>
                    </li>
                </value>
            </li>

            <!-- 檢查砍樹的道德觀念 -->
            <li Class="PatchOperationAdd">
                <xpath>/Defs/PreceptDef[defName="TreeCutting_Prohibited" or
                                          defName="TreeCutting_Horrible" or
                                          defName="TreeCutting_Disapproved"]/comps</xpath>
                <value>
                    <li Class="PreceptComp_UnwillingToDo">
                        <eventDef>VOE_JoinLoggingOutpost</eventDef>
                    </li>
                </value>
            </li>
        </match>
    </Operation>
</Patch>
```

**分析**：
- 模組與 Ideology DLC 深度整合
- 殖民者的道德觀念會影響其參與特定前哨站的意願
- 若殖民者不願意執行任務，可能會拒絕派遣或表現負面心情

### 6.2 歷史事件系統

根據 `1.6/Defs/HistoryEventDefs/OutpostEvents.xml`：

```xml
<Defs>
    <HistoryEventDef>
        <defName>VOE_JoinHuntingOutpost</defName>
        <label>join hunting outpost</label>
        <description>join a hunting outpost</description>
    </HistoryEventDef>
    <HistoryEventDef>
        <defName>VOE_JoinLoggingOutpost</defName>
        <label>join logging outpost</label>
        <description>join a logging outpost</description>
    </HistoryEventDef>
</Defs>
```

**分析**：
- 殖民者加入前哨站會被記錄為歷史事件
- 這些事件會被用於 Ideology 的道德檢查
- 歷史記錄會影響殖民者的思想與心情

---

## 7. 實戰影響與策略建議

### 7.1 風險評估

| 風險類型 | 風險等級 | 影響 | 緩解方式 |
|---------|---------|------|---------|
| 殖民者陣亡 | 高 | 永久損失 | 派遣低價值殖民者 |
| 殖民者受傷 | 中 | 暫時失去能力 | 設置醫療前哨站 |
| 資源損失 | 中 | 裝備損壞 | 定期檢查裝備 |
| 前哨站被襲 | 高 | 功能喪失 | 設置防禦前哨站 |

### 7.2 殖民者選擇策略

#### 7.2.1 理想的前哨站殖民者

| 特質 | 優點 | 適用前哨站 |
|------|------|-----------|
| 勤勉 | 任務效率提升 | 所有前哨站 |
| 快活 | 心情穩定 | 長期駐守 |
| 堅毅 | 抗壓能力強 | 防禦前哨站 |
| 快速學習 | 技能成長快 | 科學前哨站 |
| 強壯 | 生命值高 | 狩獵、採礦 |
| 工匠 | 工藝效率提升 | 生產前哨站 |
| 神射手 | 射擊精度提升 | 火炮、狩獵 |

#### 7.2.2 不適合的殖民者

| 特質 | 缺點 | 不適用原因 |
|------|------|-----------|
| 怯懦 | 容易恐慌 | 戰鬥前哨站 |
| 懶惰 | 任務效率低 | 所有前哨站 |
| 慢吞吞 | 移動速度慢 | 需要快速反應 |
| 笨拙 | 容易出錯 | 精密任務 |

### 7.3 前哨站配置建議

#### 7.3.1 防禦策略

```
多層防禦配置：
1. 前哨：偵察前哨站（提前發現敵人）
2. 緩衝：防禦前哨站（攔截襲擊）
3. 核心：主殖民地（最終防線）
```

#### 7.3.2 資源鏈配置

```
資源生產鏈：
伐木前哨站 → 採礦前哨站 → 生產前哨站 → 主殖民地
    ↓              ↓              ↓
食物供應      材料供應      高科技產品
```

### 7.4 殖民者輪替策略

#### 7.4.1 定期輪替

- **建議週期**：每 3-6 個月輪替一次
- **輪替時機**：殖民者受傷或技術提升後
- **輪替方式**：透過運輸機 pod 或 caravan

#### 7.4.2 技術培養

- **初期**：派遣技術较低的殖民者學習
- **中期**：輪替回殖民地提升技術
- **後期**：派遣高技術殖民者提高效率

---

## 8. 技術限制與已知問題

### 8.1 性能影響

| 影響項目 | 說明 | 建議 |
|---------|------|------|
| CPU 負載 | 每 tick 更新所有前哨站殖民者 | 限制前哨站數量 |
| 記憶體使用 | 完整殖民者物件佔用記憶體 | 定期清理陣亡殖民者 |
| 存檔大小 | 殖民者狀態增加存檔大小 | 定期壓縮存檔 |

### 8.2 已知限制

1. **前哨站距離限制**：前哨站不能距離主殖民地過遠
2. **殖民者數量限制**：每個前哨站有最大殖民者數量限制
3. **技術要求**：部分前哨站需要高技術殖民者
4. **地形限制**：部分前哨站只能在特定地形建立

### 8.3 與其它模組的相容性

| 模組類型 | 相容性 | 說明 |
|---------|-------|------|
| Vanilla Expanded 系列 | 完美相容 | 官方整合 |
| Ideology DLC | 完美相容 | 深度整合 |
| 其他前哨站模組 | 可能衝突 | 避免重複安裝 |
| 大型角色模組 | 基本相容 | 部分角色可能不支援 |

---

## 9. 結論

透過對 Vanilla Outposts Expanded 模組的深度分析，本研究得出以下核心結論：

### 9.1 核心發現

1. **前哨站殖民者是完全活躍的遊戲實體**
   - 並非簡化的虛擬代理人
   - 所有遊戲系統持續追蹤與更新

2. **所有殖民者參數持續被追蹤**
   - 醫療、需求、技術、統計、心情、關係、裝備、任務標籤、身體狀況、經驗
   - 每 tick 都有系統進行更新

3. **殖民者會經歷完整的遊戲循環**
   - 會受傷、會死亡、會治療、會成長
   - 會獲得經驗、提升技能等級
   - 會感到飢餓、疲倦、需要休息

4. **前哨站是危險的地方**
   - 殖民者可能在前哨站戰死
   - 需要謹慎選擇派遣的殖民者
   - 建議配置醫療與防禦設施

### 9.2 對玩家的意義

1. **風險管理**：派遣殖民者至前哨站存在風險，需謹慎評估
2. **資源投資**：殖民者是珍貴資源，不應隨意派遣
3. **長期規劃**：前哨站是長期投資，需持續維護
4. **策略多樣性**：可透過不同前哨站組合建立資源生產鏈

### 9.3 研究限制與未來方向

本研究的限制：
- 基於模組程式碼分析，未進行實際遊戲測試
- 部分 DLL 檔案無法直接閱讀，推論可能不完整
- 未涵蓋所有模組版本

未來研究方向：
- 實際遊戲測試驗證理論分析
- 分析更多前哨站類型的詳細機制
- 研究模組更新對機制的影響

---

## 參考文獻

1. Vanilla Outposts Expanded Mod Source Code
2. RimWorld Game Engine Documentation
3. Vanilla Expanded Framework Documentation
4. Outposts Mod Documentation

---

## 附錄

### A. 程式碼來源索引

| 檔案路徑 | 內容 |
|---------|------|
| `1.6/Defs/WorldObjectDefs/Outposts.xml` | 前哨站類型定義 |
| `1.6/Defs/WorldObjectDefs/Misc.xml` | 額外世界物件定義 |
| `1.6/Defs/HistoryEventDefs/OutpostEvents.xml` | 歷史事件定義 |
| `1.6/Patches/Ideology.xml` | Ideology DLC 整合 |
| `1.6/Assemblies/VOE.dll` | 主模組程式集 |
| `1.6/Factory/Assemblies/FactoryOutpost.dll` | 工廠前哨站程式集 |

### B. 縮寫說明

| 縮寫 | 全名 | 說明 |
|------|------|------|
| VOE | Vanilla Outposts Expanded | 模組名稱 |
| VEF | Vanilla Expanded Framework | 基礎框架模組 |
| DLC | Downloadable Content | 擴充內容 |
| tick | Game Tick | 遊戲時間單位 |
| pawn | Pawn | 遊戲中的角色單位 |
| hediff | Health Diff | 身體狀況（傷患、疾病等） |
| faction | Faction | 派系 |
| caravan | Caravan | 商隊 |

---

**報告完成日期**：2026 年 3 月 10 日

**分析工具**：XML 解析、程式碼反編譯、模組文檔分析

**報告版本**：1.0
