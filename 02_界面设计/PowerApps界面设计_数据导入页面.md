# Power Apps 数据导入页面设计

## 页面布局结构

### 1. 页面标题区域
```
控件类型: Label
属性配置:
- Text: "权限申请数据导入"
- Font: Open Sans Semibold
- Size: 24
- Color: RGBA(51, 51, 51, 1)
- Align: Center
- Y: 20
- Width: Parent.Width
- Height: 60
```

### 2. 导入步骤指引
```
控件类型: Container (水平布局)
包含控件:
- Step1Label: "1. 下载模板"
- Step2Label: "2. 填写数据"  
- Step3Label: "3. 上传文件"
- Step4Label: "4. 预览确认"
- Step5Label: "5. 执行导入"

样式配置:
- Direction: Horizontal
- Gap: 20
- Align: Center
- Y: 100
- Height: 50
```

### 3. 模板下载区域
```
控件类型: Container
属性配置:
- BorderColor: RGBA(166, 166, 166, 1)
- BorderStyle: Solid
- BorderThickness: 1
- Fill: RGBA(245, 245, 245, 1)
- Y: 170
- Height: 120
- Padding: 20

包含控件:
- TemplateDescLabel: "请先下载Excel导入模板，按照要求填写权限申请数据"
- DownloadTemplateButton: "下载Excel模板"
- ViewGuideButton: "查看填写说明"
```

### 4. 文件上传区域
```
控件类型: Container  
属性配置:
- BorderColor: RGBA(0, 120, 212, 1)
- BorderStyle: Dashed
- BorderThickness: 2
- Fill: RGBA(243, 248, 255, 1)
- Y: 310
- Height: 150
- Padding: 20

包含控件:
- FileUploadControl: Add Picture控件
- UploadHintLabel: "点击选择Excel文件或拖拽文件到此区域"
- SupportedFormatsLabel: "支持格式：.xlsx, .xls | 最大文件大小：10MB"
```

### 5. 文件信息显示区域
```
控件类型: Container
属性配置:
- Visible: !IsBlank(FileUploadControl.FileName)
- Y: 480
- Height: 100

包含控件:
- FileInfoIcon: Icon.DocumentPdf
- FileNameLabel: FileUploadControl.FileName
- FileSizeLabel: Round(FileUploadControl.Size / 1024, 2) & " KB"
- RemoveFileButton: "移除文件"
```

### 6. 数据验证结果区域
```
控件类型: Container
属性配置:
- Visible: varShowValidationResults
- Y: 600
- Height: 200

包含控件:
- ValidationStatusIcon: 
  - Icon.CheckMark (成功)
  - Icon.Warning (警告)
  - Icon.Cancel (错误)
- ValidationMessageLabel: 显示验证结果信息
- ErrorDetailsGallery: 错误详情列表
```

### 7. 数据预览区域
```
控件类型: Gallery
属性配置:
- Items: varPreviewData
- Visible: CountRows(varPreviewData) > 0
- Y: 820
- Height: 300
- TemplateSize: 40

模板控件:
- RowNumberLabel: ThisItem.RowNumber
- ApplicantLabel: ThisItem.Applicant
- EmailLabel: ThisItem.ApplicantEmail
- SystemLabel: ThisItem.PermissionSystem
- ObjectLabel: ThisItem.PermissionObject
```

### 8. 操作按钮区域
```
控件类型: Container (水平布局)
属性配置:
- Y: 1140
- Height: 60
- Direction: Horizontal
- Gap: 20
- Align: Center

包含控件:
- ValidateDataButton: "验证数据"
- PreviewDataButton: "预览数据"
- ImportDataButton: "执行导入"
- CancelButton: "取消"
```

## 控件详细配置

### 文件上传控件 (FileUploadControl)
```
控件类型: Add Picture
属性配置:
- Media: Self.Media
- OnChange: Set(varSelectedFile, Self); Set(varValidationResults, Blank())
- Text: "选择Excel文件"
- BorderColor: RGBA(0, 120, 212, 1)
- HoverBorderColor: RGBA(0, 90, 158, 1)
- PressedBorderColor: RGBA(0, 78, 137, 1)
```

### 下载模板按钮 (DownloadTemplateButton)
```
控件类型: Button
属性配置:
- Text: "下载Excel模板"
- OnSelect: 
  Set(varTemplateData, 
    Table(
      {申请人姓名: "张三", 申请人邮箱: "zhangsan@company.com", 权限系统: "ERP系统", 权限对象: "财务模块查询权限", 申请原因: "工作需要查看财务报表", 预计使用期限: "3个月"},
      {申请人姓名: "李四", 申请人邮箱: "lisi@company.com", 权限系统: "CRM系统", 权限对象: "客户信息管理权限", 申请原因: "负责客户关系维护", 预计使用期限: "6个月"}
    )
  );
  Export(varTemplateData, "权限申请导入模板", ExportFormat.Excel)
- Fill: RGBA(0, 120, 212, 1)
- Color: RGBA(255, 255, 255, 1)
```

