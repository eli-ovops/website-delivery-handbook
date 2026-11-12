# 卓建官网从 Payload 安装到 V1.3.5：复盘与开发偏好

本文在原《卓建官网 Payload 二开复盘与开发偏好 V1.21》基础上更新，保留完整的建设脉络，并以 V1.3.5 源码校正已经变化的事实。它不是安装教程，也不是版本宣传稿，而是回答四个问题：项目为什么这样做、哪里返过工、以后怎样提前防止、哪些技术债仍然没有解决。

## 一、当前基线与事实边界

当前权威版本为不可变 tag `v1.3.5`：commit `add4e5d0b0ba04778a500ef4281255c65c723120`，tree `6c593e879bb71468c76e53f677931687887ba8a4`。前端采用 Next.js 16.2.6、React 19.2.6；CMS 采用 Payload 3.85.1、Next.js 16.2.6、SQLite 适配器和 Lexical 富文本。

本项目的四类事实必须分开：

1. **源码事实**：Collection、Global、access、BFF、显示适配、路由、测试和构建脚本。
2. **运行资产事实**：线上 SQLite、public media、private resume、env、当前 release 与 symlink。运行资产只以当次生产只读核对为准，不能从仓库脚本反推。
3. **内容事实**：中文线上已发布内容是英文增量同步的唯一源；本地 SQLite 和旧快照不是生产权威。
4. **版本状态事实**：代码可部署、远端已 push/tag、生产已上线是三个状态，必须分别证明。

早期 `PAYLOAD_FRONTEND_DATA_CONTRACT.md` 记录了重要设计起点，但其中 `eventDate` 等字段已经被 V1.3.1 移出 Payload admin/API schema。本文以 V1.3.5 为准，旧契约只用于解释决策过程。

## 二、从 create-payload-app 开始，但不止于安装

项目最初从 Payload 官方脚手架和 `create-payload-app` 的标准形态起步。脚手架能解决项目启动、数据库适配、管理后台和生成类型，但不能替甲方回答内容模型、运营工作流和权限问题。真正的二开从“前台页面怎样消费内容”开始。

选择 Payload 3.85.1 和 SQLite 的现实理由是：部署规模有限、单机运行简单、草稿与版本能力成熟、管理后台可扩展、文件与内容模型可以放在同一个运营入口。代价也很明确：SQLite、媒体、private 文件和 env 都变成运行资产；不能把数据库随源码覆盖，也不能假设多实例共享事务或限流状态。

初始化阶段应先完成五件事：

- 锁定 Node、包管理器、Payload/Next/React 版本，避免脚手架默认升级造成无关差异；
- 配置数据库适配器、迁移策略、事务行为和生成类型；
- 约定公开媒体与私有附件目录，不把 upload 当普通文本字段；
- 定义草稿、发布、版本历史和 `publishedAt` 的业务语义；
- 建立敏感资产 Git 排除和备份/恢复清单。

卓建后来把 SQLite 事务行为设为 immediate，英文导入和发布采用单批事务。这不是性能装饰，而是为了保证第 N 条失败时整批回滚、相同输入重跑零写。

## 三、先做字段消费矩阵，再建 Collection

最早容易走错的路径是看到前台栏目就创建同名 Collection，再给每个 Collection 堆很多“可能有用”的字段。这样做会出现三个问题：后台字段很多但前台不用、排序语义重复、历史字段以后不敢删。

正确顺序是：页面区块 → 用户动作 → 数据来源 → 字段 → 显示/排序/关系消费者 → 权限 → fallback。每个字段至少要记录：

- 后台谁录入，是否必填；
- API 是否公开，草稿是否隔离；
- 前端列表、详情、首页资源位是否显示；
- 是否参与排序、筛选、语言映射或关系；
- 缺失时是空白、隐藏、返回首页还是报错；
- 删除字段会影响哪些历史数据和类型。

V1.3.1 正是通过完整 consumer 矩阵确认六个字段没有正式消费者，才从 Payload admin/API schema 移除：`Cultures.eventDate`、`SocialResponsibilities.eventDate`、`PartyBuilding.eventDate`、`Lawyers.publishedAt`、`Pages.publishedAt`、`Posts.publishedAt`。这里的“移除”不等于 drop column；历史列保留，避免隐式 schema push 破坏生产数据。

仍然保留的日期字段都有真实用途：News、Culture、Social Responsibility、Party Building、Job Opening、Practice Area 的 `publishedAt` 参与显示或排序；Job Application 的 `submittedAt`、Job Opening 的 `closingDate`、Analytics 的 `occurredAt/receivedAt` 也有明确消费者。

## 四、内容模型：Global、Collection 与稳定关系

单例站点设置使用 Global，重复条目使用 Collection。Homepage、SiteSettings、EnglishSiteSettings 需要草稿/版本和匿名已发布读取；新闻、文化、社会责任、党建、律师、专业领域、招聘岗位等是条目集合。

关系字段不能只存“看起来像 slug 的字符串”。卓建的中英文映射最终依赖稳定 `sourceRecordId` 与 `sourceRoute`；相关文章、相关律师、相关专业领域要在 mapper、snapshot、display-data 和详情视图全链路保留。V1.3 首轮审计曾发现 culture/responsibility/party 的 `relatedNews` 被 display-data 主动设成 `undefined`，视图又只对 News/Insights kind 渲染。生成器“产出了关系”并不代表用户能点击，必须测试真实消费链。

删除关系也要定义生命周期。投递申请关联私有简历：删除 application 后，只有当 resume 已不再被其他 application 使用时才删除 resume 记录及物理 private 文件；共享附件不能误删。UI 二次确认、API ACL、hook 清理和失败一致性必须一起测。

## 五、后台按前台栏目和运营任务组织

用户最稳定的偏好是：后台不是给开发看的数据库管理器，而是给运营完成工作用的工具。栏目名称、导航顺序、编辑字段和提示语应对应前台栏目与日常任务。内容编辑越少越好，但每个保留字段必须真实可用。

本项目形成了三类正式角色：

| role | 界面含义 | 主要边界 |
| --- | --- | --- |
| `super-admin` | 全功能后台管理员 | 技术与全局管理，保留最高权限 |
| `admin` | 普通管理员 | 可执行被明确授权的管理动作，包括删除 job application |
| `operator` | 运营 | 日常内容运营，不可删除 job application |

最重要的教训是不能凭“管理员账号”这种口语猜 role。V1.3 第一次实现把需求解释反了，后来必须在 V1.3.1 纠正为：`super-admin/admin` 可以删除 application，`operator` 不可以。UI 中 super/admin 显示删除并二次确认，operator 不显示；API 同样必须返回对应成功或 403。`job-resumes`、`job-openings` 和 private media 的直接删除权限不能因为内部清理而扩大。

权限验收必须显式列四层：后台导航、按钮/表单、Payload access/API、底层资源。UI 隐藏不是 ACL；内部 hook 有权清理孤立附件，也不等于角色获得直接删除 resume 的权限。

Users 管理还有独立边界：super-admin 管理全部；admin 只能管理 operator 及有限的自身信息；operator 不管理用户。最后一个 active super-admin 不能被删除、停用或降级。历史兼容逻辑仍可能把缺失 role 视作 super、缺失 `isActive` 视作 active，这是已登记 P2，不应写成已经解决。

## 六、草稿、发布、发布时间和排序

