# NiterForum 实体语义详解指南

本文档通过深入分析源代码，详细解释每个实体的真实含义、用途、和彼此之间的区别与联系。

## 核心概念理解

在开始之前，需要理解NiterForum中内容的三个维度：
1. **主体来源**：谁创建的（用户发布 vs 系统自动导入）
2. **内容形式**：是什么样的内容（长文章 vs 短微博式 vs 新闻资讯）
3. **交互方式**：如何进行互动（评论、点赞、分享等）

---

## 1. Question（问题/帖子）表详解

### 1.1 真实含义

**Question** 表代表的是 **用户发布的长篇问题或讨论贴**。

**核心特征**：
- ✅ 由用户主动创建和发布
- ✅ 内容较长（支持HTML富文本编辑）
- ✅ 聚焦于问题解答或讨论
- ✅ 支持层级评论（问题 → 一级评论 → 二级评论）
- ✅ 支持精华、置顶等内容运营

### 1.2 代码验证

**创建流程**（PublishController.java）：
```java
// 用户通过Web表单发布
@PostMapping("p/publish")
public String doPublish2(
    @RequestParam("title") String title,              // 必填：标题
    @RequestParam("description", required = false) String description,  // 必填：内容
    @RequestParam("tag") String tag,                 // 必填：标签
    @RequestParam("column2") Integer column2,        // 分类
    @RequestParam("permission") Integer permission,  // 权限
    HttpServletRequest request
){
    // 内容审核
    ResultDTO resultDTO = baiduCloudProvider.getTextCensorReult(...);
    
    // 创建Question对象
    Question question = new Question();
    question.setTitle(title);
    question.setDescription(description);  // HTML格式
    question.setTag(tag);
    question.setCreator(user.getId());
    questionService.createOrUpdate(question, user);
}
```

**URL路由**（QuestionController.java）：
```java
// 问题详情页面
@GetMapping(value = {"/p/{id}","/article/{id}"})
public String po(@PathVariable(name = "id") Long id, ...){
    QuestionDTO questionDTO = questionService.getById(id, viewUser_id);
    // 权限检查
    if(questionDTO.getPermission() > viewUser.getGroupId()){
        // 权限不足
        return "result";
    }
    // 获取该问题下的所有评论
    List<CommentDTO> comments = commentService.listByTargetId(id, CommentTypeEnum.QUESTION);
}
```

**数据库字段解析**：
```sql
CREATE TABLE question (
    id BIGINT PRIMARY KEY,                    -- 问题ID
    title VARCHAR(120) NOT NULL,              -- 问题标题
    description TEXT,                        -- 问题内容（HTML格式）
    creator BIGINT,                          -- 发布者ID（来自user表）
    tag VARCHAR(256),                        -- 多个标签，逗号分隔
    comment_count INT DEFAULT 0,             -- 评论总数
    view_count INT DEFAULT 0,                -- 浏览次数
    like_count INT DEFAULT 0,                -- 收藏/点赞数
    status INT DEFAULT 0,                    -- 状态：0=正常, 1=加精, 2=置顶, 3=精+顶
    column2 INT DEFAULT 2,                   -- 分类/栏目ID
    permission INT DEFAULT 0,                -- 权限：0=公开, >0=需要权限等级
    gmt_latest_comment BIGINT,               -- 最新评论时间（用于热门排序）
    gmt_create BIGINT,                       -- 创建时间
    gmt_modified BIGINT                      -- 修改时间
);
```

### 1.3 Question的关键特点

| 特性 | 说明 |
|-----|------|
| **内容长度** | 长文本，支持HTML富文本编辑器（WangEditor） |
| **发布者** | 普通用户 |
| **评论支持** | ✅ 支持（类型为1=回复问题） |
| **点赞支持** | ✅ 支持（类型为1） |
| **权限控制** | ✅ 支持（permission字段） |
| **内容运营** | ✅ 支持精华、置顶（status字段的位运算） |
| **编辑功能** | ✅ 支持（可修改，gmt_modified更新） |
| **前端路由** | `/p/{id}` 或 `/article/{id}` |
| **模板位置** | `templates/p/detail.html` |

### 1.4 业务示例

