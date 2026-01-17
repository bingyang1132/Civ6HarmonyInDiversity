# 马格努斯增强计划

## 目标 ✅ 已完成
给总督马格努斯的 `GOVERNOR_PROMOTION_RESOURCE_MANAGER_EXPEDITION`（马一）增加额外效果：**本城生产的所有开拓者+2移动力**

## 实现方案

参考梁的 `GOVERNOR_PROMOTION_BUILDER_GUILDMASTER` 实现方式，采用相同的机制。

### 实现步骤

#### 1. 创建能力定义（在 `UpdateDataBase/DL_UnitsAbilities.sql`）

- **能力类型**：`ABILITY_MAGNUS_TRAINED_SETTLER_MOVEMENT`
- **能力标签**：`CLASS_SETTLER`（需要确认是否存在，或使用 `CLASS_LANDCIVILIAN`）
- **能力修饰符**：`MAGNUS_EXTRA_MOVEMENT`（提供+2移动力）

#### 2. 创建修饰符系统（在 `SubMods/GovernorsDiversity/HD_Governors.sql`）

- **城市级修饰符**：`MAGNUS_TRAINED_SETTLER_MOVEMENT`
  - 类型：`MODIFIER_SINGLE_CITY_ATTACH_MODIFIER`
  - 作用：附加到城市
  
- **单位授予修饰符**：`MAGNUS_TRAINED_SETTLER_MOVEMENT_MODIFIER`
  - 类型：`MODIFIER_SINGLE_CITY_GRANT_ABILITY_FOR_TRAINED_UNITS`
  - Permanent：1（永久生效）
  - SubjectRequirementSetId：`UNIT_IS_SETTLER_REQUIREMENTS` 或 `HD_UNIT_IS_SETTLER`
  - 作用：给本城训练的开拓者授予能力

- **移动力修饰符**：`MAGNUS_EXTRA_MOVEMENT`
  - 类型：`MODIFIER_PLAYER_UNIT_ADJUST_MOVEMENT`
  - Permanent：1
  - Amount：2

#### 3. 关联到总督升级

在 `GovernorPromotionModifiers` 表中添加：
- `GOVERNOR_PROMOTION_RESOURCE_MANAGER_EXPEDITION` → `MAGNUS_TRAINED_SETTLER_MOVEMENT`

#### 4. 需求集检查

需要确认是否存在 `UNIT_IS_SETTLER_REQUIREMENTS`，如果不存在：
- 使用现有的 `HD_UNIT_IS_SETTLER`
- 或创建 `UNIT_IS_SETTLER_REQUIREMENTS`（参考 `UNIT_IS_BUILDER` 的生成方式）

### 代码结构

```
能力定义 (DL_UnitsAbilities.sql)
  └─ ABILITY_MAGNUS_TRAINED_SETTLER_MOVEMENT
      └─ MAGNUS_EXTRA_MOVEMENT (移动力+2)

修饰符链 (HD_Governors.sql)
  └─ MAGNUS_TRAINED_SETTLER_MOVEMENT (城市级)
      └─ MAGNUS_TRAINED_SETTLER_MOVEMENT_MODIFIER (授予能力)
          └─ 授予 ABILITY_MAGNUS_TRAINED_SETTLER_MOVEMENT 给本城训练的开拓者

总督关联
  └─ GOVERNOR_PROMOTION_RESOURCE_MANAGER_EXPEDITION
      └─ MAGNUS_TRAINED_SETTLER_MOVEMENT
```

### 注意事项

1. **需求集**：确认使用 `UNIT_IS_SETTLER_REQUIREMENTS` 还是 `HD_UNIT_IS_SETTLER`
2. **能力标签**：确认开拓者的单位类别标签，可能需要使用 `CLASS_SETTLER` 或 `CLASS_LANDCIVILIAN`
3. **文本描述**：需要在 `HD_Text_Governors.sql` 中更新描述文本
4. **测试**：确保只影响本城生产的开拓者，不影响其他城市生产的或通过其他方式获得的开拓者

### 参考实现

参考 `GUILDMASTER_TRAINED_BUILDER_MOVEMENT` 的实现：
- 使用 `MODIFIER_SINGLE_CITY_GRANT_ABILITY_FOR_TRAINED_UNITS`
- Permanent = 1 确保永久生效
- SubjectRequirementSetId 限制单位类型

---

## 实现方案：从所在城市的区域与建筑获得对应产出

### 概述

