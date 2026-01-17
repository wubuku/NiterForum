# NiterForum 数据模型设计详解

## 1. 项目概述

**NiterForum** 是一个基于 Spring Boot 的开源社区论坛系统，采用以下技术栈：
- 后端框架：Spring Boot 2.1.6
- ORM框架：MyBatis + MyBatis Generator
- 数据库：MySQL 5.5+
- 数据访问层：使用自动生成的mapper和自定义mapper扩展

### 项目特点
- 支持多种OAuth2登录方式（QQ、微博、百度、Github等）
- 完整的问题/评论/回复系统
- 说说板块（类似朋友圈）
- 新闻资讯板块
- 积分系统和用户等级晋升
- 消息通知系统

---

## 2. 数据库初始化

### 2.1 SQL文件位置
```
src/main/resources/niter.sql
```

### 2.2 手动初始化步骤

1. **创建数据库**
```sql
CREATE DATABASE niter1129 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE niter1129;
```

2. **导入SQL文件**
```bash
mysql -u root -p niter1129 < src/main/resources/niter.sql
```

3. **验证表是否创建成功**
```sql
SHOW TABLES;
```

应该看到以下11个表：
- `user` - 用户基本信息
- `user_account` - 用户账户积分等级信息
- `user_info` - 用户详细信息
- `question` - 帖子/问题表
- `comment` - 评论表
- `talk` - 说说表
- `notification` - 通知表
- `thumb` - 点赞/收藏表
- `news` - 新闻资讯表
- `ad` - 广告表

### 2.3 字符编码说明
所有表都使用 `utf8mb4` 字符集，支持完整的Unicode字符和emoji表情。

---

## 3. 核心数据模型详解

### 3.1 用户体系（User System）

#### 3.1.1 user 表 - 用户基本信息

**表结构：**
```sql
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `account_id` varchar(100) - 账户ID（邮箱或手机号）
  `qq_account_id` varchar(64) - QQ OAuth账户ID
  `baidu_account_id` varchar(100) - 百度OAuth账户ID
  `weibo_account_id` varchar(100) - 微博OAuth账户ID
  `name` varchar(16) - 用户昵称
  `token` char(36) - 登录token
  `password` varchar(64) - 密码哈希
  `avatar_url` varchar(255) - 头像URL
  `email` varchar(32) UNIQUE - 邮箱
  `phone` varchar(16) UNIQUE - 手机号
  `gmt_create` bigint(20) - 创建时间（毫秒时间戳）
  `gmt_modified` bigint(20) - 修改时间（毫秒时间戳）
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**对应Model：**
```java
public class User {
    private Long id;
    private String accountId;           // 主账户标识
    private String qqAccountId;         // OAuth绑定账户
    private String baiduAccountId;      // OAuth绑定账户
    private String weiboAccountId;      // OAuth绑定账户
    private String name;                // 显示昵称
    private String token;               // JWT token
    private Long gmtCreate;             // 创建时间戳
    private Long gmtModified;           // 修改时间戳
    private String avatarUrl;           // 头像URL
    private String email;               // 邮箱（可选）
    private String phone;               // 手机（可选）
    private String password;            // 密码哈希
}
```

**业务说明：**
- 支持多种账户绑定（手机/邮箱/QQ/微博/百度/Github）
- 使用长整型时间戳（毫秒）记录时间
- 头像URL默认值为 `/images/default-avatar.png`

---

#### 3.1.2 user_account 表 - 用户账户数据

**表结构：**
```sql
CREATE TABLE `user_account` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL UNIQUE - 用户ID
  `group_id` int(3) DEFAULT 1 - 用户组ID
  `vip_rank` int(1) DEFAULT 0 - VIP等级
  `score` int(11) DEFAULT 0 - 总积分
  `score1` int(11) DEFAULT 0 - 积分类型1
  `score2` int(11) DEFAULT 0 - 积分类型2
  `score3` int(11) DEFAULT 0 - 积分类型3
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**对应Model：**
```java
public class UserAccount {
    private Long id;
    private Long userId;        // 关联user表的外键
    private Integer groupId;    // 用户组等级（1-5）
    private Integer vipRank;    // VIP等级
    private Integer score;      // 总积分
    private Integer score1;     // 发布相关积分
    private Integer score2;     // 评论相关积分
    private Integer score3;     // 点赞相关积分
}
```

