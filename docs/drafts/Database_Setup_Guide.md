# NiterForum 数据库初始化和配置指南

## 1. 快速开始

### 1.1 前置要求

- MySQL 5.5+ 或 MySQL 8.0+
- 已安装 mysql-client 或 MySQL Workbench
- 拥有MySQL root权限或创建数据库的权限

### 1.2 一键初始化（推荐）

```bash
# 1. 进入项目根目录
cd NiterForum

# 2. 创建数据库并导入SQL文件
mysql -u root -p < src/main/resources/niter.sql

# 系统会提示输入MySQL密码，输入后自动创建数据库和表
```

### 1.3 手动分步初始化

如果一键初始化失败，可以按以下步骤手动操作：

**步骤1：登录MySQL**
```bash
mysql -u root -p
# 输入密码后进入MySQL命令行
```

**步骤2：创建数据库**
```sql
CREATE DATABASE niter1129 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**步骤3：切换到新数据库**
```sql
USE niter1129;
```

**步骤4：导入SQL文件**
```sql
SOURCE src/main/resources/niter.sql;
```

**步骤5：验证表创建成功**
```sql
SHOW TABLES;
```

应该看到以下输出：
```
+----------------------+
| Tables_in_niter1129  |
+----------------------+
| ad                   |
| comment              |
| news                 |
| notification         |
| question             |
| talk                 |
| thumb                |
| user                 |
| user_account         |
| user_info            |
+----------------------+
10 rows in set (0.00 sec)
```

---

## 2. Spring Boot 应用配置

### 2.1 配置文件位置

```
src/main/resources/application.properties
```

### 2.2 数据库连接配置

编辑 `application.properties` 文件，添加以下配置：

```properties
# MySQL数据库连接
spring.datasource.url=jdbc:mysql://localhost:3306/niter1129?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=你的MySQL密码
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

# MyBatis配置
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.type-aliases-package=cn.niter.forum.model
mybatis.configuration.map-underscore-to-camel-case=true

# 日志级别
logging.level.cn.niter.forum.mapper=debug
```

### 2.3 参数说明

| 参数 | 说明 | 示例 |
|-----|------|------|
| url | 数据库连接URL | jdbc:mysql://localhost:3306/niter1129 |
| username | 数据库用户名 | root |
| password | 数据库密码 | 123456 |
| useUnicode | 使用Unicode编码 | true |
| characterEncoding | 字符编码 | UTF-8 |
| useSSL | 是否使用SSL | false |
| serverTimezone | 时区 | UTC 或 Asia/Shanghai |

### 2.4 常见连接问题

**问题1：时区错误**
```
The server time zone value 'CST' is unrecognized or represents more than one time zone
```

**解决方案：** 在URL中添加 `&serverTimezone=UTC` 或 `&serverTimezone=Asia/Shanghai`

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/niter1129?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Shanghai
```

**问题2：认证插件错误**
```
Public Key Retrieval is not allowed
```

**解决方案：** 在URL中添加 `&allowPublicKeyRetrieval=true`

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/niter1129?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
```

---

## 3. MyBatis Generator 配置

### 3.1 配置文件位置

```
src/main/resources/generatorConfig.xml
```

### 3.2 配置示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <classPathEntry location="jar/mysql-connector-java-5.1.46.jar"/>
    
    <context id="default" targetRuntime="MyBatis3">
        
        <!-- 数据库连接 -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                       connectionURL="jdbc:mysql://localhost:3306/niter1129"
                       userId="root"
                       password="你的密码"/>
        
        <!-- Java模型生成器 -->
        <javaModelGenerator targetPackage="cn.niter.forum.model"
                           targetProject="src/main/java"/>
        
        <!-- XML映射文件生成器 -->
        <sqlMapGenerator targetPackage="mapper"
                        targetProject="src/main/resources"/>
        
        <!-- DAO接口生成器 -->
        <javaClientGenerator type="XMLMAPPER"
                            targetPackage="cn.niter.forum.mapper"
                            targetProject="src/main/java"/>
        
        <!-- 配置要生成的表 -->
        <table tableName="user" domainObjectName="User"/>
        <table tableName="user_account" domainObjectName="UserAccount"/>
        <table tableName="user_info" domainObjectName="UserInfo"/>
        <table tableName="question" domainObjectName="Question"/>
        <table tableName="comment" domainObjectName="Comment"/>
        <table tableName="talk" domainObjectName="Talk"/>
        <table tableName="notification" domainObjectName="Notification"/>
        <table tableName="thumb" domainObjectName="Thumb"/>
        <table tableName="news" domainObjectName="News"/>
        <table tableName="ad" domainObjectName="Ad"/>
    </context>
</generatorConfiguration>
```

