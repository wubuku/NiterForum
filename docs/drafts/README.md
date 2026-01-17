# NiterForum 数据模型文档库

本文档库提供了关于NiterForum项目数据模型设计的全面详解。

## 📚 文档目录

### 1. **DataModel.md** (主文档 - 28KB，900+行)
**完整的数据模型设计指南**

**内容包括：**
- 项目概述和技术栈
- 数据库初始化方法
- 11个核心数据表的详细解析
  - 用户体系（User, UserAccount, UserInfo）
  - 内容体系（Question, Comment, Talk, News）
  - 交互体系（Thumb, Notification）
  - 广告体系（Ad）
- 关键业务逻辑实现
  - 积分系统
  - 用户等级晋升
  - 评论层级关系
  - 通知系统
  - 点赞/收藏机制
- MyBatis映射架构
- 数据模型特点总结
- 性能优化建议
- 常见操作示例代码

**适用场景：** 理解项目整体架构，学习数据模型设计思想

**快速导航：**
- 初次使用者：先读第1-3章
- 开发人员：重点关注第4章（业务逻辑）和第10章（操作示例）
- 优化人员：参考第7章（性能优化）

---

### 2. **DataModel_Tables_Reference.md** (快速参考 - 10KB)
**所有数据表的字段速查表**

**内容包括：**
- 10个表的完整字段列表
  - 字段名、数据类型、约束、说明
  - 关键业务字段解释
- 枚举值对照表
  - comment.type 字段值
  - notification.type 字段值
  - thumb.type 字段值
- 推荐索引SQL
- 数据量估算

**适用场景：** 快速查询表结构，开发时速查字段含义

**使用方式：**
- Ctrl+F 快速查找表名
- 查看字段定义和可选值
- 复制推荐索引SQL执行

---

### 3. **Entity_Semantics_Guide.md** (实体语义详解 - 31KB)
**深入理解每个数据表的真实含义和业务用途**

**内容包括：**
- Question vs Talk vs News 的详细对比
- 每个实体的真实含义和业务场景
- 通过源代码验证的实现细节
- Comment表的多态设计（支持多种评论类型）
- 各个实体的关键特点对比表
- 整体架构和设计模式解析
- 真实的业务场景示例代码

**适用场景：** 深入理解项目业务模型，而不只是表字段

**快速导航：**
- Question vs Talk vs News：第1-3章
- Comment的复杂设计：第4章
- 整体架构理解：第6章
- 具体业务场景：各章的"业务示例"部分

---

### 4. **Database_Setup_Guide.md** (配置指南 - 13KB)
**数据库初始化和配置完整指南**

**内容包括：**
- 快速初始化步骤（一键初始化或手动分步）
- Spring Boot 应用配置
  - application.properties 配置说明
  - 常见连接问题和解决方案
- MyBatis Generator 配置和使用
- 数据库维护和备份
  - 数据备份和恢复
  - 常用维护命令
- 性能调优
  - 索引添加
  - 表优化
  - 查询性能分析
- 数据初始化和测试数据
- 常见问题排查
- 生产环境建议

**适用场景：** 首次部署、数据库配置、问题排查

**快速导航：**
- 初次安装：第1章
- 连接问题：第2.4章
- 性能问题：第5和7章
- 生产部署：第8章

---

## 🚀 快速开始

### 方案1：一分钟快速初始化
```bash
# 进入项目目录
cd NiterForum

# 导入SQL文件
mysql -u root -p < src/main/resources/niter.sql

# 配置 application.properties
# 修改数据库连接信息

# 运行项目
mvn spring-boot:run
```

### 方案2：详细步骤
1. 阅读 `DataModel.md` 第2章了解初始化方式
2. 按 `Database_Setup_Guide.md` 第1.3节手动初始化
3. 按 `Database_Setup_Guide.md` 第2章配置Spring Boot
4. 参考 `DataModel_Tables_Reference.md` 查看表结构

---

## 📖 按场景快速查询

### 我想了解...

#### Question、Talk、News到底有什么区别？
- 阅读：`Entity_Semantics_Guide.md` 第1-3章
- 对比：各章节最后的对比表格
- **这是用户最常问的问题，强烈推荐阅读**

#### 数据模型整体设计
- 阅读：`DataModel.md` 第1-3章
- 查看：`DataModel_Tables_Reference.md` 表关系部分

#### 积分系统如何实现
- 阅读：`DataModel.md` 第4.1章
- 查看：`DataModel_Tables_Reference.md` user_account表说明

#### 评论的层级结构和多态设计
- 阅读：`Entity_Semantics_Guide.md` 第4章（Comment详解）
- 验证：通过CommentService源代码的实现细节
- 查看：`DataModel.md` 第4.3章的技术说明

#### 通知系统是怎么工作的
- 阅读：`DataModel.md` 第4.4章
- 查看：`DataModel_Tables_Reference.md` notification表

#### 如何进行数据库初始化
- 阅读：`Database_Setup_Guide.md` 第1章

#### 如何配置MyBatis
- 阅读：`Database_Setup_Guide.md` 第3章

#### 数据库性能太慢
- 阅读：`Database_Setup_Guide.md` 第5和7章
- 执行：`DataModel_Tables_Reference.md` 推荐索引

#### 连接数据库出错
- 查看：`Database_Setup_Guide.md` 第7.1-7.2章

#### 需要备份数据库
- 查看：`Database_Setup_Guide.md` 第4章

---

## 📊 核心表关系图

