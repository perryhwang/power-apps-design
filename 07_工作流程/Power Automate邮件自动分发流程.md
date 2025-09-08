# Power Automate 邮件自动分发流程设计

## 流程架构概览

### 主流程：权限申请邮件分发
**流程名称**: PermissionApprovalEmailDistribution
**触发器**: SharePoint - 当项目创建或修改时
**触发条件**: PermissionRequests列表中状态变为"待审批"

## 1. 流程触发器配置

### SharePoint触发器
```json
{
  "type": "SharePointWhenAnItemIsCreatedOrModified",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "PermissionRequests",
      "since": "@utcNow()",
      "filter": "Status eq '待审批'"
    }
  },
  "conditions": {
    "triggerConditions": [
      "@equals(triggerBody()['Status']?['Value'], '待审批')",
      "@not(empty(triggerBody()['Applicant']))"
    ]
  }
}
```

## 2. 初始化变量和参数

### 全局变量初始化
```json
{
  "type": "InitializeVariable",
  "inputs": {
    "variables": [
      {
        "name": "RequestDetails",
        "type": "object",
        "value": {
          "RequestID": "@triggerBody()['RequestID']",
          "Applicant": "@triggerBody()['Applicant']", 
          "ApplicantEmail": "@triggerBody()['ApplicantEmail']",
          "PermissionSystem": "@triggerBody()['PermissionSystem']?['Value']",
          "PermissionObject": "@triggerBody()['PermissionObject']",
          "Reason": "@triggerBody()['Reason']",
          "ExpectedDuration": "@triggerBody()['ExpectedDuration']",
          "RequestDate": "@formatDateTime(triggerBody()['RequestDate'], 'yyyy年MM月dd日 HH:mm')"
        }
      },
      {
        "name": "MatchedReviewers",
        "type": "array",
        "value": []
      },
      {
        "name": "EmailsSent",
        "type": "integer", 
        "value": 0
      },
      {
        "name": "EmailErrors",
        "type": "array",
        "value": []
      }
    ]
  }
}
```

## 3. 智能审核人匹配

### 获取审核人配置
```json
{
  "type": "SharePointGetItems",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ReviewerConfig",
      "query": {
        "filter": "IsActive eq true",
        "orderby": "Priority asc"
      }
    }
  },
  "runAfter": {
    "初始化变量": ["Succeeded"]
  }
}
```

### 审核人匹配逻辑
```json
{
  "type": "ForEach",
  "foreach": "@body('获取审核人配置')?['value']",
  "actions": {
    "检查系统权限匹配": {
      "type": "Condition",
      "expression": {
        "or": [
          {
            "contains": ["@items('ForEach')?['SystemScope']", "@variables('RequestDetails')['PermissionSystem']"]
          },
          {
            "equals": ["@items('ForEach')?['SystemScope']", "全部系统"]
          }
        ]
      },
      "actions": {
        "添加匹配审核人": {
          "type": "AppendToArrayVariable",
          "inputs": {
            "name": "MatchedReviewers",
            "value": {
              "ReviewerName": "@items('ForEach')?['ReviewerName']",
              "ReviewerEmail": "@items('ForEach')?['ReviewerEmail']",
              "SystemScope": "@items('ForEach')?['SystemScope']",
              "Priority": "@items('ForEach')?['Priority']"
            }
          }
        }
      }
    }
  },
  "runAfter": {
    "获取审核人配置": ["Succeeded"]
  }
}
```

### 无匹配审核人处理
```json
{
  "type": "Condition",
  "expression": {
    "equals": ["@length(variables('MatchedReviewers'))", 0]
  },
  "actions": {
    "更新申请状态为无审核人": {
      "type": "SharePointUpdateItem",
      "inputs": {
        "parameters": {
          "site": "@parameters('SharePointSiteUrl')",
          "list": "PermissionRequests",
          "id": "@triggerBody()['ID']",
          "item": {
            "Status": {
              "Value": "无匹配审核人"
            },
            "Comments": "系统未找到负责该权限系统的审核人，请联系管理员",
            "LastModified": "@utcNow()"
          }
        }
      }
    },
    "发送无审核人通知": {
      "type": "Office365SendEmail",
      "inputs": {
        "parameters": {
          "To": "@parameters('SystemAdminEmail')",
          "Subject": "【系统警告】权限申请无匹配审核人 - @{variables('RequestDetails')['RequestID']}",
          "Body": "<p><strong>权限申请无法分配审核人</strong></p><p>申请编号：@{variables('RequestDetails')['RequestID']}</p><p>权限系统：@{variables('RequestDetails')['PermissionSystem']}</p><p>请检查审核人配置或手动指定审核人。</p>",
          "IsHtml": true,
          "Importance": "High"
        }
      }
    }
  },
  "else": {
    "actions": {
      "执行邮件分发": {}
    }
  }
}
```

## 4. 邮件模板处理