**业务说明：**
- **group_id**：用户等级
  - 1: 新手（初级）
  - 2-5: 不同等级（根据积分自动晋升）
- **积分系统**：
  - `score1`：发布帖子获得
  - `score2`：被评论时获得
  - `score3`：获得点赞/收藏时获得
  - 具体增减规则在 `application.properties` 配置
- 一对一关系：每个user对应一条user_account记录

---

#### 3.1.3 user_info 表 - 用户详细信息

**表结构：**
```sql
CREATE TABLE `user_info` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL - 用户ID
  `realname` varchar(50) - 真实姓名
  `userdetail` varchar(128) - 个人简介
  `birthday` varchar(20) - 生日
  `marriage` varchar(5) - 婚姻状态
  `sex` varchar(5) DEFAULT '男' - 性别
  `blood` varchar(5) - 血型
  `figure` varchar(5) - 体型
  `constellation` varchar(20) - 星座
  `education` varchar(20) - 教育程度
  `trade` varchar(50) - 行业
  `job` varchar(50) - 工作
  `location` varchar(30) DEFAULT '北京-北京-东城区' - 位置
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**业务说明：**
- 用户个性化信息表
- 所有字段都是可选的
- 一对一关系：每个user对应一条user_info记录
- location 使用地区级别的分层结构（省-市-区）

---

### 3.2 内容体系（Content System）

#### 3.2.1 question 表 - 帖子/问题

**表结构：**
```sql
CREATE TABLE `question` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `title` varchar(120) NOT NULL - 帖子标题
  `description` text - 帖子内容（HTML格式）
  `creator` bigint(20) NOT NULL - 创建者ID
  `gmt_create` bigint(20) NOT NULL - 创建时间
  `gmt_modified` bigint(20) NOT NULL - 修改时间
  `gmt_latest_comment` bigint(20) NOT NULL - 最新评论时间
  `comment_count` int(11) DEFAULT 0 - 评论数
  `view_count` int(11) DEFAULT 0 - 浏览数
  `like_count` int(11) DEFAULT 0 - 点赞/收藏数
  `tag` varchar(256) - 标签（逗号分隔）
  `status` int(11) DEFAULT 0 - 状态（0:正常 1:加精 2:置顶 3:精+顶）
  `column2` int(3) DEFAULT 2 - 专栏号
  `permission` int(3) DEFAULT 0 - 阅读权限
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**对应Model：**
```java
public class Question {
    private Long id;
    private String title;              // 帖子标题
    private String description;        // 帖子内容
    private Long creator;              // 创建者ID
    private Long gmtCreate;            // 创建时间戳
    private Long gmtModified;          // 修改时间戳
    private Long gmtLatestComment;     // 最新评论时间
    private Integer commentCount;      // 评论总数
    private Integer viewCount;         // 浏览次数
    private Integer likeCount;         // 点赞/收藏总数
    private String tag;                // 标签列表
    private Integer status;            // 帖子状态
    private Integer column2;           // 分类/专栏
    private Integer permission;        // 阅读权限
}
```

**业务说明：**
- **status 字段**：
  - 0: 普通帖子
  - 1: 精华帖（加精）
  - 2: 置顶帖
  - 3: 既加精又置顶
- **tag 字段**：多个标签用逗号分隔，支持智能生成
- **permission 字段**：
  - 0: 公开（所有人可见）
  - >0: 需要相应权限等级才能浏览
- **时间戳用途**：
  - `gmt_latest_comment` 用于排序，最新有评论的帖子排在前面
- 与user表通过creator字段关联

---

#### 3.2.2 comment 表 - 评论系统

