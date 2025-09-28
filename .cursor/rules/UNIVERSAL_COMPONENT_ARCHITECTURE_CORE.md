# 通用组件模块化架构核心标准 v2.2

## 🤖 Agent 执行指令

**当 Agent 接收到本架构文档时，必须严格按照以下要求执行：**

### 🚨 前后端一体化强制条款
**在实施任何包含API接口的模块时，Agent必须：**
- ✅ 同时创建前端API接口层和后端交互层的所有必需文件
- ✅ 确保后端实现可运行、可测试，包含完整的entity、dto、vo、controller、service、serviceImpl、mapper、queryBuilder
- ✅ 明确定义前后端数据交换格式和错误处理机制
- ✅ **数据访问层强制使用MyBatis-Plus + QueryWrapper技术栈，所有查询优先使用QueryWrapper**
- ✅ **ServiceImpl层使用QueryWrapper/LambdaQueryWrapper执行数据库查询，复杂查询可选择使用QueryBuilder封装**
- ✅ **🎯 接口设计严格按需实现 - 只创建前端实际需要的接口，避免批量操作等过度设计**
- ✅ **📦 逻辑集中化管理 - 优先在主组件文件内集中管理状态、事件处理、数据处理、简单工具函数等逻辑**
- ❌ 严禁仅创建前端接口而忽略后端实现
- ❌ 严禁交付不完整的后端交互层
- ❌ **严禁在数据访问层使用非MyBatis-Plus的ORM框架**
- ❌ **严禁在Mapper中编写复杂的自定义SQL，优先使用MyBatis-Plus提供的方法和QueryWrapper**
- ❌ **🚫 严禁过度设计接口 - 禁止创建前端未明确需要的批量操作、复杂查询、管理员功能等接口**
- ❌ **🚫 严禁过度文件拆分 - 避免为简单逻辑创建独立文件，包括简单的事件处理、数据处理、工具函数等**

### 🎯 强制执行规则

1. **📁 层级化页面组主导架构**
   - 页面组位置：`src/pages/{PageGroupName}/`
   - 主页面位置：`src/pages/{PageGroupName}/MainPage/`
   - 子页面位置：`src/pages/{PageGroupName}/{SubFunction}Page/`
   - 组件位置：`src/pages/{PageGroupName}/{PageType}Page/{ComponentName}/`
   - 默认嵌套化架构，支持页面组内多页面管理
   - 移除 `components/` 中间层级
   - 页面组级统一管理状态和导航

2. **🔧 完整结构要求**
   - 核心文件必需：`index.[ext]`、`types.[ext]`、`constants.[ext]`、`README.md`
   - 功能层按需创建，职责不得混合
   - **🚨 强制要求：API接口层与后端交互层必须同时完整实施，不得单独创建前端接口**

3. **🚫 禁止行为**
   - 禁止省略核心文件
   - 禁止职责混合
   - 禁止命名不规范
   - 禁止常量硬编码
   - **🚨 严禁仅实施前端API接口而忽略后端交互层实施**
   - **🚨 严禁后端交互层实施不完整（缺少entity、dto、controller、service等核心文件）**

## 🏗️ 层级化页面组集成架构

