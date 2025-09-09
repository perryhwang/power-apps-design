# Excel导入功能原型开发指南

## 开发目标
验证Power Apps能否可靠处理Excel数据导入，包括中文字符支持、数据验证和批量处理性能。

## Step 1: 环境准备 (30分钟)

### 创建SharePoint开发站点
```powershell
# 如果有管理权限，运行以下命令
Connect-SPOService -Url https://yourtenant-admin.sharepoint.com
New-SPOSite -Url "https://yourtenant.sharepoint.com/sites/PermissionApprovalDev" -Title "权限审批系统-开发环境" -Owner "your@email.com" -StorageQuota 1000 -Template "STS#3"
```

### 创建测试用SharePoint列表
在SharePoint站点中手动创建列表 "PermissionRequestsTest":

**字段配置**:
1. RequestID (单行文本) - 必填
2. Applicant (单行文本) - 必填  
3. ApplicantEmail (单行文本) - 必填
4. PermissionSystem (选择) - 选项: ERP系统,CRM系统,OA系统,财务系统,HR系统,其他
5. PermissionObject (多行文本) - 必填
6. Reason (多行文本)
7. ExpectedDuration (单行文本)
8. Status (选择) - 默认值: 已导入

## Step 2: 创建测试Excel文件 (15分钟)

### 创建标准测试文件
创建文件: `权限申请测试数据.xlsx`

**列结构**:
| A | B | C | D | E | F |
|---|---|---|---|---|---|
| 申请人姓名 | 申请人邮箱 | 权限系统 | 权限对象 | 申请原因 | 预计使用期限 |
| 张三 | zhangsan@company.com | ERP系统 | 财务模块查询权限 | 工作需要查看财务报表 | 3个月 |
| 李四 | lisi@company.com | CRM系统 | 客户信息管理权限 | 负责客户关系维护 | 6个月 |

### 创建边界测试文件
- **大文件测试**: 1000行数据
- **特殊字符测试**: 包含特殊字符和emoji
- **格式测试**: .xls和.xlsx两种格式
- **错误数据测试**: 缺失必填字段、错误邮箱格式等

## Step 3: 开发Power Apps原型 (2小时)

### 创建新的Canvas App
1. 打开 https://make.powerapps.com
2. 创建新的空白Canvas应用
3. 命名为 "权限申请导入原型"

### 设计基础界面

#### 主界面控件
```
Screen1 布局:
┌─────────────────────────────────────────┐
│ Title: "Excel导入功能验证"                │
├─────────────────────────────────────────┤
│ [文件上传控件] AddMediaButton1           │
│ 提示: "选择Excel文件 (.xlsx, .xls)"      │
├─────────────────────────────────────────┤  
│ [验证按钮] Button_Validate              │
│ [导入按钮] Button_Import                │
├─────────────────────────────────────────┤
│ [结果显示区] Gallery_Results            │
│ 显示解析结果和验证信息                    │
├─────────────────────────────────────────┤
│ [状态标签] Label_Status                 │
│ 显示当前处理状态                         │
└─────────────────────────────────────────┘
```

#### 关键Power Apps公式

**文件验证公式**:
```powershell
// Button_Validate的OnSelect属性
If(
    IsBlank(AddMediaButton1.Media),
    Notify("请先选择Excel文件", NotificationType.Error),
    If(
        !Or(
            EndsWith(Upper(AddMediaButton1.Media.Name), ".XLSX"),
            EndsWith(Upper(AddMediaButton1.Media.Name), ".XLS")
        ),
        Notify("请选择Excel文件格式(.xlsx或.xls)", NotificationType.Error),
        If(
            AddMediaButton1.Media.Size > 10*1024*1024,
            Notify("文件大小超过10MB限制", NotificationType.Error),
            // 文件验证通过
            Set(varFileValid, true);
            Notify("文件验证通过，可以开始导入", NotificationType.Success)
        )
    )
)
```

**数据导入公式** (简化版):
```powershell
// Button_Import的OnSelect属性  
If(
    varFileValid,
    // 这里需要Power Automate流程来处理Excel解析
    Set(varImporting, true);
    // 调用Power Automate流程
    // ProcessExcelImport.Run(AddMediaButton1.Media);
    Notify("开始导入处理...", NotificationType.Information),
    Notify("请先验证文件", NotificationType.Error)
)
```

