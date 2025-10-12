# PlantUML数据库设计更新说明 v5.0

> **更新时间**: 2025年10月12日  
> **参考文档**: 发现页面模块架构设计文档v1.6  
> **设计原则**: 无JSON设计，所有字段具体化

---

## 📊 更新概览

### 版本变更
- **v4.1 → v5.0**
- **表数量**: 36张 → **43张** (+7张新表)
- **核心模块**: 新增完整的发现页面社交模块

---

## 🆕 新增表结构 (7张)

### 1. **Comment** - 评论表
**设计理由**: 从ContentAction中分离评论功能，支持多级评论系统

**核心字段**:
- `parent_id`: 一级评论ID（二级回复时使用）
- `reply_to_id`: 直接回复的评论ID
- `reply_to_user_id`: 被回复的用户ID
- `comment_text`: 评论内容（最大500字）
- `like_count`: 点赞数量冗余
- `reply_count`: 回复数量冗余
- `is_top`: 是否置顶评论
- `status`: 评论状态（0=已删除,1=正常,2=审核中,3=已屏蔽）

**功能支持**:
- ✅ 一级评论 + 二级回复
- ✅ 评论点赞统计
- ✅ 评论排序（时间/热度/点赞数）
- ✅ 评论置顶功能
- ✅ 评论状态管理

---

### 2. **CommentLike** - 评论点赞表
**设计理由**: 独立的评论点赞系统，与内容点赞分离

**核心字段**:
- `comment_id`: 评论ID
- `user_id`: 点赞用户ID
- `status`: 状态（0=已取消,1=正常）

**功能支持**:
- ✅ 评论独立点赞记录
- ✅ 支持取消点赞
- ✅ 实时统计更新

---

### 3. **ContentDraft** - 内容草稿表
**设计理由**: 支持发布动态时的草稿自动保存功能

**核心字段**:
- `user_id`: 创建者ID
- `type`: 内容类型（1=动态,2=活动,3=技能服务）
- `title`: 标题草稿
- `content`: 正文草稿
- `location_name`, `location_address`, `location_lat`, `location_lng`, `city_id`: 位置信息
- `auto_save_time`: 自动保存时间
- `expire_time`: 过期时间（30天后清理）
- `status`: 草稿状态（0=已发布,1=编辑中,2=已过期,3=已删除）

**功能支持**:
- ✅ 自动保存草稿（每10秒）
- ✅ 支持所有内容类型
- ✅ 位置信息保存
- ✅ 30天过期自动清理
- ✅ 发布后清理草稿

---

### 4. **ContentTopic** - 内容话题关联表
**设计理由**: 内容和话题的多对多关联，一个内容可以有多个话题

**核心字段**:
- `content_id`: 内容ID
- `topic_id`: 话题ID
- `sort_order`: 排序顺序（多个话题时的显示顺序）

**功能支持**:
- ✅ 多对多关联（最多5个话题）
- ✅ 话题排序控制
- ✅ 话题聚合展示

---

### 5. **TopicFollow** - 用户关注话题表
**设计理由**: 用户可以关注感兴趣的话题，接收话题更新通知

**核心字段**:
- `user_id`: 用户ID
- `topic_id`: 话题ID
- `status`: 关注状态（0=已取消,1=正常）

**功能支持**:
- ✅ 用户关注话题功能
- ✅ 关注状态管理
- ✅ 支持取消关注
- ✅ 话题更新通知

---

## 🔄 完善现有表结构 (5张)

### 1. **Content** - 内容表
**新增字段**:
- `location_name`: 地点名称（场所名称）
- `location_address`: 详细地址
- `location_lat`: 纬度
- `location_lng`: 经度
- `city_id`: 所在城市ID
- `publish_time`: 发布时间（支持定时发布）

**修改说明**:
- `status`: 扩展状态枚举（0=草稿,1=已发布,2=审核中,3=已下架,4=已删除）
- `title`: 明确说明（动态标题，最大50字）
- `content`: 明确说明（正文，最大1000字）

**功能增强**:
- ✅ 完整的位置信息支持
- ✅ 定时发布功能
- ✅ 更细化的内容状态管理

---

### 2. **ContentAction** - 内容行为表
**重大调整**: 移除评论相关字段，专注于其他互动行为

**移除字段**:
- ~~`comment_text`~~（迁移到Comment表）
- ~~`reply_to_id`~~（迁移到Comment表）

**新增字段**:
- `status`: 状态（0=已取消,1=正常）
- `updated_at`: 更新时间

**功能调整**:
- `action`: 行为类型（1=点赞,2=分享,3=收藏,4=浏览）
- ❌ 移除评论功能（action=2）
- ✅ 更清晰的职责划分