普通“保存草稿”“立即发布”“发布修改”和版本历史需要保留。V1.3.1 关闭了统一 `schedulePublish` 能力和入口，但没有删除 `publishedAt`，也没有破坏普通草稿/发布。

`publishedAt` 的语义是已发布内容的业务发布时间。前端应读取发布后的时间，并按 Asia/Shanghai 格式化展示；底层数据库和 REST 仍保存标准 UTC ISO，不修改历史值。禁止直接对 UTC ISO 使用 `slice(0,10)`，否则北京时间凌晨内容可能显示成前一天。

排序优先级必须以运营意图为准：

1. 置顶和固定排名；
2. 首页推荐、栏目资源位、featured 或推荐位；
3. 显式 `sortOrder`；
4. `publishedAt` 降序，仅作为允许场景中的最低优先级或并列兜底；
5. 稳定 ID 作为最终确定性顺序。

新闻使用置顶顺序后再按发布时间；文化、社会责任和党建使用 `sortOrder` 优先、同值时 `publishedAt` 降序。V1.3 首轮审计发现三类列表虽然定义了 comparator，却没有在 API 三处调用，测试也只是正则检查函数存在。修复后测试直接传数组断言真实输出。永久规则是：测试调用链，不测试“源码里有一段看起来正确的函数”。

## 七、素材中心、富文本和媒体边界

运营素材应先进入素材中心，再由内容记录建立关系。这样可以复用、替换、审计并避免重复上传。公开图片与招聘私有简历必须物理和访问路径隔离。

公开媒体在前端通过同源 `/api/media/file` 代理读取，路径要规范化并拒绝目录穿越。Payload Media 只允许明确的栅格图片类型和大小；SVG 等高风险格式不默认开放。英文站不在运行时依赖中文 Payload media，而是由快照生成器下载、哈希并落为内容寻址的 `/english-media/<sha256>.<ext>` 静态资产。

招聘附件属于 private 资产。匿名投递只能走受控 route，不开放 Collection 通用 create；下载需要已有权限。删除 application 的内部清理必须只处理孤立 resume。当前“申请记录写入”和“附件记录/文件写入”尚未形成完整原子事务，这是明确延期 P2。

Lexical 富文本需要独立的序列化与显示适配。不要把后台 JSON 直接交给页面，也不要把正文 HTML 当不可信字符串随意插入。链接、段落、列表、图片、空节点和未知节点必须有稳定处理和安全策略。

## 八、中文动态前端与显示适配层

中文前端是动态读取：页面请求同源 `/api/site-data`，BFF 聚合 Payload 已发布资源，映射媒体、富文本和关系，再交给 display-data 转成 SiteShell 需要的稳定显示模型。SiteShell 用 hash 路由承载列表/详情和客户端交互。

这一层解决了两个问题：Payload schema 可以按运营建模，页面不用直接依赖数据库结构；前端组件只消费经过归一化的状态。不过它也容易变成“第二套数据库”。必须保证只有一个正式 display-data 真相，任何字段都不能在 BFF 生成后被显示层默默清空。

当前中文站在 Payload 请求失败时仍有 `fallbackSiteData`。这是现状，不应美化成推荐方案。新项目应把开发 fixture、静态应急页和生产 fallback 分开，生产内容不能被旧 mock 悄悄替代。

## 九、英文独立内容层

英文不是中文页面运行时翻译，而是独立发布内容层。当前静态快照包含 79 条正式英文条目（12 个专业领域、18 位律师、16 条新闻、9 条文化、10 条社会责任、10 条党建、4 个岗位）以及 4 个 Global；静态媒体和 partner logos 也有清单和哈希。

每个英文条目记录来源身份、来源路由、记录级哈希、字段级 sourceTextHash、审核状态和英文可发布状态。同步时：

- 切换前最新线上中文是唯一权威；
- source unavailable 或 stale 必须在任何写入前失败；
- 一批导入放在单个事务中，第 N 条失败全部回滚；
- 写前做深等值比较，相同输入重跑零 create、零 update、零版本、零 updatedAt 变化；
- 不允许英文页面显示中文 fallback；
- 英文没有对应内容时，关系或语言切换回 `/en`；
- 英文到中文用稳定 `sourceRoute` 回同页；
- 媒体以内容哈希落静态目录，不能引用 private 或运行时 Payload media。

V1.3 首轮审计发现多个 importer 串行写入且没有事务/等值比较，律师源请求失败还会回退本地快照。正确修复是五个正式 entry importer 共用单批事务 upsert，Global 单事务；律师正式路径在 Payload 初始化前验证在线源，正式环境禁止 snapshot fallback。本地快照只能作为显式隔离测试夹具。

当前前端 `prebuild` 会运行英文快照生成器，并存在默认 loopback 来源与重写产物路径。这会让构建隐式联网并改写快照，是明确延期 P2。未来应把“内容发布产物生成”和“应用代码构建”拆开，构建只消费已审计的不可变快照。

## 十、语言路由与无 fallback 规则

中文列表/详情是 hash route，英文是 `/en` catch-all。切换不能盲拼 slug，而应使用稳定映射：中文详情先找到 `sourceRecordId`，再找英文对应记录；找不到就回 `/en`。英文详情用 `sourceRoute` 或受控 detail route 回中文同页。

列表、详情和静态页都要覆盖。相关文章也按 locale 生成真实链接；英文关系缺目标时回英文首页，不能指向中文详情。一个链接有 href 还不够，必须点击后验证目标详情、locale、404 和返回行为。

## 十一、Asia/Shanghai 与 Analytics 自然日

数据库继续保存 UTC，这是跨系统交换和排序的标准。所有给用户或运营展示的日期统一在边界层用 `Asia/Shanghai` 格式化。中文新闻、文化、社会责任、党建的列表和详情、招聘 closingDate、后台 submittedAt 都遵守此口径；英文页面当前不展示日期，不额外增加日期 UI。

Analytics 的每日 PV/UV/点击不能用服务器 UTC 自然日切分。查询上下界和 dayKey 使用北京时间自然日，排序仍按时间戳。V1.3.1 增加了上海时区工具和凌晨跨日样例，防止本地显示与统计口径不一致。

## 十二、V1.3.4 安全维护：四条边界

### 1. Global 草稿的 REST 与 Local API 隔离

EnglishSiteSettings、Homepage、SiteSettings 允许匿名读取已发布版本，但匿名 `draft=true` 不能得到草稿。第一版只在 access 中从 `req.query/req.url` 猜 draft，独立审计用真实 `payload.findGlobal({ draft:true, overrideAccess:false, user:null })` 发现 Local API 把 draft 作为独立 operation 参数传递，因此绕过检查。

最终做法是三个 Global 共用 `beforeOperation` 边界，直接检查 `args.draft` 与 `overrideAccess`。当 `draft=true` 且 `overrideAccess=false` 时，必须有当前数据库中仍存在且 `isActive=true` 的用户；匿名、停用、已删除或用户查询异常都 fail closed。普通匿名 published 读取、活跃用户草稿和正常发布不变。

永久规则：Payload 的 REST 与 Local API 是两条真实调用路径；只直接单测 access 函数不足以证明 Local API 安全。

### 2. 停用账号旧会话

