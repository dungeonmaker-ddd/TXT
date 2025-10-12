# PlantUML 数据库设计完整编写规范指南

> 🎯 **目标**：为AI Agent提供零歧义的PlantUML编写规范，确保生成的代码可以完美渲染  
> 📅 **版本**：v1.0 | 2025-10-12  
> ✅ **状态**：生产级规范

---

## 📋 目录

1. [三层注释体系](#一三层注释体系完整规范)
2. [关系定义规范](#二关系定义完整规范)
3. [常见错误对照表](#三常见错误对照表)
4. [完整示例模板](#四完整示例模板)
5. [生成前检查清单](#五生成前检查清单)

---

## 一、三层注释体系完整规范

### 📌 Layer 1: 类内部注释（必须）

**位置**：放在类定义的分隔线 `--` 之后

**格式**：
```plantuml
class TableName {
    + field1 : Type
    + field2 : Type
    + field3 : Type
    --
    ' 表的用途说明（一句话概括）
    ' field1: 字段1说明(主键/索引/约束/枚举值)
    ' field2: 字段2说明(外键/长度限制/默认值)
    ' field3: 字段3说明(JSON结构/计算规则)
}
```

**示例**：
```plantuml
class User {
    + id : Long
    + username : String  
    + mobile : String
    + password : String
    + status : Integer
    + created_at : DateTime
    --
    ' 用户基础信息表
    ' id: 用户唯一标识(雪花ID,主键)
    ' username: 登录用户名(唯一索引,2-20字符)
    ' mobile: 手机号(唯一索引,登录凭证,11位)
    ' password: 密码哈希值(BCrypt加密,6-20位原文)
    ' status: 用户状态(0=禁用,1=正常,2=冻结)
    ' created_at: 注册时间(自动生成,索引)
}
```

**规则**：
- ✅ 每个字段必须有说明
- ✅ 说明格式：`字段含义(技术细节/枚举值/约束)`
- ✅ 主键标注：`(主键)` 或 `(雪花ID,主键)`
- ✅ 外键标注：`(FK TableName.field,索引)`
- ✅ 索引标注：`(索引)` 或 `(唯一索引)`
- ✅ 枚举值：`(0=值1,1=值2,2=值3)`
- ❌ 不要写超过3行的长注释

---

### 📌 Layer 2: 关系线上的标签（必须）

**位置**：关系定义的两端和描述部分

**格式**：
```plantuml
ClassName1 "多重性1" 关系符号 "多重性2" ClassName2 : "简短描述"
ClassName1 "角色\n中文" 关系符号 "多重性" ClassName2 : "简短描述"
```

**示例**：
```plantuml
' 组合关系
User "1" *-- "1" UserProfile : "拥有"

' 聚合关系
User "1" o-- "0..*" Transaction : "产生"

' 普通关联
User "1" -- "0..*" Content : "创建"

' 多角色关联
User "buyer\n买家" -- "0..*" ServiceOrder : "购买"
User "seller\n卖家" -- "0..*" ServiceOrder : "提供"

' 自关联
User "source\n发起者" -- "0..*" UserRelation
User "target\n目标者" -- "0..*" UserRelation

' 依赖关系（注意：不用角色标签）
ContentAction ..> Notification : "触发通知"
```

**规则**：
- ✅ 多重性必须标注：`"1"`, `"0..1"`, `"0..*"`, `"1..*"`
- ✅ 描述简短：<15字符，放在 `: "描述"` 中
- ✅ 角色标签格式：`"英文标识\n中文说明"`
- ✅ 依赖关系 `..>` 不使用角色标签
- ❌ 不要在关系线上写长描述

---

### 📌 Layer 3: 外部note注释（重要类必须）

**位置**：关联到特定类的外部说明

**格式**：
```plantuml
note right/left/top/bottom of ClassName
  **标题（加粗）**
  • 要点1
  • 要点2
  
  **子标题**
  详细说明文字
  
  **示例代码**
  ```sql
  SELECT ...
  ```
end note
```

**示例**：
```plantuml
note right of UserProfile
  **组合关系说明**
  • 用户删除时资料必须删除
  • Profile无法独立存在
  • 数据库：ON DELETE CASCADE
  
  **11项可编辑字段**
  直接字段(4): nickname, avatar, gender, birthday
  JSON字段(7): height, weight, location, wechat...
  
  **metadata结构**
  ```json
  {
    "height": 170,
    "weight": 50,
    "location": "深圳",
    "wechat": "sunny0713"
  }
  ```
end note
```

**规则**：
- ✅ 使用Markdown格式（**加粗**、列表）
- ✅ 关键类必须有note（核心业务逻辑）
- ✅ 多态关联必须有note（说明原理）
- ✅ 复杂JSON字段必须有note（展示结构）
- ✅ 可以包含SQL/JSON示例代码
- ❌ 不要使用 `note right of Class::Relation` 语法

---

## 二、关系定义完整规范

### 2.1 关系类型决策树

```
选择关系类型时问自己：

1. B删除时，A必须删除吗？
   └─ 是 → 组合关系 *--
   └─ 否 → 继续

2. B删除时，A需要保留吗？
   └─ 是 → 聚合关系 o--
   └─ 否 → 继续

3. A需要引用B的数据吗？
   └─ 是 → 关联关系 --
   └─ 否 → 继续

4. A只是临时使用B吗？
   └─ 是 → 依赖关系 ..>
```

---

### 2.2 自关联关系规范（最易出错）

**✅ 正确方式1：两条关系线**
```plantuml
User "source\n发起者" -- "0..*" UserRelation
User "target\n目标者" -- "0..*" UserRelation

note bottom of UserRelation
  **自关联说明**
  • user_id: 发起用户
  • target_id: 目标用户
  • type: 1=关注, 2=拉黑
end note
```

**✅ 正确方式2：有向箭头**
```plantuml
ChatMessage "1" <-- "0..*" ChatMessage : "reply_to"

note bottom of ChatMessage
  **消息回复链**
  • reply_to_id: 被回复的消息ID
  • 支持评论楼层显示
end note
```

**❌ 错误方式**
```plantuml
User -- UserRelation  // 没有区分角色！
User -- "0..*" UserRelation : "source"  // 只标注一端！
```

---

### 2.3 多态关联规范（最难理解）

**核心概念**：一个表的外键字段可能指向多个不同的表

**✅ 正确表示方法**

```plantuml
' 1. 正常关联举报人
User "reporter\n举报人" -- "0..*" Report

' 2. 多态目标用依赖关系
Report ..> User : "可能举报"
Report ..> Content : "可能举报"
Report ..> ContentAction : "可能举报"

' 3. 必须添加详细note
note bottom of Report
  **多态关联设计**
  ⚠️ 重要：Report.target_id是多态外键
  
  **字段说明**
  • target_id: 被举报目标的ID(多态)
  • target_type: 目标类型
    - 1 = User.id
    - 2 = Content.id
    - 3 = ContentAction.id
  
  **数据库实现**
  • 不建立FK外键约束
  • 应用层根据target_type判断
  • 查询时动态JOIN对应表
  
  **查询示例**
  ```sql
  -- 查询被举报的用户
  SELECT r.*, u.* FROM Report r
  JOIN User u ON r.target_id = u.id
  WHERE r.target_type = 1;
  
  -- 查询被举报的内容
  SELECT r.*, c.* FROM Report r
  JOIN Content c ON r.target_id = c.id
  WHERE r.target_type = 2;
  ```
end note
```

**❌ 错误表示方法**

```plantuml
' ❌ 不能用实线箭头建立3个外键
Report "0..*" --> "1" User
Report "0..*" --> "1" Content
Report "0..*" --> "1" ContentAction

' 原因：数据库层面Report只有一个target_id字段！
```

**其他多态关联场景**：

1. **Topic-Content（JSON数组关联）**
```plantuml
Topic "1" o-- "0..*" Content : "聚合"

note top of Topic
  **JSON数组关联**
  • Content.data.topic_ids: [123, 456]
  • 一个动态可关联多个话题
  • 不使用中间表
  
  **查询示例**
  ```sql
  SELECT c.* FROM Content c
  WHERE JSON_CONTAINS(c.data, '123', '$.topic_ids')
  ```
end note
```

2. **Media关联（JSON字段关联）**
```plantuml
Content "0..*" -- "0..*" Media : "包含"

note top of Media
  **JSON字段关联**
  • Content.data.media_ids: [123, 456]
  • ChatMessage.media_data.media_id: 789
  • 不使用真实FK，应用层JOIN
end note
```

---

## 三、常见错误对照表

### ❌ 错误 vs ✅ 正确

| 错误类型 | ❌ 错误代码 | ✅ 正确代码 | 说明 |
|---------|-----------|-----------|------|
| **依赖关系角色** | `Action "trigger" ..> Notify` | `Action ..> Notify : "触发"` | 依赖关系不用角色 |
| **多态外键** | `Report --> User`<br>`Report --> Content` | `Report ..> User`<br>`Report ..> Content` | 多态用依赖关系 |
| **note语法** | `note of User::Profile` | `note right of UserProfile` | 不用::语法 |
| **自关联** | `User -- UserRelation` | `User "source" -- UserRelation`<br>`User "target" -- UserRelation` | 必须标注角色 |
| **描述过长** | `: "拥有资料\n生命周期绑定\n级联删除"` | `: "拥有"`<br>+ note说明 | 描述要简短 |
| **角色格式** | `"买家\nbuyer"` (中文在前) | `"buyer\n买家"` (英文在前) | 统一格式 |
| **重复关系** | `A -- B`<br>`B o-- A` | 选择一个主要关系 | 避免重复 |

---

## 四、完整示例模板

### 4.1 单表完整示例

```plantuml
class User {
    + id : Long
    + username : String  
    + mobile : String
    + password : String
    + status : Integer
    + created_at : DateTime
    --
    ' 用户基础信息表
    ' id: 用户唯一标识(雪花ID,主键)
    ' username: 登录用户名(唯一索引,2-20字符)
    ' mobile: 手机号(唯一索引,11位,登录凭证)
    ' password: 密码哈希值(BCrypt加密,存储60字符)
    ' status: 用户状态(0=禁用,1=正常,2=冻结,默认1)
    ' created_at: 注册时间(自动生成,索引)
}

note top of User
  **系统核心实体**
  • 所有业务模块的基础
  • 支持多角色身份切换(买家/卖家/服务商)
  • 统一的用户状态管理
  
  **索引设计**
  • PRIMARY KEY (id)
  • UNIQUE KEY (username)
  • UNIQUE KEY (mobile)
  • INDEX (status, created_at)
end note
```

---

### 4.2 关系完整示例

```plantuml
' ==========================================
' 组合关系：强依赖，生命周期绑定
' ==========================================

User "1" *-- "1" UserProfile : "拥有"

note on link
  ON DELETE CASCADE
end note

note right of UserProfile
  **组合关系说明**
  • 用户删除时资料必须删除
  • Profile无法独立存在
  • 数据库：外键约束 + CASCADE
end note

' ==========================================
' 自关联关系：必须用两条线
' ==========================================

User "source\n发起者" -- "0..*" UserRelation
User "target\n目标者" -- "0..*" UserRelation

note bottom of UserRelation
  **自关联说明**
  • user_id: 关系发起者(我)
  • target_id: 关系目标者(对方)
  
  **查询示例**
  ```sql
  -- 我关注的人
  SELECT target_id FROM UserRelation 
  WHERE user_id = ? AND type = 1;
  
  -- 关注我的人(粉丝)
  SELECT user_id FROM UserRelation
  WHERE target_id = ? AND type = 1;
  ```
end note

' ==========================================
' 多角色关联：同一类扮演不同角色
' ==========================================

User "buyer\n买家" -- "0..*" ServiceOrder
User "seller\n卖家" -- "0..*" ServiceOrder

note bottom of ServiceOrder
  **多角色订单设计**
  • buyer_id: 下单用户(FK User.id)
  • seller_id: 服务提供者(FK User.id)
  • 同一用户可以既是买家又是卖家
  
  **状态流转**
  0(待付款) → 1(已付款) → 2(进行中) → 3(已完成)
             ↓
           4(已取消/退款)
end note

' ==========================================
' 多态关联：一个字段指向多个表
' ==========================================

User "reporter\n举报人" -- "0..*" Report
Report ..> User : "可能举报"
Report ..> Content : "可能举报"
Report ..> ContentAction : "可能举报"

note bottom of Report
  **多态关联设计**
  ⚠️ 核心：Report.target_id是多态外键
  
  **字段结构**
  • target_id: 被举报目标的ID(Long)
  • target_type: 目标类型(Integer)
    - 1 = User.id (举报用户)
    - 2 = Content.id (举报内容)
    - 3 = ContentAction.id (举报评论)
  
  **数据库实现**
  • 不建立真实FK约束(会报错)
  • 应用层保证数据一致性
  • 根据target_type动态JOIN
  
  **查询示例**
  ```sql
  -- 查询被举报的内容
  SELECT r.*, c.title, c.created_at
  FROM Report r
  INNER JOIN Content c ON r.target_id = c.id
  WHERE r.target_type = 2 
    AND r.status = 0
  ORDER BY r.created_at DESC;
  ```
  
  **为什么用依赖关系**
  • UML中依赖关系(..>)表示"临时使用"
  • Report"可能"指向User/Content/Action
  • 不是真实的FK关系，而是逻辑关联
end note
```

---

## 五、生成前检查清单

### ✅ 类定义检查清单

- [ ] 所有字段都有类型声明 `+ field : Type`
- [ ] 分隔线 `--` 后有完整的表说明
- [ ] 每个字段都有单独一行注释
- [ ] 字段注释包含：含义、约束、枚举值
- [ ] 主键字段标注了 `(主键)` 或 `(雪花ID,主键)`
- [ ] 外键字段标注了 `(FK TableName.field)`
- [ ] 索引字段标注了 `(索引)` 或 `(唯一索引)`
- [ ] JSON字段说明了内部结构

### ✅ 关系定义检查清单

- [ ] 所有关系都标注了多重性 `"1"`, `"0..*"` 等
- [ ] 关系类型选择正确（组合/聚合/关联/依赖）
- [ ] 依赖关系 `..>` 没有使用角色标签
- [ ] 自关联用了两条线标注source/target
- [ ] 多态关联用了依赖关系 `..>`
- [ ] 关系描述简短（<15字符）
- [ ] 角色标签格式统一 `"role\n中文"`
- [ ] 没有同一对类定义多个相同关系

### ✅ 注释检查清单

- [ ] 核心业务类都有外部note说明
- [ ] note使用正确的位置关键字 `right/left/top/bottom of`
- [ ] note使用Markdown格式（**加粗**、列表）
- [ ] 多态关联必须有note说明原理
- [ ] 复杂JSON字段有note展示结构
- [ ] note中包含必要的SQL查询示例
- [ ] 没有使用 `Class::Relation` 语法

### ✅ 语法检查清单

- [ ] 引号都是英文双引号 `"`
- [ ] 注释都是英文单引号 `'`
- [ ] 没有拼写错误（class, note, end note）
- [ ] 缩进统一（建议4空格或Tab）
- [ ] 每个section有清晰的分隔注释
- [ ] 文件以 `@startuml` 开始，`@enduml` 结束

---

## 六、给AI Agent的终极指令模板

```
请生成XY相遇派系统的PlantUML数据库设计图，严格遵循以下规范：

【类定义规范】
1. 每个类必须包含：
   - 所有字段的类型声明
   - 分隔线 -- 后的表说明
   - 每个字段单独一行的详细注释
   - 注释格式：字段含义(约束/枚举值/索引信息)

2. 字段注释必须包含：
   - 主键：标注(主键)或(雪花ID,主键)
   - 外键：标注(FK TableName.field,索引)
   - 索引：标注(索引)或(唯一索引)
   - 枚举：标注所有可能值(0=值1,1=值2)
   - 约束：长度限制、格式要求、默认值

【关系定义规范】
1. 所有关系必须标注：
   - 多重性："1", "0..1", "0..*", "1..*"
   - 关系描述：简短(<15字)
   - 角色标签(如需要)："role\n中文"

2. 关系类型选择：
   - 组合(*--): 生命周期绑定，如User-UserProfile
   - 聚合(o--): 弱依赖，如User-Transaction
   - 关联(--): 普通引用，如User-Content
   - 依赖(..>): 临时使用，如Action..>Notification

3. 特殊关系处理：
   - 自关联：必须用两条线或有向箭头
   - 多角色：每个角色独立一条关系线
   - 多态：使用依赖关系 + 详细note说明

【注释规范】
1. 类内注释：
   - 位置：分隔线--之后
   - 格式：' field: 说明
   - 内容：简洁但完整

2. 关系标签：
   - 依赖关系不用角色标签
   - 角色格式："英文\n中文"
   - 描述简短：<15字

3. 外部note：
   - 核心类必须有note
   - 多态关联必须有note
   - 使用Markdown格式
   - 可包含JSON/SQL示例
   - 使用：note right/left/top/bottom of ClassName
   - 避免：note of Class::Relation

【必须避免的错误】
❌ 依赖关系加角色标签
❌ 多态外键用实线箭头 -->
❌ 自关联只画一条线
❌ note使用 Class::Relation 语法
❌ 关系描述过长(>15字)
❌ 同一对类重复定义关系
❌ 缺少多重性标注

【参考模板】
请参考我提供的完整示例模板，确保：
1. 所有18张表都有完整的类内注释
2. 所有33条关系都有清晰的标签和描述
3. 核心类(User/Content/Report/Topic等)都有外部note
4. 多态关联(Report/Media/Topic)有详细note说明

【最终验证】
生成后请检查：
1. 能否在PlantUML工具中成功渲染
2. 关系箭头是否清晰不重叠
3. 所有note是否正确关联到类
4. 多态关联的note是否解释了原理
```

---

## 七、快速参考卡片

### 🔥 最容易犯的5个错误

1. **依赖关系加角色标签** ← 最常见
   ```plantuml
   ❌ Action "trigger" ..> Notify
   ✅ Action ..> Notify : "触发通知"
   ```

2. **多态外键用实线** ← 最严重
   ```plantuml
   ❌ Report --> User (数据库实现不了!)
   ✅ Report ..> User + note说明
   ```

3. **自关联只画一条线** ← 容易忘记
   ```plantuml
   ❌ User -- UserRelation
   ✅ User "source" -- UserRelation
      User "target" -- UserRelation
   ```

4. **note语法错误** ← 导致渲染失败
   ```plantuml
   ❌ note of User::Profile
   ✅ note right of UserProfile
   ```

5. **缺少多态说明** ← 让人困惑
   ```plantuml
   ❌ 只有 Report ..> User，没有note
   ✅ Report ..> User + 详细note解释target_type
   ```

---

### 🎯 三层注释快速检查

```
生成PlantUML后，快速检查：

Layer 1（类内注释）：
  ✓ 每个类都有 -- 分隔线
  ✓ 每个字段都有 ' 注释
  ✓ 注释包含：含义+约束+枚举值

Layer 2（关系标签）：
  ✓ 每条关系都有多重性 "1", "0..*"
  ✓ 每条关系都有描述 : "描述"
  ✓ 依赖关系没有角色标签
  ✓ 自关联有两条线或方向箭头

Layer 3（外部note）：
  ✓ 核心类有note（User/Content/Report等）
  ✓ 多态关联有note（Report/Media/Topic）
  ✓ note使用正确语法
  ✓ note包含示例代码(SQL/JSON)
```

---

## 八、在线验证工具

生成PlantUML后，请使用以下工具验证：

1. **PlantUML官方在线编辑器**
   - URL: http://www.plantuml.com/plantuml/uml/
   - 粘贴代码，立即查看渲染结果
   - 如果有语法错误会显示错误信息

2. **VS Code插件**
   - 插件名：PlantUML
   - 快捷键：Alt+D 预览图表
   - 实时预览，方便调试

3. **IntelliJ IDEA插件**
   - 插件名：PlantUML integration
   - 右键 → Show PlantUML Diagram
   - 支持导出PNG/SVG

---

## 九、完整流程指南

### AI Agent生成PlantUML的完整流程

```
Step 1: 分析需求
├─ 确定有哪些表（实体类）
├─ 确定表之间的关系类型
└─ 识别多态关联和特殊关系

Step 2: 编写类定义
├─ 写字段声明（+ field : Type）
├─ 添加分隔线（--）
├─ 写类内注释（' 表说明）
├─ 写字段注释（' field: 说明(约束)）
└─ 检查：每个字段都有注释吗？

Step 3: 编写关系定义
├─ 选择关系类型（*-- / o-- / -- / ..>）
├─ 标注多重性（"1", "0..*"等）
├─ 添加角色标签（如需要）
├─ 添加关系描述（: "描述"）
└─ 检查：依赖关系有角色标签吗？(应该没有)

Step 4: 添加外部note
├─ 核心类添加note
├─ 多态关联添加note说明
├─ 复杂JSON字段添加note示例
├─ 使用正确的note语法
└─ 检查：note使用Class::Relation语法了吗？(不应该)

Step 5: 最终验证
├─ 在PlantUML工具中渲染
├─ 检查箭头是否清晰
├─ 检查note是否正确关联
├─ 检查是否有语法错误
└─ 检查是否符合三层注释规范

Step 6: 交付
├─ 确认所有检查清单都通过
├─ 生成PNG/SVG图片
└─ 提供.puml源码文件
```

---

## 十、总结

### 🎯 核心原则

> **三层注释 + 关系规范 + 避免错误 = 完美PlantUML**

**三层注释缺一不可**：
1. 类内注释 - 字段说明
2. 关系标签 - 多重性+描述
3. 外部note - 详细说明

**关系定义必须规范**：
- 组合/聚合/关联/依赖选择正确
- 自关联两条线
- 多态用依赖
- 依赖不加角色

**避免5大常见错误**：
1. 依赖关系加角色标签
2. 多态外键用实线箭头
3. 自关联只画一条线
4. note使用Class::Relation
5. 缺少多态关联note说明

---

### 📌 记住这句话

> **PlantUML是UML图，不是代码文档！**  
> **关系线上保持简洁，复杂逻辑放到note中说明。**

---

**文档版本**: v1.0  
**创建时间**: 2025-10-12  
**适用对象**: AI Agent / 数据库设计师 / 架构师  
**参考标准**: PlantUML官方文档 + UML 2.5规范

---

© 2025 PlantUML数据库设计规范指南