```
src/pages/                                          # 页面容器层
├── {PageGroupName}/                                # 📦 页面组 (主功能模块，如 UserProfile, ProductCatalog)
│   ├── index.[ext]                                 # 🏠 页面组入口 - 导出所有页面组件
│   ├── types.[ext]                                 # 📋 页面组类型定义 - 导出所有相关类型
│   ├── constants.[ext]                             # ⚙️ 页面组常量配置 - 导出所有相关常量
│   ├── README.md                                   # 📖 页面组文档 - 包含所有页面说明
│   │
│   ├── 🏠 主页面层 (Main Page)
│   │   ├── MainPage/                               # 📱 主页面组件 - 页面组的主入口
│   │   │   ├── index.[ext]                         # 主页面实现
│   │   │   ├── types.[ext]                         # 主页面类型定义
│   │   │   ├── constants.[ext]                     # 主页面常量配置
│   │   │   ├── README.md                           # 主页面文档
│   │   │   │
│   │   │   ├── {ComponentAreaName1}/               # ✅ 组件区域1 - 顶部布局区域
│   │   │   │   ├── index.[ext]                     # 主组件文件
│   │   │   │   ├── types.[ext]                     # 组件类型定义 (可选)
│   │   │   │   ├── constants.[ext]                 # 组件常量配置 (可选)
│   │   │   │   ├── README.md                       # 组件文档
│   │   │   │   │
│   │   │   │   ├── {FunctionAreaName1}/            # 🔸 功能区域1 - 复杂的逻辑，嵌套实现
│   │   │   │   │   ├── index.[ext]                 # 区域主文件 - 集中管理组件状态
│   │   │   │   │   ├── types.[ext]                 # 区域类型定义 (可选)
│   │   │   │   │   ├── constants.[ext]             # 区域常量 (可选)
│   │   │   │   │   ├── use[LocalState].[ext]       # 区域本地状态 (仅复杂状态逻辑时创建)
│   │   │   │   │   ├── on[Action].[ext]            # 区域事件处理 (仅复杂事件逻辑时创建)
│   │   │   │   │   ├── api[Action].[ext]           # 区域API接口 (仅复杂API逻辑时创建)
│   │   │   │   │   ├── process[Data].[ext]         # 数据处理 (仅复杂数据处理时创建)
│   │   │   │   │   └── utils[Function].[ext]       # 工具函数 (仅复杂工具函数时创建)
│   │   │   │   │
│   │   │   │   ├── {FunctionAreaName2}/            # 🟢 功能区域2 - 扁平实现
│   │   │   │   │   ├── index.[ext]                 # 区域主文件
│   │   │   │   │   ├── constants.[ext]             # 区域常量
│   │   │   │   │   ├── processData.[ext]           # 数据处理 (优先在主文件内处理)
│   │   │   │   │   └── utilsDisplay.[ext]          # 显示工具 (优先在主文件内处理)
│   │   │   │   │
│   │   │   │   └── {FunctionAreaName3}/            # 🔸 功能区域3 - 复杂交互，嵌套实现
│   │   │   │       ├── index.[ext]                 # 区域主文件 - 集中管理状态和逻辑
│   │   │   │       ├── use[LocalState].[ext]       # 区域本地状态 (仅特别复杂时创建)
│   │   │   │       ├── on[Action].[ext]            # 区域事件处理 (优先在主文件内处理)
│   │   │   │       ├── processLogic.[ext]          # 逻辑处理 (优先在主文件内处理)
│   │   │   │       └── utilsHelper.[ext]           # 辅助工具 (优先在主文件内处理)
│   │   │   │
│   │   │   ├── {ComponentAreaName2}/               # ✅ 组件区域2 - 主要内容区域
│   │   │   └── {ComponentAreaName3}/               # ✅ 组件区域3 - 操作交互区域
│   │   │
│   │   ├── 🔄 页面组状态管理层 (按需管理 - 优先在主文件内管理)
│   │   │   ├── use{PageGroup}.[ext]                # 页面组主状态管理 (仅复杂跨页面状态时创建)
│   │   │   ├── use{PageGroup}Data.[ext]            # 页面组数据状态管理 (仅复杂数据逻辑时创建)
│   │   │   └── use{PageGroup}Flow.[ext]            # 页面组流程状态管理 (仅复杂流程时创建)
│   │   │
│   │   └── 🧭 页面组导航层 (统一管理)
│   │       ├── navigate{PageGroup}Flow.[ext]       # 页面组内导航流程
│   │       ├── navigateToSubPages.[ext]            # 子页面导航
│   │       └── navigateBack{PageGroup}.[ext]       # 返回导航
│   │
│   ├── 📄 子页面层 (Sub Pages) - 最多10个相关页面
│   │   ├── {SubFunction1}Page/                     # 🎯 子功能页面1 - 如详情展示、数据查看等
│   │   │   ├── index.[ext]                         # 子功能页实现
│   │   │   ├── types.[ext]                         # 子功能页类型定义
│   │   │   ├── constants.[ext]                     # 子功能页常量配置
│   │   │   ├── README.md                           # 子功能页文档
│   │   │   │
│   │   │   ├── {ComponentAreaName1}/               # ✅ 子功能页组件区域1
│   │   │   └── {ComponentAreaName2}/               # ✅ 子功能页组件区域2
│   │   │
│   │   ├── {SubFunction2}Page/                     # 🎯 子功能页面2 - 如表单输入、数据编辑等
│   │   │   ├── index.[ext]                         # 子功能页实现
│   │   │   ├── types.[ext]                         # 子功能页类型定义
│   │   │   ├── constants.[ext]                     # 子功能页常量配置
│   │   │   ├── README.md                           # 子功能页文档
│   │   │   │
│   │   │   ├── {ComponentAreaName1}/               # ✅ 子功能页组件区域1
│   │   │   ├── {ComponentAreaName2}/               # ✅ 子功能页组件区域2
│   │   │   └── {ComponentAreaName3}/               # ✅ 子功能页组件区域3
│   │   │
│   │   ├── {SubFunction3}Page/                     # 🎯 子功能页面3 - 如列表管理、搜索筛选等
│   │   ├── {SubFunction4}Page/                     # 🎯 子功能页面4 - 如设置配置、偏好管理等
│   │   ├── {SubFunction5}Page/                     # 🎯 子功能页面5 - 如操作处理、状态变更等
│   │   ├── {SubFunction...}Page/                   # 🎯 更多子功能页面 - 根据业务需要定义
│   │   └── ...                                     # 最多10个子页面
│   │
│   ├── 🌐 页面组API层 ⚠️ **强制配套实施** - 严格按前端需求设计，避免过度设计
│   │   ├── api{PageGroup}Main.[ext]                # 主页面API接口 - 前端页面实际需要的接口
│   │   ├── api{PageGroup}[SubFunction].[ext]       # 子功能页面API接口 - 对应具体前端功能
│   │   └── api{PageGroup}Aggregate.[ext]           # 聚合API接口 - 仅在前端需要时创建
│   │
│   ├── 🔌 后端交互层 ⚠️ **强制配套实施** - MyBatis-Plus标准架构
│   │   └── backend/                                # 后端代码文件夹
│   │       ├── entity{PageGroup}.[ext]             # 🏗️ 数据实体类 - MyBatis-Plus注解配置
│   │       ├── dto{PageGroup}[Action].[ext]        # 📦 数据传输对象 - 前后端数据交换格式
│   │       ├── vo{PageGroup}[Query].[ext]          # 🔍 查询视图对象 - 查询条件封装（可选）
│   │       ├── controller{PageGroup}.[ext]         # 🎯 控制器 - HTTP请求处理入口
│   │       ├── service{PageGroup}.[ext]            # ⚙️ 业务服务接口 - 业务方法定义
│   │       ├── serviceImpl{PageGroup}.[ext]        # 🔧 业务服务实现 - QueryWrapper/LambdaQueryWrapper查询
│   │       ├── mapper{PageGroup}.[ext]             # 🗄️ 数据访问接口 - 继承BaseMapper
│   │       ├── query{PageGroup}Builder.[ext]       # 🏗️ 查询构建器 (可选) - 复杂查询封装工具
│   │       └── sql{PageGroup}.xml                  # 📝 复杂SQL (可选) - 极复杂查询场景
│   │
│   ├── 🔄 页面组数据处理层 (按需实施)
│   │   ├── process{PageGroup}Data.[ext]            # 页面组数据处理
│   │   ├── processSharedLogic.[ext]                # 共享逻辑处理
│   │   └── processValidation.[ext]                 # 数据验证处理
│   │
│   └── 🛠️ 页面组工具层 (按需实施)
│       ├── utils{PageGroup}Display.[ext]           # 页面组显示工具
│       ├── utilsSharedHelper.[ext]                 # 共享辅助工具
│       └── utilsNavigation.[ext]                   # 导航工具
```

