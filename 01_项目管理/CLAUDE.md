# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

这是一个 **Power Apps权限审批控制系统** - 基于Microsoft Power Platform构建的综合企业级解决方案。该系统通过Excel数据导入、Outlook智能邮件分发和审批跟踪，实现权限申请工作流的自动化。

## 系统架构

系统采用分层架构设计：

```
前端层: Power Apps Canvas App
    ↓
业务逻辑层: Power Automate Flows  
    ↓
数据存储层: SharePoint Lists + OneDrive
    ↓
集成层: Office 365 APIs + Outlook
    ↓
分析层: Power BI + Excel Online
```

**核心数据流**: Excel导入 → 权限分配 → 邮件分发 → 审批反馈 → 状态跟踪 → 报告分析

## 关键组件

### 1. 数据导入引擎 (`PowerApps数据导入配置.json`)
- Excel文件处理和验证规则
- 支持.xlsx/.xls格式，最大1000条记录，10MB限制
- 申请人姓名、申请人邮箱、权限系统、权限对象的架构验证
- 数据清洗和重复检测

### 2. 邮件处理系统
- **出站邮件**: 基于权限系统映射的审核人自动分发
- **入站邮件**: AI驱动的邮件回复解析，识别审批决定
- **邮件模板**: 4个主要模板（审批请求、申请确认、审批结果、催办提醒）
- **决策识别**: 支持中英文关键词，带置信度评分

### 3. 状态机 (`审批流程状态机设计.md`)
覆盖完整生命周期的16个状态：
- 初始阶段: 草稿 → 已提交 → 分配审核人中
- 处理阶段: 待审批 → 审批中 → 需要补充信息
- 最终阶段: 审批通过/拒绝 → 已实施 → 已过期
- 异常处理: 超时 → 已升级 → 无匹配审核人

### 4. SharePoint数据模型
核心列表：
- **PermissionRequests**: 主要申请数据
- **ReviewerConfig**: 审核人与系统映射关系
- **ApprovalHistory**: 完整审计跟踪
- **ApprovalTracking**: 个人审核人状态

## 开发环境设置

### 前置条件
- 具有Power Platform许可证的Office 365租户
- SharePoint Online适当权限
- Power Apps和Power Automate访问权限
- 安装SharePoint PnP模块的PowerShell

### 环境配置
```powershell
# SharePoint站点创建
Connect-SPOService -Url https://tenant-admin.sharepoint.com
New-SPOSite -Url "https://tenant.sharepoint.com/sites/PermissionApproval" -Title "权限审批控制系统"

# Power Apps部署
Import-PowerApp -PackagePath $AppPackagePath -Environment $Environment
```

### 配置文件
- `PowerApps数据导入配置.json`: Excel导入验证规则和架构
- `系统集成与部署指南.md`: 完整部署程序和环境设置
- `产品需求说明书.md`: 综合功能需求和规格说明

## 系统使用

### 文件组织
代码库按功能模块组织：
- **数据导入**: `权限申请导入模板.md`, `PowerApps界面设计_数据导入页面.md`
- **邮件集成**: `Outlook邮件模板设计.md`, `Power Automate邮件自动分发流程.md` 
- **审批处理**: `邮件回复解析引擎设计.md`, `审批流程状态机设计.md`
- **报告功能**: `审批状态跟踪界面设计.md`, `报告分析功能设计.md`
- **系统管理**: `审核人管理界面设计.md`

### 关键设计模式
- **多审核人协调**: 并行审批，支持可配置决策逻辑（全部同意、任意同意、多数决定）
- **智能路由**: 基于权限系统范围的自动审核人分配
- **状态驱动工作流**: 带转换验证和回滚支持的综合状态机
- **邮件驱动交互**: 支持Web UI和基于邮件的审批，带NLP解析

### 系统集成点
- **Excel处理**: 可配置验证规则，批量处理最多1000条记录
- **Outlook集成**: 双向邮件处理，包含模板生成和回复解析
- **SharePoint存储**: 关系型数据模型，具有适当索引和权限
- **Power BI分析**: 实时仪表板和定时报告生成

## 部署环境
- **开发环境**: 完整日志记录，48小时SLA，测试数据
- **测试环境**: UAT环境，类生产配置
- **生产环境**: 72小时SLA，最小日志记录，生产安全设置

系统设计为高可用性，具有自动故障转移、全面日志记录和灾难恢复程序，详细文档记录在 `系统集成与部署指南.md` 中。