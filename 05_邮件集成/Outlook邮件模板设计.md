# Outlook邮件模板设计

## 邮件模板分类

### 1. 审批请求邮件模板
**用途**: 发送给审核人的权限申请审批请求
**模板ID**: ApprovalRequestTemplate

#### 邮件主题
```
【权限审批】{申请人姓名} 申请 {权限系统} 权限 - 申请编号: {RequestID}
```

#### 邮件正文HTML模板
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; margin: 0; padding: 20px; background-color: #f5f5f5; }
        .container { max-width: 600px; margin: 0 auto; background-color: white; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .header { background-color: #0078d4; color: white; padding: 20px; text-align: center; border-radius: 8px 8px 0 0; }
        .content { padding: 30px; }
        .section { margin-bottom: 25px; }
        .section h3 { color: #323130; margin-bottom: 10px; border-bottom: 2px solid #0078d4; padding-bottom: 5px; }
        .info-table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
        .info-table td { padding: 8px 12px; border: 1px solid #edebe9; }
        .info-table .label { background-color: #f3f2f1; font-weight: bold; width: 30%; }
        .action-buttons { text-align: center; margin: 30px 0; }
        .btn { display: inline-block; padding: 12px 30px; margin: 0 10px; text-decoration: none; border-radius: 4px; font-weight: bold; }
        .btn-approve { background-color: #107c10; color: white; }
        .btn-reject { background-color: #d13438; color: white; }
        .instructions { background-color: #fff4ce; padding: 15px; border-radius: 4px; margin-top: 20px; }
        .footer { background-color: #f3f2f1; padding: 15px; text-align: center; font-size: 12px; color: #605e5c; border-radius: 0 0 8px 8px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>权限申请审批</h2>
            <p>申请编号: {RequestID}</p>
        </div>
        
        <div class="content">
            <div class="section">
                <h3>申请信息</h3>
                <table class="info-table">
                    <tr>
                        <td class="label">申请人</td>
                        <td>{Applicant}</td>
                    </tr>
                    <tr>
                        <td class="label">申请人邮箱</td>
                        <td>{ApplicantEmail}</td>
                    </tr>
                    <tr>
                        <td class="label">权限系统</td>
                        <td>{PermissionSystem}</td>
                    </tr>
                    <tr>
                        <td class="label">权限对象</td>
                        <td>{PermissionObject}</td>
                    </tr>
                    <tr>
                        <td class="label">申请原因</td>
                        <td>{Reason}</td>
                    </tr>
                    <tr>
                        <td class="label">预计使用期限</td>
                        <td>{ExpectedDuration}</td>
                    </tr>
                    <tr>
                        <td class="label">申请时间</td>
                        <td>{RequestDate}</td>
                    </tr>
                </table>
            </div>
            
            <div class="action-buttons">
                <a href="mailto:approval-system@company.com?subject=APPROVE-{RequestID}&body=审批意见：请在此处填写您的审批意见" class="btn btn-approve">同意申请</a>
                <a href="mailto:approval-system@company.com?subject=REJECT-{RequestID}&body=拒绝原因：请在此处填写拒绝原因" class="btn btn-reject">拒绝申请</a>
            </div>
            
            <div class="instructions">
                <strong>审批说明：</strong>
                <ul>
                    <li>请点击上方按钮进行审批，或直接回复此邮件</li>
                    <li>回复邮件时，请在主题中包含 "APPROVE-{RequestID}" 或 "REJECT-{RequestID}"</li>
                    <li>请在邮件内容中填写您的审批意见或拒绝原因</li>
                    <li>如有疑问，请联系系统管理员</li>
                </ul>
            </div>
        </div>
        
        <div class="footer">
            <p>此邮件由权限审批控制系统自动发送</p>
            <p>请勿直接回复此邮件，如有问题请联系系统管理员</p>
        </div>
    </div>
</body>
</html>
```

### 2. 申请确认邮件模板
**用途**: 发送给申请人的申请确认通知
**模板ID**: ApplicationConfirmationTemplate

#### 邮件主题
```
【申请确认】您的权限申请已提交 - 申请编号: {RequestID}
```

#### 邮件正文HTML模板
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; margin: 0; padding: 20px; background-color: #f5f5f5; }
        .container { max-width: 600px; margin: 0 auto; background-color: white; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .header { background-color: #0078d4; color: white; padding: 20px; text-align: center; border-radius: 8px 8px 0 0; }
        .content { padding: 30px; }
        .section { margin-bottom: 25px; }
        .section h3 { color: #323130; margin-bottom: 10px; border-bottom: 2px solid #0078d4; padding-bottom: 5px; }
        .info-table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
        .info-table td { padding: 8px 12px; border: 1px solid #edebe9; }
        .info-table .label { background-color: #f3f2f1; font-weight: bold; width: 30%; }
        .status-info { background-color: #f0f8ff; padding: 15px; border-radius: 4px; border-left: 4px solid #0078d4; }
        .footer { background-color: #f3f2f1; padding: 15px; text-align: center; font-size: 12px; color: #605e5c; border-radius: 0 0 8px 8px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>权限申请确认</h2>
            <p>申请编号: {RequestID}</p>
        </div>
        
        <div class="content">
            <div class="section">
                <h3>申请详情</h3>
                <table class="info-table">
                    <tr>
                        <td class="label">权限系统</td>
                        <td>{PermissionSystem}</td>
                    </tr>
                    <tr>
                        <td class="label">权限对象</td>
                        <td>{PermissionObject}</td>
                    </tr>
                    <tr>
                        <td class="label">申请原因</td>
                        <td>{Reason}</td>
                    </tr>
                    <tr>
                        <td class="label">预计使用期限</td>
                        <td>{ExpectedDuration}</td>
                    </tr>
                    <tr>
                        <td class="label">申请时间</td>
                        <td>{RequestDate}</td>
                    </tr>
                    <tr>
                        <td class="label">当前状态</td>
                        <td><strong>{Status}</strong></td>
                    </tr>
                </table>
            </div>
            
            <div class="status-info">
                <h4>处理状态说明：</h4>
                <p>您的权限申请已成功提交，系统已自动分配给相应的审核人进行处理。</p>
                <p>预计处理时间：3-5个工作日</p>
                <p>您将在审批完成后收到结果通知邮件。</p>
            </div>
        </div>
        
        <div class="footer">
            <p>此邮件由权限审批控制系统自动发送</p>
            <p>如有疑问，请联系系统管理员</p>
        </div>
    </div>
</body>
</html>
```

### 3. 审批结果通知邮件模板
**用途**: 发送给申请人的最终审批结果
**模板ID**: ApprovalResultTemplate

#### 邮件主题 (同意)
```
【审批通过】您的权限申请已获批准 - 申请编号: {RequestID}
```

#### 邮件主题 (拒绝)  
```
【审批拒绝】您的权限申请未获批准 - 申请编号: {RequestID}
```

#### 邮件正文HTML模板
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; margin: 0; padding: 20px; background-color: #f5f5f5; }
        .container { max-width: 600px; margin: 0 auto; background-color: white; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .header { color: white; padding: 20px; text-align: center; border-radius: 8px 8px 0 0; }
        .header.approved { background-color: #107c10; }
        .header.rejected { background-color: #d13438; }
        .content { padding: 30px; }
        .section { margin-bottom: 25px; }
        .section h3 { color: #323130; margin-bottom: 10px; border-bottom: 2px solid #0078d4; padding-bottom: 5px; }
        .info-table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
        .info-table td { padding: 8px 12px; border: 1px solid #edebe9; }
        .info-table .label { background-color: #f3f2f1; font-weight: bold; width: 30%; }
        .result-info { padding: 15px; border-radius: 4px; margin-top: 20px; }
        .result-info.approved { background-color: #f0f8f0; border-left: 4px solid #107c10; }
        .result-info.rejected { background-color: #fef0f0; border-left: 4px solid #d13438; }
        .footer { background-color: #f3f2f1; padding: 15px; text-align: center; font-size: 12px; color: #605e5c; border-radius: 0 0 8px 8px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header {StatusClass}">
            <h2>权限申请审批结果</h2>
            <p>申请编号: {RequestID}</p>
            <p><strong>{FinalStatus}</strong></p>
        </div>
        
        <div class="content">
            <div class="section">
                <h3>申请详情</h3>
                <table class="info-table">
                    <tr>
                        <td class="label">权限系统</td>
                        <td>{PermissionSystem}</td>
                    </tr>
                    <tr>
                        <td class="label">权限对象</td>
                        <td>{PermissionObject}</td>
                    </tr>
                    <tr>
                        <td class="label">申请时间</td>
                        <td>{RequestDate}</td>
                    </tr>
                    <tr>
                        <td class="label">审批时间</td>
                        <td>{ReviewDate}</td>
                    </tr>
                    <tr>
                        <td class="label">审核人</td>
                        <td>{ReviewerName}</td>
                    </tr>
                </table>
            </div>
            
            <div class="result-info {StatusClass}">
                <h4>审批意见：</h4>
                <p>{Comments}</p>
                
                <div style="margin-top: 15px;">
                    {NextStepsContent}
                </div>
            </div>
        </div>
        
        <div class="footer">
            <p>此邮件由权限审批控制系统自动发送</p>
            <p>如有疑问，请联系系统管理员</p>
        </div>
    </div>
</body>
</html>
```

### 4. 催办提醒邮件模板
**用途**: 发送给审核人的超时提醒
**模板ID**: ReminderTemplate

#### 邮件主题
```
【催办提醒】权限申请待审批 - 申请编号: {RequestID} (已超时{OverdueDays}天)
```

#### 邮件正文HTML模板
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; margin: 0; padding: 20px; background-color: #f5f5f5; }
        .container { max-width: 600px; margin: 0 auto; background-color: white; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .header { background-color: #ff8c00; color: white; padding: 20px; text-align: center; border-radius: 8px 8px 0 0; }
        .content { padding: 30px; }
        .warning-box { background-color: #fff4ce; padding: 15px; border-radius: 4px; border-left: 4px solid #ff8c00; margin-bottom: 20px; }
        .info-table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
        .info-table td { padding: 8px 12px; border: 1px solid #edebe9; }
        .info-table .label { background-color: #f3f2f1; font-weight: bold; width: 30%; }
        .action-buttons { text-align: center; margin: 30px 0; }
        .btn { display: inline-block; padding: 12px 30px; margin: 0 10px; text-decoration: none; border-radius: 4px; font-weight: bold; }
        .btn-approve { background-color: #107c10; color: white; }
        .btn-reject { background-color: #d13438; color: white; }
        .footer { background-color: #f3f2f1; padding: 15px; text-align: center; font-size: 12px; color: #605e5c; border-radius: 0 0 8px 8px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>⚠️ 权限申请催办提醒</h2>
            <p>申请编号: {RequestID}</p>
        </div>
        
        <div class="content">
            <div class="warning-box">
                <strong>提醒：</strong>以下权限申请已超过预定审批时间 {OverdueDays} 天，请尽快处理。
            </div>
            
            <table class="info-table">
                <tr>
                    <td class="label">申请人</td>
                    <td>{Applicant}</td>
                </tr>
                <tr>
                    <td class="label">权限系统</td>
                    <td>{PermissionSystem}</td>
                </tr>
                <tr>
                    <td class="label">权限对象</td>
                    <td>{PermissionObject}</td>
                </tr>
                <tr>
                    <td class="label">申请时间</td>
                    <td>{RequestDate}</td>
                </tr>
                <tr>
                    <td class="label">超时天数</td>
                    <td><strong style="color: #d13438;">{OverdueDays} 天</strong></td>
                </tr>
            </table>
            
            <div class="action-buttons">
                <a href="mailto:approval-system@company.com?subject=APPROVE-{RequestID}&body=审批意见：" class="btn btn-approve">立即批准</a>
                <a href="mailto:approval-system@company.com?subject=REJECT-{RequestID}&body=拒绝原因：" class="btn btn-reject">拒绝申请</a>
            </div>
        </div>
        
        <div class="footer">
            <p>此邮件由权限审批控制系统自动发送</p>
            <p>如需帮助，请联系系统管理员</p>
        </div>
    </div>
</body>
</html>
```

## 邮件模板变量定义

### 通用变量
```json
{
  "RequestID": "申请编号，如：REQ20250908001",
  "Applicant": "申请人姓名",
  "ApplicantEmail": "申请人邮箱地址", 
  "PermissionSystem": "权限系统名称",
  "PermissionObject": "具体权限对象描述",
  "Reason": "申请原因",
  "ExpectedDuration": "预计使用期限",
  "RequestDate": "申请提交时间",
  "Status": "当前申请状态",
  "ReviewDate": "审批完成时间",
  "ReviewerName": "审核人姓名",
  "Comments": "审批意见或拒绝原因"
}
```

### 条件变量
```json
{
  "StatusClass": "approved|rejected - 用于CSS样式",
  "FinalStatus": "审批通过|审批拒绝",
  "NextStepsContent": "后续步骤说明HTML内容",
  "OverdueDays": "超时天数"
}
```

## 邮件模板配置参数

### 发件人配置
```json
{
  "SenderName": "权限审批系统",
  "SenderEmail": "approval-system@company.com",
  "ReplyToEmail": "approval-system@company.com",
  "NoReplyAddress": "noreply@company.com"
}
```

### 邮件优先级设置
```json
{
  "ApprovalRequest": "高优先级",
  "ApplicationConfirmation": "普通优先级", 
  "ApprovalResult": "高优先级",
  "Reminder": "紧急优先级"
}
```

### 自动回复识别关键词
```json
{
  "ApprovalKeywords": ["APPROVE", "同意", "批准", "通过"],
  "RejectionKeywords": ["REJECT", "拒绝", "驳回", "不同意"],
  "SubjectPatterns": {
    "Approve": "APPROVE-{RequestID}",
    "Reject": "REJECT-{RequestID}"
  }
}
```