JWT/cookie 中有 `req.user` 不代表账号仍有效。`authenticatedOrPublished` 与 `englishPublished` 现在重新按 user ID 查询数据库并验证 `isActive`。账号停用、删除或查询失败后，旧会话不能继续读取草稿 collection、Global、受保护的求职投递和私有简历数据；恢复 active 后正例也要验证。Form Builder 匿名 create 治理仍是延期 P2，不在本项修复范围内。

这里没有扩大写权限，也没有改变账号 schema。测试采用“登录取得会话 → 停用 → 复用旧会话 → 拒绝 → 恢复”的真实行为链。

### 3. 公共求职与 Analytics 防滥用

前端代理不再透传客户端伪造代理头；只接受当前可信代理事实能证明的单一有效地址语义。对于非法或多跳 XFF，V1.3.5 的实际行为是不采用该头、不向内层转发，限流键回退到 UA/anonymous，而不是让整个合法业务请求必然返回 4xx。job application 和 analytics 必须在 `formData()`、`json()` 或 `arrayBuffer()` 完整缓冲前执行 Content-Length 预检和流式硬字节上限；无 Content-Length 的 chunked 超限，以及请求格式或字段校验错误，才按对应规则返回稳定 4xx。

合法简历大小、正常表单字段和 analytics 响应契约保持不变。当前限流是进程内实现，不等同于分布式防滥用；不能把它宣传成跨实例全局限流。

### 4. Demo seed 生产防误写

`seed:zhuojian-demo` 保留本地隔离测试用途，但 guard 必须在 Payload config、数据库和媒体初始化前执行。production 模式、缺显式确认、非显式隔离数据库、生产形态路径均零写失败。

第一轮 guard 只检查名称和路径文本，独立审计构造“名称合规但指向隔离根外数据库的 symlink”并成功复现逃逸。最终 guard 对可信临时根和既有目标执行 `lstat/realpath`，拒绝根/父目录/文件 symlink、目录逃逸、hard link（多链接目标）、任意 `/opt` 或用户目录目标。合法目标必须是系统临时目录下命名明确的直接子目录，数据库文件名固定且显式确认。

两轮审计的共同教训是：安全测试要走真实框架行为和真实文件系统语义，不能只对 helper 传几个普通字符串。

## 十三、版本返工的主要根因

### 需求语言没有映射到真实 role

错误路径：把“管理员”当成代码 role `admin`，或沿用旧判断而不看 UI 称呼和生产角色。正确做法：先列 `super-admin/admin/operator` 三行矩阵，再对 UI/API/Collection/private 逐项验收。

### 函数存在但没有接线

错误路径：正则检查 comparator 或 mapper 的源代码。正确做法：用输入数组断言输出顺序，用真实详情视图断言链接和路由。

### 把本地快照当成生产救命 fallback

错误路径：在线源失败时继续使用本地 snapshot，造成“看似成功但来源不新”。正确做法：生产 source unavailable/stale 写前失败；fixture 只能在隔离测试中显式传入。

### 事务只覆盖发布，不覆盖 importer

错误路径：release 原子，但前置 importer 已经留下部分 draft。正确做法：每个正式写批次自己具备事务、等值比较和故障注入测试。

### 把隐藏字段等同于物理删列

错误路径：为了后台简洁直接引入破坏性 schema push。正确做法：先达成用户可见目标，从 admin/API schema 移除；物理迁移另立门禁，历史列可安全保留。

### 用伪元素或 CSS 视觉猜测代替真实 DOM

错误路径：以为移除 class 就不会显示，或只测桌面截图。V1.3.2/V1.3.3 的 Hero 进度指示器修正说明伪元素、共享节点和移动端覆盖必须检查真实 DOM、计算样式和布局。

### 安全只测 REST 或路径字符串

错误路径：忽略 Payload Local API 的独立 operation 参数，忽略 symlink/hardlink 的真实文件系统解析。正确做法：测试完整调用入口和失败后的零写不变量。

## 十四、V1.3.5 的最小变更纪律

V1.3.5 只有两个前端文件、`+8/-2`：中文 Footer 中“由原点向量提供AI技术支持”整句由一个 anchor 包裹，沿用既有 href、target、rel 和视觉样式；中文桌面 Footer nav 顶部与左侧标题做最小对齐。移动端、英文 Footer、导航内容、链接和其他页面不变。

Git 收口中 HTTPS transport 出现超时，没有改用 GitHub Git Data API 手工造 commit/ref/tag，也没有 force。最终复用本机正常 Git 凭据并通过标准 SSH transport 完成普通 non-force push，再远端回读 branch SHA、tree、annotated tag object 和 peeled commit/tree。这里的永久规则不是“遇到超时改 SSH”，而是无论 transport 怎样，必须保持普通 Git 的 ancestry、非强制和不可变 tag 语义。

部署时，release、frontend/payload symlink、env 文件和权限必须先读当前生产事实；不得拿旧脚本或旧目录猜测。生产 env 值、数据库记录和 private 内容不进入文档或日志。

## 十五、已知但仍延期的 P2

以下事项未在 V1.3.5 修复：

1. **招聘申请与附件非原子**：文件、resume 和 application 的完整写链仍可能出现跨步骤失败，需要独立事务/补偿设计。
2. **Form Builder 公开 create 治理**：通用表单的匿名 create 边界仍需单独梳理，不能夹带在求职接口修复中。
3. **prebuild 在线重生成英文快照**：构建仍可能访问来源并重写产物，需拆分内容发布和代码构建。
4. **历史账号 role/isActive fail-open**：兼容逻辑对缺失 role/isActive 的历史用户仍可能放宽，需要数据盘点、迁移和回滚计划。

这些问题必须在新批次中先做事实门禁和影响评估。把它们列入文档不是授权实施，更不能写成“已经安全”。

## 十六、用户开发偏好总结

- 后台按前台栏目和运营任务组织，不按数据库表堆菜单。
- 内容编辑字段越少越好，但每个字段必须有真实消费者。
- 角色先写正式值和权限矩阵，口语称呼只能作为线索。
- UI 隐藏、API ACL、底层文件权限必须分别证明。
- 素材先入素材中心再关联，公开媒体和 private 附件严格隔离。
- 富文本、关系、排序和发布时间都要写业务语义，不只写类型。
- 中文线上数据是英文同步唯一权威；本地数据库和旧快照不能覆盖生产。
- 英文独立发布，禁止中文 fallback；语言切换使用稳定来源映射。
- 显式资源位、置顶、推荐和 `sortOrder` 优先于 `publishedAt`。
- 底层 UTC，用户显示和 Analytics 自然日按 Asia/Shanghai。
- 历史字段隐藏不等于物理 drop；迁移必须显式、可演练、可回滚。
- 真实 DOM、移动端、伪元素、console/network/media 是浏览器验收对象。
- 每个补丁严格最小范围；第三类变化先暂停，不顺手清技术债。
- Git 使用普通 non-force push 和不可变 annotated tag；tag 冲突立即停止。
- 发布保留即时备份、运行资产权威、原子切换和回滚点。
- 当前不使用 publisher、wrapper、dry-run、doctor 或部署锁，也不为统一流程重新建设这些工具。

## 十七、留给下一位接手人的一句话

卓建官网最难的不是 Payload 字段怎么写，而是让“用户想看到什么、运营怎样录入、API允许谁做什么、前端怎样消费、版本怎样上线”始终是同一件事。任何看起来很小的修改，都先画清消费链和权限链；任何看起来已经完成的功能，都用真实入口和真实失败方式再验证一次。