**表结构：**
```sql
CREATE TABLE `comment` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `parent_id` bigint(20) NOT NULL - 父类ID（问题或评论）
  `type` int(11) NOT NULL - 类型（见下表）
  `commentator` bigint(20) NOT NULL - 评论者ID
  `gmt_create` bigint(20) NOT NULL - 创建时间
  `gmt_modified` bigint(20) NOT NULL - 修改时间
  `like_count` bigint(20) DEFAULT 0 - 点赞数
  `content` varchar(1024) NOT NULL - 评论内容
  `comment_count` int(11) DEFAULT 0 - 子评论数
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**对应Model：**
```java
public class Comment {
    private Long id;
    private Long parentId;             // 评论的目标ID
    private Integer type;              // 评论类型
    private Long commentator;          // 评论者ID
    private Long gmtCreate;            // 创建时间戳
    private Long gmtModified;          // 修改时间戳
    private Long likeCount;            // 点赞数
    private String content;            // 评论内容
    private Integer commentCount;      // 回复数
}
```

**type 字段说明**（CommentTypeEnum）：
```
1  -> QUESTION: 评论问题
2  -> COMMENT: 评论的回复（一级评论）
3  -> SUB_COMMENT: 评论回复的回复（二级评论）
11 -> TALK: 评论说说
12 -> TALK_COMMENT: 评论说说的一级回复
13 -> TALK_SUB_COMMENT: 评论说说的二级回复
```

**业务说明：**
- 支持多层级评论（楼中楼）
- 最多支持2层嵌套：评论 -> 回复 -> 再回复
- `parent_id` 在不同type下的含义：
  - type=1: parent_id为question的id
  - type=2: parent_id为问题下第一层评论的id
  - type=3: parent_id为第二层评论（回复）的id
- `comment_count` 记录该评论下的直接回复数

---

#### 3.2.3 talk 表 - 说说板块

**表结构：**
```sql
CREATE TABLE `talk` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `title` varchar(128) - 标题（可选）
  `description` varchar(1024) NOT NULL - 说说内容
  `tag` varchar(128) - 标签
  `images` varchar(2048) - 图片URL列表（JSON或逗号分隔）
  `video` varchar(128) - 视频URL
  `type` int(1) DEFAULT 1 - 类型
  `status` int(11) DEFAULT 0 - 状态
  `permission` int(11) DEFAULT 0 - 权限
  `creator` bigint(20) NOT NULL - 发布者ID
  `view_count` int(11) DEFAULT 0 - 浏览数
  `comment_count` int(11) DEFAULT 0 - 评论数
  `like_count` int(11) DEFAULT 0 - 点赞数
  `gmt_latest_comment` bigint(20) NOT NULL - 最新评论时间
  `gmt_create` bigint(20) NOT NULL - 创建时间
  `gmt_modified` bigint(20) NOT NULL - 修改时间
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**对应Model：**
```java
public class Talk {
    private Long id;
    private String title;              // 可选标题
    private String description;        // 说说内容
    private String tag;                // 标签
    private String images;             // 图片集合
    private String video;              // 视频地址
    private Integer type;              // 内容类型
    private Integer status;            // 发布状态
    private Integer permission;        // 阅读权限
    private Long creator;              // 发布者
    private Integer viewCount;         // 浏览次数
    private Integer commentCount;      // 评论总数
    private Integer likeCount;         // 点赞总数
    private Long gmtLatestComment;     // 最新评论时间
    private Long gmtCreate;            // 创建时间
    private Long gmtModified;          // 修改时间
}
```

**业务说明：**
- 轻量级内容发布，类似朋友圈
- 支持图片和视频
- 字段结构与question表类似，但更灵活
- 同样支持标签、权限、状态管理

---

#### 3.2.4 news 表 - 新闻资讯

**表结构：**
```sql
CREATE TABLE `news` (
  `news_id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `id` varchar(40) UNIQUE - 外部ID
  `title` varchar(128) NOT NULL - 新闻标题
  `description` text NOT NULL - 摘要
  `html` text NOT NULL - 完整内容
  `source` varchar(16) NOT NULL - 来源
  `link` varchar(64) NOT NULL - 原始链接
  `pub_date` varchar(24) NOT NULL - 发布日期
  `imageurl1/2/3` varchar(256) - 配图URL
  `tag` varchar(128) - 标签
  `view_count` int(11) DEFAULT 0 - 浏览数
  `comment_count` int(11) DEFAULT 0 - 评论数
  `like_count` int(11) DEFAULT 0 - 点赞数
  `gmt_latest_comment` bigint(20) NOT NULL - 最新评论时间
  `status` int(2) DEFAULT 1 - 状态
  `column2` int(2) DEFAULT 0 - 分栏
  `gmt_create` bigint(20) NOT NULL - 创建时间
  `gmt_modified` bigint(20) NOT NULL - 修改时间
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**业务说明：**
- 来自第三方新闻源的定期更新
- 支持多张配图
- 保留原始新闻链接和来源
- 用户可在新闻下评论和点赞

