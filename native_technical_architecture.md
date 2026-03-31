# Tensionote 原生技术架构文档

## 1. 文档目标

本文档定义 Tensionote V1 的原生双端技术方案，服务于 iOS 和 Android 开发团队，目标是：

- 快速交付 V1
- 保持后续 V2 接入硬件和健康平台的扩展能力
- 在保证离线可用的前提下，实现稳定、易维护的原生架构

本文档与 [requirements_prompt.md](D:\code\BloodPressionMaster\requirements_prompt.md) 和 [PRD.md](D:\code\BloodPressionMaster\PRD.md) 保持一致。

## 2. 技术选型结论

采用原生双端方案：

- iOS：Swift + SwiftUI
- Android：Kotlin + Jetpack Compose

选择原生的原因：

- V2 已明确计划接入蓝牙血压计
- V2 已明确计划接入 Apple Health 和 Health Connect
- 原生对系统权限、通知、后台行为、平台健康数据和蓝牙能力支持更直接
- 原生在后续复杂排障、性能调优、系统兼容上更稳

## 3. 架构原则

整体采用分层架构：

- Presentation
- Domain
- Data

核心设计原则：

- 业务逻辑与 UI 解耦
- 数据存储与页面解耦
- 规则引擎独立，便于测试
- 国际化资源集中管理
- 为 V2 的同步、设备接入、健康平台接入预留扩展点

## 4. 模块划分

建议的业务模块：

- `feature/home`
- `feature/record`
- `feature/history`
- `feature/trend`
- `feature/reminder`
- `feature/report`
- `feature/settings`

建议的核心模块：

- `core/database`
- `core/model`
- `core/i18n`
- `core/notification`
- `core/export`
- `core/mail`
- `core/rules`
- `core/logging`

建议的未来预留模块：

- `integration/bluetooth`
- `integration/healthkit`
- `integration/healthconnect`
- `integration/sync`

## 5. 分层说明

### 5.1 Presentation 层

职责：

- 渲染界面
- 响应用户操作
- 维护页面状态
- 处理导航

不负责：

- 直接操作数据库
- 编写业务规则
- 直接拼装复杂导出逻辑

### 5.2 Domain 层

职责：

- 定义 UseCase
- 承载业务规则
- 组织提醒、报告、异常识别等核心逻辑

核心 UseCase 示例：

- `CreateBloodPressureRecordUseCase`
- `QuickCreateBloodPressureRecordUseCase`
- `UpdateBloodPressureRecordUseCase`
- `DeleteBloodPressureRecordUseCase`
- `GetRecentTwoWeeksTrendUseCase`
- `EvaluateAlertLevelUseCase`
- `GenerateReportUseCase`
- `ScheduleReminderUseCase`

### 5.3 Data 层

职责：

- 数据库存取
- 本地查询
- DTO 和 Domain Model 转换
- Repository 实现

## 6. iOS 技术方案

### 6.1 主要技术栈

- 语言：Swift
- UI：SwiftUI
- 状态管理：Observable / Observation 或 MVVM
- 本地数据库：Core Data
- 图表：Apple Charts
- 通知：UserNotifications
- PDF 导出：PDFKit / UIGraphicsPDFRenderer
- 邮件分享：MFMailComposeViewController
- 国际化：String Catalog

说明：

- 如果最低系统版本允许且团队熟悉，可以评估 SwiftData
- 但从稳定性和迁移可控性考虑，V1 更建议 Core Data

### 6.2 iOS 目录建议

```text
ios/
  App/
  Core/
    Database/
    Models/
    Rules/
    I18n/
    Notifications/
    Export/
    Mail/
  Features/
    Home/
    Record/
    History/
    Trend/
    Reminder/
    Report/
    Settings/
  Resources/
    Localizable.xcstrings
    Fonts/
```

### 6.3 iOS 状态管理建议

- 页面层使用 `ViewModel`
- `ViewModel` 依赖 `UseCase`
- `UseCase` 依赖 `Repository`
- UI 状态使用单向数据流

## 7. Android 技术方案

### 7.1 主要技术栈