## 十八、全过程详细档案：从静态视觉到正式 CMS

这一节恢复旧版复盘的全过程信息密度。它不是对前文结论的重复，而是给第一次实际接手代码的人说明每一步应该观察什么。

### 18.1 静态视觉阶段应该保留什么

最初的静态视觉稿能够快速确认品牌、栏目和大致页面结构。它适合回答颜色、字体、留白、卡片、图片比例和移动端层次。它不适合决定正式字段、权限、草稿或关系模型。静态数组中的对象经常同时包含展示文本、模拟 ID、假链接和临时图片。

这些字段不能原样搬进 Payload。应先为每个页面标注“固定文案”“运营内容”“系统状态”“外部链接”“媒体”“关系”。固定文案未必需要 CMS。运营高频变化内容应进入 Collection 或 Global。

系统状态应由代码或 hook 产生，不给运营手填。外部链接需要协议、target、rel 和空值策略。媒体需要素材中心和 alt，而不是字符串路径。关系需要稳定 ID，而不是复制标题。

静态视觉阶段还应建立响应式基准。至少记录桌面主宽度、390 像素移动端、导航折叠、Hero 高度和卡片换行。后续动态化时要拿这些基准做“视觉未变”对照。不要把静态站的 mock 数据留作生产隐形 fallback。

如果确实需要开发 fixture，应独立命名并只在开发/测试入口使用。

### 18.2 Payload 脚手架安装清单

安装前先记录 Node 与包管理器版本。Payload、Next 和 React 必须选已验证的兼容组合。卓建当前组合是 Payload 3.85.1、Next 16.2.6、React 19.2.6。这只是本版本事实，不是所有项目的固定推荐。

运行 `create-payload-app` 后先保留最小官方结构。不要第一天就重写 admin、auth、路由和构建脚本。先验证本地启动、后台登录、生成类型和 SQLite 文件位置。确认 `payload.config.ts` 是唯一主配置入口。

确认数据库 adapter、migration 目录和事务选项。确认 upload collection 的文件目录。确认管理后台使用的 server URL 和同源策略。确认 dev、test、production 三类环境变量只记录名称，不输出值。

确认 `.gitignore` 排除 SQLite、journal/WAL、media、private 和 env。确认 seed 不会自动读取生产 DATABASE_URL。确认生产 build 不会自动执行危险 migration。确认生成类型命令不会改业务数据。

安装完成的验收不是“后台首页能开”。还要创建一条隔离记录、保存草稿、发布、读取 published，再删除 fixture。这一步验证基本生命周期和数据库写入路径。

### 18.3 工程目录怎样形成

CMS 目录负责模型、访问控制、hooks、admin、迁移、导入和测试。前端目录负责页面、BFF、显示适配、静态英文和浏览器行为。两端可以共享契约概念，但不应直接共享运行时数据库对象。CMS 生成的 Payload 类型适合后端和传输边界。

前端页面应有独立 ViewModel。脚本目录要区分 seed、正式 import、release、snapshot 和一次性校正。正式生产可达脚本必须 fail closed。测试 fixture 必须显式隔离。

一次性脚本要写适用版本、输入权威和重复运行行为。不要让测试脚本通过默认环境变量碰到生产数据库。不要把生成的 SQLite 或媒体放进源码树等待打包。

### 18.4 第一次建模的正确问题

对于每个页面，先问内容是否重复。再问运营是否需要独立发布。再问是否有列表、详情、关系和排序。再问是否需要语言 counterpart。

再问是否需要媒体、富文本或 private 附件。最后才决定 Global、Collection、数组或普通字段。Homepage 适合承载单例配置和栏目引用。News 适合 Collection，因为有列表、详情、关系和发布时间。

Lawyers 适合 Collection，因为有分类、顺序、详情和关系。SiteSettings 适合 Global，因为全站只有一个有效配置。招聘岗位适合 Collection，投递申请和简历要分开建模。不要因为页面上只有三条卡片就把它建成固定三个字段。

不要因为当前只有一个地址就把所有站点信息塞进页面组件。不要给每个模型机械加 title、slug、publishedAt、sortOrder。字段必须由消费矩阵证明。

### 18.5 字段命名和说明

字段名用于代码稳定，label 用于运营理解。内部名称应避免中文拼音和含糊缩写。后台 label 应使用自然中文。description 要说明“前台哪里显示”和“建议怎样填写”。

不要写“请输入内容”这种无信息提示。选择项要说明顺序和显示结果。关系字段要说明可多选、顺序是否重要、目标下架会怎样。日期字段要说明是发布时间、截止时间还是发生日期。

媒体字段要说明尺寸、比例、文件类型和 alt。富文本要说明允许的格式和不要粘贴什么。内部只读字段应隐藏或明确标识，不能让运营误改。系统字段 createdAt/updatedAt 不需要伪装成业务日期。

### 18.6 草稿和发布的完整语义

保存草稿允许内容尚未完整。发布必须满足必要字段和关系约束。已发布记录修改后可以形成新草稿。前台继续读取上一个 published，直到新版本发布。

匿名 API 不能因 `draft=true` 读取最新草稿。Local API 也不能因独立参数绕过。后台活跃用户按角色读取草稿。账号停用后旧 cookie/JWT 立即失去草稿能力。

普通发布与预约发布是不同能力。关闭预约发布不能顺手关闭 drafts 或 versions。发布后 `publishedAt` 应成为前端业务日期来源。如果运营可以编辑 `publishedAt`，说明其对显示和排序的影响。

如果发布动作自动补时间，说明手工值与默认值优先级。

### 18.7 后台导航产品化

后台首屏应让运营看到日常任务。菜单顺序尽量对应前台主栏目。低频技术集合不应混在运营菜单中。素材中心单独成组，避免每个内容页重复上传。

招聘投递单独呈现查看、下载和受控删除。账号管理只对有权限角色显示。Analytics 只展示业务需要的汇总和范围。后台版本文字用于确认当前 UI 版本，不代表生产验收。

导航隐藏仍需服务端 access 阻止直接 URL。自定义 Admin view 不应复制业务 access 判断。应复用共享权限 helper，同时测试 helper 的真实使用入口。

### 18.8 三角色矩阵的逐项写法

先写 `super-admin` 的职责和不可变能力。再写 `admin` 的普通管理范围。再写 `operator` 的日常运营范围。每个 Collection 分 read/create/update/delete。

每个 Global 分 read/update/draft/publish。每个自定义 route 单独列出。每个 private 文件动作单独列出。每个后台按钮与 API 结果对应。

对于 job application，当前事实是 super/admin 可删、operator 不可删。对于 job resume，内部清理与直接删除权限分开。对于 job opening，不因 application 删除能力扩大。对于 users，普通 admin 不获得最高权限角色管理能力。

对于媒体，运营内容角色可管理公开素材，但 private 简历另有边界。对于历史账号，缺 role/isActive 的兼容策略必须标为风险。每项至少一条允许正例和一条拒绝负例。

### 18.9 素材中心详细做法

媒体记录应保存文件元数据、alt 和运营可识别标题。上传限制在 CMS 端执行，前端提示只是辅助。类型检查不能只信扩展名。文件大小要同时考虑源文件和请求总大小。

图片列表需要缩略图和基本信息。关系选择器应支持搜索已有素材。替换素材时要明确旧 URL 和引用是否变化。删除素材前检查引用，避免页面 broken image。