### 获取邮件模板
```json
{
  "type": "SharePointGetItems", 
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "EmailTemplates",
      "query": {
        "filter": "TemplateID eq 'ApprovalRequestTemplate' and IsActive eq true"
      }
    }
  }
}
```

### 模板变量替换函数
```json
{
  "type": "Compose",
  "inputs": "@replace(replace(replace(replace(replace(replace(replace(body('获取邮件模板')?['value'][0]?['HTMLContent'], '{RequestID}', variables('RequestDetails')['RequestID']), '{Applicant}', variables('RequestDetails')['Applicant']), '{ApplicantEmail}', variables('RequestDetails')['ApplicantEmail']), '{PermissionSystem}', variables('RequestDetails')['PermissionSystem']), '{PermissionObject}', variables('RequestDetails')['PermissionObject']), '{Reason}', variables('RequestDetails')['Reason']), '{ExpectedDuration}', variables('RequestDetails')['ExpectedDuration'])"
}
```

## 5. 批量邮件发送

### 审核人邮件发送循环
```json
{
  "type": "ForEach",
  "foreach": "@variables('MatchedReviewers')",
  "actions": {
    "个性化邮件内容": {
      "type": "Compose", 
      "inputs": "@replace(outputs('模板变量替换'), '{ReviewerName}', items('ForEach')?['ReviewerName'])"
    },
    "发送审批请求邮件": {
      "type": "Office365SendEmail",
      "inputs": {
        "parameters": {
          "To": "@items('ForEach')?['ReviewerEmail']",
          "CC": "@if(parameters('CCToApplicant'), variables('RequestDetails')['ApplicantEmail'], '')",
          "Subject": "【权限审批】@{variables('RequestDetails')['Applicant']} 申请 @{variables('RequestDetails')['PermissionSystem']} 权限 - 申请编号: @{variables('RequestDetails')['RequestID']}",
          "Body": "@outputs('个性化邮件内容')",
          "IsHtml": true,
          "Importance": "High",
          "Headers": {
            "X-Request-ID": "@variables('RequestDetails')['RequestID']",
            "X-Approval-System": "PowerApps-PermissionControl"
          }
        }
      },
      "runAfter": {
        "个性化邮件内容": ["Succeeded"]
      },
      "runtimeConfiguration": {
        "retry": {
          "count": 3,
          "interval": "PT30S",
          "type": "exponential"
        }
      }
    },
    "记录发送成功": {
      "type": "IncrementVariable",
      "inputs": {
        "name": "EmailsSent",
        "value": 1
      },
      "runAfter": {
        "发送审批请求邮件": ["Succeeded"]
      }
    },
    "记录发送失败": {
      "type": "AppendToArrayVariable", 
      "inputs": {
        "name": "EmailErrors",
        "value": {
          "ReviewerEmail": "@items('ForEach')?['ReviewerEmail']",
          "ErrorMessage": "@body('发送审批请求邮件')['error']['message']",
          "Timestamp": "@utcNow()"
        }
      },
      "runAfter": {
        "发送审批请求邮件": ["Failed"]
      }
    }
  },
  "runAfter": {
    "模板变量替换": ["Succeeded"]
  },
  "runtimeConfiguration": {
    "concurrency": {
      "repetitions": 3
    }
  }
}
```

## 6. 申请人确认邮件

### 发送申请确认邮件
```json
{
  "type": "Office365SendEmail",
  "inputs": {
    "parameters": {
      "To": "@variables('RequestDetails')['ApplicantEmail']",
      "Subject": "【申请确认】您的权限申请已提交 - 申请编号: @{variables('RequestDetails')['RequestID']}",
      "Body": "@{申请确认邮件HTML模板}",
      "IsHtml": true,
      "Importance": "Normal"
    }
  },
  "runAfter": {
    "ForEach审核人发送": ["Succeeded", "Failed"]
  }
}
```

## 7. 更新申请状态

### 更新SharePoint记录
```json
{
  "type": "SharePointUpdateItem",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "PermissionRequests", 
      "id": "@triggerBody()['ID']",
      "item": {
        "Status": {
          "Value": "@if(greater(variables('EmailsSent'), 0), '已发送审批邮件', '邮件发送失败')"
        },
        "AssignedReviewers": "@join(select(variables('MatchedReviewers'), item()['ReviewerEmail']), ';')",
        "EmailSentCount": "@variables('EmailsSent')",
        "EmailSentTime": "@utcNow()",
        "LastModified": "@utcNow()"
      }
    }
  }
}
```

## 8. 创建审批跟踪记录