- 语言：Kotlin
- UI：Jetpack Compose
- 状态管理：ViewModel + StateFlow
- 本地数据库：Room
- 图表：Vico
- 通知：AlarmManager + BroadcastReceiver，必要时结合 WorkManager
- PDF 导出：PdfDocument
- 邮件分享：Intent `ACTION_SEND` / `ACTION_SENDTO`
- 国际化：`strings.xml`

### 7.2 Android 目录建议

```text
android/
  app/
    src/main/java/.../
      core/
        database/
        model/
        rules/
        i18n/
        notification/
        export/
        mail/
      feature/
        home/
        record/
        history/
        trend/
        reminder/
        report/
        settings/
```

### 7.3 Android 状态管理建议

- Compose 页面仅负责 UI
- ViewModel 负责状态组织
- 业务逻辑放入 UseCase
- Repository 统一提供数据访问

## 8. 数据模型设计

### 8.1 血压记录实体

字段建议：

- `id: String(UUID)`
- `systolic: Int`
- `diastolic: Int`
- `heartRate: Int`
- `measuredAt: Instant`
- `timeZone: String`
- `tags: List<String>`
- `note: String?`
- `alertLevel: String`
- `createdAt: Instant`
- `updatedAt: Instant`
- `source: String = manual`
- `syncStatus: String?`
- `deletedAt: Instant?`

### 8.2 提醒实体

- `id: String(UUID)`
- `hour: Int`
- `minute: Int`
- `repeatDays: List<Int>`
- `enabled: Boolean`
- `label: String?`
- `createdAt: Instant`
- `updatedAt: Instant`

### 8.3 应用设置实体

- `id`
- `languageCode`
- `followSystemLanguage`
- `notificationEnabled`
- `defaultReportRange`
- `onboardingCompleted`

### 8.4 报告请求实体

- `id`
- `dateFrom`
- `dateTo`
- `locale`
- `generatedAt`

## 9. 数据库设计

### 9.1 表建议

- `bp_records`
- `bp_record_tags`
- `reminders`
- `app_settings`
- `report_history` 可选

### 9.2 索引建议

- `bp_records.measured_at`
- `bp_records.alert_level`
- `reminders.enabled`

### 9.3 查询场景

重点查询包括：

- 最近 14 天记录查询
- 最近 30 天报告查询
- 单条记录详情查询
- 按时间倒序分页查询
- 提醒启用列表查询

## 10. 异常规则引擎设计

V1 不做 AI 预测，使用本地规则引擎。

规则引擎输入：

- 当前记录
- 最近 14 天记录
- 当前语言环境

规则引擎输出：

- `alertLevel`
- 直接状态文案 key
- 用于首页趋势区的当前记录解释
- 用于详情页和报告的文案 key

建议的级别：

- `normal`
- `elevated`
- `high`
- `very_high`
- `volatile`

建议拆分为独立模块：

- `SingleReadingRule`
- `RepeatedHighRule`
- `VolatilityRule`
- `TrendDeviationRule`

这样做的好处：

- 易测试
- 易维护
- 易本地化
- 后续可平滑替换为更复杂规则

实现要求：

- 不应只输出抽象的异常级别给 UI
- UI 展示层应能直接拿到明确文案，例如收缩压偏高、舒张压偏高、收缩压和舒张压偏高、处于常见正常范围

## 11. 首页与趋势图技术实现

### 11.1 首页数据聚合

首页建议由单独聚合 UseCase 提供数据：

- 快速录入默认状态
- 最近 14 天趋势点
- 当前选中点或最近一条记录的直接状态说明
- 进入详细录入页的导航状态

避免页面自行拼装多个数据源。

首页职责限制：

- 首页只承载快速录入区和最近 14 天趋势区
- 首页不承载历史记录列表、报告卡片、提醒卡片和大块摘要卡片

快速录入实现建议：

- 首页直接输入收缩压、舒张压、心率
- 保存时使用当前系统时间创建记录
- 若用户需要修改时间、标签、备注，则跳转详细录入页

### 11.2 趋势图要求

