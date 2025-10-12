@startuml

' ==========================================
' 🏗️ XY相遇派完整系统 - 综合类图设计 v7.0 (重构版)
' 🔧 重大重构说明：
' 1. 移除冗余统计字段，改用独立统计表+Redis缓存
' 2. 字符串分隔字段改为关联表设计（符合第一范式）
' 3. 地理位置字段改为空间索引类型
' 4. 添加安全增强机制（JWT黑名单、验证码防穷举）
' 5. 添加分布式事务和限流相关设计
' 6. 添加版本控制和审计功能
' 7. 消息表明确分片策略
' ==========================================
' 设计理念：生产级数据库设计，高性能、高可用、高安全
' 核心模块：登录认证、游戏服务、生活服务、活动组局、评价系统、发现页面、个人主页、消息通信等
' 表数量：60张表（新增7张核心表）
' 参考文档：发现页面模块架构设计文档v1.6、个人主页模块架构设计文档v2.0、个人信息编辑模块架构设计文档v1.0、消息模块架构设计文档v1.0、探店APP登录模块架构设计、探店APP登录模块完整功能矩阵
' ==========================================

' ==========================================
' 🔧 v7.0 重构总结
' ==========================================
' 
' **核心改进：**
' 1. 数据一致性 ✅
'    - 移除冗余统计字段（UserProfile/Content/Topic/GameService/LifeService）
'    - 新增独立统计表（UserStats/ContentStats/TopicStats/ServiceStats）
'    - 采用 Redis + MySQL 双层存储，异步同步机制
'    - 消息队列保证最终一致性
' 
' 2. 查询性能 ✅
'    - 地理位置改用 POINT 类型 + 空间索引（SPATIAL INDEX）
'    - ChatMessage 按会话ID哈希分256张表
'    - Content 冗余 user_nickname/user_avatar（避免 N+1 查询）
'    - 完善索引设计（覆盖索引、组合索引优化）
' 
' 3. 数据规范化 ✅
'    - occupation_tags 字符串改为 UserOccupation 关联表
'    - available_dates/cover_images 等改用关联表或 Media 表
'    - 新增 OccupationDict 职业字典表（统一管理枚举）
' 
' 4. 安全增强 ✅
'    - TokenBlacklist 表（JWT令牌撤销支持）
'    - PhoneVerifyLimit 表（防验证码穷举攻击）
'    - RateLimitConfig 表（统一限流管理）
'    - 敏感字段加密（id_card_encrypted 使用 AES-256）
' 
' 5. 扩展性提升 ✅
'    - 所有核心表添加 deleted_at（软删除）
'    - Content/Activity/GameService 添加 version（乐观锁）
'    - ChatMessage 添加 recalled_by（撤回操作人）
'    - 明确分片策略和归档机制
' 
' **设计原则：**
' • 统计数据与业务数据分离
' • 读写分离：Redis写入 + MySQL持久化
' • 最终一致性：消息队列 + 定时任务修正
' • 空间索引：地理查询性能提升10倍+
' • 软删除：数据可追溯、可恢复
' ==========================================

' ===== 核心用户模块 (4表) =====

class User {
    + id : Long
    + username : String  
    + mobile : String
    + region_code : String
    + email : String
    + password : String
    + password_salt : String
    + password_updated_at : DateTime
    + status : Integer
    + login_fail_count : Integer
    + login_locked_until : DateTime
    + last_login_time : DateTime
    + last_login_ip : String
    + last_login_device_id : String
    + is_two_factor_enabled : Boolean
    + two_factor_secret : String
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 用户基础信息表(含登录认证信息)
    ' id: 用户唯一标识(雪花ID)
    ' username: 登录用户名(唯一,可为空)
    ' mobile: 手机号(唯一,主要登录凭证)
    ' region_code: 地区代码(如+86,默认+86)
    ' email: 邮箱(唯一,辅助登录凭证,可为空)
    ' password: 密码哈希值(BCrypt加密)
    ' password_salt: 密码盐值(随机生成)
    ' password_updated_at: 密码最后更新时间
    ' status: 用户状态(0=禁用,1=正常,2=冻结,3=待激活)
    ' login_fail_count: 登录失败次数(用于防暴力破解)
    ' login_locked_until: 账户锁定截止时间(5次失败锁定30分钟)
    ' last_login_time: 最后登录时间
    ' last_login_ip: 最后登录IP地址
    ' last_login_device_id: 最后登录设备ID
    ' is_two_factor_enabled: 是否启用双因子认证
    ' two_factor_secret: 双因子认证密钥(TOTP)
    ' created_at: 注册时间
    ' updated_at: 更新时间
}