---

### 3. **UserProfile** - 用户资料扩展表
**新增字段**:
- `is_vip`: 是否VIP用户
- `is_popular`: 是否人气用户（系统认证）
- `vip_level`: VIP等级（1-5级）
- `vip_expire_time`: VIP过期时间
- `follower_count`: 粉丝数量
- `following_count`: 关注数量
- `content_count`: 发布内容数量（动态数量）
- `total_like_count`: 获赞总数

**功能增强**:
- ✅ 完整的用户标识系统
- ✅ VIP会员体系
- ✅ 人气用户认证
- ✅ 统计数据冗余（提升查询性能）

---

### 4. **Topic** - 话题标签表
**新增字段**:
- `category`: 话题分类（1=游戏,2=娱乐,3=生活,4=美食,5=运动,6=其他）
- `view_count`: 总浏览次数
- `like_count`: 总点赞数量
- `comment_count`: 总评论数量
- `share_count`: 总分享数量
- `follow_count`: 关注人数
- `trend_score`: 趋势分数（热度增长率）
- `today_post_count`: 今日动态数
- `week_post_count`: 本周动态数
- `month_post_count`: 本月动态数
- `last_post_time`: 最后发布动态时间
- `is_hot`: 是否热门话题
- `is_trending`: 是否趋势话题
- `updated_at`: 更新时间

**功能增强**:
- ✅ 话题分类管理
- ✅ 完整的多维度统计
- ✅ 时间维度统计（今日/本周/本月）
- ✅ 热门话题/趋势话题标识
- ✅ 热度计算和趋势分析

---

## 🔗 关系定义更新

### 新增关系
```plantuml
' 评论系统关系
Content "1" o-- "0..*" Comment : "内容评论\n多级评论"
User "评论者" -- "0..*" Comment : "发表评论\n评论创建"
Comment "父评论" -- "0..*" Comment : "子评论回复\n评论层级"
Comment "1" o-- "0..*" CommentLike : "评论点赞\n互动统计"
User "1" -- "0..*" CommentLike : "点赞评论\n用户互动"

' 草稿关系
User "1" o-- "0..*" ContentDraft : "创建草稿\n自动保存"

' 话题关联关系
Topic "1" o-- "0..*" ContentTopic : "话题内容\n多对多关联"
Content "1" o-- "0..*" ContentTopic : "内容话题\n话题标签"
Topic "1" o-- "0..*" TopicFollow : "话题关注\n用户订阅"
User "1" -- "0..*" TopicFollow : "关注话题\n话题订阅"
User "创建者" -- "0..*" Topic : "创建话题\n话题管理"

' 位置关系
City "1" -- "0..*" Content : "内容城市\n地理位置"
City "1" -- "0..*" ContentDraft : "草稿城市\n地理位置"

' 通知关系
Comment "触发通知" ..> Notification : "评论通知\n评论回复"
CommentLike "触发通知" ..> Notification : "评论点赞\n点赞通知"
TopicFollow "触发通知" ..> Notification : "话题通知\n话题更新"

' 举报关系
Comment "被举报" -- "0..*" Report : "评论举报\n评论管理"
```

---

## 📋 索引建议 (发现页面模块)

### Content表索引
```sql
-- 用户内容查询
INDEX idx_content_user (user_id, type, status, created_at)

-- 同城内容查询
INDEX idx_content_city (city_id, type, status, publish_time)

-- 地理位置查询
INDEX idx_content_location (location_lat, location_lng, status)

-- 热门内容查询
INDEX idx_content_hot (status, is_hot, created_at)
```

### Comment表索引
```sql
-- 内容评论列表
INDEX idx_comment_content (content_id, status, created_at)

-- 子评论查询
INDEX idx_comment_parent (parent_id, status, created_at)

-- 用户评论列表
INDEX idx_comment_user (user_id, status, created_at)
```

### CommentLike表索引
```sql
-- 评论点赞查询
INDEX idx_comment_like (comment_id, user_id, status)
```

### ContentDraft表索引
```sql
-- 用户草稿列表
INDEX idx_draft_user (user_id, type, status, updated_at)
```

### ContentTopic表索引
```sql
-- 内容话题查询
INDEX idx_content_topic_content (content_id, topic_id)

-- 话题内容查询
INDEX idx_content_topic_topic (topic_id, created_at)
```

### Topic表索引
```sql
-- 热门话题查询
INDEX idx_topic_hot (status, is_hot, heat_score)

-- 分类话题查询
INDEX idx_topic_category (category, status, heat_score)

-- 趋势话题查询
INDEX idx_topic_trending (status, is_trending, trend_score)
```