```java
// 典型场景：用户提问某个技术问题
{
    "title": "SpringBoot项目中如何集成MyBatis？",
    "description": "<p>我的项目想用MyBatis来做ORM...</p><p>怎么配置呢？</p>",
    "tag": "spring,mybatis,java",
    "creator": 123,              // 提问者ID
    "commentCount": 5,           // 已有5条评论
    "viewCount": 127,            // 已有127次浏览
    "likeCount": 8,              // 8个人收藏了
    "status": 1,                 // 已被加精
    "permission": 0              // 公开内容，所有人可见
}
```

---

## 2. Talk（说说）表详解

### 2.1 真实含义

**Talk** 表代表的是 **用户发布的轻量级、短篇幅内容，类似朋友圈**。

**核心特征**：
- ✅ 由用户主动创建和发布
- ✅ 内容较短（可以没有标题，只有内容）
- ✅ 支持文字、图片、视频混合
- ✅ 不强调问答，更多是分享和讨论
- ✅ 同样支持精华、置顶等运营功能

### 2.2 代码验证

**创建流程**（TalkService.java）：
```java
@Transactional
public int insert(TalkDTO talkDTO) {
    Talk talk = new Talk();
    BeanUtils.copyProperties(talkDTO, talk);
    talk.setCreator(talkDTO.getUser().getId());  // 发布者ID
    return talkMapper.insertSelective(talk);
}
```

**URL路由和展示**（TalkController.java）：
```java
// 说说详情页
@GetMapping(value = {"/t/{id}"})
public String comment(@PathVariable(name = "id") Long id, HttpServletRequest request, Model model){
    TalkVO talkVO;
    TalkQueryDTO talkQueryDTO = new TalkQueryDTO();
    talkQueryDTO.setId(id);
    PaginationDTO paginationDTO = talkService.list(talkQueryDTO, viewUser);
    talkVO = (TalkVO) paginationDTO.getData().get(0);
    
    // 获取该说说下的评论
    List<CommentDTO> comments = commentService.listByTargetId(id, CommentTypeEnum.TALK);
    
    model.addAttribute("talk", talkVO);
    model.addAttribute("comments", comments);
    return "t/detail";  // 说说详情模板
}

// 说说列表首页
@GetMapping("/talk")
public String tIndex(HttpServletRequest request, Model model,
                     @RequestParam(name = "page", defaultValue = "1") Integer page,
                     @RequestParam(name = "size", defaultValue = "10") Integer size){
    // 显示说说列表
    model.addAttribute("navtype", "talknav");
    return "t/index";
}
```

**TalkService中的转换逻辑**（TalkService.java）：
```java
public TalkVO convert(TalkVO talkVO, UserDTO view_user){
    // 处理状态位
    if((talkVO.getStatus()&1)==1) talkVO.setEssence(true);      // 是否加精
    if((talkVO.getStatus()&2)==2) talkVO.setSticky(true);       // 是否置顶
    
    // 处理图片
    if(StringUtils.isNotBlank(talkVO.getImages())){
        talkVO.setImageUrls(talkVO.getImages().split(","));  // 图片以逗号分隔
    }
    
    // 检查权限
    if(view_user != null){
        // 检查当前用户是否已点赞
        LikeQueryDTO likeQueryDTO = new LikeQueryDTO();
        likeQueryDTO.setTargetId(talkVO.getId());
        likeQueryDTO.setType(LikeTypeEnum.TALK.getType());
        likeQueryDTO.setLiker(view_user.getId());
        PaginationDTO paginationDTO = likeService.list(likeQueryDTO);
        if(paginationDTO.getTotalCount()==1)
            talkVO.setFavorite(true);  // 用户已点赞此说说
    }
    return talkVO;
}
```

**数据库字段解析**：
```sql
CREATE TABLE talk (
    id BIGINT PRIMARY KEY,                    -- 说说ID
    title VARCHAR(128),                       -- 标题（可选，通常为空）
    description VARCHAR(1024) NOT NULL,       -- 说说内容（短文本）
    tag VARCHAR(128),                         -- 标签（可选）
    images VARCHAR(2048),                     -- 图片URL列表（逗号分隔）
    video VARCHAR(128),                       -- 视频URL（可选）
    type INT DEFAULT 1,                       -- 内容类型
    status INT DEFAULT 0,                     -- 状态：0=正常, 1=加精, 2=置顶, 3=精+顶
    permission INT DEFAULT 0,                 -- 权限：0=公开
    creator BIGINT NOT NULL,                 -- 发布者ID
    view_count INT DEFAULT 0,                -- 浏览数
    comment_count INT DEFAULT 0,             -- 评论数
    like_count INT DEFAULT 0,                -- 点赞数
    gmt_latest_comment BIGINT,               -- 最新评论时间
    gmt_create BIGINT NOT NULL,              -- 创建时间
    gmt_modified BIGINT NOT NULL             -- 修改时间
);
```