class UserProfile {
    + user_id : Long
    + nickname : String
    + avatar : String
    + avatar_thumbnail : String
    + background_image : String
    + gender : Integer
    + birthday : Date
    + age : Integer
    + city_id : Long
    + location : String
    + address : String
    + ip_location : String
    + bio : String
    + height : Integer
    + weight : Integer
    ' ❌ 删除：occupation_tags : String (改用关联表 UserOccupation)
    + real_name : String
    + id_card_encrypted : String
    + wechat : String
    + wechat_unlock_condition : Integer
    + is_real_verified : Boolean
    + is_god_verified : Boolean
    + is_activity_expert : Boolean
    + is_vip : Boolean
    + is_popular : Boolean
    + vip_level : Integer
    + vip_expire_time : DateTime
    + online_status : Integer
    + last_online_time : DateTime
    ' ❌ 删除以下冗余统计字段（改用 UserStats 表 + Redis 缓存）
    ' follower_count : Integer
    ' following_count : Integer
    ' content_count : Integer
    ' total_like_count : Integer
    ' total_collect_count : Integer
    ' activity_organizer_count : Integer
    ' activity_participant_count : Integer
    ' activity_success_count : Integer
    ' activity_cancel_count : Integer
    ' activity_organizer_score : Decimal
    ' activity_success_rate : Decimal
    + profile_completeness : Integer
    + last_edit_time : DateTime
    + deleted_at : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 用户资料扩展表(支持个人主页和编辑功能)
    ' user_id: 关联用户ID
    ' nickname: 用户昵称(显示名,1-20字符)
    ' avatar: 头像URL(正方形裁剪,CDN地址)
    ' avatar_thumbnail: 头像缩略图URL(用于列表显示)
    ' background_image: 个人主页背景图URL
    ' gender: 性别(1=男,2=女,3=其他,0=未设置)
    ' birthday: 生日(YYYY-MM-DD格式)
    ' age: 年龄(根据生日自动计算)
    ' city_id: 所在城市ID(关联City表)
    ' location: 城市位置信息(如"广东 深圳")
    ' address: 详细地址
    ' ip_location: IP归属地(如"广东 深圳",用于显示)
    ' bio: 个人简介(0-30字符,多行文本)
    ' height: 身高(cm,范围140-200)
    ' weight: 体重(kg,范围30-150)
    ' ❌ 删除 occupation_tags: 改用 UserOccupation 关联表（支持高效查询和管理）
    ' real_name: 真实姓名(实名认证使用)
    ' id_card_encrypted: 身份证号(AES-256加密存储,使用KMS密钥管理)
    ' wechat: 微信号(6-20位字母数字下划线)
    ' wechat_unlock_condition: 微信解锁条件(0=公开,1=关注后可见,2=付费可见,3=私密)
    ' is_real_verified: 是否实名认证
    ' is_god_verified: 是否大神认证
    ' is_activity_expert: 是否组局达人认证
    ' is_vip: 是否VIP用户
    ' is_popular: 是否人气用户(系统认证)
    ' vip_level: VIP等级(1-5级)
    ' vip_expire_time: VIP过期时间
    ' online_status: 在线状态(0=离线,1=在线,2=忙碌,3=隐身)
    ' last_online_time: 最后在线时间
    ' ❌ 删除冗余统计字段说明：
    ' follower_count/following_count/content_count/total_like_count/total_collect_count
    ' activity_organizer_count/activity_participant_count/activity_success_count
    ' activity_cancel_count/activity_organizer_score/activity_success_rate
    ' → 改用 UserStats 表存储，配合 Redis 缓存提升性能，避免事务冲突
    ' → 优势：1.解决高并发更新冲突 2.支持异步同步 3.数据一致性可控
    ' profile_completeness: 资料完整度(0-100百分比,用于推荐算法)
    ' last_edit_time: 最后编辑资料时间
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class UserWallet {
    + user_id : Long
    + balance : Long
    + frozen : Long
    + coin_balance : Long
    + total_income : Long
    + total_expense : Long
    + version : Integer
    + updated_at : DateTime
    --
    ' 用户钱包表
    ' user_id: 关联用户ID
    ' balance: 可用余额(分)
    ' frozen: 冻结金额(分)
    ' coin_balance: 金币余额
    ' total_income: 累计收入(分)
    ' total_expense: 累计支出(分)
    ' version: 乐观锁版本号
    ' updated_at: 最后更新时间
}

class Transaction {
    + id : Long
    + user_id : Long
    + amount : Long
    + type : String
    + ref_type : String
    + ref_id : Long
    + status : Integer
    + payment_method : String
    + payment_no : String
    + description : String
    + created_at : DateTime
    --
    ' 统一交易流水表
    ' id: 交易记录ID
    ' user_id: 用户ID
    ' amount: 交易金额(正负表示收支)
    ' type: 交易类型(recharge/consume/refund/withdraw)
    ' ref_type: 关联类型(order/activity/reward)
    ' ref_id: 关联业务ID
    ' status: 交易状态(0=处理中,1=成功,2=失败)
    ' payment_method: 支付方式(wechat/alipay/balance)
    ' payment_no: 支付流水号
    ' description: 交易描述
    ' created_at: 交易时间
}

' ===== 登录认证模块 (6表) =====

class LoginSession {
    + id : Long
    + user_id : Long
    + device_id : String
    + access_token : String
    + refresh_token : String
    + token_type : String
    + expires_at : DateTime
    + refresh_expires_at : DateTime
    + login_type : Integer
    + login_ip : String
    + login_location : String
    + user_agent : String
    + device_type : String
    + device_name : String
    + os_type : String
    + os_version : String
    + app_version : String
    + network_type : String
    + is_trusted_device : Boolean
    + last_active_time : DateTime
    + status : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 登录会话表(支持多设备并发登录)
    ' id: 会话唯一ID
    ' user_id: 用户ID
    ' device_id: 设备唯一标识(设备指纹)
    ' access_token: 访问令牌(JWT格式,2小时有效期)
    ' refresh_token: 刷新令牌(30天有效期)
    ' token_type: 令牌类型(Bearer)
    ' expires_at: 访问令牌过期时间
    ' refresh_expires_at: 刷新令牌过期时间
    ' login_type: 登录方式(1=密码登录,2=验证码登录,3=第三方登录)
    ' login_ip: 登录IP地址
    ' login_location: 登录地理位置(IP解析)
    ' user_agent: 用户代理字符串
    ' device_type: 设备类型(iOS/Android/Web)
    ' device_name: 设备名称(如iPhone 13 Pro)
    ' os_type: 操作系统类型(iOS/Android/Windows等)
    ' os_version: 操作系统版本
    ' app_version: 应用版本号
    ' network_type: 网络类型(WiFi/4G/5G)
    ' is_trusted_device: 是否可信设备
    ' last_active_time: 最后活跃时间(用于心跳更新)
    ' status: 会话状态(0=已过期,1=正常,2=已注销,3=异常)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class SmsVerification {
    + id : Long
    + mobile : String
    + region_code : String
    + sms_code : String
    + sms_token : String
    + verification_type : Integer
    + scene : String
    + template_code : String
    + send_status : Integer
    + verify_status : Integer
    + verify_count : Integer
    + max_verify_count : Integer
    + ip_address : String
    + device_id : String
    + send_time : DateTime
    + expire_time : DateTime
    + verify_time : DateTime
    + created_at : DateTime
    --
    ' 短信验证码表(支持登录、注册、重置密码等场景)
    ' id: 验证码记录ID
    ' mobile: 目标手机号
    ' region_code: 地区代码(如+86)
    ' sms_code: 验证码内容(6位数字)
    ' sms_token: 短信令牌(用于验证时绑定)
    ' verification_type: 验证类型(1=登录,2=注册,3=重置密码,4=修改手机号)
    ' scene: 业务场景标识(user_login/password_reset等)
    ' template_code: 短信模板代码
    ' send_status: 发送状态(0=待发送,1=发送成功,2=发送失败)
    ' verify_status: 验证状态(0=未验证,1=验证成功,2=验证失败,3=已过期)
    ' verify_count: 验证尝试次数
    ' max_verify_count: 最大尝试次数(默认3次)
    ' ip_address: 请求IP地址
    ' device_id: 设备标识
    ' send_time: 发送时间
    ' expire_time: 过期时间(5分钟有效期)
    ' verify_time: 验证时间
    ' created_at: 创建时间
}

class PasswordResetSession {
    + id : Long
    + session_id : String
    + user_id : Long
    + mobile : String
    + region_code : String
    + reset_token : String
    + sms_verification_id : Long
    + current_step : Integer
    + step_status : String
    + verify_count : Integer
    + max_verify_count : Integer
    + ip_address : String
    + device_id : String
    + user_agent : String
    + session_status : Integer
    + expire_time : DateTime
    + completed_time : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 密码重置会话表(忘记密码流程管理)
    ' id: 会话记录ID
    ' session_id: 会话唯一标识(UUID)
    ' user_id: 用户ID
    ' mobile: 手机号
    ' region_code: 地区代码
    ' reset_token: 重置令牌(验证通过后生成,15分钟有效)
    ' sms_verification_id: 关联的验证码ID
    ' current_step: 当前步骤(1=手机号输入,2=验证码发送,3=验证码验证,4=密码重置,5=完成)
    ' step_status: 步骤状态(逗号分隔,如"1:completed,2:in_progress")
    ' verify_count: 验证尝试次数
    ' max_verify_count: 最大尝试次数(3次)
    ' ip_address: 请求IP地址
    ' device_id: 设备标识
    ' user_agent: 用户代理
    ' session_status: 会话状态(0=已取消,1=进行中,2=已完成,3=已过期,4=已失败)
    ' expire_time: 会话过期时间(30分钟)
    ' completed_time: 完成时间
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class UserDevice {
    + id : Long
    + user_id : Long
    + device_id : String
    + device_fingerprint : String
    + device_name : String
    + device_type : String
    + device_brand : String
    + device_model : String
    + os_type : String
    + os_version : String
    + app_version : String
    + screen_resolution : String
    + network_type : String
    + is_trusted : Boolean
    + trust_expire_time : DateTime
    + first_login_time : DateTime
    + last_login_time : DateTime
    + login_count : Integer
    + last_login_ip : String
    + last_login_location : String
    + status : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 用户设备表(设备管理和信任设备)
    ' id: 设备记录ID
    ' user_id: 用户ID
    ' device_id: 设备唯一标识
    ' device_fingerprint: 设备指纹(多维度组合生成)
    ' device_name: 设备名称(如iPhone 13 Pro)
    ' device_type: 设备类型(iOS/Android/Web/Tablet)
    ' device_brand: 设备品牌(Apple/Samsung/Huawei等)
    ' device_model: 设备型号
    ' os_type: 操作系统类型
    ' os_version: 操作系统版本
    ' app_version: 应用版本号
    ' screen_resolution: 屏幕分辨率
    ' network_type: 网络类型
    ' is_trusted: 是否信任设备
    ' trust_expire_time: 信任过期时间(默认30天)
    ' first_login_time: 首次登录时间
    ' last_login_time: 最后登录时间
    ' login_count: 登录次数统计
    ' last_login_ip: 最后登录IP
    ' last_login_location: 最后登录位置
    ' status: 设备状态(0=已禁用,1=正常,2=异常)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class LoginLog {
    + id : Long
    + user_id : Long
    + mobile : String
    + login_type : Integer
    + login_method : String
    + login_status : Integer
    + fail_reason : String
    + ip_address : String
    + location : String
    + device_id : String
    + device_type : String
    + device_name : String
    + os_type : String
    + os_version : String
    + app_version : String
    + user_agent : String
    + network_type : String
    + login_duration : Integer
    + session_id : String
    + risk_level : Integer
    + risk_tags : String
    + created_at : DateTime
    --
    ' 登录日志表(登录行为审计)
    ' id: 日志记录ID
    ' user_id: 用户ID(登录成功时记录)
    ' mobile: 手机号
    ' login_type: 登录方式(1=密码登录,2=验证码登录,3=第三方登录)
    ' login_method: 登录方法(password/sms_code/wechat/apple等)
    ' login_status: 登录状态(0=失败,1=成功)
    ' fail_reason: 失败原因(密码错误/验证码错误/账户锁定等)
    ' ip_address: 登录IP地址
    ' location: 地理位置(IP解析)
    ' device_id: 设备标识
    ' device_type: 设备类型
    ' device_name: 设备名称
    ' os_type: 操作系统类型
    ' os_version: 操作系统版本
    ' app_version: 应用版本号
    ' user_agent: 用户代理字符串
    ' network_type: 网络类型
    ' login_duration: 登录耗时(毫秒)
    ' session_id: 会话ID(成功时记录)
    ' risk_level: 风险等级(0=正常,1=低风险,2=中风险,3=高风险)
    ' risk_tags: 风险标签(逗号分隔,如"异地登录,新设备,频繁失败")
    ' created_at: 登录时间
}

class SecurityLog {
    + id : Long
    + user_id : Long
    + log_type : Integer
    + action : String
    + action_category : String
    + description : String
    + old_value : String
    + new_value : String
    + result : Integer
    + fail_reason : String
    + ip_address : String
    + location : String
    + device_id : String
    + device_type : String
    + user_agent : String
    + risk_level : Integer
    + is_sensitive : Boolean
    + created_at : DateTime
    --
    ' 安全操作日志表(敏感操作审计)
    ' id: 日志记录ID
    ' user_id: 用户ID
    ' log_type: 日志类型(1=密码修改,2=手机号修改,3=设备管理,4=安全设置,5=账户操作)
    ' action: 操作动作(password_change/phone_change/device_logout/two_factor_enable等)
    ' action_category: 操作分类(authentication/profile/security/account)
    ' description: 操作描述
    ' old_value: 旧值(脱敏处理)
    ' new_value: 新值(脱敏处理)
    ' result: 操作结果(0=失败,1=成功)
    ' fail_reason: 失败原因
    ' ip_address: 操作IP地址
    ' location: 地理位置
    ' device_id: 设备标识
    ' device_type: 设备类型
    ' user_agent: 用户代理
    ' risk_level: 风险等级(0=正常,1=低风险,2=中风险,3=高风险)
    ' is_sensitive: 是否敏感操作(需要二次验证)
    ' created_at: 操作时间
}

' 🔧 新增：JWT令牌黑名单表（支持令牌主动撤销）
class TokenBlacklist {
    + id : Long
    + token : String
    + user_id : Long
    + token_type : String
    + reason : String
    + expire_time : DateTime
    + created_at : DateTime
    --
    ' JWT令牌黑名单表（解决JWT无法撤销问题）
    ' id: 记录ID
    ' token: 令牌字符串（唯一索引）
    ' user_id: 用户ID
    ' token_type: 令牌类型（access_token/refresh_token）
    ' reason: 加入黑名单原因（user_logout/admin_ban/security_issue）
    ' expire_time: 过期时间（与原令牌过期时间一致，过期后自动清理）
    ' created_at: 创建时间
    ' 使用场景：
    ' 1. 用户主动退出登录
    ' 2. 用户远程注销其他设备
    ' 3. 管理员封禁用户
    ' 4. 检测到异常登录行为
    ' 性能优化：配合Redis使用，黑名单令牌存入Redis（自动过期）
}

' 🔧 新增：手机号验证限制表（防验证码穷举攻击）
class PhoneVerifyLimit {
    + mobile : String
    + daily_verify_count : Integer
    + daily_send_count : Integer
    + last_verify_time : DateTime
    + last_reset_date : Date
    + is_blocked : Boolean
    + block_until : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 手机号验证限制表（防止验证码穷举攻击）
    ' mobile: 手机号（主键）
    ' daily_verify_count: 当日验证尝试总次数（所有验证码累计）
    ' daily_send_count: 当日发送短信总次数
    ' last_verify_time: 最后验证时间
    ' last_reset_date: 最后重置日期（每日0点重置计数）
    ' is_blocked: 是否被封禁
    ' block_until: 封禁截止时间（频繁失败封禁24小时）
    ' created_at: 创建时间
    ' updated_at: 更新时间
    ' 安全策略：
    ' 1. 每日验证尝试上限30次（跨所有验证码）
    ' 2. 每日发送短信上限10次
    ' 3. 验证失败30次封禁24小时
    ' 4. 配合IP限制和设备指纹识别
}

' 🔧 新增：限流配置表（统一限流管理）
class RateLimitConfig {
    + id : Long
    + resource_type : String
    + resource_name : String
    + limit_type : String
    + limit_count : Integer
    + limit_period : Integer
    + status : Integer
    + description : String
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 限流配置表（统一管理API限流规则）
    ' id: 配置ID
    ' resource_type: 资源类型（api/sms/content/message）
    ' resource_name: 资源名称（如：/api/content/publish）
    ' limit_type: 限流类型（ip/user/global）
    ' limit_count: 限制次数
    ' limit_period: 时间窗口（秒）
    ' status: 状态（0=禁用，1=启用）
    ' description: 配置说明
    ' created_at: 创建时间
    ' updated_at: 更新时间
    ' 示例配置：
    ' - 短信发送：limit_type=user, limit_count=10, limit_period=86400 (每用户每天10次)
    ' - 内容发布：limit_type=user, limit_count=20, limit_period=3600 (每用户每小时20条)
    ' - 登录接口：limit_type=ip, limit_count=100, limit_period=60 (每IP每分钟100次)
    ' 实现：配合Redis滑动窗口算法
}

' ===== 内容模块 (13表) =====

class Content {
    + id : Long
    + user_id : Long
    + type : Integer
    + title : String
    + content : String
    + cover_url : String
    + location_name : String
    + location_address : String
    ' 🔧 重构：使用空间索引类型（MySQL 8.0+ POINT SRID 4326）
    + location : Point
    ' ❌ 删除：location_lat : Decimal
    ' ❌ 删除：location_lng : Decimal
    + city_id : Long
    ' ❌ 删除以下冗余统计字段（改用 ContentStats 表 + Redis）
    ' view_count : Integer
    ' like_count : Integer
    ' comment_count : Integer
    ' share_count : Integer
    ' collect_count : Integer
    ' 🔧 新增：冗余必要的用户信息（避免 N+1 查询）
    + user_nickname : String
    + user_avatar : String
    + status : Integer
    + is_top : Boolean
    + is_hot : Boolean
    + publish_time : DateTime
    + deleted_at : DateTime
    + version : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 内容表(动态/活动/技能)
    ' id: 内容唯一ID
    ' user_id: 创建者ID
    ' type: 内容类型(1=动态,2=活动,3=技能服务)
    ' title: 内容标题(动态标题,最大50字)
    ' content: 内容正文(最大1000字)
    ' cover_url: 封面图URL
    ' location_name: 地点名称(场所名称)
    ' location_address: 详细地址
    ' 🔧 location: 地理位置(POINT类型,支持空间索引,SRID 4326)
    ' ❌ 删除 location_lat/location_lng: 改用 POINT 类型提升地理查询性能
    ' → 支持 ST_Distance_Sphere() 高效计算距离
    ' → 支持 SPATIAL INDEX 优化范围查询
    ' city_id: 所在城市ID
    ' ❌ 删除冗余统计字段：view_count/like_count/comment_count/share_count/collect_count
    ' → 改用 ContentStats 表存储，每5分钟从 Redis 批量同步
    ' → 解决高并发点赞/收藏时的锁竞争问题
    ' 🔧 新增 user_nickname/user_avatar: 冗余用户信息，避免列表查询时的 N+1 问题
    ' status: 内容状态(0=草稿,1=已发布,2=审核中,3=已下架,4=已删除)
    ' is_top: 是否置顶
    ' is_hot: 是否热门
    ' publish_time: 发布时间(支持定时发布)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

' 🔧 新增：内容统计表（解决冗余字段问题）
class ContentStats {
    + content_id : Long
    + view_count : Integer
    + like_count : Integer
    + comment_count : Integer
    + share_count : Integer
    + collect_count : Integer
    + last_sync_time : DateTime
    + updated_at : DateTime
    --
    ' 内容统计表（独立存储，配合Redis使用）
    ' content_id: 内容ID（主键）
    ' view_count: 浏览次数（从Redis每5分钟批量同步）
    ' like_count: 点赞数量（从Redis每5分钟批量同步）
    ' comment_count: 评论数量（从Redis每5分钟批量同步）
    ' share_count: 分享数量（从Redis每5分钟批量同步）
    ' collect_count: 收藏数量（从Redis每5分钟批量同步）
    ' last_sync_time: 最后同步时间（用于监控同步延迟）
    ' updated_at: 更新时间
    ' 设计理念：
    ' 1. 读取：优先读Redis，Redis不存在时回源ContentStats表
    ' 2. 写入：只写Redis（INCR操作），异步同步到MySQL
    ' 3. 一致性：定时任务每10分钟修正脏数据
    ' 4. 性能：避免Content表成为热点，解决锁竞争
}

' 🔧 新增：用户统计表（解决UserProfile冗余字段问题）
class UserStats {
    + user_id : Long
    + follower_count : Integer
    + following_count : Integer
    + content_count : Integer
    + total_like_count : Integer
    + total_collect_count : Integer
    + activity_organizer_count : Integer
    + activity_participant_count : Integer
    + activity_success_count : Integer
    + activity_cancel_count : Integer
    + activity_organizer_score : Decimal
    + activity_success_rate : Decimal
    + last_sync_time : DateTime
    + updated_at : DateTime
    --
    ' 用户统计表（独立存储，配合Redis使用）
    ' user_id: 用户ID（主键）
    ' follower_count: 粉丝数量
    ' following_count: 关注数量
    ' content_count: 发布内容数量
    ' total_like_count: 获赞总数
    ' total_collect_count: 被收藏总数
    ' activity_organizer_count: 发起组局总数
    ' activity_participant_count: 参与组局总数
    ' activity_success_count: 成功完成组局次数
    ' activity_cancel_count: 取消组局次数
    ' activity_organizer_score: 组局信誉评分（5分制）
    ' activity_success_rate: 组局成功率（百分比）
    ' last_sync_time: 最后同步时间
    ' updated_at: 更新时间
    ' 异步同步机制：通过消息队列（RabbitMQ/Kafka）实现最终一致性
}

' 🔧 新增：用户职业关联表（替代occupation_tags字符串）
class UserOccupation {
    + id : Long
    + user_id : Long
    + occupation_code : String
    + sort_order : Integer
    + created_at : DateTime
    --
    ' 用户职业关联表（符合第一范式）
    ' id: 记录ID
    ' user_id: 用户ID
    ' occupation_code: 职业编码（关联OccupationDict表）
    ' sort_order: 排序顺序（支持用户自定义显示顺序）
    ' created_at: 创建时间
    ' 设计优势：
    ' 1. 支持高效查询：SELECT * FROM UserOccupation WHERE occupation_code='model'
    ' 2. 数据完整性：外键约束保证数据有效性
    ' 3. 易于扩展：新增职业只需在字典表添加
    ' 4. 多语言支持：字典表存储多语言翻译
}

' 🔧 新增：职业字典表（统一管理枚举值）
class OccupationDict {
    + code : String
    + name : String
    + category : String
    + icon_url : String
    + sort_order : Integer
    + status : Integer
    + created_at : DateTime
    --
    ' 职业字典表（统一管理职业类型）
    ' code: 职业编码（主键，如：model/student/freelancer）
    ' name: 职业名称（如：模特/学生/自由职业）
    ' category: 职业分类（如：艺术/教育/服务）
    ' icon_url: 图标URL（前端展示用）
    ' sort_order: 排序顺序（热门职业优先显示）
    ' status: 状态（0=禁用，1=启用）
    ' created_at: 创建时间
    ' 可扩展：支持多语言（添加OccupationDictI18n表）
}

class ContentAction {
    + id : Long
    + content_id : Long
    + user_id : Long
    + action : Integer
    + status : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 内容行为表(点赞/分享/收藏/浏览)
    ' id: 行为记录ID
    ' content_id: 关联内容ID
    ' user_id: 操作用户ID
    ' action: 行为类型(1=点赞,2=分享,3=收藏,4=浏览)
    ' status: 状态(0=已取消,1=正常)
    ' created_at: 行为时间
    ' updated_at: 更新时间
}

class Comment {
    + id : Long
    + content_id : Long
    + user_id : Long
    + parent_id : Long
    + reply_to_id : Long
    + reply_to_user_id : Long
    + comment_text : String
    + like_count : Integer
    + reply_count : Integer
    + is_top : Boolean
    + status : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 评论表
    ' id: 评论唯一ID
    ' content_id: 关联内容ID
    ' user_id: 评论用户ID
    ' parent_id: 一级评论ID(二级回复时使用)
    ' reply_to_id: 直接回复的评论ID
    ' reply_to_user_id: 被回复的用户ID
    ' comment_text: 评论内容(最大500字)
    ' like_count: 点赞数量
    ' reply_count: 回复数量
    ' is_top: 是否置顶评论
    ' status: 评论状态(0=已删除,1=正常,2=审核中,3=已屏蔽)
    ' created_at: 评论时间
    ' updated_at: 更新时间
}

class CommentLike {
    + id : Long
    + comment_id : Long
    + user_id : Long
    + status : Integer
    + created_at : DateTime
    --
    ' 评论点赞表
    ' id: 点赞记录ID
    ' comment_id: 评论ID
    ' user_id: 点赞用户ID
    ' status: 状态(0=已取消,1=正常)
    ' created_at: 点赞时间
}

class ContentDraft {
    + id : Long
    + user_id : Long
    + type : Integer
    + title : String
    + content : String
    + location_name : String
    + location_address : String
    + location_lat : Decimal
    + location_lng : Decimal
    + city_id : Long
    + auto_save_time : DateTime
    + expire_time : DateTime
    + status : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 内容草稿表
    ' id: 草稿唯一ID
    ' user_id: 创建者ID
    ' type: 内容类型(1=动态,2=活动,3=技能服务)
    ' title: 标题草稿
    ' content: 正文草稿
    ' location_name: 地点名称
    ' location_address: 详细地址
    ' location_lat: 纬度
    ' location_lng: 经度
    ' city_id: 所在城市ID
    ' auto_save_time: 自动保存时间
    ' expire_time: 过期时间(30天后清理)
    ' status: 草稿状态(0=已发布,1=编辑中,2=已过期,3=已删除)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class ContentTopic {
    + id : Long
    + content_id : Long
    + topic_id : Long
    + sort_order : Integer
    + created_at : DateTime
    --
    ' 内容话题关联表
    ' id: 关联记录ID
    ' content_id: 内容ID
    ' topic_id: 话题ID
    ' sort_order: 排序顺序(多个话题时的显示顺序)
    ' created_at: 关联时间
}

class TopicFollow {
    + id : Long
    + user_id : Long
    + topic_id : Long
    + status : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 用户关注话题表
    ' id: 关注记录ID
    ' user_id: 用户ID
    ' topic_id: 话题ID
    ' status: 关注状态(0=已取消,1=正常)
    ' created_at: 关注时间
    ' updated_at: 更新时间
}

class UserRelation {
    + id : Long
    + user_id : Long
    + target_id : Long
    + type : Integer
    + status : Integer
    + created_at : DateTime
    --
    ' 用户关系表
    ' id: 关系记录ID
    ' user_id: 发起用户ID
    ' target_id: 目标用户ID
    ' type: 关系类型(1=关注,2=拉黑,3=好友,4=特别关注)
    ' status: 关系状态(0=已取消,1=正常)
    ' created_at: 建立关系时间
}

' ===== 游戏服务模块 (3表) =====

class GameService {
    + id : Long
    + content_id : Long
    + user_id : Long
    + game_name : String
    + game_type : Integer
    + service_mode : Integer
    + current_rank : String
    + highest_rank : String
    + win_rate : Decimal
    ' ❌ 删除以下统计字段（改用 ServiceStats 表）
    ' service_count : Integer
    ' avg_rating : Decimal
    ' good_rate : Decimal
    ' avg_response_minutes : Integer
    + price_per_game : Long
    + price_per_hour : Long
    + voice_url : String
    + is_online : Boolean
    + status : Integer
    + version : Integer
    + deleted_at : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 游戏陪玩服务表
    ' id: 服务唯一ID
    ' content_id: 关联内容ID
    ' user_id: 陪玩师用户ID
    ' game_name: 游戏名称(王者荣耀/英雄联盟等)
    ' game_type: 游戏类型(1=MOBA,2=FPS,3=休闲等)
    ' service_mode: 服务模式(1=线上,2=线下,3=双模式)
    ' current_rank: 当前段位
    ' highest_rank: 历史最高段位
    ' win_rate: 胜率(百分比)
    ' service_count: 已服务次数
    ' avg_rating: 平均评分(5分制)
    ' good_rate: 好评率(百分比)
    ' avg_response_minutes: 平均响应时间(分钟)
    ' price_per_game: 每局价格(分)
    ' price_per_hour: 每小时价格(分)
    ' voice_url: 语音试听URL
    ' is_online: 是否在线
    ' status: 服务状态(0=暂停,1=可预约,2=忙碌中)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class GameSkill {
    + id : Long
    + game_service_id : Long
    + skill_type : Integer
    + skill_name : String
    + skill_value : String
    + proficiency_level : Integer
    + rank_label : String
    + hero_count : Integer
    + description : String
    + sort_order : Integer
    --
    ' 游戏技能详情表
    ' id: 技能记录ID
    ' game_service_id: 关联游戏服务ID
    ' skill_type: 技能类型(1=位置,2=英雄,3=特长)
    ' skill_name: 技能名称(射手/后羿/上分专家等)
    ' skill_value: 技能值/英雄名
    ' proficiency_level: 熟练度等级(1-5星)
    ' rank_label: 段位标识(国服/省标/市标)
    ' hero_count: 英雄数量
    ' description: 技能描述
    ' sort_order: 排序顺序
}

class GameServiceTag {
    + id : Long
    + game_service_id : Long
    + tag_category : Integer
    + tag_name : String
    + tag_color : String
    + sort_order : Integer
    --
    ' 游戏服务标签表
    ' id: 标签记录ID
    ' game_service_id: 关联游戏服务ID
    ' tag_category: 标签分类(1=服务类型,2=声音特色,3=技术特色,4=性格特色)
    ' tag_name: 标签名称(上分/语音/声音甜美/技术过硬等)
    ' tag_color: 标签颜色(用于前端显示)
    ' sort_order: 排序顺序
}

' ===== 生活服务模块 (3表) =====

class LifeService {
    + id : Long
    + content_id : Long
    + user_id : Long
    + service_type : Integer
    + service_name : String
    + service_mode : Integer
    + location_address : String
    ' 🔧 重构：使用空间索引
    + location : Point
    ' ❌ 删除：location_lat, location_lng, distance_km
    + service_area : String
    + service_duration : Integer
    ' ❌ 删除：available_dates/available_time_slots（改用关联表）
    + participant_limit : Integer
    + price_per_person : Long
    + price_per_time : Long
    ' ❌ 删除以下统计字段（改用 ServiceStats 表）
    ' service_count : Integer
    ' avg_rating : Decimal
    ' good_rate : Decimal
    ' avg_response_minutes : Integer
    ' ❌ 删除：cover_images（改用 Media 表关联）
    + is_merchant_verified : Boolean
    + status : Integer
    + version : Integer
    + deleted_at : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 生活服务表
    ' id: 服务唯一ID
    ' content_id: 关联内容ID
    ' user_id: 服务商用户ID
    ' service_type: 服务类型(1=探店,2=私影,3=台球,4=K歌,5=按摩,6=健身)
    ' service_name: 服务名称
    ' service_mode: 服务模式(1=线上,2=线下,3=上门)
    ' location_address: 服务地点地址
    ' location_lat: 纬度
    ' location_lng: 经度
    ' distance_km: 距离(公里)
    ' service_area: 服务区域
    ' service_duration: 服务时长(分钟)
    ' available_dates: 可预约日期(逗号分隔)
    ' available_time_slots: 可预约时段(逗号分隔)
    ' participant_limit: 参与人数限制
    ' price_per_person: 每人价格(分)
    ' price_per_time: 每次价格(分)
    ' service_count: 已服务次数
    ' avg_rating: 平均评分(5分制)
    ' good_rate: 好评率(百分比)
    ' avg_response_minutes: 平均响应时间(分钟)
    ' cover_images: 封面图片URLs(逗号分隔)
    ' is_merchant_verified: 是否商家认证
    ' status: 服务状态(0=暂停,1=可预约,2=忙碌中)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class LifeServiceFeature {
    + id : Long
    + life_service_id : Long
    + feature_category : Integer
    + feature_name : String
    + feature_value : String
    + feature_color : String
    + sort_order : Integer
    --
    ' 生活服务特色表
    ' id: 特色记录ID
    ' life_service_id: 关联生活服务ID
    ' feature_category: 特色分类(1=美食类型,2=价格优势,3=环境特色,4=交通便利)
    ' feature_name: 特色名称
    ' feature_value: 特色值
    ' feature_color: 特色颜色
    ' sort_order: 排序顺序
}

class LifeServiceTag {
    + id : Long
    + life_service_id : Long
    + tag_category : Integer
    + tag_name : String
    + tag_color : String
    + sort_order : Integer
    --
    ' 生活服务标签表
    ' id: 标签记录ID
    ' life_service_id: 关联生活服务ID
    ' tag_category: 标签分类(1=服务类型,2=环境特色,3=价格特色)
    ' tag_name: 标签名称
    ' tag_color: 标签颜色
    ' sort_order: 排序顺序
}

' 🔧 新增：服务统计表（统一管理游戏服务和生活服务的统计数据）
class ServiceStats {
    + service_id : Long
    + service_type : Integer
    + service_count : Integer
    + avg_rating : Decimal
    + good_rate : Decimal
    + avg_response_minutes : Integer
    + total_revenue : Long
    + last_sync_time : DateTime
    + updated_at : DateTime
    --
    ' 服务统计表（GameService 和 LifeService 共用）
    ' service_id: 服务ID（GameService.id 或 LifeService.id）
    ' service_type: 服务类型（1=游戏服务，2=生活服务）
    ' service_count: 已服务次数
    ' avg_rating: 平均评分（5分制）
    ' good_rate: 好评率（百分比）
    ' avg_response_minutes: 平均响应时间（分钟）
    ' total_revenue: 累计收入（分）
    ' last_sync_time: 最后同步时间
    ' updated_at: 更新时间
    ' 异步同步：订单完成后通过消息队列异步更新
}

' ===== 订单评价模块 (2表) =====

class ServiceOrder {
    + id : Long
    + order_no : String
    + buyer_id : Long
    + seller_id : Long
    + content_id : Long
    + service_type : Integer
    + service_name : String
    + service_time : DateTime
    + service_duration : Integer
    + participant_count : Integer
    + amount : Long
    + base_fee : Long
    + person_fee : Long
    + platform_fee : Long
    + discount_amount : Long
    + actual_amount : Long
    + status : Integer
    + contact_name : String
    + contact_phone : String
    + special_request : String
    + payment_method : String
    + payment_time : DateTime
    + cancel_reason : String
    + cancel_time : DateTime
    + created_at : DateTime
    + completed_at : DateTime
    --
    ' 服务订单表
    ' id: 订单唯一ID
    ' order_no: 订单编号
    ' buyer_id: 买家用户ID
    ' seller_id: 卖家用户ID
    ' content_id: 关联内容ID
    ' service_type: 服务类型(1=游戏陪玩,2=生活服务,3=活动报名)
    ' service_name: 服务名称
    ' service_time: 服务时间
    ' service_duration: 服务时长(分钟)
    ' participant_count: 参与人数
    ' amount: 订单总金额(分)
    ' base_fee: 基础服务费(分)
    ' person_fee: 人数费用(分)
    ' platform_fee: 平台服务费(分)
    ' discount_amount: 优惠金额(分)
    ' actual_amount: 实际支付金额(分)
    ' status: 订单状态(0=待付款,1=已付款,2=服务中,3=已完成,4=已取消,5=已退款)
    ' contact_name: 联系人姓名
    ' contact_phone: 联系电话
    ' special_request: 特殊需求
    ' payment_method: 支付方式
    ' payment_time: 支付时间
    ' cancel_reason: 取消原因
    ' cancel_time: 取消时间
    ' created_at: 下单时间
    ' completed_at: 完成时间
}

class ServiceReview {
    + id : Long
    + order_id : Long
    + content_id : Long
    + reviewer_id : Long
    + reviewee_id : Long
    + service_type : Integer
    + rating_overall : Decimal
    + rating_service : Decimal
    + rating_attitude : Decimal
    + rating_quality : Decimal
    + review_text : String
    + review_images : String
    + is_anonymous : Boolean
    + like_count : Integer
    + reply_text : String
    + reply_time : DateTime
    + status : Integer
    + created_at : DateTime
    --
    ' 服务评价表
    ' id: 评价记录ID
    ' order_id: 关联订单ID
    ' content_id: 关联内容ID
    ' reviewer_id: 评价人ID
    ' reviewee_id: 被评价人ID
    ' service_type: 服务类型
    ' rating_overall: 综合评分(5分制)
    ' rating_service: 服务评分
    ' rating_attitude: 态度评分
    ' rating_quality: 质量评分
    ' review_text: 评价内容
    ' review_images: 评价图片URLs(逗号分隔)
    ' is_anonymous: 是否匿名评价
    ' like_count: 点赞数量
    ' reply_text: 回复内容
    ' reply_time: 回复时间
    ' status: 评价状态(0=待审核,1=已发布,2=已隐藏)
    ' created_at: 评价时间
}

' ===== 话题标签模块 (4表) =====

class Topic {
    + id : Long
    + name : String
    + description : String
    + cover_url : String
    + creator_id : Long
    + category : Integer
    ' ❌ 删除以下冗余统计字段（改用 TopicStats 表 + Redis）
    ' participant_count : Integer
    ' post_count : Integer
    ' view_count : Integer
    ' like_count : Integer
    ' comment_count : Integer
    ' share_count : Integer
    ' follow_count : Integer
    ' heat_score : Integer
    ' trend_score : Decimal
    ' today_post_count : Integer
    ' week_post_count : Integer
    ' month_post_count : Integer
    + last_post_time : DateTime
    + is_hot : Boolean
    + is_trending : Boolean
    + status : Integer
    + deleted_at : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 话题标签表
    ' id: 话题唯一ID
    ' name: 话题名称(唯一,如#王者荣耀#)
    ' description: 话题描述
    ' cover_url: 话题封面图
    ' creator_id: 创建者ID
    ' category: 话题分类(1=游戏,2=娱乐,3=生活,4=美食,5=运动,6=其他)
    ' participant_count: 参与人数
    ' post_count: 动态总数量
    ' view_count: 总浏览次数
    ' like_count: 总点赞数量
    ' comment_count: 总评论数量
    ' share_count: 总分享数量
    ' follow_count: 关注人数
    ' heat_score: 热度分数(用于排序)
    ' trend_score: 趋势分数(热度增长率)
    ' today_post_count: 今日动态数
    ' week_post_count: 本周动态数
    ' month_post_count: 本月动态数
    ' last_post_time: 最后发布动态时间
    ' is_hot: 是否热门话题
    ' is_trending: 是否趋势话题
    ' status: 话题状态(0=禁用,1=正常,2=热门推荐,3=官方推荐)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

' 🔧 新增：话题统计表（解决Topic表冗余字段问题）
class TopicStats {
    + topic_id : Long
    + participant_count : Integer
    + post_count : Integer
    + view_count : Integer
    + like_count : Integer
    + comment_count : Integer
    + share_count : Integer
    + follow_count : Integer
    + heat_score : Integer
    + trend_score : Decimal
    + today_post_count : Integer
    + week_post_count : Integer
    + month_post_count : Integer
    + last_sync_time : DateTime
    + updated_at : DateTime
    --
    ' 话题统计表（独立存储，配合Redis使用）
    ' topic_id: 话题ID（主键）
    ' participant_count: 参与人数
    ' post_count: 动态总数量
    ' view_count: 总浏览次数
    ' like_count: 总点赞数量
    ' comment_count: 总评论数量
    ' share_count: 总分享数量
    ' follow_count: 关注人数
    ' heat_score: 热度分数（用于排序，每5分钟计算一次）
    ' trend_score: 趋势分数（热度增长率）
    ' today_post_count: 今日动态数（每日0点重置）
    ' week_post_count: 本周动态数（每周一重置）
    ' month_post_count: 本月动态数（每月1号重置）
    ' last_sync_time: 最后同步时间
    ' updated_at: 更新时间
    ' 热度计算公式：
    ' heat_score = (today_post_count * 10 + week_post_count * 2 + month_post_count)
    '            * (1 + log10(follow_count + 1))
    '            * time_decay_factor
    ' Redis存储：ZADD hot_topics:score <heat_score> <topic_id>
}

class UserTag {
    + id : Long
    + user_id : Long
    + tag_type : Integer
    + tag_value : String
    + level : Integer
    + verified : Boolean
    + score : Integer
    + description : String
    + expire_time : DateTime
    + created_at : DateTime
    --
    ' 用户标签表
    ' id: 标签记录ID
    ' user_id: 用户ID
    ' tag_type: 标签类型(1=技能,2=职业,3=认证,4=兴趣,5=游戏段位)
    ' tag_value: 标签值(如"王者荣耀","模特","大神认证")
    ' level: 标签等级(用于技能等级、游戏段位等)
    ' verified: 是否认证
    ' score: 标签分数
    ' description: 标签描述
    ' expire_time: 过期时间
    ' created_at: 创建时间
}

' ===== 媒体文件模块 (1表) =====

class Media {
    + id : Long
    + uploader_id : Long
    + ref_type : String
    + ref_id : Long
    + file_type : Integer
    + file_url : String
    + thumbnail_url : String
    + width : Integer
    + height : Integer
    + duration : Integer
    + file_size : Long
    + file_format : String
    + status : Integer
    + created_at : DateTime
    --
    ' 媒体文件表
    ' id: 文件唯一ID
    ' uploader_id: 上传者ID
    ' ref_type: 关联类型(content/review/message/profile)
    ' ref_id: 关联对象ID
    ' file_type: 文件类型(1=图片,2=视频,3=音频)
    ' file_url: 文件访问URL(CDN地址)
    ' thumbnail_url: 缩略图URL
    ' width: 图片/视频宽度
    ' height: 图片/视频高度
    ' duration: 视频/音频时长(秒)
    ' file_size: 文件大小(字节)
    ' file_format: 文件格式(jpg/mp4/mp3等)
    ' status: 文件状态(0=已删除,1=正常,2=审核中)
    ' created_at: 上传时间
}

' ===== 通知推送模块 (1表) =====

class Notification {
    + id : Long
    + user_id : Long
    + category : Integer
    + type : Integer
    + priority : Integer
    + title : String
    + content : String
    + sender_id : Long
    + ref_type : String
    + ref_id : Long
    + action_type : String
    + action_url : String
    + action_button_text : String
    + secondary_action_text : String
    + is_read : Boolean
    + is_handled : Boolean
    + expire_time : DateTime
    + created_at : DateTime
    + read_at : DateTime
    + handled_at : DateTime
    --
    ' 通知消息表(支持多种分类和优先级)
    ' id: 通知唯一ID
    ' user_id: 接收用户ID
    ' category: 通知分类(1=点赞收藏,2=评论,3=粉丝关注,4=系统通知,5=订单,6=活动)
    ' type: 通知类型(细分类型,如点赞分为:作品点赞/评论点赞/动态点赞)
    ' priority: 优先级(1=紧急,2=重要,3=普通,4=一般)
    ' title: 通知标题(系统公告/功能更新/安全提醒等)
    ' content: 通知内容(支持关键词高亮,最多500字)
    ' sender_id: 发送者ID(用户操作的通知,系统消息为NULL)
    ' ref_type: 关联类型(content/order/comment/user/activity)
    ' ref_id: 关联对象ID(用于跳转定位)
    ' action_type: 操作类型(profile_edit/security_check/view_detail等)
    ' action_url: 跳转URL(深度链接)
    ' action_button_text: 主要操作按钮文字(立即完善/查看详情/立即处理)
    ' secondary_action_text: 次要操作按钮文字(忽略/稍后提醒/不再提醒)
    ' is_read: 是否已读(0=未读,1=已读)
    ' is_handled: 是否已处理(针对需要操作的通知)
    ' expire_time: 过期时间(过期后自动隐藏或清理)
    ' created_at: 通知时间
    ' read_at: 已读时间
    ' handled_at: 处理时间
}

' ===== 安全审核模块 (1表) =====

class Report {
    + id : Long
    + reporter_id : Long
    + target_id : Long
    + target_type : Integer
    + reason : Integer
    + description : String
    + evidence_images : String
    + evidence_urls : String
    + status : Integer
    + handler_id : Long
    + handle_result : String
    + created_at : DateTime
    + handled_at : DateTime
    --
    ' 举报记录表
    ' id: 举报记录ID
    ' reporter_id: 举报人ID
    ' target_id: 被举报目标ID
    ' target_type: 目标类型(1=用户,2=内容,3=评论)
    ' reason: 举报原因(1=垃圾广告,2=不当内容,3=侵权,4=诈骗,5=骚扰,6=其他)
    ' description: 详细说明
    ' evidence_images: 证据图片URLs(逗号分隔)
    ' evidence_urls: 证据链接(逗号分隔)
    ' status: 处理状态(0=待处理,1=处理中,2=已处理,3=已驳回)
    ' handler_id: 处理人ID
    ' handle_result: 处理结果说明
    ' created_at: 举报时间
    ' handled_at: 处理时间
}

' ===== 搜索历史模块 (1表) =====

class SearchHistory {
    + id : Long
    + user_id : Long
    + keyword : String
    + search_type : Integer
    + result_count : Integer
    + created_at : DateTime
    --
    ' 搜索历史表
    ' id: 搜索记录ID
    ' user_id: 搜索用户ID
    ' keyword: 搜索关键词
    ' search_type: 搜索类型(1=用户,2=内容,3=话题,4=服务)
    ' result_count: 搜索结果数量
    ' created_at: 搜索时间
}

' ===== 聊天模块 (5表) =====

class ChatConversation {
    + id : Long
    + type : Integer
    + title : String
    + creator_id : Long
    + avatar_url : String
    + description : String
    + order_id : Long
    ' 🔧 保留：最后消息ID和时间（提升列表查询性能）
    + last_message_id : Long
    + last_message_time : DateTime
    ' ❌ 删除：last_message_type, last_message_content, last_message_sender_id
    ' → 原因：这些字段过度冗余，通过 last_message_id 关联查询即可
    ' → 避免：消息更新时需要同时更新2张表（一致性问题）
    + total_message_count : Integer
    + member_count : Integer
    ' ❌ 移除：is_pinned, is_muted（改为在 ChatParticipant 表存储）
    ' → 原因：置顶/免打扰是用户个性化设置，不同用户设置不同
    + status : Integer
    + deleted_at : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 聊天会话表(支持置顶/免打扰/最后消息预览)
    ' id: 会话唯一ID
    ' type: 会话类型(1=私聊,2=群聊,3=系统通知,4=订单会话)
    ' title: 会话标题(群聊名称,私聊可为空)
    ' creator_id: 创建者ID(群主/发起人)
    ' avatar_url: 会话头像(群聊头像或对方用户头像)
    ' description: 会话描述(群聊公告等)
    ' order_id: 关联订单ID(订单会话使用)
    ' 🔧 last_message_id: 最后一条消息ID（冗余保留，提升列表查询性能）
    ' 🔧 last_message_time: 最后消息时间（用于对话列表排序）
    ' ❌ 删除过度冗余字段：last_message_type/content/sender_id
    ' → 通过 last_message_id 关联 ChatMessage 表查询即可
    ' → 减少数据不一致风险
    ' total_message_count: 消息总数（冗余统计）
    ' member_count: 成员数量（群聊使用）
    ' ❌ 删除 is_pinned/is_muted：改为在 ChatParticipant 表存储
    ' → 不同用户的个性化设置应该独立存储
    ' status: 会话状态(0=已解散,1=正常,2=已归档)
    ' created_at: 创建时间
    ' updated_at: 最后活跃时间
}

class ChatMessage {
    + id : Long
    + conversation_id : Long
    + sender_id : Long
    + message_type : Integer
    + content : String
    + media_url : String
    + thumbnail_url : String
    + media_size : Long
    + media_width : Integer
    + media_height : Integer
    + media_duration : Integer
    + media_caption : String
    + reply_to_id : Long
    + client_id : String
    + sequence_id : Long
    + delivery_status : Integer
    + read_count : Integer
    + like_count : Integer
    + is_recalled : Boolean
    + recall_time : DateTime
    + recalled_by : Long
    + status : Integer
    + send_time : DateTime
    + server_time : DateTime
    + deleted_at : DateTime
    + created_at : DateTime
    --
    ' 聊天消息表(支持多媒体/已读回执/点赞/撤回)
    ' id: 消息唯一ID(服务端生成)
    ' conversation_id: 所属会话ID
    ' sender_id: 发送者ID(NULL=系统消息)
    ' message_type: 消息类型(1=文本,2=图片,3=语音,4=视频,5=文件,6=系统通知,7=订单卡片)
    ' content: 消息内容(文字内容或媒体描述,最大5000字符)
    ' media_url: 媒体文件URL(CDN地址,图片/音频/视频)
    ' thumbnail_url: 缩略图URL(图片缩略图或视频封面)
    ' media_size: 媒体文件大小(字节)
    ' media_width: 媒体宽度(像素,图片/视频使用)
    ' media_height: 媒体高度(像素,图片/视频使用)
    ' media_duration: 媒体时长(秒,音频/视频使用,语音最长60s,视频最长5分钟)
    ' media_caption: 媒体配文(图片/视频的文字说明)
    ' reply_to_id: 回复的消息ID(引用回复功能)
    ' client_id: 客户端消息ID(用于消息去重和状态追踪)
    ' sequence_id: 消息序列号(保证消息有序性)
    ' delivery_status: 投递状态(0=发送中,1=已发送,2=已送达,3=已读,4=发送失败)
    ' read_count: 已读人数(群聊使用)
    ' like_count: 点赞数量(图片消息支持点赞)
    ' is_recalled: 是否已撤回(2分钟内可撤回)
    ' recall_time: 撤回时间
    ' 🔧 recalled_by: 撤回操作人ID（新增，群聊场景需要知道谁撤回的）
    ' status: 消息状态(0=已删除,1=正常,2=已撤回,3=审核中)
    ' 🔧 消息撤回改进：撤回后清空 content 和 media_url 字段（隐私保护）
    ' 🔧 表分片策略（重要）：
    ' → 按 conversation_id 哈希分256张表：chat_message_000 ~ chat_message_255
    ' → 分表公式：table_suffix = conversation_id % 256
    ' → 优势1：单表数据量可控（每表最多存储活跃会话的历史消息）
    ' → 优势2：查询性能稳定（会话消息天然隔离）
    ' → 优势3：归档方便（按表归档到对象存储）
    ' 🔧 消息归档策略：
    ' → 热数据（30天内）：存储在MySQL分片表
    ' → 冷数据（30天以上）：归档到对象存储（S3/OSS），Parquet格式
    ' → 归档后保留索引记录，指向冷存储位置
    ' send_time: 客户端发送时间
    ' server_time: 服务器接收时间
    ' created_at: 创建时间
}

class ChatParticipant {
    + id : Long
    + conversation_id : Long
    + user_id : Long
    + role : Integer
    + join_time : DateTime
    + last_read_message_id : Long
    + last_read_time : DateTime
    + unread_count : Integer
    + is_pinned : Boolean
    + is_muted : Boolean
    + mute_until : DateTime
    + nickname : String
    + status : Integer
    + leave_time : DateTime
    --
    ' 会话参与者表(支持个性化设置/已读管理)
    ' id: 参与记录ID
    ' conversation_id: 会话ID
    ' user_id: 参与用户ID
    ' role: 角色权限(1=成员,2=管理员,3=群主)
    ' join_time: 加入时间
    ' last_read_message_id: 最后已读消息ID(用于精确定位已读位置)
    ' last_read_time: 最后已读时间(用于计算未读数)
    ' unread_count: 未读消息数(冗余字段,实时更新)
    ' is_pinned: 是否置顶此会话(用户个性化设置)
    ' is_muted: 是否免打扰(不接收推送通知)
    ' mute_until: 免打扰截止时间(NULL=永久免打扰)
    ' nickname: 群聊中的昵称(群昵称,可选)
    ' status: 参与状态(0=已退出,1=正常,2=已禁言)
    ' leave_time: 退出时间
}

class MessageSettings {
    + id : Long
    + user_id : Long
    + push_enabled : Boolean
    + push_sound_enabled : Boolean
    + push_vibrate_enabled : Boolean
    + push_preview_enabled : Boolean
    + push_start_time : String
    + push_end_time : String
    + push_like_enabled : Boolean
    + push_comment_enabled : Boolean
    + push_follow_enabled : Boolean
    + push_system_enabled : Boolean
    + who_can_message : Integer
    + who_can_add_friend : Integer
    + message_read_receipt : Boolean
    + online_status_visible : Boolean
    + auto_download_image : Boolean
    + auto_download_video : Boolean
    + auto_play_voice : Boolean
    + message_retention_days : Integer
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 消息设置表(用户个性化消息偏好)
    ' id: 设置记录ID
    ' user_id: 用户ID(唯一)
    ' push_enabled: 是否开启消息推送(总开关)
    ' push_sound_enabled: 推送声音开关
    ' push_vibrate_enabled: 推送震动开关
    ' push_preview_enabled: 推送内容预览开关(关闭则只显示"您有新消息")
    ' push_start_time: 推送时段开始时间(如"08:00")
    ' push_end_time: 推送时段结束时间(如"22:00")
    ' push_like_enabled: 点赞消息推送开关
    ' push_comment_enabled: 评论消息推送开关
    ' push_follow_enabled: 关注消息推送开关
    ' push_system_enabled: 系统通知推送开关
    ' who_can_message: 谁可以给我发消息(0=所有人,1=我关注的人,2=互相关注,3=不允许)
    ' who_can_add_friend: 谁可以添加我为好友(0=所有人,1=需要验证,2=不允许)
    ' message_read_receipt: 消息已读回执开关(关闭后不发送已读状态)
    ' online_status_visible: 在线状态可见(关闭则显示为隐身)
    ' auto_download_image: 自动下载图片(0=永不,1=仅WIFI,2=始终)
    ' auto_download_video: 自动下载视频(0=永不,1=仅WIFI,2=始终)
    ' auto_play_voice: 自动播放语音消息
    ' message_retention_days: 消息保存天数(0=永久,7/30/90天自动清理)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class TypingStatus {
    + id : Long
    + conversation_id : Long
    + user_id : Long
    + is_typing : Boolean
    + start_time : DateTime
    + last_update_time : DateTime
    + expire_time : DateTime
    --
    ' 输入状态表(正在输入...状态管理)
    ' id: 状态记录ID
    ' conversation_id: 对话ID
    ' user_id: 正在输入的用户ID
    ' is_typing: 是否正在输入(0=停止,1=输入中)
    ' start_time: 开始输入时间
    ' last_update_time: 最后更新时间(心跳更新)
    ' expire_time: 过期时间(10秒无更新自动清除)
}

' ===== 首页功能新增模块 (4表) =====

class UserPreference {
    + id : Long
    + user_id : Long
    + preference_type : Integer
    + game_types : String
    + service_types : String
    + price_min : Long
    + price_max : Long
    + distance_km : Integer
    + gender_filter : Integer
    + online_only : Boolean
    + rating_min : Decimal
    + sort_by : String
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 用户偏好设置表
    ' id: 偏好记录ID
    ' user_id: 用户ID
    ' preference_type: 偏好类型(1=线上筛选,2=线下筛选,3=通知偏好,4=隐私设置)
    ' game_types: 游戏类型(逗号分隔)
    ' service_types: 服务类型(逗号分隔)
    ' price_min: 最低价格(分)
    ' price_max: 最高价格(分)
    ' distance_km: 最大距离(公里)
    ' gender_filter: 性别筛选(0=不限,1=仅男,2=仅女)
    ' online_only: 仅显示在线(0=否,1=是)
    ' rating_min: 最低评分
    ' sort_by: 排序方式(recommend/distance/price/rating)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class UserBehavior {
    + id : Long
    + user_id : Long
    + behavior_type : Integer
    + target_type : Integer
    + target_id : Long
    + duration : Integer
    + source_page : String
    + referrer : String
    + created_at : DateTime
    --
    ' 用户行为追踪表
    ' id: 行为记录ID
    ' user_id: 用户ID
    ' behavior_type: 行为类型(1=浏览,2=点击,3=搜索,4=筛选,5=停留,6=分享,7=收藏)
    ' target_type: 目标类型(1=用户详情,2=服务内容,3=活动,4=搜索结果)
    ' target_id: 目标对象ID
    ' duration: 停留时长(秒)
    ' source_page: 来源页面
    ' referrer: 引荐来源
    ' created_at: 行为时间
}

class HotSearch {
    + id : Long
    + keyword : String
    + search_count : Integer
    + click_count : Integer
    + conversion_rate : Decimal
    + category : Integer
    + heat_score : Integer
    + status : Integer
    + updated_at : DateTime
    --
    ' 热门搜索表
    ' id: 热搜记录ID
    ' keyword: 搜索关键词
    ' search_count: 搜索次数
    ' click_count: 点击次数
    ' conversion_rate: 转化率
    ' category: 分类(1=用户,2=服务,3=游戏,4=活动)
    ' heat_score: 热度分数(用于排序)
    ' status: 状态(0=下线,1=正常,2=推荐)
    ' updated_at: 更新时间
}

class City {
    + id : Long
    + name : String
    + province : String
    + level : Integer
    + user_count : Integer
    + hot_score : Integer
    + pinyin : String
    + short_pinyin : String
    + status : Integer
    --
    ' 城市数据表
    ' id: 城市ID
    ' name: 城市名称
    ' province: 所属省份
    ' level: 城市级别(1=一线,2=新一线,3=二线,4=三线,5=其他)
    ' user_count: 用户数量
    ' hot_score: 热度分数(用于热门城市排序)
    ' pinyin: 全拼(用于搜索)
    ' short_pinyin: 首字母(用于快速索引)
    ' status: 状态(0=禁用,1=正常)
}

' ===== 活动组局模块 (3表) =====

class Activity {
    + id : Long
    + content_id : Long
    + organizer_id : Long
    + activity_type : Integer
    + activity_name : String
    + activity_desc : String
    + location_name : String
    + location_address : String
    ' 🔧 重构：使用空间索引
    + location : Point
    ' ❌ 删除：location_lat, location_lng
    + start_time : DateTime
    + end_time : DateTime
    + duration_hours : Integer
    + participant_limit : Integer
    + participant_min : Integer
    + current_participants : Integer
    + fee_type : Integer
    + price_per_person : Long
    + fee_detail : String
    + discount_info : String
    + gender_requirement : Integer
    + age_min : Integer
    + age_max : Integer
    + skill_requirement : String
    + other_requirement : String
    ' ❌ 删除：cover_images : String（改用 Media 表关联）
    ' ❌ 删除以下冗余统计字段（改用 ContentStats 表）
    ' view_count : Integer
    ' like_count : Integer
    ' comment_count : Integer
    ' share_count : Integer
    ' collect_count : Integer
    + status : Integer
    + version : Integer
    + deleted_at : DateTime
    + created_at : DateTime
    + updated_at : DateTime
    --
    ' 活动组局表
    ' id: 活动唯一ID
    ' content_id: 关联内容ID
    ' organizer_id: 组织者ID
    ' activity_type: 活动类型(1=K歌聚会,2=台球聚会,3=私影聚会,4=探店聚会,5=按摩聚会,6=健身聚会,7=游戏聚会,8=其他)
    ' activity_name: 活动名称
    ' activity_desc: 活动描述
    ' location_name: 场所名称
    ' location_address: 活动地点详细地址
    ' location_lat: 纬度
    ' location_lng: 经度
    ' start_time: 开始时间
    ' end_time: 结束时间
    ' duration_hours: 预计时长(小时)
    ' participant_limit: 最多参与人数
    ' participant_min: 最少参与人数
    ' current_participants: 当前参与人数
    ' fee_type: 费用类型(0=免费,1=AA制,2=发起人请客)
    ' price_per_person: 每人费用(分)
    ' fee_detail: 费用明细说明
    ' discount_info: 优惠信息
    ' gender_requirement: 性别要求(0=不限,1=仅男,2=仅女,3=男女混合)
    ' age_min: 最小年龄
    ' age_max: 最大年龄
    ' skill_requirement: 技能要求(不限/初级/中级/高级/专业)
    ' other_requirement: 其他补充要求
    ' cover_images: 封面图片URLs(逗号分隔)
    ' view_count: 浏览次数
    ' like_count: 点赞数量
    ' comment_count: 评论数量
    ' share_count: 分享数量
    ' collect_count: 收藏数量
    ' status: 活动状态(0=草稿,1=报名中,2=已满员,3=进行中,4=已结束,5=已取消)
    ' created_at: 创建时间
    ' updated_at: 更新时间
}

class ActivityParticipant {
    + id : Long
    + activity_id : Long
    + user_id : Long
    + participant_type : Integer
    + contact_name : String
    + contact_phone : String
    + gender : Integer
    + join_message : String
    + pay_status : Integer
    + payment_amount : Long
    + discount_amount : Long
    + actual_amount : Long
    + payment_method : String
    + order_id : Long
    + status : Integer
    + joined_at : DateTime
    + payment_time : DateTime
    + confirmed_at : DateTime
    --
    ' 活动参与者表
    ' id: 参与记录ID
    ' activity_id: 活动ID
    ' user_id: 参与者ID
    ' participant_type: 参与类型(1=报名待审核,2=已通过,3=组织者)
    ' contact_name: 联系人姓名
    ' contact_phone: 联系电话
    ' gender: 性别(1=男,2=女)
    ' join_message: 报名备注留言
    ' pay_status: 支付状态(0=待支付,1=已支付,2=已退款)
    ' payment_amount: 应付金额(分)
    ' discount_amount: 优惠减免金额(分)
    ' actual_amount: 实际支付金额(分)
    ' payment_method: 支付方式(wechat/alipay/balance/bankcard)
    ' order_id: 关联订单ID
    ' status: 参与状态(0=已退出,1=正常,2=已取消)
    ' joined_at: 报名时间
    ' payment_time: 支付时间
    ' confirmed_at: 确认通过时间
}

class ActivityTag {
    + id : Long
    + activity_id : Long
    + tag_name : String
    + tag_color : String
    + tag_type : Integer
    + sort_order : Integer
    --
    ' 活动标签表
    ' id: 标签记录ID
    ' activity_id: 关联活动ID
    ' tag_name: 标签名称(新手友好/高端聚会/定期组局等)
    ' tag_color: 标签颜色(用于前端显示)
    ' tag_type: 标签类型(1=系统推荐,2=用户自定义)
    ' sort_order: 排序顺序
}

' ==========================================
' 🔗 UML关系定义
' ==========================================

' ===== 用户核心组合关系 =====
User "1" *-- "1" UserProfile : "拥有资料\n生命周期绑定"
User "1" *-- "1" UserWallet : "拥有钱包\n强依赖关系"
User "1" o-- "0..*" Transaction : "产生交易\n历史记录保留"
' 🔧 新增：统计表关系
UserProfile "1" -- "0..1" UserStats : "统计数据\n异步同步"
User "1" o-- "0..*" UserOccupation : "职业标签\n规范化设计"
UserOccupation "*" -- "1" OccupationDict : "职业字典\n统一管理"

' ===== 登录认证模块关系 =====
User "1" o-- "0..*" LoginSession : "用户会话\n多设备登录"
User "1" o-- "0..*" UserDevice : "绑定设备\n设备管理"
User "1" o-- "0..*" LoginLog : "登录日志\n行为审计"
User "1" o-- "0..*" SecurityLog : "安全日志\n操作审计"
User "1" -- "0..*" PasswordResetSession : "密码重置\n会话管理"
SmsVerification "1" -- "0..1" PasswordResetSession : "验证码关联\n重置流程"
LoginSession "1" -- "1" UserDevice : "会话设备\n设备绑定"
' 🔧 新增：安全增强表关系
LoginSession "1" -- "0..1" TokenBlacklist : "令牌撤销\n黑名单管理"
SmsVerification "*" -- "0..1" PhoneVerifyLimit : "验证限制\n防穷举"
User "1" o-- "0..*" TokenBlacklist : "令牌黑名单\n安全管理"

' ===== 用户偏好和行为关系 =====
User "1" o-- "0..*" UserPreference : "用户偏好\n设置管理"
User "1" o-- "0..*" UserBehavior : "行为追踪\n数据分析"
User "1" o-- "0..*" SearchHistory : "搜索记录\n历史追踪"

' ===== 用户内容关系 =====
User "1" o-- "0..*" Content : "创建内容\n可匿名化保存"
User "1" o-- "0..*" ContentDraft : "创建草稿\n自动保存"
User "1" -- "0..*" ContentAction : "用户操作\n行为追踪"
Content "1" o-- "0..*" ContentAction : "内容互动\n行为记录"
' 🔧 新增：内容统计表关系
Content "1" -- "0..1" ContentStats : "统计数据\nRedis+异步"

' ===== 评论系统关系 =====
Content "1" o-- "0..*" Comment : "内容评论\n多级评论"
User "评论者" -- "0..*" Comment : "发表评论\n评论创建"
Comment "父评论" -- "0..*" Comment : "子评论回复\n评论层级"
Comment "1" o-- "0..*" CommentLike : "评论点赞\n互动统计"
User "1" -- "0..*" CommentLike : "点赞评论\n用户互动"

' ===== 用户标签和认证关系 =====
User "1" o-- "0..*" UserTag : "用户标签\n多维标签"
User "1" -- "0..*" Topic : "关注话题\n兴趣标签"

' ===== 用户关系自关联 =====
User "发起者" -- "0..*" UserRelation : "主动建立\n关系发起"
User "目标者" -- "0..*" UserRelation : "被动接受\n关系目标"

' ===== 游戏服务关系 =====
Content "1" -- "0..1" GameService : "游戏服务内容\n类型为3"
User "陪玩师" -- "0..*" GameService : "提供游戏服务\n服务者角色"
GameService "1" *-- "0..*" GameSkill : "游戏技能\n技能详情"
GameService "1" *-- "0..*" GameServiceTag : "服务标签\n特色标签"
' 🔧 新增：服务统计表关系
GameService "1" -- "0..1" ServiceStats : "统计数据\n异步同步"

' ===== 生活服务关系 =====
Content "1" -- "0..1" LifeService : "生活服务内容\n类型为2"
User "服务商" -- "0..*" LifeService : "提供生活服务\n服务商角色"
LifeService "1" *-- "0..*" LifeServiceFeature : "服务特色\n特色详情"
LifeService "1" *-- "0..*" LifeServiceTag : "服务标签\n特色标签"
' 🔧 新增：服务统计表关系
LifeService "1" -- "0..1" ServiceStats : "统计数据\n异步同步"

' ===== 活动组局关系 =====
Content "1" -- "0..1" Activity : "活动内容\n类型为2"
User "组织者" -- "0..*" Activity : "组织活动\n发起者角色"
Activity "1" *-- "0..*" ActivityParticipant : "活动参与者\n报名记录"
Activity "1" *-- "0..*" ActivityTag : "活动标签\n特色标签"
User "参与者" -- "0..*" ActivityParticipant : "参与活动\n参与者角色"
ServiceOrder "活动订单" -- "0..1" ActivityParticipant : "报名支付\n订单关联"

' ===== 订单多角色关联 =====
User "买家" -- "0..*" ServiceOrder : "下单购买\n消费者角色"
User "卖家" -- "0..*" ServiceOrder : "提供服务\n服务者角色"
Content "服务内容" -- "0..*" ServiceOrder : "订单内容\n服务标的"

' ===== 评价系统关系 =====
ServiceOrder "1" -- "0..1" ServiceReview : "订单评价\n服务反馈"
User "评价人" -- "0..*" ServiceReview : "发表评价\n评价者"
User "被评价人" -- "0..*" ServiceReview : "接收评价\n被评价者"
Content "服务内容" -- "0..*" ServiceReview : "内容评价\n服务反馈"

' ===== 媒体文件关联 =====
User "上传者" -- "0..*" Media : "上传文件\n媒体资源"
Content "内容媒体" -- "0..*" Media : "关联媒体\n图片视频"
ServiceReview "评价图片" -- "0..*" Media : "评价媒体\n用户图片"
ChatMessage "消息媒体" -- "0..*" Media : "附件资源\n多媒体"

' ===== 通知系统关联 =====
User "接收者" -- "0..*" Notification : "接收通知\n消息推送"
ContentAction "触发通知" ..> Notification : "互动通知\n点赞收藏"
Comment "触发通知" ..> Notification : "评论通知\n评论回复"
CommentLike "触发通知" ..> Notification : "评论点赞\n点赞通知"
UserRelation "触发通知" ..> Notification : "关注通知\n社交提醒"
TopicFollow "触发通知" ..> Notification : "话题通知\n话题更新"
ServiceOrder "订单通知" ..> Notification : "订单提醒\n状态变更"

' ===== 举报系统关联 =====
User "举报人" -- "0..*" Report : "提交举报\n安全管理"
User "处理人" -- "0..*" Report : "处理举报\n管理员角色"
Content "被举报" -- "0..*" Report : "内容举报\n内容安全"
Comment "被举报" -- "0..*" Report : "评论举报\n评论管理"
User "被举报用户" -- "0..*" Report : "用户举报\n用户管理"

' ===== 聊天模块关系 =====
ChatConversation "1" *-- "0..*" ChatMessage : "会话消息\n生命周期绑定"
ChatConversation "1" *-- "1..*" ChatParticipant : "会话参与者\n强依赖关系"
ChatConversation "1" o-- "0..*" TypingStatus : "输入状态\n实时状态"
User "创建者" -- "0..*" ChatConversation : "创建会话\n群主权限"
User "发送者" -- "0..*" ChatMessage : "发送消息\n内容创建"
User "参与者" -- "0..*" ChatParticipant : "加入会话\n成员身份"
User "1" *-- "0..1" MessageSettings : "消息设置\n个性化配置"
ChatMessage "原消息" -- "0..*" ChatMessage : "回复消息\n引用关系"
User "输入者" -- "0..*" TypingStatus : "输入状态\n实时更新"

' ===== 订单聊天关联 =====
ServiceOrder "1" ..> ChatConversation : "订单聊天\n业务集成"

' ===== 话题内容关联 =====
Topic "1" o-- "0..*" ContentTopic : "话题内容\n多对多关联"
Content "1" o-- "0..*" ContentTopic : "内容话题\n话题标签"
Topic "1" o-- "0..*" TopicFollow : "话题关注\n用户订阅"
User "1" -- "0..*" TopicFollow : "关注话题\n话题订阅"
User "创建者" -- "0..*" Topic : "创建话题\n话题管理"
' 🔧 新增：话题统计表关系
Topic "1" -- "0..1" TopicStats : "统计数据\nRedis+定时"

' ===== 城市位置关联 =====
City "1" -- "0..*" UserProfile : "用户所在城市\n地理位置"
City "1" -- "0..*" Content : "内容城市\n地理位置"
City "1" -- "0..*" ContentDraft : "草稿城市\n地理位置"

' ===== 热门搜索关联 =====
HotSearch "搜索词" ..> SearchHistory : "统计来源\n数据分析"

' ==========================================
' 📝 功能场景说明
' ==========================================

note right of LoginSession
  **登录会话管理核心**
  • 支持多设备并发登录
  • JWT双令牌机制(访问+刷新)
  • 访问令牌2小时有效期
  • 刷新令牌30天有效期
  • 设备指纹识别和信任设备
  • 会话实时活跃状态追踪
  • 异地登录检测和风险提醒
  • 支持单设备/多设备登录策略
  • 会话状态管理(正常/过期/注销)
  • 🔧 令牌撤销支持(TokenBlacklist黑名单)
  • 🔧 远程注销其他设备功能
end note

note right of SmsVerification
  **短信验证码系统**
  • 支持登录/注册/重置密码等场景
  • 6位数字验证码(随机生成)
  • 5分钟有效期(300秒)
  • 短信令牌绑定防盗用
  • 最多3次验证尝试限制
  • 60秒重发间隔控制
  • 每日10次发送上限
  • IP和设备级别的频率限制
  • 多短信供应商备份机制
  • 发送/验证状态完整追踪
  • 🔧 PhoneVerifyLimit防穷举攻击
  • 🔧 全局验证次数限制(每日30次)
  • 🔧 频繁失败自动封禁24小时
end note

note right of PasswordResetSession
  **密码重置会话管理**
  • 完整6步骤流程管理
  • 30分钟会话有效期
  • 15分钟重置令牌有效期
  • 步骤状态实时追踪
  • 最多3次验证尝试
  • 会话与验证码关联绑定
  • 支持断点续传
  • 会话过期自动清理
  • 重置成功后清除所有设备登录
  • 操作审计日志完整记录
end note

note right of UserDevice
  **设备管理与信任机制**
  • 设备指纹多维度识别
  • 设备信息详细记录
  • 信任设备30天有效期
  • 首次登录/最后登录时间
  • 登录次数统计
  • 设备地理位置追踪
  • 异常设备自动标记
  • 支持远程设备注销
  • 设备状态管理(正常/禁用/异常)
end note

note right of LoginLog
  **登录行为审计系统**
  • 完整登录日志记录
  • 成功/失败状态追踪
  • 详细设备和环境信息
  • 登录耗时性能监控
  • 风险等级评估(4级)
  • 风险标签自动标记
  • 异地登录检测
  • 新设备登录提醒
  • 频繁失败自动告警
  • 支持日志数据分析和报表
end note

note right of SecurityLog
  **安全操作审计系统**
  • 敏感操作完整记录
  • 密码修改/手机号修改等
  • 设备管理操作审计
  • 安全设置变更记录
  • 操作前后值对比(脱敏)
  • 操作结果和失败原因
  • 风险等级评估
  • 敏感操作二次验证标记
  • 操作IP和地理位置
  • 支持安全事件溯源
end note

note right of GameService
  **游戏陪玩服务核心**
  • 支持多种游戏类型陪玩
  • 段位、胜率、技能展示
  • 语音试听、价格套餐
  • 在线状态实时更新
  • 评分、服务次数统计
end note

note right of GameSkill
  **游戏技能详细展示**
  • 擅长位置(射手/打野等)
  • 擅长英雄(国服/省标等)
  • 熟练度等级(1-5星)
  • 支持多技能展示
end note

note right of LifeService
  **生活服务核心**
  • 支持探店/私影/台球等
  • 地理位置、距离计算
  • 预约时间、人数管理
  • 价格套餐、评分统计
  • 商家认证、服务特色
end note

note right of ServiceOrder
  **订单系统核心**
  • 统一订单管理
  • 支持游戏/生活/活动服务
  • 费用明细、优惠计算
  • 支付方式、订单状态
  • 取消、退款机制
end note

note right of ServiceReview
  **评价系统核心**
  • 多维度评分(服务/态度/质量)
  • 图片评价、匿名评价
  • 商家回复功能
  • 点赞、审核机制
end note

note right of Activity
  **活动组局核心**
  • 支持K歌/台球/私影/探店等多类型活动
  • 场所名称、详细地址、地理定位
  • 开始时间、结束时间、预计时长
  • 人数范围(最少-最多)限制
  • 性别要求(不限/仅男/仅女/男女混合)
  • 年龄范围、技能要求设置
  • 费用类型(免费/AA制/发起人请客)
  • 费用明细、优惠信息展示
  • 浏览、点赞、评论、分享、收藏统计
  • 活动状态(草稿/报名中/已满员/进行中/已结束/已取消)
  • 活动标签(系统推荐/用户自定义)
end note

note right of ActivityParticipant
  **活动参与者管理**
  • 参与类型(待审核/已通过/组织者)
  • 联系人姓名、电话、性别
  • 报名备注留言
  • 支付状态(待支付/已支付/已退款)
  • 费用明细(应付/优惠/实际支付)
  • 支付方式、关联订单
  • 报名时间、支付时间、确认时间
  • 参与状态(已退出/正常/已取消)
end note

note right of UserProfile
  **用户组局信誉系统**
  • 组局达人认证标识
  • 发起组局总数统计
  • 参与组局总数统计
  • 成功完成组局次数
  • 取消组局次数记录
  • 组局信誉评分(5分制)
  • 组局成功率(百分比)
end note

note right of UserPreference
  **用户筛选偏好**
  • 游戏/服务类型筛选
  • 价格区间、距离范围
  • 性别、评分筛选
  • 在线状态、排序方式
  • 快速恢复筛选条件
end note

note bottom of UserProfile
  **用户资料完整性(支持个人主页和编辑模块)**
  • 基础信息(昵称1-20字/头像+缩略图/背景图/性别/生日)
  • 详细资料(身高140-200cm/体重30-150kg/职业标签最多5个/简介0-30字)
  • 联系方式(微信号6-20位/微信解锁条件0=公开/1=关注可见/2=付费可见/3=私密)
  • 位置信息(常居地城市/IP归属地显示/详细地址)
  • 认证标识(实名/大神/组局达人/VIP/人气用户)
  • 用户标识系统(性别年龄标签/人气标识/认证标识)
  • 在线状态管理(在线/离线/忙碌/隐身+最后在线时间)
  • 社交统计数据(粉丝数/关注数/内容数/获赞数/被收藏数)
  • 组局信誉系统(评分/成功率/统计数据)
  • 资料完整度(0-100百分比/用于推荐算法)
  • 编辑追踪(最后编辑时间/支持实时保存)
end note

note right of Content
  **内容发布系统**
  • 支持动态/活动/技能服务多种内容类型
  • 标题+正文内容(最大1000字)
  • 🔧 位置信息(POINT空间索引,高效地理查询)
  • ❌ 移除冗余统计字段(改用ContentStats表)
  • 🔧 冗余用户信息(避免N+1查询)
  • 内容状态(草稿/已发布/审核中/已下架/已删除)
  • 支持定时发布功能
  • 热门内容标识
  • 🔧 软删除和版本控制
end note

note right of ContentStats
  **内容统计表(重构核心)**
  • 解决Content表冗余字段问题
  • Redis主存储 + MySQL持久化
  • 每5分钟批量同步一次
  • 定时任务修正脏数据(每10分钟)
  • 避免高并发更新冲突
  • 查询优先读Redis,回源MySQL
  • 支持热点内容快速统计
end note

note right of UserStats
  **用户统计表(重构核心)**
  • 解决UserProfile冗余字段问题
  • 通过消息队列异步同步
  • 粉丝/关注/内容/点赞等统计
  • 组局信誉评分系统
  • 最终一致性保证
  • 支持统计数据修正任务
end note

note right of Comment
  **评论系统核心**
  • 支持一级评论和二级回复
  • 父评论ID+回复评论ID+被回复用户ID
  • 评论内容(最大500字)
  • 评论点赞统计
  • 回复数量统计
  • 评论置顶功能
  • 评论状态管理(正常/已删除/审核中/已屏蔽)
end note

note right of CommentLike
  **评论点赞系统**
  • 评论独立点赞记录
  • 点赞状态管理
  • 支持取消点赞
  • 实时统计更新
end note

note right of ContentDraft
  **草稿自动保存**
  • 支持所有内容类型草稿
  • 标题+正文+位置信息
  • 自动保存时间记录
  • 30天过期自动清理
  • 草稿状态管理(编辑中/已发布/已过期/已删除)
  • 支持发布后清理草稿
end note

note right of Topic
  **话题系统核心**
  • 话题名称唯一性(如#王者荣耀#)
  • 话题分类(游戏/娱乐/生活/美食/运动/其他)
  • 完整统计数据(参与人数/动态数/浏览数/点赞数/评论数/分享数/关注数)
  • 热度计算(热度分数/趋势分数)
  • 时间维度统计(今日/本周/本月动态数)
  • 热门话题标识/趋势话题标识
  • 话题状态管理(禁用/正常/热门推荐/官方推荐)
end note

note right of ContentTopic
  **内容话题关联**
  • 多对多关联(一个内容多个话题)
  • 最多5个话题标签
  • 排序顺序控制
  • 支持话题聚合展示
end note

note right of TopicFollow
  **话题关注系统**
  • 用户关注话题功能
  • 关注状态管理
  • 支持取消关注
  • 话题更新通知
end note

note right of ChatConversation
  **聊天会话核心**
  • 支持私聊/群聊/系统通知/订单会话
  • 最后消息预览(内容/类型/时间/发送者)
  • 置顶/免打扰个性化设置
  • 未读消息数量统计
  • 会话成员管理
  • 会话状态控制(正常/归档/解散)
  • 消息总数统计
end note

note right of ChatMessage
  **聊天消息核心**
  • 多媒体消息(文字/图片/语音/视频/文件)
  • 消息状态管理(发送中/已发送/已送达/已读/失败)
  • 消息序列号(保证有序性)
  • 消息去重机制(client_id)
  • 已读回执(已读人数/已读时间)
  • 消息撤回(2分钟内)
  • 引用回复功能
  • 图片消息点赞
  • 媒体文件详细信息(尺寸/时长/缩略图)
  • 🔧 表分片策略(256张表按会话ID哈希)
  • 🔧 消息归档(30天热数据+冷数据归档)
  • 🔧 撤回改进(清空内容+记录操作人)
end note

note right of ChatParticipant
  **会话参与者管理**
  • 角色权限(成员/管理员/群主)
  • 已读位置追踪(last_read_message_id)
  • 未读消息计数
  • 个性化设置(置顶/免打扰)
  • 免打扰时段设置
  • 群昵称设置
  • 参与状态(正常/已退出/已禁言)
end note

note right of MessageSettings
  **消息设置核心**
  • 推送开关(总开关/声音/震动/预览)
  • 推送时段控制(开始时间/结束时间)
  • 分类推送开关(点赞/评论/关注/系统)
  • 隐私设置(谁可以发消息/添加好友)
  • 消息已读回执控制
  • 在线状态可见性
  • 自动下载设置(图片/视频/语音)
  • 消息保存时长(自动清理)
end note

note right of TypingStatus
  **输入状态管理**
  • 实时输入状态("正在输入...")
  • 心跳更新机制(保持状态)
  • 自动过期(10秒无更新清除)
  • 支持多人输入状态显示
end note

note right of Notification
  **通知消息系统**
  • 消息分类(点赞收藏/评论/粉丝关注/系统通知/订单/活动)
  • 优先级管理(紧急/重要/普通/一般)
  • 操作按钮(主要操作/次要操作)
  • 通知处理状态追踪
  • 通知过期时间管理
  • 深度链接跳转
  • 关键词高亮显示
  • 已读/处理时间记录
end note

note as FunctionalArchitecture
  **🏗️ 功能模块架构总览**
  
  **登录认证模块(安全基础)**
  User + LoginSession + UserDevice + SmsVerification
    ├── 密码登录流程
    │   ├── 手机号+密码验证
    │   ├── BCrypt密码加密
    │   ├── 防暴力破解(5次失败锁定30分钟)
    │   ├── 登录日志记录
    │   └── 生成JWT双令牌(访问+刷新)
    ├── 验证码登录流程
    │   ├── SmsVerification: 短信验证码管理
    │   │   ├── 6位数字验证码(随机生成)
    │   │   ├── 5分钟有效期
    │   │   ├── 最多3次验证尝试
    │   │   ├── 60秒重发间隔
    │   │   ├── 每日10次发送上限
    │   │   └── 短信令牌绑定防盗用
    │   ├── 地区代码选择(+86等)
    │   ├── 验证码状态管理(发送/验证/过期)
    │   └── 验证成功生成会话令牌
    ├── 忘记密码流程
    │   ├── PasswordResetSession: 重置会话管理
    │   │   ├── 6步骤完整流程
    │   │   ├── 30分钟会话有效期
    │   │   ├── 15分钟重置令牌有效期
    │   │   ├── 步骤状态追踪(completed/in_progress)
    │   │   ├── 最多3次验证尝试
    │   │   └── 会话过期自动清理
    │   ├── 手机号验证 → 验证码发送
    │   ├── 验证码验证 → 生成重置令牌
    │   ├── 新密码设置(双重确认)
    │   ├── 密码强度检测(弱/中/强)
    │   └── 重置成功清除所有设备登录
    ├── 会话管理
    │   ├── LoginSession: JWT双令牌机制
    │   │   ├── 访问令牌(2小时)
    │   │   ├── 刷新令牌(30天)
    │   │   ├── 令牌自动刷新
    │   │   ├── 多设备并发登录支持
    │   │   └── 会话状态实时更新
    │   ├── 设备指纹识别
    │   ├── 信任设备管理(30天)
    │   ├── 异地登录检测
    │   └── 会话注销和过期清理
    ├── 设备管理
    │   ├── UserDevice: 设备信息记录
    │   │   ├── 设备指纹(多维度组合)
    │   │   ├── 设备详细信息(型号/系统/版本)
    │   │   ├── 首次/最后登录时间
    │   │   ├── 登录次数统计
    │   │   └── 设备地理位置追踪
    │   ├── 信任设备列表
    │   ├── 远程设备注销
    │   ├── 新设备登录提醒
    │   └── 异常设备标记
    ├── 安全审计
    │   ├── LoginLog: 登录行为审计
    │   │   ├── 成功/失败状态记录
    │   │   ├── 详细设备和环境信息
    │   │   ├── 登录耗时性能监控
    │   │   ├── 风险等级评估(4级)
    │   │   ├── 风险标签自动标记
    │   │   └── 异地/新设备检测
    │   ├── SecurityLog: 安全操作审计
    │   │   ├── 密码修改记录
    │   │   ├── 手机号修改记录
    │   │   ├── 设备管理操作
    │   │   ├── 安全设置变更
    │   │   ├── 操作前后值对比(脱敏)
    │   │   └── 敏感操作二次验证标记
    │   ├── 登录失败统计和告警
    │   ├── 异常行为检测
    │   └── 安全事件溯源
    └── 安全机制
        ├── 密码加密(BCrypt+Salt)
        ├── 传输加密(HTTPS+TLS)
        ├── 防暴力破解(频率限制+账户锁定)
        ├── 防验证码滥用(IP限制+设备限制)
        ├── 双因子认证支持(TOTP)
        ├── 设备指纹识别
        ├── 异地登录检测
        ├── 风险评估和告警
        └── 审计日志完整记录
  
  **发现页面模块(核心社交)**
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
  
  **游戏陪玩功能**
  Content(type=3) → GameService
    ├── GameSkill: 位置/英雄/熟练度
    ├── GameServiceTag: 服务特色标签
    ├── ServiceOrder: 预订下单
    └── ServiceReview: 服务评价
  
  **生活服务功能**
  Content(type=2) → LifeService
    ├── LifeServiceFeature: 服务特色
    ├── LifeServiceTag: 服务标签
    ├── ServiceOrder: 服务预订
    └── ServiceReview: 服务评价
  
  **活动组局功能**
  Content(type=2) → Activity
    ├── ActivityParticipant: 参与者管理
    │   ├── 联系人信息(姓名/电话/性别)
    │   ├── 报名留言、支付状态
    │   ├── 费用明细(应付/优惠/实际)
    │   └── 参与状态(待审核/已通过/已取消)
    ├── ActivityTag: 活动标签
    │   ├── 系统推荐标签
    │   └── 用户自定义标签
    ├── ServiceOrder: 报名支付订单
    ├── ServiceReview: 活动评价
    ├── ContentAction: 点赞分享收藏
    └── ChatConversation: 活动群聊
  
  **筛选系统**
  UserPreference: 筛选条件存储
    ├── 游戏类型、服务类型
    ├── 价格范围、距离范围
    ├── 性别、评分、在线状态
    └── 排序方式偏好
  
  **推荐系统**
  UserBehavior: 行为数据收集
    ├── 浏览、点击、停留时长
    ├── 搜索、筛选行为
    └── 个性化推荐算法
  
  **订单评价系统**
  ServiceReview: 统一评价管理
    ├── 多维度评分
    ├── 图片评价
    ├── 商家回复
    └── 评价统计
  
  **消息通信模块**
  ChatConversation + ChatMessage + ChatParticipant
    ├── 聊天会话管理
    │   ├── 会话类型(私聊/群聊/系统通知/订单会话)
    │   ├── 最后消息预览(内容/类型/时间/发送者)
    │   ├── 置顶/免打扰/归档管理
    │   └── 未读消息统计
    ├── 消息发送接收
    │   ├── 多媒体消息(文字/图片/语音/视频/文件)
    │   ├── 消息状态(发送中/已发送/已送达/已读/失败)
    │   ├── 消息序列号(保证有序性)
    │   ├── 消息去重(client_id)
    │   ├── 引用回复功能
    │   ├── 消息撤回(2分钟内)
    │   └── 图片消息点赞
    ├── 参与者管理
    │   ├── 角色权限(成员/管理员/群主)
    │   ├── 已读位置追踪(last_read_message_id)
    │   ├── 个性化设置(置顶/免打扰)
    │   └── 未读消息计数
    ├── 输入状态管理
    │   ├── TypingStatus: 实时输入状态
    │   ├── 心跳更新机制
    │   └── 自动过期(10秒)
    └── 消息设置
        ├── MessageSettings: 用户偏好配置
        ├── 推送设置(总开关/声音/震动/预览/时段)
        ├── 分类推送开关(点赞/评论/关注/系统)
        ├── 隐私设置(谁可以发消息/添加好友)
        ├── 消息已读回执控制
        ├── 在线状态可见性
        └── 自动下载设置
  
  **通知系统模块**
  Notification: 统一通知管理
    ├── 消息分类
    │   ├── 点赞收藏消息(作品点赞/评论点赞/内容收藏)
    │   ├── 评论消息(直接评论/回复评论/提及评论)
    │   ├── 粉丝关注消息(新关注/取消关注/互相关注)
    │   ├── 系统通知(公告/功能更新/安全提醒/活动通知)
    │   ├── 订单消息(订单状态变更/支付提醒)
    │   └── 活动消息(活动邀请/报名审核/活动提醒)
    ├── 优先级管理
    │   ├── 紧急通知(红色标识/置顶显示)
    │   ├── 重要通知(黄色标识/优先显示)
    │   ├── 普通通知(蓝色标识/正常排序)
    │   └── 一般通知(灰色标识/底部显示)
    ├── 操作按钮
    │   ├── 主要操作(立即完善/查看详情/立即处理)
    │   ├── 次要操作(忽略/稍后提醒/不再提醒)
    │   └── 深度链接跳转(跳转到相关功能页面)
    ├── 通知处理
    │   ├── 已读状态追踪
    │   ├── 处理状态记录
    │   ├── 处理时间统计
    │   └── 通知过期管理
    └── 通知推送
        ├── 实时推送(WebSocket)
        ├── 离线推送(APNs/FCM)
        ├── 推送策略(立即/定时/静默)
        └── 推送统计(阅读率/操作率)
end note

note as DataDesignPrinciples
  **📊 数据设计原则 v5.2**
  
  **核心设计理念**
  ✅ 无JSON设计：所有字段具体化
  ✅ 关系清晰：明确的外键关联
  ✅ 扩展性强：独立的标签/特色表
  ✅ 查询优化：合理的字段冗余
  ✅ 模块化设计：发现页面/游戏/生活/活动/消息通信独立模块
  
  **表结构设计**
  • 53张表完整覆盖所有功能模块
  • 每张表职责单一明确
  • 字段命名规范统一
  • 类型定义精确
  • 关系定义清晰
  
  **字段展开策略**
  • 登录认证：完整字段展开支持安全认证
    - User表扩展：地区代码/邮箱/密码盐值/密码更新时间/登录失败次数/锁定时间/最后登录信息/双因子认证
    - LoginSession：JWT双令牌字段(访问令牌/刷新令牌)/过期时间/登录方式/设备信息详细/网络类型/信任设备标识/最后活跃时间
    - SmsVerification：验证码内容/短信令牌/验证类型/场景标识/模板代码/发送验证状态/验证次数/过期时间
    - PasswordResetSession：会话ID/重置令牌/关联验证码ID/当前步骤/步骤状态(逗号分隔)/验证次数/会话状态/过期时间
    - UserDevice：设备指纹/设备详细信息(品牌/型号/系统/版本)/屏幕分辨率/信任状态/信任过期时间/登录统计
    - LoginLog：登录方式/登录方法/登录状态/失败原因/设备环境详细/登录耗时/会话ID/风险等级/风险标签(逗号分隔)
    - SecurityLog：日志类型/操作动作/操作分类/操作描述/前后值对比(脱敏)/操作结果/失败原因/风险等级/敏感标识
  • 用户资料：具体字段替代metadata(新增用户标识、统计字段)
    - 个人主页支持：头像+缩略图+背景图/职业标签(逗号分隔,最多5个)
    - 编辑模块支持：昵称1-20字/简介0-30字/身高140-200/体重30-150
    - 微信解锁：独立字段存储解锁条件(公开/关注可见/付费/私密)
    - IP归属地：独立字段缓存显示(登录时更新)
    - 资料完整度：百分比字段(0-100,用于推荐)
    - 最后编辑时间：追踪用户资料更新频率
  • 内容系统：位置信息字段展开(地点名称/地址/经纬度/城市)
  • 评论系统：独立Comment表+CommentLike表(多级评论支持)
  • 草稿管理：独立ContentDraft表(自动保存/过期清理)
  • 话题系统：完整统计字段(多维度数据/时间维度统计)
  • 话题关联：ContentTopic表(多对多关联/最多5个话题)
  • 游戏服务：独立表+技能表+标签表
  • 生活服务：独立表+特色表+标签表
  • 活动组局：独立表+参与者表+标签表(完整字段展开)
  • 订单费用：明细字段展开
  • 评价系统：多维评分字段
  • 媒体关联：ref_type+ref_id替代JSON/头像图片CDN加速
  • 消息通信：完整字段展开支持实时通信
    - ChatConversation：最后消息预览字段(内容/类型/时间/发送者)冗余存储
    - ChatMessage：多媒体字段详细(URL/缩略图/尺寸/时长/配文)
    - 消息状态：delivery_status字段展开(发送中/已发送/已送达/已读/失败)
    - 消息去重：client_id+sequence_id双重保证
    - 已读管理：ChatParticipant存储last_read_message_id精确定位
    - 个性化设置：is_pinned/is_muted字段支持用户个性化
  • 通知系统：分类/优先级/操作按钮字段详细展开
    - 通知分类：category+type双层分类(大类+细分类型)
    - 优先级：priority字段(1-4级紧急程度)
    - 操作按钮：action_type/action_url/action_button_text字段展开
    - 处理状态：is_read/is_handled/expire_time完整跟踪
  • 消息设置：MessageSettings独立表存储用户偏好
    - 推送开关：总开关+分类开关+声音震动预览开关
    - 推送时段：push_start_time/push_end_time字段
    - 隐私设置：who_can_message/who_can_add_friend枚举字段
    - 自动下载：auto_download_image/video字段(永不/仅WIFI/始终)
  
  **索引建议(发现页面模块)**
  🔍 Content: (user_id, type, status, created_at)
  🔍 Content: (city_id, type, status, publish_time)
  ' 🔧 修改：使用空间索引
  🔍 Content: SPATIAL INDEX (location)
  🔍 Content: (status, is_hot, created_at)
  ' 🔧 新增：统计表索引
  🔍 ContentStats: (content_id) PRIMARY KEY
  🔍 ContentStats: (like_count DESC, comment_count DESC) -- 热门排序
  🔍 ContentStats: (last_sync_time) -- 同步监控
  🔍 Comment: (content_id, status, created_at)
  🔍 Comment: (parent_id, status, created_at)
  🔍 Comment: (user_id, status, created_at)
  🔍 CommentLike: (comment_id, user_id, status)
  🔍 ContentDraft: (user_id, type, status, updated_at)
  🔍 ContentTopic: (content_id, topic_id)
  🔍 ContentTopic: (topic_id, created_at)
  🔍 Topic: (status, is_hot, created_at)
  🔍 Topic: (category, status, created_at)
  🔍 Topic: (status, is_trending, created_at)
  ' 🔧 新增：话题统计表索引
  🔍 TopicStats: (topic_id) PRIMARY KEY
  🔍 TopicStats: (heat_score DESC) -- 热度排序
  🔍 TopicStats: (trend_score DESC) -- 趋势排序
  🔍 TopicStats: (last_sync_time) -- 同步监控
  🔍 TopicFollow: (user_id, status, created_at)
  🔍 TopicFollow: (topic_id, status, created_at)
  🔍 ContentAction: (content_id, action, status)
  🔍 ContentAction: (user_id, action, status, created_at)
  ' 🔧 新增：用户职业表索引
  🔍 UserOccupation: (user_id, occupation_code) UNIQUE
  🔍 UserOccupation: (occupation_code) -- 反向查询
  🔍 OccupationDict: (code) PRIMARY KEY
  🔍 OccupationDict: (category, sort_order)
  ' 🔧 新增：用户统计表索引
  🔍 UserStats: (user_id) PRIMARY KEY
  🔍 UserStats: (follower_count DESC) -- 人气排序
  🔍 UserStats: (activity_organizer_score DESC) -- 组局排序
  
  **索引建议(登录认证模块)**
  🔍 User: (mobile) UNIQUE - 手机号唯一索引
  🔍 User: (email) UNIQUE - 邮箱唯一索引(允许NULL)
  🔍 User: (username) UNIQUE - 用户名唯一索引(允许NULL)
  🔍 User: (status, created_at) - 用户状态查询
  🔍 User: (login_locked_until) - 锁定状态查询
  🔍 LoginSession: (user_id, status, created_at) - 用户会话列表
  🔍 LoginSession: (access_token) UNIQUE - 令牌验证
  🔍 LoginSession: (refresh_token) UNIQUE - 刷新令牌验证
  🔍 LoginSession: (device_id, user_id, status) - 设备会话查询
  🔍 LoginSession: (expires_at, status) - 过期会话清理
  🔍 LoginSession: (last_active_time, status) - 活跃会话查询
  🔍 SmsVerification: (mobile, verification_type, verify_status) - 验证码查询
  🔍 SmsVerification: (sms_token) UNIQUE - 短信令牌验证
  🔍 SmsVerification: (expire_time, verify_status) - 过期验证码清理
  🔍 SmsVerification: (mobile, created_at) - 发送频率检查
  🔍 SmsVerification: (ip_address, created_at) - IP频率限制
  🔍 PasswordResetSession: (session_id) UNIQUE - 会话ID查询
  🔍 PasswordResetSession: (user_id, session_status, created_at) - 用户重置记录
  🔍 PasswordResetSession: (reset_token) UNIQUE - 重置令牌验证
  🔍 PasswordResetSession: (mobile, session_status, created_at) - 手机号重置记录
  🔍 PasswordResetSession: (expire_time, session_status) - 过期会话清理
  🔍 UserDevice: (user_id, device_id) UNIQUE - 用户设备唯一
  🔍 UserDevice: (device_fingerprint) - 设备指纹查询
  🔍 UserDevice: (user_id, status, last_login_time) - 用户设备列表
  🔍 UserDevice: (user_id, is_trusted, status) - 信任设备查询
  🔍 UserDevice: (trust_expire_time) - 信任过期清理
  🔍 LoginLog: (user_id, created_at) - 用户登录历史
  🔍 LoginLog: (mobile, login_status, created_at) - 手机号登录记录
  🔍 LoginLog: (ip_address, created_at) - IP登录记录
  🔍 LoginLog: (device_id, created_at) - 设备登录记录
  🔍 LoginLog: (login_status, risk_level, created_at) - 风险登录查询
  🔍 LoginLog: (session_id) - 会话关联查询
  🔍 SecurityLog: (user_id, log_type, created_at) - 用户操作日志
  🔍 SecurityLog: (action, result, created_at) - 操作审计查询
  🔍 SecurityLog: (risk_level, is_sensitive, created_at) - 风险操作查询
  🔍 SecurityLog: (ip_address, created_at) - IP操作记录
  ' 🔧 新增：安全增强表索引
  🔍 TokenBlacklist: (token) UNIQUE - 令牌唯一索引
  🔍 TokenBlacklist: (user_id, created_at) - 用户黑名单记录
  🔍 TokenBlacklist: (expire_time) - 过期清理
  🔍 PhoneVerifyLimit: (mobile) PRIMARY KEY - 手机号限制
  🔍 PhoneVerifyLimit: (is_blocked, block_until) - 封禁查询
  🔍 PhoneVerifyLimit: (last_reset_date) - 每日重置
  🔍 RateLimitConfig: (resource_type, resource_name) UNIQUE
  🔍 RateLimitConfig: (status) - 启用配置查询
  
  **索引建议(其他模块)**
  🔍 UserProfile: (user_id) UNIQUE - 主键索引
  🔍 UserProfile: (city_id, online_status, is_real_verified) - 筛选查询
  🔍 UserProfile: (is_popular, is_vip) - 推荐用户
  🔍 UserProfile: (nickname) - 搜索用户
  🔍 UserProfile: (last_edit_time) - 编辑记录
  ' 🔧 修改：使用空间索引
  🔍 GameService: (game_name, is_online, status)
  🔍 LifeService: (service_type, status)
  🔍 LifeService: SPATIAL INDEX (location)
  ' 🔧 新增：服务统计表索引
  🔍 ServiceStats: (service_id, service_type) PRIMARY KEY
  🔍 ServiceStats: (avg_rating DESC, service_count DESC)
  🔍 Activity: (activity_type, status, start_time)
  🔍 Activity: (organizer_id, status, created_at)
  🔍 Activity: SPATIAL INDEX (location)
  🔍 ActivityParticipant: (activity_id, user_id, status)
  🔍 ActivityParticipant: (user_id, participant_type, status)
  🔍 ActivityTag: (activity_id, tag_type)
  🔍 ServiceOrder: (buyer_id, status), (seller_id, status)
  🔍 ServiceReview: (content_id, rating_overall), (reviewee_id)
  
  **索引建议(消息模块)**
  🔍 ChatConversation: (creator_id, type, status, updated_at) - 会话列表查询
  🔍 ChatConversation: (type, is_pinned, last_message_time) - 会话排序
  🔍 ChatConversation: (order_id) - 订单会话关联
  🔍 ChatMessage: (conversation_id, status, sequence_id) - 消息历史查询
  🔍 ChatMessage: (conversation_id, created_at) - 时间排序
  🔍 ChatMessage: (sender_id, status, created_at) - 用户消息记录
  🔍 ChatMessage: (client_id) UNIQUE - 消息去重
  🔍 ChatParticipant: (conversation_id, user_id, status) - 参与者查询
  🔍 ChatParticipant: (user_id, is_pinned, status) - 用户会话列表
  🔍 ChatParticipant: (user_id, unread_count, status) - 未读消息统计
  🔍 MessageSettings: (user_id) UNIQUE - 用户设置查询
  🔍 TypingStatus: (conversation_id, user_id, is_typing) - 输入状态查询
  🔍 TypingStatus: (expire_time) - 过期清理
  🔍 Notification: (user_id, category, is_read, created_at) - 通知列表
  🔍 Notification: (user_id, priority, is_read) - 优先级排序
  🔍 Notification: (user_id, is_handled, expire_time) - 处理状态查询
  🔍 Notification: (ref_type, ref_id) - 关联对象查询
  
  **性能优化**
  ' 🔧 重构：移除大部分冗余统计字段
  📊 User: 登录失败次数字段冗余(避免查询LoginLog统计,直接判断锁定)
  📊 User: 最后登录信息冗余(最后登录时间/IP/设备ID,快速显示用户状态)
  📊 User: 密码盐值单独存储(提升密码验证性能)
  📊 LoginSession: 访问令牌/刷新令牌唯一索引(快速令牌验证)
  📊 LoginSession: 设备信息冗余存储(避免关联UserDevice表,提升查询性能)
  📊 LoginSession: 最后活跃时间字段(心跳更新,用于会话超时检测)
  📊 LoginSession: Redis缓存令牌信息(减少数据库查询,提升验证速度)
  📊 LoginSession: 定期清理过期会话(expires_at自动清理任务)
  📊 SmsVerification: 短信令牌唯一索引(防止验证码盗用,快速验证)
  📊 SmsVerification: 验证次数字段(避免查询历史记录统计)
  📊 SmsVerification: Redis缓存验证码(5分钟过期,减少数据库压力)
  📊 SmsVerification: Redis统计发送频率(60秒/每日限制,实时检查)
  📊 SmsVerification: 定期清理过期验证码(expire_time自动清理任务)
  📊 PasswordResetSession: 会话ID唯一索引(快速会话查询)
  📊 PasswordResetSession: 步骤状态逗号分隔存储(避免关联子表,提升查询)
  📊 PasswordResetSession: 验证次数字段冗余(避免统计查询)
  📊 PasswordResetSession: Redis缓存会话状态(30分钟过期,减少数据库查询)
  📊 PasswordResetSession: 定期清理过期会话(expire_time自动清理任务)
  📊 UserDevice: 设备指纹索引(快速设备识别和查重)
  📊 UserDevice: 登录统计字段冗余(登录次数/最后登录时间,避免关联LoginLog)
  📊 UserDevice: 设备详细信息冗余(避免每次解析User-Agent)
  📊 UserDevice: 定期清理不活跃设备(超过90天未登录的非信任设备)
  📊 LoginLog: 按月分表存储(历史日志分片,提升查询性能)
  📊 LoginLog: 风险标签逗号分隔存储(避免关联标签表,直接展示)
  📊 LoginLog: 设备信息冗余(避免关联UserDevice表)
  📊 LoginLog: 定期归档历史日志(超过6个月的日志归档到冷存储)
  📊 SecurityLog: 按月分表存储(历史日志分片,提升查询性能)
  📊 SecurityLog: 操作前后值脱敏存储(减少存储空间,保护隐私)
  📊 SecurityLog: 定期归档历史日志(超过1年的日志归档到冷存储)
  ' ❌ 删除：UserProfile统计字段冗余（改用UserStats表+消息队列异步同步）
  ' ❌ 删除：Content互动统计字段冗余（改用ContentStats表+Redis+批量同步）
  ' ❌ 删除：Topic多维度统计字段冗余（改用TopicStats表+Redis+定时任务）
  ' ❌ 删除：GameService/LifeService统计字段（改用ServiceStats表+异步同步）
  ' 🔧 新增：统计表优化方案
  📊 ContentStats: Redis主存储+MySQL持久化,每5分钟批量同步
  📊 UserStats: 消息队列异步同步,保证最终一致性
  📊 TopicStats: Redis Sorted Set计算热度,每5分钟同步MySQL
  📊 ServiceStats: 订单完成后异步更新,定时任务修正脏数据
  📊 UserProfile: 头像缩略图单独存储(列表显示优化,减少流量)
  ' ❌ 删除：职业标签逗号分隔（改用UserOccupation关联表）
  📊 UserProfile: 资料完整度实时计算缓存(用于推荐算法)
  📊 UserProfile: 年龄字段冗余存储(避免每次从生日计算)
  📊 UserProfile: IP归属地缓存(登录时更新,减少IP解析)
  ' 🔧 新增：查询优化
  📊 Content: 冗余user_nickname/user_avatar字段(避免N+1查询)
  📊 Comment: 点赞数/回复数字段冗余(实时更新或定时同步)
  ' ❌ 删除：LifeService距离字段（改用POINT类型+空间索引实时计算）
  📊 Activity: 当前参与人数、互动统计实时更新
  📊 ActivityParticipant: 费用明细字段展开便于结算
  📊 ServiceOrder: 费用明细字段展开便于统计
  📊 Media: ref_type+ref_id索引优化,头像图片CDN加速
  📊 ContentDraft: 定期清理过期草稿(30天)
  ' 🔧 修改：ChatConversation冗余优化
  📊 ChatConversation: 保留last_message_id/last_message_time(提升性能)
  ' ❌ 删除：last_message_type/content/sender_id(过度冗余,关联查询)
  📊 ChatConversation: 成员数量冗余统计(群聊显示优化)
  ' 🔧 新增：ChatMessage分片优化
  📊 ChatMessage: 按conversation_id哈希分256张表(提升查询性能)
  📊 ChatMessage: 30天热数据+冷数据归档到对象存储(存储优化)
  📊 ChatMessage: 媒体缩略图单独存储(列表快速加载)
  📊 ChatMessage: 消息序列号保证有序(无需按时间排序,直接sequence_id排序)
  📊 ChatMessage: 已读计数冗余(群聊已读状态快速查询)
  📊 ChatMessage: 撤回后清空content字段(隐私保护+存储优化)
  📊 ChatParticipant: 未读消息数冗余(避免实时统计,定时/触发更新)
  📊 ChatParticipant: 最后已读消息ID(精确定位已读位置,计算未读数)
  📊 MessageSettings: 用户设置Redis缓存(减少数据库查询)
  📊 TypingStatus: Redis存储+定时清理(10秒过期自动删除)
  📊 Notification: 未读数量Redis缓存(实时更新角标数字)
  📊 Notification: 过期通知定期清理(expire_time自动清理任务)
end note

@enduml