该方案用于实现：**当城市拥有特定区域或建筑时，改良设施（或其他对象）获得对应的产出加成**。

参考实现：`HD_Governors_Late.sql` 中梁右4（新版市立公园）的实现。

### 核心数据表

#### 1. `DistrictCorrespondingYieldType_HD` 表

该表定义了区域类型与产出类型的对应关系：

```sql
-- 表结构
DistrictType        TEXT    -- 区域类型
YieldType           TEXT    -- 产出类型（YIELD_SCIENCE, YIELD_CULTURE 等）
Amount              INT     -- 产出数量
RequiresPopulation  BOOLEAN -- 是否需要人口要求（1=专业化区域，0=非专业化区域）
HasAdjacency        BOOLEAN -- 是否有相邻加成
```

该表在 `UpdateDataBase/DL_EarlySetup.sql` 中定义和填充数据。

### 实现步骤

#### 步骤1：区域加产实现

当城市拥有某个区域时，改良设施获得对应产出。

**1.1 关联修饰符到改良设施**

```sql
insert or replace into ImprovementModifiers
    (ImprovementType, ModifierId)
select
    'IMPROVEMENT_CITY_PARK', 
    'PARKS_RECREATION_' || DistrictType || '_YIELD_BONUS'
from DistrictCorrespondingYieldType_HD 
where RequiresPopulation = 1;
```

- `ImprovementType`: 目标改良设施类型
- `ModifierId`: 动态生成的修饰符ID，格式为 `PARKS_RECREATION_<DistrictType>_YIELD_BONUS`

**1.2 创建修饰符定义**

```sql
insert or replace into Modifiers
    (ModifierId, ModifierType, SubjectRequirementSetId)
select
    'PARKS_RECREATION_' || DistrictType || '_YIELD_BONUS',
    'MODIFIER_SINGLE_PLOT_ADJUST_PLOT_YIELDS',
    'REQUIRES_CITY_HAS_' || DistrictType || '_UDMET'
from DistrictCorrespondingYieldType_HD 
where RequiresPopulation = 1;
```

- `ModifierType`: `MODIFIER_SINGLE_PLOT_ADJUST_PLOT_YIELDS` - 调整地块产出
- `SubjectRequirementSetId`: `REQUIRES_CITY_HAS_<DistrictType>_UDMET` - 检查城市是否拥有该区域（包括替代区域）

**1.3 设置产出类型**

```sql
insert or replace into ModifierArguments
    (ModifierId, Name, Value)
select
    'PARKS_RECREATION_' || DistrictType || '_YIELD_BONUS',
    'YieldType',
    YieldType
from DistrictCorrespondingYieldType_HD 
where RequiresPopulation = 1;
```

**1.4 设置产出数量**

```sql
insert or replace into ModifierArguments
    (ModifierId, Name, Value)
select
    'PARKS_RECREATION_' || DistrictType || '_YIELD_BONUS',
    'Amount',
    Amount
from DistrictCorrespondingYieldType_HD 
where RequiresPopulation = 1;
```

#### 步骤2：建筑加成实现

当城市拥有某个建筑时，改良设施获得对应产出。

**2.1 关联修饰符到改良设施**

```sql
insert or replace into ImprovementModifiers
    (ImprovementType, ModifierId)
select
    'IMPROVEMENT_CITY_PARK',
    'PARKS_RECREATION_' || a.BuildingType || '_YIELD_BONUS'
from Buildings a 
inner join DistrictCorrespondingYieldType_HD b on a.PrereqDistrict = b.DistrictType
where b.RequiresPopulation = 1
    and a.BuildingType not in (select BuildingType from HD_DUMMY_BUILDINGS)
    and a.BuildingType not in (select CivUniqueBuildingType from BuildingReplaces);
```

- 通过 `Buildings` 表与 `DistrictCorrespondingYieldType_HD` 表关联，获取建筑所在区域的产出类型
- 排除虚拟建筑和文明特色建筑（避免重复）

**2.2 创建修饰符定义**

```sql
insert or replace into Modifiers
    (ModifierId, ModifierType, SubjectRequirementSetId)
select
    'PARKS_RECREATION_' || a.BuildingType || '_YIELD_BONUS',
    'MODIFIER_SINGLE_PLOT_ADJUST_PLOT_YIELDS',
    'CITY_HAS_' || a.BuildingType || '_REQUIREMENTS'
from Buildings a 
inner join DistrictCorrespondingYieldType_HD b on a.PrereqDistrict = b.DistrictType
where b.RequiresPopulation = 1
    and a.BuildingType not in (select BuildingType from HD_DUMMY_BUILDINGS)
    and a.BuildingType not in (select CivUniqueBuildingType from BuildingReplaces);
```