公开媒体部署时保留共享目录和权限。英文静态媒体由发布流程复制并哈希，不直接引用中文运行时 URL。private 简历不进入素材中心公共列表。文档和测试不能包含真实简历内容。

媒体验收包括上传、选择、显示、404、缓存、移动端比例和删除关系。

### 18.10 富文本详细做法

工具栏只开放业务需要的节点。标题层级要避免破坏页面信息结构。粘贴 Word/网页内容要清理多余样式。外部链接要处理 target 与 rel。

内部链接优先使用受控 route 或关系。图片节点要引用媒体记录并具备 alt。空段落和连续换行要稳定处理。列表嵌套和长链接要检查移动端。

未知 Lexical 节点应安全忽略或显示可识别降级。显示适配不能在不同页面各自实现一套 rich text renderer。富文本 schema 更新要验证旧记录兼容。运营验收要实际编辑、保存草稿、发布并在前台查看。

### 18.11 关系字段详细做法

关系的 source 和 target Collection 必须明确。单向展示不代表不需要反向查询。多选关系的顺序是否影响页面必须说明。关系目标只读取 published，草稿目标不应泄漏到前台。

目标下架后应隐藏链接或按产品规则回退。目标删除时应有引用检查或安全降级。跨语言关系不能直接复用中文 target。需要通过 source identity 找英文 target。

英文 target 不存在时不能显示中文标题。mapper 生成后还要检查 display-data 和 view 是否保留。测试既检查数据结构，也点击真实 locale route。

### 18.12 首页推荐与资源位

首页栏目通常不是普通列表的前几条。运营需要决定哪些内容进入哪个位置。featured 表示入选，不一定决定内部顺序。sortOrder 表示显式顺序。

pin 表示固定在普通列表顶部。category 可能决定首页分栏。publishedAt 只在上述规则之后兜底。资源位容量满时要定义是否自动替换。

同一内容能否进入多个资源位要定义。下架后资源位是否自动空缺要定义。BFF 不应把所有 featured 重新按最新排序覆盖运营意图。测试要覆盖同 sortOrder、相同时间和无时间记录。

### 18.13 律师模型收敛

律师详情通常包含姓名、职位、照片、专业领域、简介和联系信息。分类 tabs 必须来自生产分类事实或稳定配置。英文 tabs 镜像中文集合时需要准确翻译和顺序。律师排序使用显式 sortOrder、姓名和稳定 ID。

V1.3.1 已确认 Lawyers.publishedAt 无正式 consumer，因此移出 schema。不要为了“所有 Collection 一致”重新加回日期。相关律师关系需要 locale counterpart。照片通过素材中心管理。

空简介、无专业领域和无英文 counterpart 都要有明确页面行为。

### 18.14 SQLite 运行资产详细治理

SQLite 主文件、WAL 和 journal 都可能影响一致性。备份前确认数据库运行模式和活跃写入。副本演练应复制成独立路径，不在生产文件上试验。`quick_check` 用于结构健康，foreign key 检查用于关系完整。

应用启动不能默认自动 drop 历史列。生产 migration 需要显式开关和版本记录。导入脚本必须接受明确数据库目标。测试数据库必须位于隔离临时根。

seed guard 在导入 Payload config 前运行。生产形态路径、用户目录、系统目录和链接逃逸都应拒绝。数据库文件不得进入 Git diff。回滚时不能用旧数据库覆盖切换后的新业务写入。

### 18.15 Migration 的设计与演练

先写为什么需要 migration。再写只改 schema、转换数据还是清理记录。统计受影响记录只保存非敏感汇总。forward 和 rollback 都要写。

大表操作评估锁和时间。关系变化检查 FK。字段重命名考虑旧 API 和生成类型。字段删除先确认所有消费者为零。

如果用户目标只是后台不显示，可以保留历史列。在生产数据副本启动当前应用，证明不会隐式 drop。migration 失败要保持事务或可恢复检查点。生产不是 migration 调试环境。

### 18.16 中文动态前端的接线细节

根页面只负责加载 SiteShell。SiteShell 管理数据请求、hash route、语言切换和 analytics。BFF 并发读取独立资源减少瀑布。BFF 只保留 `_status=published`。

媒体转换成同源公开路径。Lexical 转成页面可消费结构。关系转成轻量稳定对象。排序在明确的业务入口执行。

display-data 将传输对象转成 ViewModel。views 只负责渲染和局部交互。当前 `views.tsx` 已偏大，新复杂模块应拆分。当前 fallbackSiteData 是历史兼容，不是新项目推荐。

请求失败不能与有效空数据混淆。

### 18.17 英文内容导入细节

每个来源条目先建立稳定 source key。记录 sourceRecordId 和 sourceRoute。计算 record hash。对每个可翻译字段记录 fieldPath 和 sourceTextHash。

只发布完成审核且 enPublishable 的记录。关系目标要从 source key 转成英文 route。媒体下载后计算内容 hash。静态文件名由 hash 决定。

禁止 private 路径和未知 MIME。五类正式 entry importer 使用共同事务 upsert。Global importer 也使用单事务。写前深等值比较，完全相同就不写。

source unavailable 在 Payload 初始化前失败。正式律师导入不回退本地 snapshot。隔离测试可显式传 fixture，但不能成为生产可达默认。

### 18.18 英文 release 与 snapshot

release 前验证四个 Globals。验证预期条目集合和数量。验证所有 source hash 未 stale。验证关系目标有效。

验证媒体存在且 hash 匹配。验证无汉字泄漏。验证无 private 路径。所有条件满足后在一个事务内发布。

失败全部回滚。相同输入重跑零写。snapshot 只读取已发布英文内容。生成 translationMetadata、release manifest 和 staticMedia 清单。

当前 prebuild 会触发 snapshot generator，是延期 P2。推荐将 snapshot 生成为独立、显式、审计后的内容发布阶段。

### 18.19 双语语言切换细节

中文详情先取得当前 sourceRecordId。在英文快照找相同来源记录。找到则进入稳定英文 route。找不到则回 `/en`。

不使用中文 slug 猜英文 slug。英文详情读取 sourceRoute。再返回中文 hash detail。静态页也要在映射表中。

列表页切换到对应 locale 列表。相关文章使用相同映射函数。英文关系缺目标回英文首页或隐藏，禁止中文 fallback。测试覆盖刷新、直接 URL、返回、404 和双向点击。

### 18.20 时间与日期细节

createdAt/updatedAt 是系统审计时间。publishedAt 是业务发布时间。closingDate 是岗位截止日期。submittedAt 是投递发生时间。

occurredAt/receivedAt 是 analytics 事件与接收时间。它们不能因为类型都是 date 就共用一个业务语义。底层统一 UTC ISO。显示层明确 `Asia/Shanghai`。

排序使用时间戳。统计自然日用上海零点到下一日零点。凌晨样例必须覆盖 UTC 日期与上海日期不同的情况。英文当前不显示日期，不顺手新增 UI。

### 18.21 公开写接口的逐层保护

浏览器请求先到前端同源 route。前端 route 不透传任意 XFF/X-Real-IP。可信地址只根据实际代理链允许的语义生成。声明 Content-Length 超限立即拒绝。

无声明长度仍用流式 reader 计数。超过硬上限停止读取并返回 4xx。只有通过上限后才调用 formData/json。再校验字段、MIME 和文件大小。

