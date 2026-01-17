# NiterForum 数据表字段快速参考

## 表1：user - 用户基本信息表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 用户ID |
| account_id | varchar(100) | - | 账户ID（邮箱或手机） |
| qq_account_id | varchar(64) | - | QQ OAuth账户 |
| baidu_account_id | varchar(100) | - | 百度OAuth账户 |
| weibo_account_id | varchar(100) | - | 微博OAuth账户 |
| name | varchar(16) | - | 昵称 |
| token | char(36) | - | JWT Token |
| password | varchar(64) | - | 密码哈希 |
| avatar_url | varchar(255) | - | 头像URL（默认:/images/default-avatar.png） |
| email | varchar(32) | UQ | 邮箱 |
| phone | varchar(16) | UQ | 手机号 |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| gmt_modified | bigint(20) | - | 修改时间戳(ms) |

**关键操作：**
- 登录验证：account_id + password
- OAuth绑定：更新对应的*_account_id字段
- 头像更新：avatar_url字段

---

## 表2：user_account - 用户账户积分表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 账户ID |
| user_id | bigint(20) | FK, UQ | 用户ID（外键） |
| group_id | int(3) | - | 用户组等级(1-5) |
| vip_rank | int(1) | - | VIP等级 |
| score | int(11) | - | 总积分 |
| score1 | int(11) | - | 发布积分 |
| score2 | int(11) | - | 互动积分 |
| score3 | int(11) | - | 点赞积分 |

**用途：**
- 积分管理：score1/score2/score3分别对应不同操作类型
- 等级晋升：根据group_id判断用户权限
- VIP体系：vip_rank记录VIP等级

---

## 表3：user_info - 用户详细信息表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 信息ID |
| user_id | bigint(20) | FK | 用户ID（外键） |
| realname | varchar(50) | - | 真实姓名 |
| userdetail | varchar(128) | - | 个人简介 |
| birthday | varchar(20) | - | 生日 |
| marriage | varchar(5) | - | 婚姻状态 |
| sex | varchar(5) | - | 性别(默认:男) |
| blood | varchar(5) | - | 血型 |
| figure | varchar(5) | - | 体型 |
| constellation | varchar(20) | - | 星座 |
| education | varchar(20) | - | 教育程度 |
| trade | varchar(50) | - | 行业 |
| job | varchar(50) | - | 工作 |
| location | varchar(30) | - | 位置(省-市-区格式) |

**用途：** 用户个性化信息，所有字段可选

---

## 表4：question - 问题/帖子表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 问题ID |
| title | varchar(120) | NOT NULL | 标题 |
| description | text | - | 内容（HTML格式） |
| creator | bigint(20) | FK | 创建者ID |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| gmt_modified | bigint(20) | - | 修改时间戳(ms) |
| gmt_latest_comment | bigint(20) | - | 最新评论时间 |
| comment_count | int(11) | - | 评论总数 |
| view_count | int(11) | - | 浏览总数 |
| like_count | int(11) | - | 点赞/收藏总数 |
| tag | varchar(256) | - | 标签(逗号分隔) |
| status | int(11) | - | 状态(0:正常,1:加精,2:置顶,3:精+顶) |
| column2 | int(3) | - | 专栏/分类ID |
| permission | int(3) | - | 权限等级(0:公开) |

**关键字段说明：**
- `status`：用位运算组合(1=加精，2=置顶，3=两者)
- `tag`：支持智能生成，用逗号分隔多个标签
- `gmt_latest_comment`：用于热门排序

---

## 表5：comment - 评论表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 评论ID |
| parent_id | bigint(20) | FK | 父对象ID（问题或评论） |
| type | int(11) | - | 评论类型(见下表) |
| commentator | bigint(20) | FK | 评论者ID |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| gmt_modified | bigint(20) | - | 修改时间戳(ms) |
| like_count | bigint(20) | - | 点赞数 |
| content | varchar(1024) | - | 评论内容 |
| comment_count | int(11) | - | 直接回复数 |

**type字段值表：**
| 值 | 含义 | parent_id指向 |
|----|------|--------------|
| 1 | 回复问题 | question.id |
| 2 | 一级回复（对问题） | comment.id (type=1) |
| 3 | 二级回复（对一级评论） | comment.id (type=2) |
| 11 | 回复说说 | talk.id |
| 12 | 一级回复（对说说） | comment.id (type=11) |
| 13 | 二级回复（对说说评论） | comment.id (type=12) |

---

## 表6：talk - 说说表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 说说ID |
| title | varchar(128) | - | 标题（可选） |
| description | varchar(1024) | NOT NULL | 说说内容 |
| tag | varchar(128) | - | 标签 |
| images | varchar(2048) | - | 图片URL列表 |
| video | varchar(128) | - | 视频URL |
| type | int(1) | - | 内容类型 |
| status | int(11) | - | 发布状态 |
| permission | int(11) | - | 权限等级 |
| creator | bigint(20) | FK | 发布者ID |
| view_count | int(11) | - | 浏览数 |
| comment_count | int(11) | - | 评论数 |
| like_count | int(11) | - | 点赞数 |
| gmt_latest_comment | bigint(20) | - | 最新评论时间 |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| gmt_modified | bigint(20) | - | 修改时间戳(ms) |

**用途：** 轻量级内容发布，类似朋友圈

---

## 表7：notification - 通知表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 通知ID |
| notifier | bigint(20) | FK | 触发者ID（谁发起的通知） |
| receiver | bigint(20) | FK | 接收者ID（谁收到通知） |
| outerid | bigint(20) | - | 关联对象ID（评论/点赞等） |
| type | int(1) | - | 通知类型(见下表) |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| status | int(1) | - | 已读状态(0:未读,1:已读) |
| notifier_name | varchar(50) | - | 触发者昵称(冗余字段) |
| outer_title | varchar(1024) | - | 关联对象标题(冗余字段) |