### 2.3 Talk的关键特点

| 特性 | 说明 |
|-----|------|
| **内容长度** | 短文本（1024字符），不支持复杂HTML |
| **发布者** | 普通用户 |
| **评论支持** | ✅ 支持（类型为11=回复说说） |
| **点赞支持** | ✅ 支持（类型为11） |
| **权限控制** | ✅ 支持（permission字段） |
| **内容运营** | ✅ 支持精华、置顶（status字段） |
| **媒体支持** | ✅ 图片（images字段，逗号分隔）、视频 |
| **编辑功能** | ✅ 支持 |
| **前端路由** | `/t/{id}` |
| **模板位置** | `templates/t/detail.html` |

### 2.4 业务示例

```java
// 典型场景：用户分享一张有趣的代码截图和感悟
{
    "title": null,                           // 说说通常不需要标题
    "description": "今天bug调了一整天，终于解决了！💪",
    "tag": "debug,happy",                    // 标签
    "images": "https://cdn.niter.cn/img1.jpg,https://cdn.niter.cn/img2.jpg",  // 图片列表
    "video": null,                           // 没有视频
    "creator": 456,                          // 发布者ID
    "commentCount": 12,                      // 12条评论
    "likeCount": 28,                         // 28个人点赞
    "status": 0,                             // 正常内容
    "permission": 0                          // 公开
}
```

### 2.5 Question vs Talk 对比

| 维度 | Question | Talk |
|-----|----------|------|
| **目的** | 提问、讨论、知识分享 | 日常分享、心情表达、想法碎片 |
| **内容长度** | 长文本（支持HTML）| 短文本（1024字符） |
| **标题** | 必填 | 可选 |
| **媒体** | 不支持图片/视频 | 支持图片/视频 |
| **典型场景** | "如何用SpringBoot连接MySQL？" | "今天天气真好💙" |
| **前端页面** | `/p/{id}` | `/t/{id}` |
| **模板** | `p/detail.html` | `t/detail.html` |
| **导航栏** | "社区" (communitynav) | "说说" (talknav) |
| **主要用户** | 求知者、技术分享者 | 所有用户 |

---

## 3. News（新闻）表详解

### 3.1 真实含义

**News** 表代表的是 **来自第三方新闻源的自动导入资讯**。

**核心特征**：
- ❌ **NOT** 由用户创建（系统自动抓取）
- ✅ 来自外部新闻源（如新闻API）
- ✅ 定期更新（定时任务自动导入）
- ✅ 可以被用户阅读和评论
- ✅ 展现资讯性而非讨论性内容

### 3.2 代码验证

**自动更新机制**（NewsUpdateTasks.java - 定时任务）：
```java
@Component
@Slf4j
public class NewsUpdateTasks {
    @Autowired
    private AliProvider aliProvider;

    // 每天10点7分自动更新电脑新闻
    @Scheduled(cron = "0 7 10 * * ?")
    public void updateDiannaoNewsSchedule() {
        log.info("updateDiannaoNewsSchedule start");
        aliProvider.autoGetNews(NewsColumnEnum.NEWS_COLUMN_DIANNAO.getStrId(), 1);
        log.info("updateDiannaoNewsSchedule stop");
    }
    
    // 每天8/13/18/23点的02分更新国内新闻
    @Scheduled(cron = "0 2 8,13,18,23 * * ?")
    public void updateGuoneiNewsSchedule() {
        log.info("updateGuoneiNewsSchedule start");
        aliProvider.autoGetNews(NewsColumnEnum.NEWS_COLUMN_GUONEI.getStrId(), 20);
        log.info("updateGuoneiNewsSchedule stop");
    }
    
    // 还有互联网、科技、科普、数码、体育、娱乐等频道
    // ...多个定时任务自动更新不同分类的新闻
}
```