## Step 4: 开发Power Automate解析流程 (3小时)

### 创建Excel处理流程
1. 在Power Automate中创建新的即时流程
2. 命名为 "Excel数据解析验证流程"
3. 触发器选择 "PowerApps"

### 流程步骤设计

#### Step 1: 接收Power Apps文件
```json
{
  "trigger": {
    "kind": "PowerApps",
    "inputs": {
      "schema": {
        "type": "object", 
        "properties": {
          "file": {
            "type": "string",
            "format": "binary"
          }
        }
      }
    }
  }
}
```

#### Step 2: 解析Excel内容
```json
{
  "action": "Get_tables",
  "inputs": {
    "source": "@triggerBody()['file']",
    "table": "Sheet1"
  }
}
```

#### Step 3: 数据验证循环
```json
{
  "action": "Apply_to_each",
  "foreach": "@body('Get_tables')?['rows']",
  "actions": {
    "验证必填字段": {
      "type": "Condition",
      "expression": {
        "and": [
          "@not(empty(item()['申请人姓名']))",
          "@not(empty(item()['申请人邮箱']))", 
          "@not(empty(item()['权限系统']))",
          "@not(empty(item()['权限对象']))"
        ]
      }
    },
    "验证邮箱格式": {
      "type": "Condition",
      "expression": "@contains(item()['申请人邮箱'], '@')"
    }
  }
}
```

#### Step 4: 批量创建SharePoint记录
```json
{
  "action": "Apply_to_each_valid_record", 
  "foreach": "@variables('ValidRecords')",
  "actions": {
    "Create_item": {
      "inputs": {
        "site": "https://yourtenant.sharepoint.com/sites/PermissionApprovalDev",
        "list": "PermissionRequestsTest",
        "item": {
          "Title": "@concat('REQ', formatDateTime(utcNow(), 'yyyyMMdd'), padLeft(string(add(variables('RecordIndex'), 1)), 4, '0'))",
          "Applicant": "@item()['申请人姓名']",
          "ApplicantEmail": "@item()['申请人邮箱']",
          "PermissionSystem": "@item()['权限系统']", 
          "PermissionObject": "@item()['权限对象']",
          "Reason": "@item()['申请原因']",
          "ExpectedDuration": "@item()['预计使用期限']",
          "Status": "已导入"
        }
      }
    }
  }
}
```

## Step 5: 测试验证 (1小时)

### 功能测试清单
- [ ] 上传.xlsx格式文件 - 正常处理
- [ ] 上传.xls格式文件 - 正常处理  
- [ ] 上传非Excel文件 - 错误提示
- [ ] 上传超大文件(>10MB) - 大小限制提示
- [ ] 包含中文字符数据 - 正确解析
- [ ] 缺失必填字段数据 - 验证错误提示
- [ ] 错误邮箱格式 - 格式验证错误
- [ ] 1000条记录批量处理 - 性能测试

### 性能基准测试
- **小文件(10行)**: 处理时间 < 10秒
- **中文件(100行)**: 处理时间 < 30秒  
- **大文件(1000行)**: 处理时间 < 2分钟

### 错误处理测试
- 网络中断恢复
- SharePoint连接失败处理
- 部分数据导入失败处理

## 预期验证结果

### 成功标准
✅ 支持.xlsx和.xls两种格式
✅ 正确处理中文字符和特殊字符  
✅ 数据验证规则准确执行
✅ 1000条记录在2分钟内处理完成
✅ 错误信息准确定位到具体行和字段

### 可能遇到的问题和解决方案

**问题1**: Excel解析中文乱码
- **解决**: 确保Power Automate使用UTF-8编码处理

**问题2**: 大文件处理超时
- **解决**: 实现分批处理，每批处理100-200条记录

**问题3**: SharePoint批量插入性能差
- **解决**: 使用批处理API，减少单次插入调用

## 下一步行动
完成此原型后，我们将基于验证结果评估：
1. 技术可行性确认度
2. 性能优化需求
3. 开发复杂度估算
4. 进入下个验证阶段的条件

准备好开始这个原型开发了吗？需要我提供任何具体步骤的详细指导？