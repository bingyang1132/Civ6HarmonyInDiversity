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