**查询和展示**（NewsController.java）：
```java
@Controller
public class NewsController {
    @Autowired
    private NewsService newsService;

    // 新闻首页列表
    @GetMapping("/news")
    public String newsIndex(HttpServletRequest request, Model model,
                           @RequestParam(name = "page", defaultValue = "1") Integer page,
                           @RequestParam(name = "size", defaultValue = "10") Integer size,
                           @RequestParam(name = "column", defaultValue = "0") Integer column2){
        model.addAttribute("navtype", "newsnav");
        return "news/index";  // 新闻列表页面
    }

    // 新闻详情页
    @GetMapping("/news/{id}")
    public String po(@PathVariable(name = "id") Long id, 
                     HttpServletRequest request, Model model){
        NewsDTO newsDTO = newsService.getById(id);
        // 获取相关新闻（同分类）
        PaginationDTO more = newsService.listAllNews("", "", "new", 1, 10, newsDTO.getColumn2());
        List<NewsDTO> relatedNews = more.getData();
        
        // 累加阅读数
        newsService.incView(id);
        
        model.addAttribute("news", newsDTO);
        model.addAttribute("relatedNews", relatedNews);
        model.addAttribute("navtype", "newsnav");
        return "news/detail";
    }

    // API: 新闻列表查询
    @GetMapping("/news/list")
    @ResponseBody
    public Map<String, Object> newsList(@RequestParam Integer page,
                                        @RequestParam Integer size,
                                        @RequestParam Integer column2){
        PaginationDTO pagination = newsService.listAllNews(search, tag, sort, page, size, column2);
        map.put("news", pagination.getData());
        map.put("totalPage", pagination.getTotalPage());
        return map;
    }
}
```

**数据库字段解析**：
```sql
CREATE TABLE news (
    news_id BIGINT PRIMARY KEY AUTO_INCREMENT,    -- 新闻ID
    id VARCHAR(40) UNIQUE,                        -- 外部唯一ID（来自新闻源）
    title VARCHAR(128) NOT NULL,                  -- 新闻标题
    description TEXT NOT NULL,                    -- 摘要
    html TEXT NOT NULL,                           -- 完整内容（HTML格式）
    source VARCHAR(16) NOT NULL,                  -- 来源（如"新华网"、"央视"等）
    link VARCHAR(64) NOT NULL,                    -- 原始新闻链接（指向外部网站）
    pub_date VARCHAR(24) NOT NULL,               -- 发布日期
    imageurl1/2/3 VARCHAR(256),                  -- 配图URL（可以有多张）
    tag VARCHAR(128),                            -- 标签
    view_count INT DEFAULT 0,                    -- 浏览数
    comment_count INT DEFAULT 0,                 -- 评论数
    like_count INT DEFAULT 0,                    -- 点赞数
    gmt_latest_comment BIGINT NOT NULL,          -- 最新评论时间
    status INT(2) DEFAULT 1,                     -- 发布状态
    column2 INT(2) DEFAULT 0,                    -- 分栏分类
    gmt_create BIGINT NOT NULL,                  -- 创建时间（导入时间）
    gmt_modified BIGINT NOT NULL                 -- 修改时间
);
```

### 3.3 News的关键特点

| 特性 | 说明 |
|-----|------|
| **发布者** | 系统（自动导入，不是用户） |
| **来源** | 外部新闻API（阿里新闻、新华网等） |
| **更新方式** | 定时任务自动导入（定时任务）|
| **内容形式** | 标题 + 摘要 + HTML + 配图 |
| **评论支持** | ❌ **NOT** 支持用户评论（SQL显示comment_count但代码中不支持） |
| **点赞支持** | ✅ 支持 |
| **权限控制** | ✅ 支持（但通常为公开） |
| **编辑功能** | ❌ 不支持（只读内容） |
| **前端路由** | `/news/{id}` |
| **模板位置** | `templates/news/detail.html` |
| **导航栏** | "看看" (newsnav) |
| **外链** | ✅ 有原始链接（link字段指向源网站）|

### 3.4 自动导入的分类

系统定时任务会自动抓取以下分类的新闻（每天固定时间导入）：