### 💡 架构说明

- **📦 页面组**: 主功能模块，包含最多10个相关页面
- **📱 主页面**: 页面组的主入口，负责集成和协调所有子组件
- **📄 子页面**: `{SubFunction}Page/` 通用化命名，根据具体业务功能定义
- **🔄 页面组状态管理**: 优先在主文件内管理状态，避免过度抽象为hooks
- **🧭 页面组导航**: 统一管理页面组级导航
- **✅ ComponentAreaName**: 使用通用的组件区域命名，通过注释详细描述具体的布局功能和职责
- **🔸🟢 FunctionAreaName**: 使用通用的功能区域命名，通过注释详细描述具体的业务功能、实现复杂度和技术细节

### 🔧 MyBatis-Plus最佳实践架构说明

- **🏗️ Entity**: 数据实体类，使用MyBatis-Plus注解配置表映射关系
- **📦 DTO**: 数据传输对象，用于前后端数据交换，与前端API一致
- **🔍 VO (Query)**: 查询视图对象（可选），用于封装复杂查询条件，简单查询可直接使用DTO
- **⚙️ Service**: 业务服务接口，定义业务方法签名
- **🔧 ServiceImpl**: 业务服务实现类，使用QueryWrapper/LambdaQueryWrapper执行数据库查询
- **🏗️ QueryBuilder**: 查询构建器（可选），用于封装和复用复杂查询逻辑
- **🗄️ Mapper**: 数据访问接口，继承BaseMapper<Entity>，利用MyBatis-Plus内置CRUD方法
- **📝 SQL文件**: 仅在极复杂查询场景下使用，优先使用MyBatis-Plus提供的方法