```
┌─────────────────────────────────────────┐
│              User (用户)                 │
│  id, name, email, phone, password       │
└────────┬────────────────────┬───────────┘
         │                    │
    ┌────▼─────────┐   ┌─────▼────────┐
    │ UserAccount  │   │  UserInfo    │
    │(积分/等级)  │   │ (个人信息)   │
    └──────────────┘   └──────────────┘

         Content Types
    ┌────────┬───────────┬──────────┐
    │        │           │          │
┌───▼─┐  ┌──▼──┐  ┌───┐ ┌─▼───┐  ┌─▼──┐
│Qst  │  │Talk │  │Ad │ │News │  │    │
└──┬──┘  └──┬──┘  └───┘ └─────┘  └────┘
   │        │
┌──▼────────▼─────┐
│    Comment      │
│ (type=1/2/3..)  │
└────────┬────────┘
         │
    ┌────▼─────┐
    │   Thumb  │
    │ (点赞)  │
    └──────────┘

Notification (通知)
├─ receiver -> User
├─ notifier -> User
└─ outerid -> Comment/Thumb等
```

---

## 🔍 文档使用技巧

### 1. 快速定位表信息
```
DataModel.md 中搜索表名：
- Ctrl+F: "### 3.2.1 question 表"
- Ctrl+F: "CREATE TABLE `question`"

DataModel_Tables_Reference.md 中：
- Ctrl+F: "## 表4：question"
```

### 2. 快速查找字段说明
```
假设要了解 comment 表的 type 字段：

步骤1：打开 DataModel_Tables_Reference.md
步骤2：Ctrl+F 搜索 "comment" 
步骤3：查看 "type字段值表" 部分
```

### 3. 快速查找业务逻辑实现
```
假设要了解评论创建后如何生成通知：

步骤1：打开 DataModel.md
步骤2：Ctrl+F 搜索 "通知系统实现"
步骤3：查看代码示例和流程说明
```

---

## 💡 主要设计特点

### 时间戳设计
- **单位**：毫秒级（System.currentTimeMillis()）
- **字段**：gmt_create, gmt_modified, gmt_latest_comment
- **用途**：排序、性能优化、时间差计算

### 枚举值设计
- **CommentTypeEnum**：评论类型（1-3为问题评论，11-13为说说评论）
- **NotificationTypeEnum**：通知类型（20种细分）
- **LikeTypeEnum**：点赞类型（问题/评论/说说）

### 冗余字段设计
- **notification.notifier_name**：避免JOIN查询
- **notification.outer_title**：性能优化
- **原理**：以更新成本换取查询性能

### 权限管理
- **permission 字段**：0=公开，1-5=权限等级
- **group_id 字段**：用户等级（1-5）
- **机制**：只能查看权限 >= 自己等级的内容

---

## 🛠️ 常用SQL命令速查

### 数据库操作
```sql
-- 创建数据库
CREATE DATABASE niter1129 CHARACTER SET utf8mb4;

-- 查看表列表
SHOW TABLES;

-- 查看表结构
DESC question;

-- 查看自增ID
SHOW TABLE STATUS LIKE 'question'\G

-- 导出数据
mysqldump -u root -p niter1129 > backup.sql

-- 导入数据
mysql -u root -p niter1129 < backup.sql
```

### 索引操作
```sql
-- 查看表索引
SHOW INDEX FROM question;

-- 添加索引
ALTER TABLE question ADD INDEX idx_creator(creator);

-- 删除索引
ALTER TABLE question DROP INDEX idx_creator;

-- 分析索引性能
ANALYZE TABLE question;
```

### 性能分析
```sql
-- 查看SQL执行计划
EXPLAIN SELECT * FROM question WHERE creator = 1;

-- 查看表大小
SELECT table_name, ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema = 'niter1129'
ORDER BY size_mb DESC;

-- 查看表记录数
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = 'niter1129'
ORDER BY table_rows DESC;
```

---

## ⚠️ 重要提醒

### 修改数据库前
- ✅ 备份数据库
- ✅ 在测试环境验证修改
- ✅ 确保有回滚方案

### 生产环境部署
- ✅ 参考 `Database_Setup_Guide.md` 第8章
- ✅ 创建专用数据库用户（不用root）
- ✅ 启用慢查询日志
- ✅ 定期备份

### 性能优化前
- ✅ 使用EXPLAIN分析查询计划
- ✅ 测试索引效果
- ✅ 监控资源使用

---

## 📞 获取帮助

### 文档相关问题
- 查看本README中的"按场景快速查询"部分
- 使用Ctrl+F在相应文档中搜索关键词

### 数据模型相关问题
- 参考 `DataModel.md` 相应章节
- 查看 `DataModel_Tables_Reference.md` 字段说明

### 配置和部署问题
- 参考 `Database_Setup_Guide.md` 的故障排除部分
- 参考本README的"常用SQL命令速查"

### 代码示例
- 查看 `DataModel.md` 第10章的操作示例
- 查看 `Database_Setup_Guide.md` 第6章的数据初始化例子

---

## 📝 文档版本

| 版本 | 日期 | 说明 |
|-----|------|------|
| 1.0 | 2026-01-17 | 初始版本，包含完整的数据模型分析 |

---

## 📄 许可协议

本文档是NiterForum项目的一部分，遵循与项目相同的协议。

**文档统计：5份详细文档，2,800+行，90KB+内容**

---

**Happy coding with NiterForum! 🎉**