```
NEWS_COLUMN_DIANNAO      - 电脑新闻       (每天 10:07)
NEWS_COLUMN_GUONEI       - 国内新闻       (每天 08:02, 13:02, 18:02, 23:02)
NEWS_COLUMN_HULIANWANG   - 互联网新闻     (每天 11:08)
NEWS_COLUMN_KEJI         - 科技新闻       (每天 12:09)
NEWS_COLUMN_KEPU         - 科普新闻       (每天 13:10)
NEWS_COLUMN_SHUMA        - 数码新闻       (每天 06:06, 13:06, 20:06)
NEWS_COLUMN_TIYU         - 体育新闻       (每天 07:07, 15:07, 23:07)
NEWS_COLUMN_YULE         - 娱乐新闻       (每天 12:08, 22:08)
```

### 3.5 业务示例

```java
// 典型场景：系统自动导入的新闻
{
    "id": "abc123def456",                   // 来自新闻源的唯一ID
    "title": "中国科学院院士当选国家工程院院士",
    "description": "今日获悉，8位中国科学院院士当选...",
    "html": "<p>今日获悉，8位中国科学院院士当选...</p>",
    "source": "新华网",                      // 新闻源
    "link": "http://news.xinhuanet.com/...", // 指向源网站的链接
    "pubDate": "2023-11-29",
    "imageurl1": "http://cdn.xinhuanet.com/img1.jpg",
    "tag": "科学,院士",
    "viewCount": 527,
    "likeCount": 12,
    "column2": 3,                           // 科技分栏
    "status": 1,                            // 已发布
    "gmtCreate": 1606641726000              // 导入时间
}
```

### 3.6 Question vs Talk vs News 对比

| 维度 | Question | Talk | News |
|-----|----------|------|------|
| **发布者** | 👤 用户 | 👤 用户 | 🤖 系统 |
| **来源** | 自原创 | 自原创 | 第三方新闻源 |
| **更新方式** | 用户手动发布 | 用户手动发布 | 定时任务自动导入 |
| **内容性质** | 知识问答型 | 日常分享型 | 资讯阅读型 |
| **内容长度** | 长文本(HTML) | 短文本(1024) | 长文本(HTML) |
| **标题** | 必填 | 可选 | 必填 |
| **媒体** | 无 | 图/视频 | 多张图片 |
| **可编辑** | ✅ 是 | ✅ 是 | ❌ 否 |
| **支持评论** | ✅ 是 | ✅ 是 | ❌ 否* |
| **支持点赞** | ✅ 是 | ✅ 是 | ✅ 是 |
| **前端路由** | `/p/{id}` | `/t/{id}` | `/news/{id}` |
| **导航栏** | "社区" | "说说" | "看看" |
| **用户群体** | 求知分享者 | 全部用户 | 信息消费者 |

**注：News虽然数据库有comment_count字段，但前端和业务代码中不支持用户评论新闻。

---

## 4. Comment（评论）表详解

### 4.1 真实含义

**Comment** 表代表的是 **对上述内容的回复和讨论**。它是一个 **灵活的多态表**，通过 `type` 字段支持对不同类型内容的评论。

**核心特征**：
- ✅ 由用户发起（回复其他内容的用户）
- ✅ 支持多种评论类型（通过type字段区分）
- ✅ 支持层级评论（一级评论、二级回复）
- ✅ 支持点赞
- ✅ 每条评论都会触发通知机制

### 4.2 Comment的type字段含义

```java
// CommentTypeEnum 枚举
public enum CommentTypeEnum {
    QUESTION(1, "回复问题"),              // 对Question的评论
    COMMENT(2, "回复评论（一级）"),       // 对QUESTION评论的回复（二级）
    SUB_COMMENT(3, "回复评论（二级）"),   // 对二级评论的回复（三级，但实际限制为二级）
    
    TALK(11, "回复说说"),                 // 对Talk的评论
    TALK_COMMENT(12, "回复说说评论"),     // 对TALK评论的回复
    TALK_SUB_COMMENT(13, "回复说说评论2"), // 对说说二级评论的回复
}
```

**重要的parent_id映射关系**：
```
对Question的评论:
  ├─ type=1:   parent_id = question.id
  ├─ type=2:   parent_id = comment.id (该comment的type=1)
  └─ type=3:   parent_id = comment.id (该comment的type=2)

对Talk的评论:
  ├─ type=11:  parent_id = talk.id
  ├─ type=12:  parent_id = comment.id (该comment的type=11)
  └─ type=13:  parent_id = comment.id (该comment的type=12)

对News的评论:
  ❌ 不支持（代码中没有实现）
```