- 双线展示收缩压和舒张压
- 背景展示正常范围带
- 异常点单独着色
- 支持点选查看直接状态说明
- X 轴时间密度需控制
- 小屏设备上优先可读性，不追求复杂交互

状态文案要求：

- 趋势图点选后，图表下方或固定说明区直接显示状态说明
- 文案优先说明具体含义，而不是只显示异常

## 12. 通知提醒技术实现

### 12.1 iOS

- 使用 `UNUserNotificationCenter`
- 每个提醒转换为本地重复通知
- 用户修改提醒后重新注册
- App 启动时校验本地提醒与数据库一致性

### 12.2 Android

- 使用 `AlarmManager` 注册定时提醒
- `BroadcastReceiver` 响应通知
- 开机广播后重建提醒
- 必要时增加 `WorkManager` 做兜底校验

### 12.3 统一策略

- 数据库存真相源
- 系统通知为派生结果
- 修改提醒时先改数据库，再同步系统任务

## 13. PDF 与邮件分享架构

### 13.1 报告生成流程

1. 从数据库读取时间范围内记录
2. 计算统计摘要
3. 生成趋势图快照
4. 填充本地化模板
5. 生成 PDF 文件
6. 返回文件路径给分享模块

### 13.2 邮件分享流程

1. 生成本地化邮件主题
2. 生成本地化正文摘要
3. 附带 PDF 文件
4. 调起系统邮件客户端

### 13.3 风险点

- Android 设备可能没有可用邮件客户端
- 中文和印地语 PDF 需要嵌入可用字体
- 报告图表导出需要保证清晰度和分页稳定

## 14. 国际化架构

### 14.1 语言支持

- `zh-CN`
- `en`
- `hi`

### 14.2 语言策略

- 默认跟随系统语言
- 用户可在设置页手动切换
- 系统语言不命中时回退到英文

### 14.3 国际化要求

- 所有 UI 文案统一 key 化
- 异常提示统一使用文案 key，不直接写字符串
- 标签、报告字段、邮件模板统一本地化
- 日期、时间、数字格式使用 locale 能力
- 健康提示需人工校对翻译

## 15. 日志与可观测性

V1 建议保留基础日志能力：

- 记录保存失败
- 报告生成失败
- 邮件分享失败
- 提醒注册失败

注意：

- 不记录敏感正文
- 不上传完整健康数据
- 本地日志用于开发和测试定位问题

## 16. 安全与隐私

V1 为本地存储，但仍需遵循最小化原则：

- 仅存储实现功能所需的数据
- 导出文件使用临时目录并考虑清理策略
- 不申请无关权限
- 设置页提供隐私政策、用户协议和免责声明

## 17. 测试策略

### 17.1 单元测试

- 异常规则引擎
- 报告聚合逻辑
- 时间范围查询
- 提醒调度逻辑

### 17.2 集成测试

- 录入到首页刷新链路
- 编辑记录后趋势刷新
- 生成报告
- 邮件分享参数正确性

### 17.3 UI 测试

- 首次启动
- 快速录入
- 趋势页查看
- 提醒设置
- 语言切换

### 17.4 兼容测试

- 小屏和大屏
- 深浅色模式基础兼容
- 中英文印地语界面长度变化
- Android 不同品牌机型通知行为

## 18. 开发阶段建议

### 阶段 1：基础架构

- 工程搭建
- 分层架构
- 本地数据库
- 多语言框架
- 基础设计系统组件

### 阶段 2：核心功能

- 快速录入
- 历史记录
- 首页聚合
- 趋势图
- 异常规则

### 阶段 3：增强功能

- 提醒
- 报告导出
- 邮件分享
- 设置页

### 阶段 4：质量与发布

- 测试修复
- 多语言校对
- 商店材料与合规文案

## 19. 为 V2 预留的技术接口

建议从 V1 开始预留以下扩展点：

- 数据来源枚举：手动、蓝牙、HealthKit、Health Connect
- Repository 扩展层：未来接入同步仓库
- 平台集成层：蓝牙、健康平台单独隔离
- 权限管理层：为后续健康数据读取授权预留统一入口

这样可以保证 V2 接入时不需要推翻 V1 核心结构。