### 3.3 运行代码生成

修改配置后，执行以下Maven命令：

```bash
# 生成Model和Mapper（覆盖现有文件）
mvn -Dmybatis.generator.overwrite=true mybatis-generator:generate

# 或者不覆盖现有文件
mvn mybatis-generator:generate
```

生成的文件位置：
- Model类：`src/main/java/cn/niter/forum/model/*.java`
- Mapper接口：`src/main/java/cn/niter/forum/mapper/*Mapper.java`
- XML映射：`src/main/resources/mapper/*Mapper.xml`

---

## 4. 数据库维护

### 4.1 备份数据库

```bash
# 备份所有表
mysqldump -u root -p niter1129 > backup_$(date +%Y%m%d_%H%M%S).sql

# 备份指定表
mysqldump -u root -p niter1129 user question comment > backup_content.sql
```

### 4.2 恢复数据库

```bash
# 从备份文件恢复
mysql -u root -p niter1129 < backup_20220101_120000.sql
```

### 4.3 常用维护命令

**查看表大小**
```sql
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS size_mb
FROM information_schema.tables 
WHERE table_schema = 'niter1129'
ORDER BY size_mb DESC;
```

**查看表记录数**
```sql
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = 'niter1129'
ORDER BY table_rows DESC;
```

**查看自增ID当前值**
```sql
SELECT auto_increment 
FROM information_schema.tables 
WHERE table_name = 'question' AND table_schema = 'niter1129';
```

**重置自增ID**
```sql
ALTER TABLE question AUTO_INCREMENT = 500;
```

**查看表索引**
```sql
SHOW INDEX FROM question;
```

**查看表结构**
```sql
DESC question;
-- 或
SHOW FULL COLUMNS FROM question;
```

---

## 5. 性能调优

### 5.1 添加推荐索引

执行以下SQL优化查询性能：

```sql
-- 用户表索引
ALTER TABLE user ADD INDEX idx_phone(phone);
ALTER TABLE user ADD INDEX idx_email(email);

-- 问题表索引
ALTER TABLE question ADD INDEX idx_creator(creator);
ALTER TABLE question ADD INDEX idx_gmt_latest_comment(gmt_latest_comment);
ALTER TABLE question ADD INDEX idx_tag(tag);

-- 评论表索引
ALTER TABLE comment ADD INDEX idx_parent_id(parent_id);
ALTER TABLE comment ADD INDEX idx_type(type);
ALTER TABLE comment ADD INDEX idx_commentator(commentator);

-- 点赞表索引
ALTER TABLE thumb ADD INDEX idx_target_id(target_id);
ALTER TABLE thumb ADD INDEX idx_liker(liker);

-- 通知表索引
ALTER TABLE notification ADD INDEX idx_receiver(receiver);
ALTER TABLE notification ADD INDEX idx_status(status);
```

### 5.2 执行表优化

```sql
-- 整理表碎片
OPTIMIZE TABLE question;
OPTIMIZE TABLE comment;

-- 或批量操作
OPTIMIZE TABLE user, user_account, user_info, question, comment, talk, notification, thumb, news, ad;
```

### 5.3 查询性能分析

```sql
-- 分析SQL执行计划
EXPLAIN SELECT * FROM question WHERE creator = 1 LIMIT 10;

-- 使用EXTENDED获取更详细信息
EXPLAIN EXTENDED SELECT * FROM question WHERE gmt_latest_comment > 1606641726000;
```

---

## 6. 数据初始化和测试数据

### 6.1 初始数据说明

SQL文件中已包含的初始数据：

```sql
-- 系统用户（机器人）
INSERT INTO user VALUES (1, NULL, NULL, NULL, NULL, 'NiterBot', NULL, 1606641726000, 1606641726000, '/images/default-avatar.png', NULL, NULL, NULL);

-- 对应的账户数据
INSERT INTO user_account VALUES (1, 1, 1, 0, 20, 10, 2, 1);

-- 对应的用户信息
INSERT INTO user_info VALUES (1, 1, NULL, '大家好，这里是NiterForum官方机器人', NULL, NULL, '男', NULL, NULL, NULL, NULL, NULL, NULL, '北京-北京-东城区');

-- 欢迎问题
INSERT INTO question VALUES (1, '欢迎访问，又一个NiterForum社区', '...', 1606641726000, 1606641726000, 1, 0, 0, 0, NULL, 1606641726000, 0, 2, 0);

-- 新闻数据（15个新闻记录）
INSERT INTO news VALUES ...
```