**type字段值表：**
| 值 | 含义 | 说明 |
|----|------|------|
| 1 | 回复问题 | 有人回复了您的问题 |
| 2 | 回复评论 | 有人回复了您的评论 |
| 21 | 评论了回复 | 有人评论了您的回复 |
| 3 | 收藏问题 | 有人收藏了您的问题 |
| 4 | 点赞评论 | 有人点赞了您的回复 |
| 5 | 点赞评论2 | 有人点赞了您的评论 |
| 11 | 回复说说 | 有人回复了您的说说 |
| 12 | 回复说说评论 | 有人回复了您的说说评论 |
| 14 | 点赞说说 | 有人点赞了您的说说 |
| 15 | 点赞说说评论 | 有人点赞了您的说说评论 |

---

## 表8：thumb - 点赞/收藏表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint(20) | PK, AI | 点赞ID |
| target_id | bigint(20) | FK | 目标ID（问题/评论/说说） |
| type | int(11) | - | 目标类型(见下表) |
| liker | bigint(20) | FK | 点赞者ID |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| gmt_modified | bigint(20) | - | 修改时间戳(ms) |

**type字段值表（LikeTypeEnum）：**
| 值 | 含义 | 目标对象 |
|----|------|---------|
| 1 | 问题点赞 | question.id |
| 2 | 评论点赞 | comment.id (type=1) |
| 3 | 回复点赞 | comment.id (type=2/3) |
| 11 | 说说点赞 | talk.id |

**注：说说评论的点赞（type=12/13）在代码中未实现，LikeTypeEnum 只支持上述4种类型**

---

## 表9：news - 新闻资讯表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| news_id | bigint(20) | PK, AI | 新闻ID |
| id | varchar(40) | UQ | 外部唯一ID |
| title | varchar(128) | NOT NULL | 新闻标题 |
| description | text | - | 摘要文本 |
| html | text | - | 完整HTML内容 |
| source | varchar(16) | - | 新闻来源 |
| link | varchar(64) | - | 原始链接 |
| pub_date | varchar(24) | - | 发布日期 |
| imageurl1 | varchar(256) | - | 配图1 |
| imageurl2 | varchar(256) | - | 配图2 |
| imageurl3 | varchar(256) | - | 配图3 |
| tag | varchar(128) | - | 标签 |
| view_count | int(11) | - | 浏览数 |
| comment_count | int(11) | - | 评论数 |
| like_count | int(11) | - | 点赞数 |
| gmt_latest_comment | bigint(20) | - | 最新评论时间 |
| status | int(2) | - | 发布状态 |
| column2 | int(2) | - | 分栏类别 |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| gmt_modified | bigint(20) | - | 修改时间戳(ms) |

**用途：** 来自第三方新闻源的定期更新

---

## 表10：ad - 广告表

| 字段名 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | int(11) | PK, AI | 广告ID |
| title | varchar(256) | - | 广告标题 |
| url | varchar(512) | - | 广告链接 |
| image | varchar(256) | - | 广告图片URL |
| gmt_start | bigint(20) | - | 投放开始时间戳(ms) |
| gmt_end | bigint(20) | - | 投放结束时间戳(ms) |
| gmt_create | bigint(20) | - | 创建时间戳(ms) |
| gmt_modified | bigint(20) | - | 修改时间戳(ms) |
| status | int(11) | - | 状态（0:停用,1:启用） |
| pos | varchar(10) | - | 投放位置(如header,sidebar等) |

**用途：** 时间限制的广告管理

---

## 索引建议

```sql
-- 用户表索引
ALTER TABLE user ADD INDEX idx_phone(phone);
ALTER TABLE user ADD INDEX idx_email(email);
ALTER TABLE user ADD INDEX idx_account_id(account_id);

-- 问题表索引
ALTER TABLE question ADD INDEX idx_creator(creator);
ALTER TABLE question ADD INDEX idx_gmt_latest_comment(gmt_latest_comment);
ALTER TABLE question ADD INDEX idx_tag(tag);
ALTER TABLE question ADD INDEX idx_status(status);

-- 评论表索引
ALTER TABLE comment ADD INDEX idx_parent_id(parent_id);
ALTER TABLE comment ADD INDEX idx_type(type);
ALTER TABLE comment ADD INDEX idx_commentator(commentator);

-- 说说表索引
ALTER TABLE talk ADD INDEX idx_creator(creator);
ALTER TABLE talk ADD INDEX idx_gmt_latest_comment(gmt_latest_comment);

-- 点赞表索引
ALTER TABLE thumb ADD INDEX idx_target_id(target_id);
ALTER TABLE thumb ADD INDEX idx_liker(liker);
ALTER TABLE thumb ADD INDEX idx_type(type);

-- 通知表索引
ALTER TABLE notification ADD INDEX idx_receiver(receiver);
ALTER TABLE notification ADD INDEX idx_status(status);
ALTER TABLE notification ADD INDEX idx_notifier(notifier);

-- 新闻表索引
ALTER TABLE news ADD INDEX idx_column2(column2);
```

---

## 数据量估算

| 表名 | 初始数据 | 预期增长 |
|-----|--------|--------|
| user | 1 | 日增10-100 |
| user_account | 1 | 同user增长 |
| user_info | 1 | 同user增长 |
| question | 1 | 日增50-200 |
| comment | ~1100 | 日增100-500 |
| talk | ~50 | 日增20-50 |
| notification | ~1400 | 日增200-1000 |
| thumb | ~500 | 日增100-300 |
| news | ~15700 | 日增50-100 |
| ad | 4 | 低增长 |