CMS 专用 route 重复执行必要保护，不能只信前端。rate limit key 不应由可伪造头决定。当前限流是进程内，不宣传为分布式能力。Analytics 失败不能阻塞页面主流程。

求职成功响应不能泄露 private 路径。

### 18.22 招聘删除全链路

用户点击删除先出现二次确认。operator 不显示入口。operator 直接 DELETE 返回 403。admin/super DELETE application 成功。

服务端查询关联 resume。再检查是否有其他 application 引用。有共享引用则只删 application。没有共享引用才删 resume 记录。

由 upload 删除流程精确清理 private 物理文件。失败时不能出现 application 已删、共享文件误删。三个角色直接 DELETE job-resumes 保持原权限。内部清理使用服务端能力，不对角色开放新 ACL。

### 18.23 安全审计方法

先由实现线程提供完整 changed-files 和复现。审计线程只读，不改候选。审计回读基线 tag object、commit 和 tree。审计搜索正式调用链，而不是只看新增测试。

审计构造框架真实入口。Global 同时测 REST 和 `payload.findGlobal`。账号状态使用真实登录会话。body 上限使用 chunked 和伪造声明长度。

seed 使用真实 symlink、parent symlink 和 hardlink。失败后对比 DB hash、mtime、size 和 schema。发现 P1 后冻结 push/tag/deploy。只允许阻断项最小修复。

第二轮复审确认无第五项漂移。

### 18.24 Git 版本收口细节

从不可变 tag 建 clean branch/worktree。回读 HEAD commit/tree 和 ancestry。remote 必须唯一正确。检查远端分支是否已存在和是否可快进。

检查同名 tag 是否存在。tag 冲突立即停止，不能覆盖或移动。候选形成单一清晰 commit。普通 non-force push 明确分支。

创建 annotated tag 精确指向 commit。普通 push tag。fetch 后回读 branch SHA、tree、tag object、peeled commit/tree。三路一致后 worktree 应 clean。

HTTPS 超时时可使用正常 Git SSH transport。不能用 Git Data API 手工模拟普通 push。不能 force 或改写历史。

### 18.25 生产部署细节

生产动作必须有当前明确授权。先确认没有其他活动切换或迁移。从全新即时完整备份开始。只记录非敏感元数据和行数。

不读取或复制 env 值。不读取 private 内容。新 release 从已回读 tag 构建。数据库、public media、private、env 使用线上权威资产。

在服务器副本完成必要流程。loopback 检查页面、API 和后台。识别备份后新增业务写入。只有全部门禁通过才原子切换 symlink。

切换后做真实浏览器和 DB/媒体不变量。失败立即按代码/数据边界回滚。当前不使用 publisher、wrapper、dry-run、doctor 或部署锁。

## 十九、用户偏好的可执行说明

### 19.1 用户说“其他不要动”时

先写业务 allowlist。

再写 changed-files allowlist。

相关测试文件属于允许范围，但必须解释为何必要。

生成文件如无内容变化却只改 generatedAt，应恢复原 blob。

QA symlink、临时数据库和截图不能进入候选。

发现第三类文件立即暂停，不自行合理化。

### 19.2 用户说“后台简单一点”时

不是删除功能，而是减少无 consumer 字段和无关入口。

保留草稿、发布、关系、素材和必要权限。

用中文 label 和帮助说明降低学习成本。

危险动作二次确认。

列表列只显示运营判断需要的信息。

### 19.3 用户说“管理员”时

马上确认界面称呼和正式 role。

列 super-admin/admin/operator 当前事实。

列 UI 与 API 证据。

任何既有能力保持不变。

只给目标角色补目标动作。

不顺带扩大相关 Collection 或 private 权限。

### 19.4 用户说“按发布时间”时

先核对置顶、资源位、featured、sortOrder。

明确 publishedAt 只是默认还是并列兜底。

确认显示使用哪个已发布时间。

确认时区。

对输入数组做真实输出断言。

不要统一替换所有栏目 comparator。

### 19.5 用户说“中英文同步”时

先确认切换时中文生产权威。

列新增、变更、删除和关系 delta。

逐字段核对 provenance/hash。

stale 写前失败。

事务和幂等零写。

媒体和 locale route 同批。

无对应内容不中文 fallback。

### 19.6 用户说“上线”时

确认授权是否包含生产写入。

确认 Git tag 已远端回读。

确认当前 production release。

即时备份、服务器副本、loopback、原子切换。

真实浏览器验收。

回报 live 或 rolled back，不能用 deployable 替代。

## 二十、完整验收样例目录

### 20.1 字段样例

选择一个标题字段验证后台录入、API、列表和详情。

选择一个 publishedAt 验证 UTC、上海显示和排序兜底。

选择一个零 consumer 字段验证 admin/API 不见、历史列不 drop。

选择一个关系验证 target 下架和 locale route。

选择一个媒体验证素材中心、代理和移动端。

### 20.2 权限样例

三角色逐个登录。

记录导航和按钮差异。

直接调用同一 API。

对比 200/403 和数据不变量。

停用其中一个账号并复用旧会话。

恢复后验证正例。

### 20.3 事务样例

在隔离数据库准备 N 条输入。

在第 N 条注入失败。

验证所有 draft/版本/updatedAt 零变化。

成功运行一次。

相同输入重跑。

验证零 create/update/version。

### 20.4 浏览器样例

桌面 1440 打开首页、列表、详情和后台。

移动 390 重复关键路径。

检查导航、Hero、Footer、长英文、图片和关系。

检查 console 和 network。

检查 broken image、404 和 overflow。

真实点击语言切换和相关链接。

### 20.5 发布样例

备份前记录当前 release 和非敏感资产摘要。

副本 build 与 loopback。

原子切换。

浏览器和数据库检查。

记录新 release 与回滚点。

验证旧 release 仍可用。

## 二十一、文档维护规则

版本事实来自不可变 tag 和源码。

生产布局来自当次只读核对。

旧契约只作历史背景。

角色矩阵、字段消费和延期 P2 必须随版本更新。

不要把终端完整日志复制进复盘。

不要记录 env 值、cookie、密码、私有附件或数据库明细。

事故记录应写根因、影响、修复和永久规则。

不要用“已优化”“已完善”代替可核验事实。

任何新版本都补状态：代码可部署、已 push/tag、已上线。

## 二十二、典型决策记录：为什么最后采用这些做法

### 22.1 为什么选择“独立前端 + Payload”，而不是让 Payload 页面直接渲染全部站点

卓建前台已经形成一套稳定视觉和交互，中文页面包含较多定制 Shell、hash 路由、首页资源位和响应式规则。让运营后台和前台页面完全耦合，会使每次 CMS 字段调整都直接冲击视图，也会让后台技术结构反过来决定页面信息架构。因此最终采用独立 Next 前端，通过同源 BFF 消费 Payload published 内容。这个选择带来额外的契约和适配工作，但换来了三个边界：运营模型可以围绕录入任务设计；前端可以围绕页面 ViewModel 设计；公开浏览器不必知道 Payload 原始 schema 和内部地址。

这并不意味着“独立前端”天然更高级。新项目如果页面简单、团队很小、SEO 和预览要求适合 Payload 自带 Next 形态，也可以采用一体化。真正应复用的是先确认团队、发布、预览和契约成本，再决定形态。无论哪种形态，published/draft、媒体、关系和权限语义都必须只有一个权威。