## 📋 命名规范

### 🏷️ 核心文件
| 文件类型 | 命名格式 | 职责描述 |
|---------|---------|----------|
| **主组件** | `index.[ext]` | 组件的主要UI实现和功能组装 |
| **类型定义** | `types.[ext]` | 数据结构、接口、类型定义 |
| **常量定义** | `constants.[ext]` | 组件相关的常量配置 |
| **组件文档** | `README.md` | 组件使用说明和API文档 |

### 📁 页面组命名
|| 层级类型 | 命名格式 | 示例 | 说明 |
||---------|---------|------|------|
|| **页面组** | `{PageGroupName}/` | `UserProfile/`, `ProductCatalog/` | 主功能模块，最多10个相关页面 |
|| **主页面** | `MainPage/` | `MainPage/` | 固定命名，页面组的主入口 |
|| **子功能页面** | `{SubFunction}Page/` | `DetailPage/`, `FormPage/`, `EditPage/` | 通用化子功能页面命名 |
|| **通用示例** | `{SubFunction}Page/` | `ReportPage/`, `SharePage/`, `SettingsPage/` | 根据具体业务功能定义 |

### 🔄 状态管理 (优先在主文件内管理)
| 功能类型 | 命名格式 | 示例 | 创建条件 |
|---------|---------|------|----------|
| **页面组状态** | `use{PageGroup}.[ext]` | `useUserProfile.[ext]` | 仅复杂跨页面状态时创建 |
| **页面组数据** | `use{PageGroup}Data.[ext]` | `useUserProfileData.[ext]` | 仅复杂数据逻辑时创建 |
| **页面组流程** | `use{PageGroup}Flow.[ext]` | `useUserProfileFlow.[ext]` | 仅复杂流程时创建 |
| **子功能页状态** | `use{SubFunction}.[ext]` | `useDetail.[ext]`, `useForm.[ext]` | 仅特别复杂的状态逻辑时创建 |
| **本地状态** | `use[LocalState].[ext]` | `useToggle.[ext]` | 避免创建，优先在主文件内管理 |

### 🎯 事件处理 (优先在主文件内管理)
| 事件类型 | 命名格式 | 示例 | 创建条件 |
|---------|---------|------|----------|
| **基础交互** | `on[Action].[ext]` | `onClick.[ext]` | 避免创建，优先在主文件内定义 |
| **复杂操作** | `on[UserAction].[ext]` | `onLongPress.[ext]` | 仅特别复杂的事件逻辑时创建 |

### 🧭 导航处理
| 导航类型 | 命名格式 | 示例 |
|---------|---------|------|
| **页面组内导航** | `navigate{PageGroup}Flow.[ext]` | `navigateUserProfileFlow.[ext]` |
| **子页面导航** | `navigateToSubPages.[ext]` | `navigateToSubPages.[ext]` |
| **具体页面导航** | `navigateTo{SubFunction}.[ext]` | `navigateToDetail.[ext]`, `navigateToForm.[ext]` |
| **返回导航** | `navigateBack{PageGroup}.[ext]` | `navigateBackUserProfile.[ext]` |