### 4.3 代码验证

**评论创建**（CommentService.java）：
```java
// 当用户提交评论时
public Comment insertComment(Comment comment, CommentTypeEnum typeEnum) {
    comment.setCommentator(currentUser.getId());  // 评论者
    comment.setGmtCreate(System.currentTimeMillis());
    comment.setGmtModified(System.currentTimeMillis());
    comment.setType(typeEnum.getType());  // 设置评论类型
    comment.setParentId(targetId);        // parent_id是目标的ID
    return commentMapper.insertSelective(comment);
}
```

**触发通知机制**（CommentService中调用NotificationService）：
```java
// 评论创建后，系统会创建通知
// 如果用户A评论了用户B的问题，B会收到通知
Notification notification = new Notification();
notification.setNotifier(commentatorId);        // A（评论者）
notification.setReceiver(questionCreatorId);    // B（被评论者）
notification.setOuterid(comment.getId());       // 评论ID
notification.setType(NotificationTypeEnum.REPLY_QUESTION.getType());  // type=1
notificationMapper.insert(notification);
```

**数据库字段解析**：
```sql
CREATE TABLE comment (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,        -- 评论ID
    parent_id BIGINT NOT NULL,                   -- 父对象ID（question/talk/comment）
    type INT NOT NULL,                           -- 评论类型（1/2/3/11/12/13）
    commentator BIGINT NOT NULL,                 -- 评论者ID
    gmt_create BIGINT NOT NULL,                  -- 创建时间
    gmt_modified BIGINT NOT NULL,                -- 修改时间
    like_count BIGINT DEFAULT 0,                 -- 点赞数
    content VARCHAR(1024) NOT NULL,              -- 评论内容
    comment_count INT DEFAULT 0                  -- 直接子评论数
);
```

### 4.4 Comment的关键特点

| 特性 | 说明 |
|-----|------|
| **内容长度** | 短文本（1024字符） |
| **发布者** | 普通用户 |
| **支持对象** | Question、Talk（News不支持）|
| **层级关系** | 最多二层（评论 → 回复） |
| **点赞支持** | ✅ 支持（类型2/3/12/13）|
| **编辑功能** | ✅ 支持 |
| **触发通知** | ✅ 是（每条新评论都会触发通知）|
| **评论计数** | ✅ 父对象的comment_count会+1 |

### 4.5 业务示例

```java
// 典型场景1：用户A评论用户B的问题
{
    "id": 1001,
    "parentId": 100,                  // question.id
    "type": 1,                        // 回复问题
    "commentator": 456,               // A的ID
    "content": "试试用@Autowired注入试试",
    "likeCount": 5,
    "commentCount": 2,                // 有2条对此评论的回复
    "gmtCreate": 1606641726000
}

// 触发通知：用户B收到"有人回复了您的问题"的通知

// 典型场景2：用户C回复用户A的评论
{
    "id": 1002,
    "parentId": 1001,                 // comment.id（对问题的评论）
    "type": 2,                        // 回复评论（一级）
    "commentator": 789,               // C的ID
    "content": "@A 好建议，我试试",
    "likeCount": 1,
    "gmtCreate": 1606641830000
}

// 触发通知：用户A收到"有人回复了您的评论"的通知
```

---

## 5. 其他关键实体简介

### 5.1 Thumb（点赞/收藏）表

- **用途**：记录用户的点赞和收藏行为
- **type字段**（LikeTypeEnum 中定义）：
  - 1: 对问题的点赞
  - 2-3: 对评论的点赞
  - 11: 对说说的点赞
  - **注：说说评论的点赞（type=12/13）在代码中未实现**
- **防止重复**：通过(target_id, type, liker)的组合唯一约束
- **同步操作**：删除/添加点赞时同时更新目标表的like_count

### 5.2 Notification（通知）表

- **用途**：系统通知用户有关他们的内容的事件
- **type字段**：15种细分类型（1-5为问题相关，11-15为说说相关）
- **status字段**：0=未读，1=已读
- **冗余字段**：notifier_name和outer_title用于性能优化

### 5.3 User、UserAccount、UserInfo表