### 验证数据按钮 (ValidateDataButton)
```
控件类型: Button
属性配置:
- Text: "验证数据"
- Visible: !IsBlank(FileUploadControl.FileName)
- OnSelect: 
  Set(varValidationResults, ValidateExcelData(FileUploadControl));
  Set(varShowValidationResults, true)
- Fill: RGBA(0, 120, 212, 1)
- Color: RGBA(255, 255, 255, 1)
```

### 预览数据按钮 (PreviewDataButton)
```
控件类型: Button
属性配置:
- Text: "预览数据"
- Visible: varValidationResults.IsValid
- OnSelect: 
  Set(varPreviewData, ParseExcelData(FileUploadControl))
- Fill: RGBA(16, 124, 16, 1)
- Color: RGBA(255, 255, 255, 1)
```

### 执行导入按钮 (ImportDataButton)
```
控件类型: Button
属性配置:
- Text: "执行导入"
- Visible: CountRows(varPreviewData) > 0
- OnSelect: 
  Set(varImporting, true);
  ForAll(varPreviewData,
    Patch(PermissionRequests, Defaults(PermissionRequests),
      {
        Title: GenerateRequestID(),
        Applicant: ThisRecord.Applicant,
        ApplicantEmail: ThisRecord.ApplicantEmail,
        PermissionSystem: ThisRecord.PermissionSystem,
        PermissionObject: ThisRecord.PermissionObject,
        Reason: ThisRecord.Reason,
        ExpectedDuration: ThisRecord.ExpectedDuration,
        Status: "待审批",
        RequestDate: Now()
      }
    )
  );
  Set(varImporting, false);
  Set(varImportComplete, true);
  Notify("数据导入完成！共导入 " & CountRows(varPreviewData) & " 条记录", NotificationType.Success)
- Fill: RGBA(0, 176, 80, 1)
- Color: RGBA(255, 255, 255, 1)
```

## 页面变量定义

```
全局变量:
- varSelectedFile: 选中的文件信息
- varValidationResults: 数据验证结果
- varShowValidationResults: 是否显示验证结果
- varPreviewData: 预览数据集合
- varImporting: 导入进行中标记
- varImportComplete: 导入完成标记
- varTemplateData: 模板数据

集合变量:
- colValidationErrors: 验证错误集合
- colPreviewData: 预览数据集合
```

## 数据验证逻辑

### 文件格式验证
```
If(
  Right(Upper(FileUploadControl.FileName), 4) In [".XLS", "XLSX"],
  {IsValid: true, Message: "文件格式正确"},
  {IsValid: false, Message: "请选择Excel文件格式(.xlsx或.xls)"}
)
```

### 文件大小验证
```
If(
  FileUploadControl.Size <= 10 * 1024 * 1024,
  {IsValid: true, Message: "文件大小符合要求"},
  {IsValid: false, Message: "文件大小超过10MB限制"}
)
```

### 必填字段验证
```
ForAll(ParsedData,
  If(
    IsBlank(Applicant) || IsBlank(ApplicantEmail) || IsBlank(PermissionSystem) || IsBlank(PermissionObject),
    Collect(colValidationErrors, 
      {
        RowNumber: RowIndex,
        ErrorType: "必填字段缺失",
        ErrorField: "申请人姓名/邮箱/权限系统/权限对象",
        ErrorContent: "存在空值",
        Suggestion: "请检查并填写所有必填字段"
      }
    )
  )
)
```

### 邮箱格式验证
```
ForAll(ParsedData,
  If(
    !IsMatch(ApplicantEmail, "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"),
    Collect(colValidationErrors,
      {
        RowNumber: RowIndex,
        ErrorType: "格式错误",
        ErrorField: "申请人邮箱",
        ErrorContent: ApplicantEmail,
        Suggestion: "请输入正确的邮箱格式"
      }
    )
  )
)
```

## 用户体验优化

### 进度指示器
```
控件类型: Progress Bar
属性配置:
- Min: 0
- Max: 100
- Value: 
  Switch(
    true,
    IsBlank(FileUploadControl.FileName), 0,
    varShowValidationResults, 40,
    CountRows(varPreviewData) > 0, 80,
    varImportComplete, 100,
    20
  )
- Visible: varImporting
```

### 加载动画
```
控件类型: Loading Spinner
属性配置:
- Visible: varImporting
- Size: LoadingSpinnerSize.Medium
- Color: RGBA(0, 120, 212, 1)
```

### 成功提示
```
控件类型: Container
属性配置:
- Visible: varImportComplete
- Fill: RGBA(223, 246, 221, 1)
- BorderColor: RGBA(40, 167, 69, 1)

包含控件:
- SuccessIcon: Icon.CheckMark
- SuccessMessage: "数据导入成功完成！"
- ViewResultsButton: "查看导入结果"
```