### 批量创建审批记录
```json
{
  "type": "ForEach",
  "foreach": "@variables('MatchedReviewers')",
  "actions": {
    "创建审批跟踪": {
      "type": "SharePointCreateItem",
      "inputs": {
        "parameters": {
          "site": "@parameters('SharePointSiteUrl')",
          "list": "ApprovalTracking",
          "item": {
            "Title": "@concat(variables('RequestDetails')['RequestID'], '-', items('ForEach')?['ReviewerEmail'])",
            "RequestID": "@variables('RequestDetails')['RequestID']",
            "ReviewerEmail": "@items('ForEach')?['ReviewerEmail']",
            "ReviewerName": "@items('ForEach')?['ReviewerName']",
            "AssignedDate": "@utcNow()",
            "Status": "待审批",
            "DueDate": "@addDays(utcNow(), parameters('ApprovalTimeoutDays'))"
          }
        }
      }
    }
  }
}
```

## 9. 定时催办机制

### 催办流程触发器
**流程名称**: ApprovalReminderFlow
**触发器**: 计划触发器 - 每日检查

```json
{
  "type": "Recurrence",
  "inputs": {
    "frequency": "Day",
    "interval": 1,
    "schedule": {
      "hours": ["09", "14"],
      "minutes": [0]
    },
    "timeZone": "China Standard Time"
  }
}
```

### 获取超时申请
```json
{
  "type": "SharePointGetItems",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ApprovalTracking",
      "query": {
        "filter": "Status eq '待审批' and DueDate lt '@{utcNow()}'",
        "orderby": "DueDate asc"
      }
    }
  }
}
```

### 发送催办邮件
```json
{
  "type": "ForEach",
  "foreach": "@body('获取超时申请')?['value']",
  "actions": {
    "计算超时天数": {
      "type": "Compose",
      "inputs": "@div(sub(ticks(utcNow()), ticks(items('ForEach')?['DueDate'])), 864000000000)"
    },
    "发送催办邮件": {
      "type": "Office365SendEmail",
      "inputs": {
        "parameters": {
          "To": "@items('ForEach')?['ReviewerEmail']",
          "CC": "@parameters('SystemAdminEmail')",
          "Subject": "【催办提醒】权限申请待审批 - 申请编号: @{items('ForEach')?['RequestID']} (已超时@{outputs('计算超时天数')}天)",
          "Body": "@{催办邮件HTML模板}",
          "IsHtml": true,
          "Importance": "High"
        }
      }
    },
    "更新催办记录": {
      "type": "SharePointUpdateItem",
      "inputs": {
        "parameters": {
          "site": "@parameters('SharePointSiteUrl')",
          "list": "ApprovalTracking",
          "id": "@items('ForEach')?['ID']",
          "item": {
            "ReminderCount": "@add(if(empty(items('ForEach')?['ReminderCount']), 0, items('ForEach')?['ReminderCount']), 1)",
            "LastReminderDate": "@utcNow()"
          }
        }
      }
    }
  }
}
```

## 10. 错误处理和日志记录

### 全局错误处理
```json
{
  "type": "Scope",
  "actions": {
    "所有邮件分发逻辑": "..."
  },
  "runAfter": {},
  "runtimeConfiguration": {
    "trackedProperties": {
      "RequestID": "@variables('RequestDetails')['RequestID']",
      "Applicant": "@variables('RequestDetails')['Applicant']"
    }
  }
}
```

### 错误恢复处理  
```json
{
  "type": "Scope",
  "actions": {
    "记录错误日志": {
      "type": "SharePointCreateItem",
      "inputs": {
        "parameters": {
          "site": "@parameters('SharePointSiteUrl')",
          "list": "SystemErrorLogs",
          "item": {
            "Title": "邮件分发失败-@{variables('RequestDetails')['RequestID']}",
            "RequestID": "@variables('RequestDetails')['RequestID']",
            "ErrorType": "EmailDistribution",
            "ErrorMessage": "@{result('Scope')[0]['error']['message']}",
            "ErrorDetails": "@{result('Scope')[0]['error']}",
            "OccurredTime": "@utcNow()"
          }
        }
      }
    },
    "发送系统错误通知": {
      "type": "Office365SendEmail",
      "inputs": {
        "parameters": {
          "To": "@parameters('SystemAdminEmail')",
          "Subject": "【系统错误】权限申请邮件分发失败",
          "Body": "<p>申请编号：@{variables('RequestDetails')['RequestID']}</p><p>错误信息：@{result('Scope')[0]['error']['message']}</p><p>请及时处理。</p>",
          "IsHtml": true,
          "Importance": "High"
        }
      }
    }
  },
  "runAfter": {
    "Scope": ["Failed", "TimedOut", "Cancelled"]
  }
}
```

## 11. 性能优化配置

### 流程配置参数
```json
{
  "parameters": {
    "SharePointSiteUrl": "https://company.sharepoint.com/sites/PermissionApproval",
    "SystemAdminEmail": "admin@company.com",
    "ApprovalTimeoutDays": 3,
    "CCToApplicant": false,
    "MaxConcurrentEmails": 3,
    "EmailRetryAttempts": 3,
    "ReminderIntervalHours": 24
  }
}
```

### 并发和限流控制
```json
{
  "runtimeConfiguration": {
    "concurrency": {
      "repetitions": 3
    },
    "operationOptions": "singleInstance"
  }
}