见前文档的详细说明。简单说：
- **User**：用户基本登录信息
- **UserAccount**：用户积分和等级
- **UserInfo**：用户个性化信息

---

## 6. 总体架构理解

### 6.1 内容类型分层

```
内容体系结构
│
├─ 用户生成内容（User-Generated Content）
│  ├─ Question（长文本讨论）
│  ├─ Talk（短文本分享）
│  └─ Comment（评论回复）
│
├─ 系统导入内容（System-Imported Content）
│  └─ News（资讯阅读）
│
└─ 用户交互行为
   ├─ Thumb（点赞/收藏）
   ├─ Notification（通知）
   └─ Comment（评论）
```

### 6.2 数据关系图

```
User(id)
├─ 1:1 → UserAccount (user_id)
├─ 1:1 → UserInfo (user_id)
├─ 1:N → Question (creator)
├─ 1:N → Talk (creator)
└─ 1:N → Comment (commentator)

Question(id)
├─ 1:N → Comment (parent_id, type=1)
└─ 1:N → Thumb (target_id, type=1)

Talk(id)
├─ 1:N → Comment (parent_id, type=11)
└─ 1:N → Thumb (target_id, type=11)

Comment(id)
├─ 1:N → Comment (parent_id, type=2/3)
├─ 1:N → Thumb (target_id, type=2/3)
└─ 1:N → Notification (outerid)

News(id)
└─ 1:N → Thumb (target_id, type=?) [不支持评论]
```

### 6.3 前端导航对应关系

```
首页 (/)
├─ 社区 (/forum)           → Question列表 + 顶部热门问题
├─ 说说 (/talk)            → Talk列表
├─ 看看 (/news)            → News列表
├─ 个人中心 (/user/home)   → 用户资料 + 发布历史
└─ 发布 (/p/publish)       → 发布Question表单
```

---

## 7. 查询场景总结

### 7.1 获取Question详情页需要的数据

```java
// 场景：用户访问 /p/{id}
1. Question 表：获取标题、内容、标签、统计数据
2. User 表：通过creator_id获取发布者信息
3. Comment 表：WHERE parent_id=question_id AND type=1，获取一级评论
4. Comment 表：获取每条一级评论的二级回复（type=2）
5. Thumb 表：检查当前用户是否已点赞
6. Notification 表：无需查询（评论时自动生成）
```

### 7.2 获取Talk详情页需要的数据

```java
// 场景：用户访问 /t/{id}
1. Talk 表：获取内容、统计数据
2. User 表：通过creator_id获取发布者信息
3. Comment 表：WHERE parent_id=talk_id AND type=11，获取评论
4. Comment 表：获取每条评论的回复（type=12）
5. Thumb 表：检查当前用户是否已点赞
```

### 7.3 获取News详情页需要的数据

```java
// 场景：用户访问 /news/{id}
1. News 表：获取标题、内容、图片、链接
2. Thumb 表：检查当前用户是否已点赞
3. News 表：获取同分类的相关新闻（通过column2字段）
注意：不支持用户评论
```

---

## 8. 核心设计模式

### 8.1 多态表设计（Comment）

通过 `type` 字段实现单表支持多种评论类型：
- ✅ **优点**：灵活、易于扩展新的评论类型
- ⚠️  **缺点**：type值较多，需要枚举维护

### 8.2 位运算状态标志（status）

```
status = 0 → 二进制: 00 → 正常
status = 1 → 二进制: 01 → 加精 (bit 0 = 1)
status = 2 → 二进制: 10 → 置顶 (bit 1 = 1)
status = 3 → 二进制: 11 → 加精+置顶 (bit 0 & 1 = 1)

检查是否加精：(status & 1) == 1
检查是否置顶：(status & 2) == 2
```

### 8.3 冗余字段优化（Notification）

`notifier_name` 和 `outer_title` 是冗余字段，存储在Notification中：
- ✅ 避免每次查询都JOIN多个表
- ✅ 加快列表页面的加载速度
- ⚠️  需要在源数据更新时同步更新冗余字段

### 8.4 时间戳排序（gmt_latest_comment）

在Question和Talk表中都有 `gmt_latest_comment` 字段：
- 用途：快速排序出"最新有评论的内容"
- 优化：避免COUNT等聚合查询
- 使用：ORDER BY gmt_latest_comment DESC