### 🌐 API接口
| 接口类型 | 命名格式 | 示例 |
|---------|---------|------|
| **主页面API** | `api{PageGroup}Main.[ext]` | `apiUserProfileMain.[ext]` |
| **子功能页API** | `api{PageGroup}[SubFunction].[ext]` | `apiUserProfileDetail.[ext]`, `apiUserProfileForm.[ext]` |
| **聚合API** | `api{PageGroup}Aggregate.[ext]` | `apiUserProfileAggregate.[ext]` |

### 🔄 数据处理 (优先在主文件内管理)
| 处理类型 | 命名格式 | 示例 | 创建条件 |
|---------|---------|------|----------|
| **数据处理** | `processData.[ext]` | `processData.[ext]` | 仅复杂数据处理逻辑时创建 |
| **逻辑处理** | `processLogic.[ext]` | `processLogic.[ext]` | 仅复杂业务逻辑时创建 |
| **内容处理** | `processContent.[ext]` | `processContent.[ext]` | 仅复杂内容处理时创建 |
| **验证处理** | `processValidation.[ext]` | `processValidation.[ext]` | 仅复杂验证逻辑时创建 |

### 🛠️ 工具函数 (优先在主文件内管理)
| 工具类型 | 命名格式 | 示例 | 创建条件 |
|---------|---------|------|----------|
| **显示工具** | `utilsDisplay.[ext]` | `utilsDisplay.[ext]` | 仅复杂显示逻辑时创建 |
| **辅助工具** | `utilsHelper.[ext]` | `utilsHelper.[ext]` | 仅复杂辅助逻辑时创建 |
| **动画工具** | `utilsAnimation.[ext]` | `utilsAnimation.[ext]` | 仅复杂动画逻辑时创建 |
| **格式化工具** | `utilsFormat.[ext]` | `utilsFormat.[ext]` | 仅复杂格式化逻辑时创建 |

## 📝 八段式文件内代码编写规范

### 🎯 八段式结构标准

所有 `index.[ext]` 主文件都必须按照以下八段式结构组织代码：

```
// #region 1. File Banner & TOC
/**
 * 组件名称 - 功能描述
 * 
 * TOC (快速跳转):
 * [1] Imports
 * [2] Types & Schema  
 * [3] Constants & Config
 * [4] Utils & Helpers
 * [5] State Management
 * [6] Domain Logic
 * [7] UI Components & Rendering
 * [8] Exports
 */
// #endregion

// #region 2. Imports
// React/框架核心导入
// 第三方库导入
// 内部模块导入 (./types, ./constants, ./use*, ./on*, etc.)
// #endregion

// #region 3. Types & Schema
// Props接口定义
// 本地类型定义 (如果types.[ext]不够用)
// 运行时Schema (可选)
// #endregion

// #region 4. Constants & Config
// 本地常量 (如果constants.[ext]不够用)
// 配置项
// #endregion

// #region 5. Utils & Helpers
// 本地工具函数
// 格式化函数
// 计算函数
// #endregion

// #region 6. State Management
// 状态初始化
// useEffect副作用
// 状态派生和计算
// #endregion

// #region 7. Domain Logic
// 事件处理函数
// 业务逻辑函数
// API调用逻辑
// #endregion

// #region 8. UI Components & Rendering
// 子组件定义 (如果简单的话)
// 主渲染JSX
// 条件渲染逻辑
// 样式定义
// #endregion

// #region 9. Exports
// export default Component
// 可选导出 (types, utils等)
// #endregion
```

### 💡 八段式编写原则

- **严格分段**: 每段职责单一，使用 `#region` 标记
- **顺序固定**: 必须按1-8的顺序组织代码
- **可折叠导航**: 便于快速定位和代码审查
- **自包含**: 避免跨段的复杂依赖关系

## 🤖 Agent 重构操作指南

### 📋 重构执行步骤