### TopicFollow表索引
```sql
-- 用户关注话题
INDEX idx_topic_follow_user (user_id, status, created_at)

-- 话题粉丝列表
INDEX idx_topic_follow_topic (topic_id, status, created_at)
```

### ContentAction表索引
```sql
-- 内容互动查询
INDEX idx_action_content (content_id, action, status)

-- 用户互动历史
INDEX idx_action_user (user_id, action, status, created_at)
```

---

## 🎯 性能优化策略

### 1. 字段冗余优化
```
✅ UserProfile: 统计字段冗余
   - follower_count, following_count, content_count, total_like_count
   
✅ Content: 互动统计字段冗余
   - view_count, like_count, comment_count, share_count, collect_count
   
✅ Comment: 统计字段冗余
   - like_count, reply_count
   
✅ Topic: 多维度统计字段冗余
   - participant_count, post_count, view_count, like_count, comment_count
   - share_count, follow_count, today_post_count, week_post_count, month_post_count
```

### 2. 数据库分表策略
```
🔄 Comment表建议分表（按content_id或创建时间）
🔄 ContentAction表建议分表（按用户ID或时间）
🔄 ContentDraft表定期清理过期数据（30天）
```

### 3. 缓存策略
```
📦 热门话题缓存：Redis，TTL=15分钟
📦 热门动态缓存：Redis，TTL=30分钟
📦 用户资料缓存：Redis，TTL=1小时
📦 评论列表缓存：Redis，TTL=5分钟
```

---

## 🎨 功能模块架构

### 发现页面模块 (核心社交)
```
Content(type=1动态) + User + Topic
├── ContentDraft: 草稿自动保存(每10秒)
├── ContentTopic: 内容话题关联(最多5个话题)
├── Comment: 评论系统
│   ├── 一级评论+二级回复
│   ├── CommentLike: 评论点赞
│   ├── 评论排序(时间/热度/点赞数)
│   └── 评论状态管理
├── ContentAction: 互动行为
│   ├── 点赞/分享/收藏/浏览
│   └── 状态管理
├── Topic: 话题系统
│   ├── 话题统计(参与人数/动态数/热度)
│   ├── TopicFollow: 话题关注
│   ├── 热门话题/趋势话题标识
│   └── 时间维度统计(今日/本周/本月)
├── Media: 媒体管理
│   ├── 图片(最多9张)+视频(1个)
│   ├── 媒体上传处理
│   └── 缩略图生成
├── UserProfile: 用户标识系统
│   ├── 性别年龄标识
│   ├── 人气用户标识
│   ├── VIP/认证标识
│   └── 统计数据(粉丝/关注/内容/获赞)
└── Report: 举报安全系统
    ├── 6类举报理由
    ├── 举报状态跟踪
    └── 24小时处理机制
```

---

## ✅ 设计原则验证

### ✅ 无JSON设计
- 所有字段均为具体类型
- 位置信息展开为独立字段（location_name, location_address, location_lat, location_lng, city_id）
- 没有使用任何JSON/JSONB字段

### ✅ 关系清晰
- 明确的外键关联
- 多对多关系通过中间表实现（ContentTopic）
- 自关联关系明确（Comment父子关系）

### ✅ 扩展性强
- 评论系统独立，易于扩展
- 话题系统独立管理
- 草稿系统独立，不影响正式内容

### ✅ 查询优化
- 合理的字段冗余（统计数据）
- 完善的索引建议
- 清晰的数据分离（评论/草稿/话题）

### ✅ 模块化设计
- 发现页面模块独立完整
- 与其他模块（游戏/生活/活动）解耦
- 清晰的模块边界

---

## 📊 表数量统计

### 各模块表数量分布
```
📋 核心用户模块: 4张表
   - User, UserProfile, UserWallet, Transaction

📋 内容模块: 9张表 (新增6张)
   - Content, ContentAction, Comment, CommentLike
   - ContentDraft, ContentTopic, TopicFollow, UserRelation
   - Media (共享表)

📋 话题标签模块: 4张表 (扩展2张)
   - Topic, UserTag, ContentTopic, TopicFollow

📋 游戏服务模块: 3张表
   - GameService, GameSkill, GameServiceTag

📋 生活服务模块: 3张表
   - LifeService, LifeServiceFeature, LifeServiceTag

📋 订单评价模块: 2张表
   - ServiceOrder, ServiceReview

📋 活动组局模块: 3张表
   - Activity, ActivityParticipant, ActivityTag

📋 通知推送模块: 1张表
   - Notification

📋 安全审核模块: 1张表
   - Report

📋 搜索历史模块: 1张表
   - SearchHistory

📋 聊天模块: 3张表
   - ChatConversation, ChatMessage, ChatParticipant

📋 首页功能模块: 4张表
   - UserPreference, UserBehavior, HotSearch, City

总计: 43张表
```