- `SubjectRequirementSetId`: `CITY_HAS_<BuildingType>_REQUIREMENTS` - 检查城市是否拥有该建筑

**2.3 设置产出类型和数量**

```sql
-- 产出类型
insert or replace into ModifierArguments
    (ModifierId, Name, Value)
select
    'PARKS_RECREATION_' || a.BuildingType || '_YIELD_BONUS',
    'YieldType',
    b.YieldType
from Buildings a 
inner join DistrictCorrespondingYieldType_HD b on a.PrereqDistrict = b.DistrictType
where b.RequiresPopulation = 1
    and a.BuildingType not in (select BuildingType from HD_DUMMY_BUILDINGS)
    and a.BuildingType not in (select CivUniqueBuildingType from BuildingReplaces);

-- 产出数量
insert or replace into ModifierArguments
    (ModifierId, Name, Value)
select
    'PARKS_RECREATION_' || a.BuildingType || '_YIELD_BONUS',
    'Amount',
    b.Amount
from Buildings a 
inner join DistrictCorrespondingYieldType_HD b on a.PrereqDistrict = b.DistrictType
where b.RequiresPopulation = 1
    and a.BuildingType not in (select BuildingType from HD_DUMMY_BUILDINGS)
    and a.BuildingType not in (select CivUniqueBuildingType from BuildingReplaces);
```

### 需求集说明

#### 区域需求集：`REQUIRES_CITY_HAS_<DistrictType>_UDMET`

- **生成位置**：`UpdateDataBase/DL_Requirements.sql` 第240-244行
- **作用**：检查城市是否拥有指定区域（包括文明特色替代区域）
- **UDMET 含义**：Unique District Matching Extended Type（唯一区域匹配扩展类型）
- **类型**：`REQUIREMENTSET_TEST_ANY` - 满足任一条件即可

#### 建筑需求集：`CITY_HAS_<BuildingType>_REQUIREMENTS`

- **生成位置**：`UpdateDataBase/DL_Requirements.sql` 第305-308行
- **作用**：检查城市是否拥有指定建筑
- **类型**：`REQUIREMENTSET_TEST_ANY` - 满足任一条件即可

### 关键要点

1. **修饰符类型**：使用 `MODIFIER_SINGLE_PLOT_ADJUST_PLOT_YIELDS` 来调整地块产出
2. **动态生成**：通过 SQL 的 `SELECT` 语句动态生成所有区域/建筑的修饰符，避免手动编写
3. **需求集检查**：使用自动生成的需求集来检查城市状态
4. **数据源**：从 `DistrictCorrespondingYieldType_HD` 表获取区域与产出的对应关系
5. **建筑过滤**：排除虚拟建筑（`HD_DUMMY_BUILDINGS`）和文明特色建筑（`BuildingReplaces`），避免重复计算

### 适用场景

- 改良设施根据城市区域/建筑获得产出
- 单位根据城市区域/建筑获得加成
- 其他需要基于城市区域/建筑状态提供效果的情况

### 代码结构示例

```
ImprovementModifiers (关联层)
  └─ IMPROVEMENT_CITY_PARK
      └─ PARKS_RECREATION_<DistrictType>_YIELD_BONUS
          └─ Modifiers (修饰符定义)
              ├─ ModifierType: MODIFIER_SINGLE_PLOT_ADJUST_PLOT_YIELDS
              ├─ SubjectRequirementSetId: REQUIRES_CITY_HAS_<DistrictType>_UDMET
              └─ ModifierArguments (参数)
                  ├─ YieldType: <从DistrictCorrespondingYieldType_HD获取>
                  └─ Amount: <从DistrictCorrespondingYieldType_HD获取>
```

### 参考代码位置

- **实现位置**：`SubMods/GovernorsDiversity/HD_Governors_Late.sql` 第70-132行
- **数据表定义**：`UpdateDataBase/DL_EarlySetup.sql` 第185-235行
- **需求集生成**：`UpdateDataBase/DL_Requirements.sql` 第240-244行（区域）、第305-308行（建筑）

---

## 马格努斯右四（纵向一体化）增强计划

### 目标

修改 `GOVERNOR_PROMOTION_RESOURCE_MANAGER_VERTICAL_INTEGRATION`（马右四）的效果：

**当前效果**：本城从6环内来自其他城市的区域获得对应产出。