---

### 3.3 交互体系（Interaction System）

#### 3.3.1 thumb 表 - 点赞/收藏

**表结构：**
```sql
CREATE TABLE `thumb` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `target_id` bigint(20) NOT NULL - 点赞目标ID
  `type` int(11) NOT NULL - 目标类型
  `liker` bigint(20) NOT NULL - 点赞者ID
  `gmt_create` bigint(20) NOT NULL - 创建时间
  `gmt_modified` bigint(20) NOT NULL - 修改时间
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**对应Model：**
```java
public class Thumb {
    private Long id;
    private Long targetId;             // 被点赞的目标ID
    private Integer type;              // 目标类型（LikeTypeEnum）
    private Long liker;                // 点赞者ID
    private Long gmtCreate;            // 创建时间
    private Long gmtModified;          // 修改时间
}
```

**业务说明：**
- 记录用户的点赞和收藏行为
- 通过type字段区分不同类型的目标
- 防止重复点赞（unique约束应该在代码层面检查）
- 与user表通过liker字段关联

---

#### 3.3.2 notification 表 - 通知系统

**表结构：**
```sql
CREATE TABLE `notification` (
  `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `notifier` bigint(20) NOT NULL - 通知触发者ID
  `receiver` bigint(20) NOT NULL - 接收者ID
  `outerid` bigint(20) NOT NULL - 外部关联ID
  `type` int(1) NOT NULL - 通知类型
  `gmt_create` bigint(20) NOT NULL - 创建时间
  `status` int(1) DEFAULT 0 - 状态（0:未读 1:已读）
  `notifier_name` varchar(50) - 通知者名称（冗余字段）
  `outer_title` varchar(1024) - 外部标题（冗余字段）
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**对应Model：**
```java
public class Notification {
    private Long id;
    private Long notifier;             // 谁触发的通知
    private Long receiver;             // 谁接收的通知
    private Long outerid;              // 关联的对象ID
    private Integer type;              // 通知类型
    private Long gmtCreate;            // 创建时间
    private Integer status;            // 已读状态
    private String notifierName;       // 触发者昵称（查询性能优化）
    private String outerTitle;         // 关联对象标题（查询性能优化）
}
```

**type 字段说明**（NotificationTypeEnum）：
```
1  -> REPLY_QUESTION: 有人回复了您的问题
2  -> REPLY_COMMENT: 有人回复了您的评论
21 -> REPLY_SUB_COMMENT: 有人评论了您的回复
3  -> LIKE_QUESTION: 有人收藏了您的问题
4  -> LIKE_COMMENT: 有人点赞了您的回复
5  -> LIKE_SUB_COMMENT: 有人点赞了您的评论
11 -> REPLY_TALK: 有人回复了您的说说
12 -> REPLY_TALK_COMMENT: 有人回复了您的说说评论
14 -> LIKE_TALK: 有人点赞了您的说说
15 -> LIKE_TALK_COMMENT: 有人点赞了您的说说评论
```

**业务说明：**
- 通知系统用于提醒用户相关事件
- `status` 字段实现已读/未读功能
- `notifier_name` 和 `outer_title` 是冗余字段，避免查询时需要JOIN多个表
- 一个用户可能有多条不同类型的通知

---

### 3.4 广告体系（Ad System）

#### 3.4.1 ad 表 - 广告

**表结构：**
```sql
CREATE TABLE `ad` (
  `id` int(11) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `title` varchar(256) - 广告标题
  `url` varchar(512) - 广告链接
  `image` varchar(256) - 广告图片
  `gmt_start` bigint(20) NOT NULL - 开始时间
  `gmt_end` bigint(20) NOT NULL - 结束时间
  `gmt_create` bigint(20) NOT NULL - 创建时间
  `gmt_modified` bigint(20) NOT NULL - 修改时间
  `status` int(11) NOT NULL - 状态
  `pos` varchar(10) NOT NULL - 投放位置
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**业务说明：**
- 支持有时间限制的广告投放
- 按位置(pos)展示不同的广告
- status字段控制广告是否生效

---

## 4. 关键业务逻辑与数据模型结合

### 4.1 积分系统实现

积分系统通过 `user_account` 表的四个字段实现：

```java
// QuestionService.java - 发布帖子时增加积分
Integer score1PublishInc;   // 发布帖子增加score1
Integer score2PublishInc;   // 发布帖子增加score2（用户被评论时）
Integer score3PublishInc;   // 发布帖子增加score3（用户被点赞时）
```

**具体流程：**
1. 用户发布问题 -> score1增加（鼓励发布内容）
2. 有人评论用户的问题 -> 被评论者的score2增加（鼓励互动）
3. 有人点赞用户的内容 -> 被点赞者的score3增加（鼓励优质内容）

通过配置文件控制每种操作的积分增减值。

### 4.2 用户等级晋升系统

基于积分的自动晋升机制：

```java
// user_group.r1.max: 等级1的最大积分
// user_group.r2.max: 等级2的最大积分
// ...
// 当用户总积分超过某个阈值，自动晋升
```

**等级结构：**
- group_id=1: 初级用户
- group_id=2-5: 中高级用户（逐级提升）

不同等级享受不同特权（如发布权限、浏览权限等）。

### 4.3 评论的层级关系

通过 `comment` 表的 `type` 和 `parent_id` 实现复杂的评论层级：

```
问题（Question）
  ├─ 一级评论（type=2, parent_id=question.id）
  │  ├─ 二级评论（type=3, parent_id=一级评论.id）
  │  └─ 二级评论（type=3, parent_id=一级评论.id）
  └─ 一级评论（type=2, parent_id=question.id）

说说（Talk）
  ├─ 一级评论（type=12, parent_id=talk.id）
  │  └─ 二级评论（type=13, parent_id=一级评论.id）
  └─ 一级评论（type=12, parent_id=talk.id）
```

### 4.4 通知系统实现

通知创建时的关键逻辑（CommentService）：

```java
// 用户A评论了用户B的问题 -> 为用户B创建通知
Notification notification = new Notification();
notification.setNotifier(commentatorId);        // A
notification.setReceiver(questionCreatorId);    // B
notification.setOuterid(commentId);             // 评论ID
notification.setType(NotificationTypeEnum.REPLY_QUESTION.getType());
notification.setGmtCreate(System.currentTimeMillis());
notification.setStatus(NotificationStatusEnum.UNREAD.getStatus());  // 0 = 未读
notification.setNotifierName(commentatorName);  // 冗余字段
notification.setOuterTitle(questionTitle);      // 冗余字段
```

冗余字段设计的原因：
- 避免每次查询通知时都需要JOIN多个表
- 提升查询性能，特别是通知列表查询
- 即使原始内容被删除，通知仍然可读

### 4.5 点赞/收藏机制

通过 `thumb` 表和相关的 `like_count` 字段实现：

```
用户点赞/收藏 -> 插入/删除thumb表记录 -> 同时更新目标表的like_count
```

**流程示例：**
1. 用户点赞问题 -> 向thumb表插入一条记录
2. 同时将question表的like_count +1
3. 查询时可以根据 `type` 字段知道是对问题、评论还是说说的点赞

---

## 5. MyBatis映射与数据访问

### 5.1 Mapper文件位置

```
src/main/resources/mapper/
├── AdMapper.xml
├── CommentMapper.xml
├── CommentExtMapper.xml
├── NewsMapper.xml
├── NewsExtMapper.xml
├── NotificationMapper.xml
├── QuestionMapper.xml
├── QuestionExtMapper.xml
├── TalkMapper.xml
├── TalkExtMapper.xml
├── ThumbMapper.xml
├── ThumbExtMapper.xml
├── UserMapper.xml
├── UserExtMapper.xml
├── UserInfoMapper.xml
├── UserAccountMapper.xml
└── UserAccountExtMapper.xml
```

### 5.2 Mapper命名规则

- **标准Mapper**（如 `UserMapper.xml`）：由 MyBatis Generator 自动生成
  - 包含基本的CRUD操作
  - 使用Example模式进行条件查询
- **扩展Mapper**（如 `UserExtMapper.xml`）：手工编写的扩展
  - 包含复杂的业务查询
  - 支持多表JOIN等高级查询

### 5.3 常用查询模式

**示例：按标签查询问题列表**

```java
// QuestionExtMapper 中的自定义查询
List<Question> listByTag(@Param("tag") String tag, 
                         RowBounds rowBounds);
```

**示例：获取用户信息和积分**

```java
// UserService 中通常需要关联查询
User user = userMapper.selectByPrimaryKey(userId);
UserAccount account = userAccountMapper.selectByPrimaryKey(userId);
UserInfo info = userInfoMapper.selectByPrimaryKey(userId);
```

---

## 6. 数据模型特点总结

### 6.1 时间戳设计

所有表都使用 `gmt_create` 和 `gmt_modified` 字段记录时间：
- **单位**：毫秒级时间戳（使用 `System.currentTimeMillis()`）
- **优势**：
  - 跨时区兼容性好
  - 排序和比较性能高
  - 便于时间差计算

### 6.2 字段冗余设计

部分表中存在冗余字段（如notification表的notifier_name）：
- **目的**：减少查询时的JOIN操作，提升性能
- **维护**：需要在更新原始数据时同步更新冗余字段
- **取舍**：在高并发场景下，冗余字段值得牺牲一些更新性能

### 6.3 状态字段设计

多个表都使用状态字段（status）来标记对象的状态：
- question: 0(正常), 1(加精), 2(置顶), 3(精+顶)
- notification: 0(未读), 1(已读)
- 通过位标志的设计，可以在一个int字段中表示多个状态

### 6.4 权限管理

通过 `permission` 字段实现阅读权限控制：
- 0: 公开（所有人可见）
- 1-5: 不同权限等级（对应group_id）
- 用户只能查看权限等级 >= 自己group_id的内容

### 6.5 分类与标签

- **column2字段**：问题的分类/专栏（整数型，便于索引）
- **tag字段**：逗号分隔的字符串，支持多个标签
- 标签可以自动生成（通过NLP分析内容）

---

## 7. 性能优化建议

### 7.1 索引建议

重要的索引应该包括：

```sql
-- 用户相关
CREATE INDEX idx_user_phone ON user(phone);
CREATE INDEX idx_user_email ON user(email);

-- 内容相关
CREATE INDEX idx_question_creator ON question(creator);
CREATE INDEX idx_question_tag ON question(tag);
CREATE INDEX idx_question_gmt_latest_comment ON question(gmt_latest_comment);

-- 评论相关
CREATE INDEX idx_comment_parent_id ON comment(parent_id);
CREATE INDEX idx_comment_type ON comment(type);
CREATE INDEX idx_comment_commentator ON comment(commentator);

-- 交互相关
CREATE INDEX idx_thumb_target_id ON thumb(target_id);
CREATE INDEX idx_thumb_liker ON thumb(liker);

-- 通知相关
CREATE INDEX idx_notification_receiver ON notification(receiver);
CREATE INDEX idx_notification_status ON notification(status);
```

### 7.2 查询优化

- 使用分页查询 (RowBounds) 避免一次加载过多数据
- 根据 `gmt_latest_comment` 排序，实现"热门"功能
- 对用户列表、评论列表等高频查询使用缓存
- 避免在SELECT中使用LIKE '%keyword%'，改用全文搜索或专门的搜索引擎

### 7.3 表设计改进建议

- 考虑对大数据量表进行分表（如按时间分表）
- 对于高并发的计数字段（如like_count），考虑异步更新
- 对于用户头像等不常变的数据，考虑使用缓存

---

## 8. 数据库配置

### 8.1 配置文件位置

```
src/main/resources/application.properties
```

关键配置项：

```properties
# 数据库连接
spring.datasource.url=jdbc:mysql://localhost:3306/niter1129?characterEncoding=utf-8
spring.datasource.username=root
spring.datasource.password=your_password

# MyBatis配置
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.type-aliases-package=cn.niter.forum.model

# 积分系统配置
score1.publish.inc=10
score2.publish.inc=5
score3.publish.inc=3
score1.comment.inc=2
# ... 更多配置
```

### 8.2 MyBatis Generator配置

```
src/main/resources/generatorConfig.xml
```

用于生成Model和Mapper类。配置修改后执行：

```bash
mvn -Dmybatis.generator.overwrite=true mybatis-generator:generate
```

---

## 9. 快速参考

### 9.1 表关系图

```
User (id)
  ├─ 1:1 ─→ UserAccount (user_id)
  ├─ 1:1 ─→ UserInfo (user_id)
  ├─ 1:N ─→ Question (creator)
  ├─ 1:N ─→ Comment (commentator)
  ├─ 1:N ─→ Talk (creator)
  └─ 1:N ─→ Thumb (liker)

Question (id)
  ├─ 1:N ─→ Comment (parent_id, type=1)
  └─ 1:N ─→ Thumb (target_id, type=1)

Comment (id)
  ├─ 1:N ─→ Comment (parent_id, type=2/3)
  ├─ 1:N ─→ Thumb (target_id, type=2/3)
  └─ 1:N ─→ Notification (outerid)

Notification (id)
  ├─ N:1 ─→ User (notifier) [发起者]
  └─ N:1 ─→ User (receiver) [接收者]
```

### 9.2 常用字段值参考

| 字段 | 含义 | 可选值 |
|-----|------|--------|
| comment.type | 评论类型 | 1(问题), 2(一级评论), 3(二级评论), 11(说说), 12(说说一级), 13(说说二级) |
| question.status | 问题状态 | 0(正常), 1(加精), 2(置顶), 3(精+顶) |
| notification.type | 通知类型 | 1(回问), 2(回评), 3(收藏), 4(点赞), 等等 |
| notification.status | 通知状态 | 0(未读), 1(已读) |
| user_account.group_id | 用户等级 | 1-5 |
| permission | 权限等级 | 0(公开), 1-5(不同权限等级) |

---

## 10. 常见操作示例

### 10.1 发布一个问题

```java
// 1. 创建Question对象
Question question = new Question();
question.setTitle("我的第一个问题");
question.setDescription("<p>问题内容</p>");
question.setCreator(userId);
question.setGmtCreate(System.currentTimeMillis());
question.setGmtModified(System.currentTimeMillis());
question.setGmtLatestComment(System.currentTimeMillis());
question.setTag("java,spring");

// 2. 插入数据库
questionMapper.insert(question);

// 3. 更新用户积分
UserAccount account = userAccountMapper.selectByPrimaryKey(userId);
account.setScore1(account.getScore1() + score1PublishInc);
userAccountMapper.updateByPrimaryKey(account);
```

### 10.2 添加评论并创建通知

```java
// 1. 创建Comment对象
Comment comment = new Comment();
comment.setParentId(questionId);
comment.setType(CommentTypeEnum.QUESTION.getType()); // type=1
comment.setCommentator(userId);
comment.setContent("很好的问题");
comment.setGmtCreate(System.currentTimeMillis());
comment.setGmtModified(System.currentTimeMillis());

// 2. 插入评论
commentMapper.insert(comment);

// 3. 更新问题的评论计数和最新评论时间
question.setCommentCount(question.getCommentCount() + 1);
question.setGmtLatestComment(System.currentTimeMillis());
questionMapper.updateByPrimaryKeySelective(question);

// 4. 为提问者创建通知
Notification notification = new Notification();
notification.setNotifier(userId);
notification.setReceiver(question.getCreator());
notification.setOuterid(comment.getId());
notification.setType(NotificationTypeEnum.REPLY_QUESTION.getType());
notification.setGmtCreate(System.currentTimeMillis());
notification.setStatus(0); // 未读
notificationMapper.insert(notification);
```

### 10.3 点赞操作

```java
// 1. 检查是否已点赞
ThumbExample example = new ThumbExample();
example.createCriteria()
    .andTargetIdEqualTo(questionId)
    .andTypeEqualTo(LikeTypeEnum.QUESTION.getType())
    .andLikerEqualTo(userId);
List<Thumb> thumbs = thumbMapper.selectByExample(example);

if (thumbs.isEmpty()) {
    // 2. 未点赞，则添加点赞记录
    Thumb thumb = new Thumb();
    thumb.setTargetId(questionId);
    thumb.setType(LikeTypeEnum.QUESTION.getType());
    thumb.setLiker(userId);
    thumb.setGmtCreate(System.currentTimeMillis());
    thumb.setGmtModified(System.currentTimeMillis());
    thumbMapper.insert(thumb);
    
    // 3. 更新问题的like_count
    question.setLikeCount(question.getLikeCount() + 1);
    questionMapper.updateByPrimaryKeySelective(question);
} else {
    // 取消点赞
    thumbMapper.deleteByExample(example);
    question.setLikeCount(Math.max(0, question.getLikeCount() - 1));
    questionMapper.updateByPrimaryKeySelective(question);
}
```