---

## 🔄 迁移建议

### 1. 数据迁移步骤
```sql
-- Step 1: 创建新表
CREATE TABLE Comment (...);
CREATE TABLE CommentLike (...);
CREATE TABLE ContentDraft (...);
CREATE TABLE ContentTopic (...);
CREATE TABLE TopicFollow (...);

-- Step 2: 迁移评论数据
INSERT INTO Comment (content_id, user_id, comment_text, ...)
SELECT content_id, user_id, comment_text, ...
FROM ContentAction
WHERE action = 2; -- 评论类型

-- Step 3: 更新Content表
ALTER TABLE Content ADD COLUMN location_name VARCHAR(200);
ALTER TABLE Content ADD COLUMN location_address VARCHAR(500);
ALTER TABLE Content ADD COLUMN location_lat DECIMAL(10, 7);
ALTER TABLE Content ADD COLUMN location_lng DECIMAL(10, 7);
ALTER TABLE Content ADD COLUMN city_id BIGINT;
ALTER TABLE Content ADD COLUMN publish_time DATETIME;

-- Step 4: 更新UserProfile表
ALTER TABLE UserProfile ADD COLUMN is_vip BOOLEAN DEFAULT FALSE;
ALTER TABLE UserProfile ADD COLUMN is_popular BOOLEAN DEFAULT FALSE;
ALTER TABLE UserProfile ADD COLUMN vip_level INT DEFAULT 0;
ALTER TABLE UserProfile ADD COLUMN vip_expire_time DATETIME;
ALTER TABLE UserProfile ADD COLUMN follower_count INT DEFAULT 0;
ALTER TABLE UserProfile ADD COLUMN following_count INT DEFAULT 0;
ALTER TABLE UserProfile ADD COLUMN content_count INT DEFAULT 0;
ALTER TABLE UserProfile ADD COLUMN total_like_count INT DEFAULT 0;

-- Step 5: 更新Topic表
ALTER TABLE Topic ADD COLUMN category INT DEFAULT 6;
ALTER TABLE Topic ADD COLUMN view_count INT DEFAULT 0;
-- ... 其他字段

-- Step 6: 删除ContentAction中的评论数据
DELETE FROM ContentAction WHERE action = 2;

-- Step 7: 创建索引
CREATE INDEX idx_comment_content ON Comment(content_id, status, created_at);
-- ... 其他索引

-- Step 8: 初始化统计数据
UPDATE UserProfile SET 
  follower_count = (SELECT COUNT(*) FROM UserRelation WHERE target_id = user_id AND type = 1 AND status = 1),
  following_count = (SELECT COUNT(*) FROM UserRelation WHERE user_id = UserProfile.user_id AND type = 1 AND status = 1),
  content_count = (SELECT COUNT(*) FROM Content WHERE user_id = UserProfile.user_id AND status = 1);
```

### 2. 兼容性考虑
- ✅ ContentAction表保留，移除评论功能
- ✅ 新增字段默认值设置
- ✅ 索引逐步创建，避免锁表
- ✅ 分批迁移数据，避免长事务

---

## 📝 总结

### 主要改进
1. ✅ 评论系统独立化（Comment + CommentLike）
2. ✅ 草稿自动保存功能（ContentDraft）
3. ✅ 话题系统完善（Topic + ContentTopic + TopicFollow）
4. ✅ 用户标识系统增强（UserProfile新增8个字段）
5. ✅ 位置信息完整化（Content新增5个位置字段）
6. ✅ 定时发布支持（Content.publish_time）
7. ✅ 多维度统计完善（Topic新增15个统计字段）

### 设计亮点
- 🎯 **模块化设计**：发现页面模块完整独立
- 🎯 **无JSON原则**：所有字段具体化，查询性能优异
- 🎯 **关系清晰**：多对多、一对多、自关联关系明确
- 🎯 **可扩展性强**：新增功能不影响现有结构
- 🎯 **性能优化**：合理的字段冗余和索引设计

### 适用场景
- ✅ 社交内容发布平台（微博、小红书类）
- ✅ 社区论坛系统
- ✅ 话题讨论平台
- ✅ 用户互动社区
- ✅ 内容聚合平台

---

**设计完成时间**: 2025年10月12日  
**设计师**: AI Assistant  
**参考标准**: PlantUML Best Practices + Database Normalization  
**版本**: v5.0