**新效果**：
1. 本城从9环内的区域获得对应产出（6环→9环）
2. 解除本城限制（可以包括本城的区域）
3. 解锁"行政部门"市政后效果翻倍（额外获得一份产出）

### 实现方案

参考 `MAGNUS_REGIONAL_EARLY_FOOD` 和 `MAGNUS_REGIONAL_LATE_FOOD` 的实现方式。

#### 当前实现分析

**位置**：`SubMods/GovernorsDiversity/HD_Governors_Late.sql` 第168-191行

**当前结构**：
```
GovernorPromotionModifiers
  └─ GOVERNOR_PROMOTION_RESOURCE_MANAGER_VERTICAL_INTEGRATION
      └─ HD_VERTICAL_INTEGRATION_<DistrictType>_ATTACH
          └─ Modifiers (MODIFIER_PLAYER_DISTRICTS_ATTACH_MODIFIER)
              ├─ SubjectRequirementSetId: DISTRICT_IS_<DistrictType>_WITHIN_6_TILES_REQUIREMENTS
              └─ ModifierId: HD_VERTICAL_INTEGRATION_<DistrictType>
                  └─ Modifiers (MODIFIER_PLAYER_CITIES_ADJUST_CITY_YIELD_CHANGE)
                      ├─ OwnerRequirementSetId: CITY_HAS_NO_VERTICAL_INTEGRATION_REQUIREMENTS (排除本城)
                      └─ SubjectRequirementSetId: CITY_HAS_VERTICAL_INTEGRATION_REQUIREMENTS (需要纵向一体化)
```

#### 修改步骤

**步骤1：创建9环区域需求集**

检查 `DISTRICT_IS_<DistrictType>_WITHIN_9_TILES_REQUIREMENTS` 是否存在，如果不存在，需要在 `UpdateDataBase/DL_Requirements.sql` 中添加（参考6环的实现）。

**步骤2：修改基础修饰符**

1. 将 `DISTRICT_IS_<DistrictType>_WITHIN_6_TILES_REQUIREMENTS` 改为 `DISTRICT_IS_<DistrictType>_WITHIN_9_TILES_REQUIREMENTS`
2. 移除 `OwnerRequirementSetId: CITY_HAS_NO_VERTICAL_INTEGRATION_REQUIREMENTS`（解除本城限制）

**步骤3：添加解锁行政部门后的额外产出**

参考 `MAGNUS_REGIONAL_LATE_FOOD` 的实现：
- 创建新的修饰符：`HD_VERTICAL_INTEGRATION_<DistrictType>_LATE`
- 使用 `OwnerRequirementSetId: PLAYER_HAS_CIVIC_CIVIL_SERVICE_REQUIREMENTS`（行政部门市政）
- 使用相同的 `SubjectRequirementSetId` 和产出参数
- 在 `GovernorPromotionModifiers` 中关联

#### 实现细节

**修饰符结构（修改后）**：

```
基础效果（解锁前）：
  HD_VERTICAL_INTEGRATION_<DistrictType>
    ├─ ModifierType: MODIFIER_PLAYER_CITIES_ADJUST_CITY_YIELD_CHANGE
    ├─ OwnerRequirementSetId: null (移除排除本城的限制)
    └─ SubjectRequirementSetId: CITY_HAS_VERTICAL_INTEGRATION_REQUIREMENTS

额外效果（解锁行政部门后）：
  HD_VERTICAL_INTEGRATION_<DistrictType>_LATE
    ├─ ModifierType: MODIFIER_PLAYER_CITIES_ADJUST_CITY_YIELD_CHANGE
    ├─ OwnerRequirementSetId: PLAYER_HAS_CIVIC_CIVIL_SERVICE_REQUIREMENTS
    └─ SubjectRequirementSetId: CITY_HAS_VERTICAL_INTEGRATION_REQUIREMENTS
```

**需求集说明**：
- `PLAYER_HAS_CIVIC_CIVIL_SERVICE_REQUIREMENTS`：检查玩家是否解锁"行政部门"市政（已在 `DL_Requirements.sql` 中定义）
- `DISTRICT_IS_<DistrictType>_WITHIN_9_TILES_REQUIREMENTS`：检查9环内是否有指定区域（需要创建）

### 参考实现

- **马格努斯左三（工业家）**：`HD_Governors.sql` 第58-61行、第72-75行、第99-106行
- **当前纵向一体化**：`HD_Governors_Late.sql` 第168-191行