### 22.2 为什么 BFF 必须做正式契约，而不是简单转发 REST

Payload REST 中的关系可能是 ID，也可能是展开对象；media 可能带内部 URL；Lexical 是编辑器结构；日期是 UTC；Collection 还包含草稿状态和系统字段。浏览器若直接使用这些对象，每个页面就会分别处理 null、关系展开、媒体和日期，久而久之出现多个不一致实现。卓建 BFF 因而承担 published 过滤、媒体同源化、富文本转换、关系轻量化和业务排序。

BFF 的风险是变成看不见的第二个业务数据库。V1.3 审计发现 relatedNews 在前面已经产生，却在显示层被丢弃，说明每次映射都可能破坏契约。以后新增字段时必须把传输模型和 ViewModel diff 一起审查；删除字段时必须有消费矩阵；资源位和排序不能因“统一整理响应”而改变。

### 22.3 为什么中文和英文采用不同运行时读取方式

中文是运营高频更新的权威站，运行时通过 BFF 读取已发布 Payload 内容。英文需要独立审核、无中文 fallback、静态媒体和可追溯来源，因此采用发布后快照。这样英文公开访问不依赖中文 CMS 实时可用性，也能在发布前扫描汉字、关系和媒体。

代价是英文内容发布链更长：提取中文来源、生成 provenance、导入 draft、审核、原子 release、生成 snapshot、复制静态媒体、构建和路由验收。任何一步使用旧快照兜底都会破坏“最新中文生产是唯一权威”。当前 prebuild 仍会运行 snapshot generator，说明发布产物和应用构建还没有彻底解耦，这正是为什么它被保留为 P2，而不是在普通前端维护中顺手修。

### 22.4 为什么 provenance 要做到字段级

只记录“这篇英文来自哪篇中文”不足以判断翻译是否 stale。中文正文、标题、摘要和关系可能分别变化，整条记录的更新时间也可能因为无关字段改变。字段级 `fieldPath/sourceTextHash` 让审核者知道哪一段来源发生变化；记录级 hash 用于快速判断整体；sourceRecordId/sourceRoute 用于身份和路由。

字段级治理的成本是清单更长、导入器更复杂。为避免形式主义，只有正式翻译字段进入 provenance；系统字段和派生字段不必人工审核。写入前比较来源 hash，批次内任何 stale 都整体失败，避免一半新、一半旧。相同输入重跑必须零写，否则版本历史和 updatedAt 会制造虚假变更。

### 22.5 为什么不允许中文 fallback

英文页面显示中文内容看似比空白友好，实际会破坏语言承诺、审核边界和品牌一致性，也会掩盖翻译遗漏。卓建规定英文无对应内容时回英文首页或不显示关系目标，绝不把中文正文塞进英文详情。中文到英文同页切换找不到 counterpart 也回 `/en`。

无 fallback 不等于用户看到错误。列表可以只呈现已发布英文条目；关系可以在生成阶段剔除无目标链接；route 不存在应给明确 404 或 locale 首页。关键是所有缺失行为由产品规则决定，不能由某个组件临时拿中文字段补齐。

### 22.6 为什么六个字段只从 schema 移除，不物理删列

消费矩阵证明六个字段在正式前端和后台业务链都没有 consumer，继续展示只会增加运营困惑。但生产数据库已有历史值，Payload/SQLite 的 schema 行为如果在上线时自动 drop，风险远大于用户收益。V1.3.1 因此把字段从 Collection 配置、admin/API 和生成类型移除，同时保留历史列。

这个决策体现“用户可见目标”和“物理数据治理”分开。以后如确需清理历史列，应新建 migration 批次，统计非敏感数量，在生产副本验证 forward/rollback、启动行为、quick_check 和 FK，再取得明确授权。不能因为列目前不可见就宣称数据库已经清理，也不能因为历史列还在就说用户需求没完成。

### 22.7 为什么显示用北京时间、存储仍用 UTC

运营和公众按北京时间理解发布日期、截止时间和每日统计。如果直接截取 UTC ISO 的日期部分，北京时间凌晨会显示为前一天，Analytics 也会按错误自然日切分。V1.3.1 在显示和统计边界统一 `Asia/Shanghai`，覆盖中文公开日期、岗位截止、投递时间和 Analytics dayKey。

底层仍存 UTC，原因是它适合跨系统交换、时间戳比较和夏令时无关的持久化。排序用数值时间戳，不用格式化后的 `YYYY-MM-DD`。英文当前不显示日期，因此没有为“统一”新增英文 UI。时间治理的通用原则是分别写清存储时区、显示时区和统计自然日，而不是把服务器本地时区当默认。

### 22.8 为什么排序不能全站统一成“最新优先”

官网排序不仅表达时间，还表达运营排版。置顶确保重要新闻固定；首页推荐和栏目资源位决定展示集合；`sortOrder` 让运营控制同组顺序；`publishedAt` 只解决没有更强意图时的先后。若全站统一 `-publishedAt`，后台看起来“更自动”，实际会破坏已安排的首页和专题位置。

因此每个列表都有决策表和 comparator。文化、社会责任、党建只有在相同 `sortOrder` 时才按发布时间降序。新闻先处理 pin。首页四栏先筛 published + featured 和 category。测试必须把同顺序、同日期、无日期和稳定 ID 放进一组乱序输入，验证正式 API 调用后的输出。

### 22.9 为什么招聘申请和简历要分模型

application 是业务记录，包含岗位、提交时间和表单信息；resume 是私有上传资产，涉及文件存储、下载 ACL 和物理删除。两者生命周期不同：一个 resume 可能被引用，application 删除也不必然意味着文件立即删除。分模型后可以在 hook 中检查孤立关系，并让 private 文件沿 upload 删除流程清理。

这种设计也暴露跨步骤原子性问题。匿名请求需要先处理文件，再建立记录和关系；某一步失败可能留下孤立资产。当前版本通过精确清理和失败测试降低风险，但没有把整个申请与附件写入统一为原子事务，所以明确标为 P2。文档必须区分“删除链已正确”与“创建链完全原子”这两个不同结论。

### 22.10 为什么权限 helper 仍需真实入口测试

权限 helper 可以减少重复，但 helper 名称和真实行为可能漂移。例如一个历史名称看似只允许 super/operator，实际为了正式内容角色已覆盖三类 active 用户。只看函数名会得出错误权限结论。另一个例子是 Global access 从 req.query 猜 draft，对 REST 有效，却漏掉 Local API 的独立 `args.draft`。

因此权限审计先画调用链：Admin view 使用哪个条件、Collection access 使用哪个 helper、自定义 route 是否绕过、Local API 是否设置 `overrideAccess`、底层文件如何读取。单元测试 helper 之后，还要用真实登录、真实 REST 和真实 `payload.findGlobal`。命名债可以登记重构，但不能在安全补丁中顺手改大范围。

### 22.11 为什么停用账号要重新查数据库

JWT/cookie 是某个时刻签发的身份声明。账号随后停用、删除或降权时，旧 token 仍可能包含旧 `req.user`。如果 access 只检查对象存在，停用账号就会继续读取草稿或受保护投递。V1.3.4 的 active helper 按 ID 查询当前数据库，检查存在和 `isActive`，查询异常也拒绝。