1. **分析阶段**: 分析现有组件功能模块，规划完整架构结构
2. **创建阶段**: 创建组件根目录和核心文件，根据功能需求创建对应文件
3. **前后端一体化实施阶段**: 
   - 🌐 **前端API层创建**: 创建所有必需的API接口文件
   - 🔌 **后端交互层创建**: 同步创建完整的后端实现文件
   - 📝 **接口文档编写**: 明确定义前后端数据交换格式
4. **重构阶段**: 按职责拆分代码到对应文件，遵循简洁实用原则和八段式结构
5. **验证阶段**: 检查文件命名、职责分离、类型完整性、八段式结构完整性、**前后端一体化完整性**

### 🎯 重构质量标准

- 文件命名符合规范，职责单一明确
- 类型定义完整，常量全部提取  
- 主组件简洁清晰，文档完整
- 所有 `index.[ext]` 文件严格遵循八段式结构
- **🚨 前后端一体化完整性检查清单（强制执行）**：
  - ✅ **🎯 接口按需创建检查 - 所有API接口都有对应的前端页面功能需求**
  - ✅ API接口层文件完整创建（仅创建前端实际需要的接口）
  - ✅ 后端交互层核心文件完整创建（entity、dto、vo、controller、service、serviceImpl、mapper）
  - ✅ 前后端数据类型定义一致性
  - ✅ API接口与后端service方法对应关系明确
  - ✅ 数据传输对象（DTO）仅包含前端实际使用的字段
  - ✅ **查询视图对象（VO）用于封装查询条件（可选，简单查询可直接使用DTO）**
  - ✅ **ServiceImpl层使用QueryWrapper/LambdaQueryWrapper执行数据库查询操作**
  - ✅ **复杂查询逻辑可选择使用QueryBuilder进行封装和复用**
  - ✅ **Entity类正确配置MyBatis-Plus注解（@TableName、@TableId、@TableField等）**
  - ✅ **Mapper接口继承BaseMapper<Entity>，利用MyBatis-Plus内置方法**
  - ✅ 错误处理和异常情况的前后端统一处理
  - ❌ **🚫 过度设计检查 - 确认未创建批量操作、管理员功能、复杂报表等前端未明确需要的接口**
  - ❌ **🚫 过度文件拆分检查 - 确认未为简单逻辑创建不必要的独立文件，状态、事件、数据处理、工具函数优先集中在主文件内**

## 🎯 核心原则

### 🏗️ 架构原则
1. **层级化页面组主导** - 所有页面按页面组层级化架构
2. **移除中间层** - 禁止使用 `components/` 中间层级
3. **多页面并存** - 页面组支持多个相关页面并存
4. **统一标准** - 所有页面组按完整架构实施
5. **功能区域导向** - 根据UI功能区域进行模块划分
6. **🚨 前后端一体化强制原则** - API接口层与后端交互层必须同时完整实施，确保可用性
7. **🔧 技术栈标准化原则** - 数据访问层统一使用MyBatis-Plus + QueryWrapper，通过QueryBuilder封装查询逻辑
8. **🎯 按需设计原则** - 接口设计严格遵循YAGNI原则，只实现前端明确需要的功能，避免过度设计
9. **📦 逻辑集中化原则** - 优先在主文件内管理状态、事件处理、数据处理、简单工具函数，减少文件碎片化

### 💻 实施原则  
10. **架构完整 + 代码简洁** - 结构完整但实现简洁
11. **强制执行** - Agent 严格按标准执行
12. **渐进演化** - 支持从简单到复杂的演化
13. **🚨 前后端同步交付** - 不得单独交付前端接口，必须确保后端实现可运行可测试
14. **🔧 技术栈一致性** - 统一使用MyBatis-Plus + QueryWrapper + QueryBuilder模式，实现类型安全的查询封装
15. **🎯 最小可用原则** - 每次实施只创建当前前端页面实际需要的接口，后续可按需扩展
16. **📦 逻辑集中管理** - 避免过度文件拆分，优先在主文件内管理状态、事件、数据处理、简单工具函数等逻辑

---

**版本**: 2.2.0  
**更新日期**: 2024年  
**适用范围**: 所有现代前端框架和编程语言  
**维护者**: 架构团队