### 6.2 添加测试用户

```sql
-- 创建测试用户
INSERT INTO user (account_id, name, password, avatar_url, gmt_create, gmt_modified)
VALUES ('test@example.com', 'TestUser', SHA2('password123', 256), '/images/default-avatar.png', NOW(3), NOW(3));

-- 为新用户创建账户数据
INSERT INTO user_account (user_id, group_id, vip_rank, score, score1, score2, score3)
SELECT id, 1, 0, 0, 0, 0, 0 FROM user WHERE account_id = 'test@example.com';

-- 为新用户创建用户信息
INSERT INTO user_info (user_id, sex, location)
SELECT id, '男', '北京-北京-东城区' FROM user WHERE account_id = 'test@example.com';
```

### 6.3 批量生成测试数据

```sql
-- 生成100条测试问题
INSERT INTO question (title, description, creator, gmt_create, gmt_modified, gmt_latest_comment, tag)
SELECT 
    CONCAT('测试问题 ', @row := @row + 1),
    '<p>这是测试内容</p>',
    1,
    NOW(3),
    NOW(3),
    NOW(3),
    'test,spring,java'
FROM (SELECT @row := 0) AS init,
      information_schema.tables AS t
LIMIT 100;
```

---

## 7. 常见数据库问题排查

### 7.1 连接问题

**现象：** `Connection refused`

**排查步骤：**
1. 确认MySQL服务正在运行
2. 检查MySQL监听端口（默认3306）
3. 确认防火墙没有阻止连接
4. 验证用户名和密码正确

```bash
# Linux/Mac: 检查MySQL状态
sudo systemctl status mysql
# 或
ps aux | grep mysqld

# 启动MySQL
sudo systemctl start mysql
```

### 7.2 字符编码问题

**现象：** 中文显示为乱码

**解决方案：**
```sql
-- 检查数据库字符编码
SHOW CREATE DATABASE niter1129;

-- 检查表字符编码
SHOW CREATE TABLE question;

-- 转换数据库字符编码
ALTER DATABASE niter1129 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 转换表字符编码
ALTER TABLE question CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 7.3 性能问题

**现象：** 查询速度慢

**排查步骤：**
1. 检查是否有必要的索引
2. 使用EXPLAIN分析执行计划
3. 检查表大小和记录数
4. 考虑添加缓存层

```sql
-- 查看慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;

-- 查看当前运行的查询
SHOW PROCESSLIST;

-- 杀死长时间运行的查询
KILL query_id;
```

---

## 8. 生产环境建议

### 8.1 安全建议

- 修改root默认密码
- 创建专用的应用用户（不使用root）
- 限制用户的访问权限
- 定期备份数据库
- 启用MySQL的日志功能

```sql
-- 创建应用用户
CREATE USER 'niter'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON niter1129.* TO 'niter'@'localhost';
FLUSH PRIVILEGES;

-- 查看用户权限
SHOW GRANTS FOR 'niter'@'localhost';
```

### 8.2 监控建议

- 监控数据库连接数
- 监控磁盘空间使用率
- 监控慢查询
- 设置合理的连接超时时间

```properties
# application.properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

### 8.3 备份策略

- 每天执行完整备份
- 每小时执行增量备份
- 定期测试备份恢复
- 将备份存储在不同的物理位置

```bash
#!/bin/bash
# 每天凌晨2点执行备份
BACKUP_DIR="/backup/niter_$(date +%Y%m%d_%H%M%S).sql"
mysqldump -u niter -p'password' niter1129 > $BACKUP_DIR

# 只保留最近7天的备份
find /backup -name "niter_*.sql" -mtime +7 -delete
```

---

## 9. 快速参考

| 操作 | 命令 |
|-----|------|
| 创建数据库 | `CREATE DATABASE niter1129 CHARACTER SET utf8mb4;` |
| 导入SQL | `mysql -u root -p niter1129 < niter.sql` |
| 导出数据 | `mysqldump -u root -p niter1129 > backup.sql` |
| 查看表 | `SHOW TABLES;` |
| 查看表结构 | `DESC table_name;` |
| 查看索引 | `SHOW INDEX FROM table_name;` |
| 添加索引 | `ALTER TABLE table_name ADD INDEX idx_name(column);` |
| 分析性能 | `EXPLAIN SELECT ...;` |
| 优化表 | `OPTIMIZE TABLE table_name;` |
| 修复表 | `REPAIR TABLE table_name;` |