这不是完整的全站 token 撤销系统，而是对高风险读取边界的最小修复。测试必须真实登录得到会话，再停用并复用同一会话；随后恢复 active 验证正例。写权限、private resume 和账号 schema没有因此扩大。历史缺失 isActive 的兼容仍是另一项 P2。

### 22.12 为什么 body 上限必须在 formData/json 之前

如果先调用 `formData()` 或 `json()`，框架已经把完整请求缓冲进内存，之后检查文件大小只能保护业务字段，不能防止大请求占用内存。攻击者还可以省略 Content-Length 或使用 chunked 传输，绕过只看声明长度的检查。V1.3.4 同时做声明预检和流式实际字节计数，在完整解析前停止超限请求。

前端同源 route 和 CMS 接收 route 都要保护，避免攻击者绕过其中一层。上限不能任意调小到破坏合法简历；Analytics 使用更小的 JSON 上限。响应保持稳定 4xx，不泄露内部堆栈。测试覆盖正常请求、声明超限、无长度 chunked 超限和边界值。

### 22.13 为什么不能信任 X-Forwarded-For 第一项

客户端可以直接发送 XFF，反向代理还可能追加地址。如果应用无条件取第一项，攻击者就能伪造 rate limit key。正确做法依赖实际可信代理链：前端不透传客户端提供的代理头，只根据受信入口能够证明的单一地址语义生成转发信息；V1.3.5 对多跳或畸形值不采用、不向内层转发，并将限流键回退到 UA/anonymous，不因此单独拒绝其他方面合法的业务请求。

这一修复不能脱离部署事实凭空设计。若当前 Caddy/代理如何改写地址无法证明，应停止并请求最小必要事实，而不是假装修复。文档不需要记录基础设施敏感配置，只记录“哪一层负责清洗、应用接受什么形态、负例怎样返回”。

### 22.14 为什么 demo seed 要检查真实文件系统链接

路径字符串看起来位于临时目录，不代表真实目标也在临时目录。文件 symlink、父目录 symlink 和 hardlink 都可能让 SQLite 写到隔离根外。第一轮 guard 只验证名称和路径，独立审计用真实 symlink 复现了外部写入。最终 guard 在任何 Payload/config/media/DB 初始化前使用 `lstat/realpath`，校验临时根、父目录、目标和 link count。

合法 fixture 必须位于系统临时目录下命名明确的直接子目录，使用固定测试数据库名和显式确认。production、生产形态路径、任意用户目录、`/opt`、缺确认、根/父/文件 symlink、hardlink 都失败。失败测试比较目标 hash、mtime、size 和 schema，证明不是“报错但已经写了一部分”。

### 22.15 为什么 V1.3.5 仍值得单独版本

两个 Footer 细节很小，但用户要求同步线上，仍需要可审计版本。把整句技术支持文字放进单一 anchor 影响可点击范围；桌面 nav 对齐影响特定 locale 和 viewport。若直接修改 main 或移动旧 tag，日后无法回答生产究竟包含哪个视觉版本。

V1.3.5 用两个文件、`+8/-2` 证明范围，没有修改后台版本、英文 Footer、移动端或内容。HTTPS transport 超时只改变传输路径，没有改变普通 Git 语义。标准 SSH push、annotated tag 和远端三路回读保证版本可追溯。微小不等于可以省略版本纪律，反而更适合展示最小发布的标准形态。

## 二十三、如果新项目从零开始，前八周怎样安排

### 第 1 周：需求与页面盘点

完成页面树、用户旅程、内容来源和明确不做项。把视觉稿中的所有文字、图片、列表、详情、表单和链接标成固定或动态。组织产品、运营、前端和后端共同填写第一版消费矩阵。此周不追求完整 schema，也不承诺上线日期之外的技术方案。

验收物是能回答“哪个页面的哪个内容由谁在什么后台编辑”。如果仍出现“后台都可以改”“管理员都可以”这样的模糊语句，不能进入下一阶段。

### 第 2 周：模型、权限和运行资产设计

确定 Collection/Global、草稿/发布、媒体/附件、关系和排序。把口语角色映射为正式 role，完成 UI/API/文件权限矩阵。定义数据库、public media、private、env 和备份的权威位置与 Git 边界。设计迁移策略，但不在此时编造生产目录。

验收物是字段消费矩阵、模型表、权限矩阵、运行资产表和风险清单。零 consumer 字段在此阶段删掉最便宜。

### 第 3 周：Payload 基线和运营后台

用官方脚手架搭建兼容版本，完成 auth、基础 Collection/Global、草稿和生成类型。后台按运营任务组织导航，加入自然中文 label/description、素材中心和必要的列表列。先用 fixture 完成创建、草稿、发布、关系和媒体闭环。

验收不只看 admin 能开，还要让运营同事按说明独立完成一次录入和发布。

### 第 4 周：BFF、显示适配和核心页面

建立 published-only BFF、媒体代理、富文本转换和唯一 ViewModel。优先完成首页、一个列表和一个详情的纵向切片，而不是先搭所有空页面。处理 loading、空、错误、404 和媒体失败。建立排序与 route 的纯函数测试。

验收以真实动态数据从后台发布到页面为准，不接受静态 mock 截图。

### 第 5 周：关系、媒体、表单和移动端

完成相关内容、律师/专业领域等关系链，加入公开写 route 和 private 附件。实现解析前 body limit、可信代理地址和 ACL。覆盖 1440/390，处理长文本、富文本和图片比例。删除/共享附件生命周期同时设计，不留到上线前。

### 第 6 周：双语或其他内容层

如果项目需要双语，冻结来源权威和审核责任，建立 stable identity、字段级 provenance、事务导入、route map 和静态媒体。先用小批次验证 stale、回滚和零写，再扩到全量。没有双语的项目可用此周完成搜索、Analytics 和可访问性。

### 第 7 周：迁移、审计和运营试用

在生产数据副本演练 migration、导入、快照和 build。执行权限、旧会话、公开写、链接文件和失败不变量测试。让运营使用接近正式的数据完成录入、修改、发布、下架、媒体和关系。独立审计只读检查真实调用链。

### 第 8 周：Git 收口、发布和交接

从审计通过候选形成单一 commit 和不可变 tag，远端回读。生产从即时备份和只读事实开始，在服务器副本完成 loopback，再原子切换。上线后真实浏览器、后台、DB/media/private/env 不变量通过。交付操作说明、版本日志、回滚点和延期 P2。

八周只是节奏示例，不是固定工期。小项目可以合并阶段，大项目可以延长，但阶段输出和门禁不应消失。

## 二十四、如何判断文档和代码已经一致

先从文档随机选择五条事实，在 V1.3.5 源码中找到对应入口：角色删除矩阵、Global draft hook、三类列表排序、语言 route map、seed guard。再从源码随机选择五个关键入口，在文档中找到解释：BFF、English snapshot generator、JobApplications hook、Shanghai time、public write reader。

如果文档写“已修复”，测试应存在真实失败转成功证据；如果文档写“延期”，源码不应被描述成已提供完整保证；如果文档写“生产事实”，证据必须来自当次发布或只读核对，而不是旧脚本。字段和角色一旦变化，先更新矩阵，再更新复盘和 Skill 案例。

文档不应成为第二套配置。版本号、条目数、角色矩阵和路径只在项目复盘中记录，通用 Skill 使用占位和决策流程。这样同事既能准确接手卓建，也不会把卓建的偶然选择复制到下一个客户。
