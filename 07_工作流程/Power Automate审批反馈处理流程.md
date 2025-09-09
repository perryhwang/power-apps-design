# Power Automate 审批反馈处理流程

## 流程架构概览

### 主要处理流程
1. **邮件监听流程**: 监控审批回复邮件
2. **内容解析流程**: 解析邮件内容获取审批决定
3. **状态更新流程**: 更新申请和审核人状态
4. **结果通知流程**: 发送审批结果通知
5. **后续处理流程**: 触发权限实施或其他后续动作

## 1. 邮件监听和初步处理

### 主流程触发器
```json
{
  "name": "ApprovalFeedbackProcessor",
  "trigger": {
    "type": "Office365WhenANewEmailArrives",
    "inputs": {
      "parameters": {
        "folderPath": "Inbox",
        "to": "approval-system@company.com",
        "subjectFilter": "REQ|APPROVE|REJECT|权限审批",
        "importance": "Any",
        "includeAttachments": false,
        "onlyWithAttachment": false
      }
    }
  }
}
```

### 初始数据提取
```json
{
  "type": "InitializeVariable",
  "inputs": {
    "variables": [
      {
        "name": "EmailData",
        "type": "object",
        "value": {
          "Subject": "@triggerBody()['Subject']",
          "Body": "@triggerBody()['Body']",
          "From": "@triggerBody()['From']",
          "ReceivedTime": "@triggerBody()['DateTimeReceived']",
          "MessageId": "@triggerBody()['Id']",
          "HasAttachments": "@triggerBody()['HasAttachments']"
        }
      },
      {
        "name": "ProcessingResult",
        "type": "object",
        "value": {
          "RequestID": "",
          "Decision": "",
          "Comments": "",
          "Confidence": 0,
          "IsValid": false,
          "ErrorMessage": ""
        }
      },
      {
        "name": "NotificationTargets",
        "type": "array",
        "value": []
      }
    ]
  }
}
```

## 2. 邮件内容解析和验证

### 申请编号提取
```json
{
  "type": "Compose",
  "inputs": {
    "RequestIDPattern": "@{
      if(
        contains(variables('EmailData')['Subject'], 'REQ'),
        first(
          array(
            replace(
              first(
                split(
                  last(
                    split(variables('EmailData')['Subject'], 'REQ')
                  ),
                  ' '
                )
              ),
              '-',
              ''
            )
          )
        ),
        ''
      )
    }"
  }
}
```

### 审批决定识别
```json
{
  "type": "Compose",
  "inputs": {
    "DecisionAnalysis": "@{
      if(
        or(
          contains(upper(variables('EmailData')['Subject']), 'APPROVE'),
          contains(variables('EmailData')['Body'], '同意'),
          contains(variables('EmailData')['Body'], '批准'),
          contains(variables('EmailData')['Body'], '通过')
        ),
        json('{\"Decision\":\"APPROVE\",\"Confidence\":0.9}'),
        if(
          or(
            contains(upper(variables('EmailData')['Subject']), 'REJECT'),
            contains(variables('EmailData')['Body'], '拒绝'),
            contains(variables('EmailData')['Body'], '驳回'),
            contains(variables('EmailData')['Body'], '不同意')
          ),
          json('{\"Decision\":\"REJECT\",\"Confidence\":0.9}'),
          json('{\"Decision\":\"UNCLEAR\",\"Confidence\":0.3}')
        )
      )
    }"
  }
}
```

### 审批意见提取
```json
{
  "type": "Compose",
  "inputs": {
    "ExtractedComments": "@{
      coalesce(
        if(
          contains(variables('EmailData')['Body'], '审批意见：'),
          trim(
            first(
              split(
                last(
                  split(variables('EmailData')['Body'], '审批意见：')
                ),
                '\n'
              )
            )
          ),
          ''
        ),
        if(
          contains(variables('EmailData')['Body'], '拒绝原因：'),
          trim(
            first(
              split(
                last(
                  split(variables('EmailData')['Body'], '拒绝原因：')
                ),
                '\n'
              )
            )
          ),
          ''
        ),
        if(
          greater(length(variables('EmailData')['Body']), 50),
          substring(variables('EmailData')['Body'], 0, min(200, length(variables('EmailData')['Body']))),
          variables('EmailData')['Body']
        )
      )
    }"
  }
}
```

## 3. 审核人身份验证

### 验证审核人权限
```json
{
  "type": "SharePointGetItems",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ReviewerConfig",
      "query": {
        "filter": "ReviewerEmail eq '@{variables('EmailData')['From']}' and IsActive eq true"
      }
    }
  },
  "runAfter": {
    "提取邮件信息": ["Succeeded"]
  }
}
```

### 验证申请存在和状态
```json
{
  "type": "SharePointGetItems",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "PermissionRequests", 
      "query": {
        "filter": "RequestID eq '@{outputs('申请编号提取')}' and Status eq '已发送审批邮件'"
      }
    }
  }
}
```

### 验证审核人分配关系
```json
{
  "type": "SharePointGetItems",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ApprovalTracking",
      "query": {
        "filter": "RequestID eq '@{outputs('申请编号提取')}' and ReviewerEmail eq '@{variables('EmailData')['From']}' and Status eq 'Assigned'"
      }
    }
  }
}
```

## 4. 业务逻辑验证和处理

### 多层验证逻辑
```json
{
  "type": "Condition",
  "expression": {
    "and": [
      {
        "greater": ["@length(body('验证审核人权限')?['value'])", 0]
      },
      {
        "greater": ["@length(body('验证申请存在和状态')?['value'])", 0]
      },
      {
        "greater": ["@length(body('验证审核人分配关系')?['value'])", 0]
      },
      {
        "not": {
          "equals": ["@outputs('申请编号提取')", ""]
        }
      },
      {
        "greater": ["@outputs('审批决定识别')['Confidence']", 0.7]
      }
    ]
  },
  "actions": {
    "处理有效审批": {
      "type": "Scope",
      "actions": {
        "更新处理结果": {
          "type": "SetVariable",
          "inputs": {
            "name": "ProcessingResult",
            "value": {
              "RequestID": "@outputs('申请编号提取')",
              "Decision": "@outputs('审批决定识别')['Decision']",
              "Comments": "@outputs('审批意见提取')",
              "Confidence": "@outputs('审批决定识别')['Confidence']",
              "IsValid": true,
              "ErrorMessage": ""
            }
          }
        }
      }
    }
  },
  "else": {
    "actions": {
      "处理无效审批": {
        "type": "Scope",
        "actions": {
          "记录错误信息": {
            "type": "SetVariable",
            "inputs": {
              "name": "ProcessingResult",
              "value": {
                "RequestID": "@outputs('申请编号提取')",
                "Decision": "ERROR",
                "Comments": "",
                "Confidence": 0,
                "IsValid": false,
                "ErrorMessage": "@{
                  if(empty(body('验证审核人权限')?['value']), '发件人不是有效审核人；',
                  if(empty(body('验证申请存在和状态')?['value']), '申请不存在或状态不正确；',
                  if(empty(body('验证审核人分配关系')?['value']), '该审核人未被分配此申请；',
                  if(equals(outputs('申请编号提取'), ''), '无法识别申请编号；',
                  if(lessOrEquals(outputs('审批决定识别')['Confidence'], 0.7), '无法确定审批决定', '')))))
                }"
              }
            }
          }
        }
      }
    }
  }
}
```

## 5. 审批状态更新处理

### 更新个人审批状态
```json
{
  "type": "SharePointUpdateItem",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ApprovalTracking",
      "id": "@first(body('验证审核人分配关系')?['value'])?['ID']",
      "item": {
        "Status": "@if(equals(variables('ProcessingResult')['Decision'], 'APPROVE'), 'Approved', 'Rejected')",
        "Decision": "@variables('ProcessingResult')['Decision']",
        "Comments": "@variables('ProcessingResult')['Comments']",
        "ResponseDate": "@utcNow()",
        "ResponseSource": "Email",
        "OriginalEmailId": "@variables('EmailData')['MessageId']"
      }
    }
  },
  "runAfter": {
    "验证业务逻辑": ["Succeeded"]
  },
  "runtimeConfiguration": {
    "retry": {
      "count": 3,
      "interval": "PT30S",
      "type": "exponential"
    }
  }
}
```

### 计算整体审批状态
```json
{
  "type": "SharePointGetItems",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ApprovalTracking",
      "query": {
        "filter": "RequestID eq '@{variables('ProcessingResult')['RequestID']}'"
      }
    }
  }
}
```

### 审批决策逻辑
```json
{
  "type": "Compose",
  "inputs": {
    "OverallDecision": "@{
      if(
        greater(
          length(
            filter(
              body('获取所有审核状态')?['value'],
              equals(item()['Status'], 'Rejected')
            )
          ),
          0
        ),
        'REJECTED',
        if(
          equals(
            length(
              filter(
                body('获取所有审核状态')?['value'],
                equals(item()['Status'], 'Approved')
              )
            ),
            length(body('获取所有审核状态')?['value'])
          ),
          'APPROVED',
          'PENDING'
        )
      )
    }"
  }
}
```

### 更新主申请状态
```json
{
  "type": "Condition",
  "expression": {
    "not": {
      "equals": ["@outputs('计算整体审批状态')", "PENDING"]
    }
  },
  "actions": {
    "更新申请最终状态": {
      "type": "SharePointUpdateItem",
      "inputs": {
        "parameters": {
          "site": "@parameters('SharePointSiteUrl')",
          "list": "PermissionRequests",
          "id": "@first(body('验证申请存在和状态')?['value'])?['ID']",
          "item": {
            "Status": "@if(equals(outputs('计算整体审批状态'), 'APPROVED'), '审批通过', '审批拒绝')",
            "FinalDecision": "@outputs('计算整体审批状态')",
            "ReviewDate": "@utcNow()",
            "ReviewComments": "@join(select(filter(body('获取所有审核状态')?['value'], not(empty(item()['Comments']))), item()['Comments']), '; ')",
            "ProcessedBy": "System-EmailProcessor"
          }
        }
      }
    }
  }
}
```

## 6. 创建审批历史记录

### 详细审批记录
```json
{
  "type": "SharePointCreateItem",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ApprovalHistory",
      "item": {
        "Title": "@concat(variables('ProcessingResult')['RequestID'], '-', formatDateTime(utcNow(), 'yyyyMMdd-HHmmss'))",
        "RequestID": "@variables('ProcessingResult')['RequestID']",
        "ReviewerEmail": "@variables('EmailData')['From']",
        "ReviewerName": "@first(body('验证审核人权限')?['value'])?['ReviewerName']",
        "Decision": "@variables('ProcessingResult')['Decision']",
        "Comments": "@variables('ProcessingResult')['Comments']",
        "ReviewDate": "@utcNow()",
        "Source": "Email",
        "EmailSubject": "@variables('EmailData')['Subject']",
        "ProcessingConfidence": "@variables('ProcessingResult')['Confidence']",
        "ResponseTimeMinutes": "@div(sub(ticks(utcNow()), ticks(first(body('验证审核人分配关系')?['value'])?['AssignedDate'])), 600000000)"
      }
    }
  }
}
```

## 7. 通知处理

### 确认邮件给审核人
```json
{
  "type": "Office365SendEmail",
  "inputs": {
    "parameters": {
      "To": "@variables('EmailData')['From']",
      "Subject": "【确认收到】您的审批意见已处理 - @{variables('ProcessingResult')['RequestID']}",
      "Body": "感谢您的审批！\n\n申请编号：@{variables('ProcessingResult')['RequestID']}\n审批结果：@{if(equals(variables('ProcessingResult')['Decision'], 'APPROVE'), '同意', if(equals(variables('ProcessingResult')['Decision'], 'REJECT'), '拒绝', '待定'))}\n审批意见：@{variables('ProcessingResult')['Comments']}\n处理时间：@{formatDateTime(utcNow(), 'yyyy年MM月dd日 HH:mm')}\n\n您的审批意见已成功记录到系统中。",
      "IsHtml": false,
      "Importance": "Normal"
    }
  },
  "runAfter": {
    "创建审批历史记录": ["Succeeded"]
  }
}
```

### 申请人状态通知
```json
{
  "type": "Condition",
  "expression": {
    "not": {
      "equals": ["@outputs('计算整体审批状态')", "PENDING"]
    }
  },
  "actions": {
    "发送最终结果通知": {
      "type": "CallChildFlow",
      "inputs": {
        "host": {
          "triggerName": "manual",
          "workflow": {
            "id": "/workflows/SendApprovalResultNotification"
          }
        },
        "body": {
          "RequestID": "@variables('ProcessingResult')['RequestID']",
          "FinalDecision": "@outputs('计算整体审批状态')",
          "ApplicantEmail": "@first(body('验证申请存在和状态')?['value'])?['ApplicantEmail']",
          "AllComments": "@join(select(filter(body('获取所有审核状态')?['value'], not(empty(item()['Comments']))), concat(item()['ReviewerName'], ': ', item()['Comments'])), '\n')"
        }
      }
    }
  },
  "else": {
    "actions": {
      "发送进度更新通知": {
        "type": "Office365SendEmail",
        "inputs": {
          "parameters": {
            "To": "@first(body('验证申请存在和状态')?['value'])?['ApplicantEmail']",
            "Subject": "【审批进度】您的权限申请有新进展 - @{variables('ProcessingResult')['RequestID']}",
            "Body": "您好！\n\n您的权限申请有新的审批进展：\n\n申请编号：@{variables('ProcessingResult')['RequestID']}\n最新审批：@{first(body('验证审核人权限')?['value'])?['ReviewerName']} - @{if(equals(variables('ProcessingResult')['Decision'], 'APPROVE'), '同意', '拒绝')}\n审批意见：@{variables('ProcessingResult')['Comments']}\n\n@{if(equals(outputs('计算整体审批状态'), 'PENDING'), '还有其他审核人正在处理您的申请，请耐心等待。', '所有审批已完成。')}\n\n您可以登录系统查看详细进度。",
            "IsHtml": false
          }
        }
      }
    }
  }
}
```

## 8. 后续处理触发

### 权限实施流程触发
```json
{
  "type": "Condition",
  "expression": {
    "equals": ["@outputs('计算整体审批状态')", "APPROVED"]
  },
  "actions": {
    "触发权限实施流程": {
      "type": "CallChildFlow",
      "inputs": {
        "host": {
          "triggerName": "manual",
          "workflow": {
            "id": "/workflows/PermissionImplementationFlow"
          }
        },
        "body": {
          "RequestID": "@variables('ProcessingResult')['RequestID']",
          "ApplicantEmail": "@first(body('验证申请存在和状态')?['value'])?['ApplicantEmail']",
          "PermissionSystem": "@first(body('验证申请存在和状态')?['value'])?['PermissionSystem']",
          "PermissionObject": "@first(body('验证申请存在和状态')?['value'])?['PermissionObject']",
          "ApprovalDate": "@utcNow()",
          "ExpectedDuration": "@first(body('验证申请存在和状态')?['value'])?['ExpectedDuration']"
        }
      }
    }
  }
}
```

### 拒绝申请处理
```json
{
  "type": "Condition",
  "expression": {
    "equals": ["@outputs('计算整体审批状态')", "REJECTED"]
  },
  "actions": {
    "处理拒绝申请": {
      "type": "Scope",
      "actions": {
        "更新申请为已关闭": {
          "type": "SharePointUpdateItem",
          "inputs": {
            "parameters": {
              "site": "@parameters('SharePointSiteUrl')",
              "list": "PermissionRequests",
              "id": "@first(body('验证申请存在和状态')?['value'])?['ID']",
              "item": {
                "IsClosed": true,
                "ClosedDate": "@utcNow()",
                "ClosedReason": "审批被拒绝"
              }
            }
          }
        },
        "创建重新申请指导": {
          "type": "SharePointCreateItem",
          "inputs": {
            "parameters": {
              "site": "@parameters('SharePointSiteUrl')",
              "list": "ReapplicationGuidance",
              "item": {
                "OriginalRequestID": "@variables('ProcessingResult')['RequestID']",
                "RejectionReasons": "@join(select(filter(body('获取所有审核状态')?['value'], equals(item()['Decision'], 'REJECT')), item()['Comments']), '; ')",
                "SuggestedImprovements": "请根据拒绝原因修改申请内容后重新提交",
                "CanReapply": true,
                "ReapplicationAllowedAfter": "@addDays(utcNow(), 7)"
              }
            }
          }
        }
      }
    }
  }
}
```

## 9. 错误处理和日志记录

### 全局错误处理
```json
{
  "type": "Scope",
  "actions": {
    "所有审批处理逻辑": "..."
  },
  "runAfter": {},
  "runtimeConfiguration": {
    "trackedProperties": {
      "RequestID": "@variables('ProcessingResult')['RequestID']",
      "ReviewerEmail": "@variables('EmailData')['From']",
      "ProcessingTime": "@utcNow()"
    }
  }
}
```

### 错误恢复处理
```json
{
  "type": "Scope",
  "actions": {
    "记录处理错误": {
      "type": "SharePointCreateItem",
      "inputs": {
        "parameters": {
          "site": "@parameters('SharePointSiteUrl')",
          "list": "EmailProcessingErrors",
          "item": {
            "Title": "审批邮件处理失败",
            "EmailFrom": "@variables('EmailData')['From']",
            "EmailSubject": "@variables('EmailData')['Subject']",
            "EmailBody": "@variables('EmailData')['Body']",
            "ErrorMessage": "@{result('主处理逻辑')?['error']?['message']}",
            "ErrorCode": "@{result('主处理逻辑')?['error']?['code']}",
            "ProcessingTime": "@utcNow()",
            "RequiresManualReview": true
          }
        }
      }
    },
    "发送系统管理员警告": {
      "type": "Office365SendEmail",
      "inputs": {
        "parameters": {
          "To": "@parameters('SystemAdminEmail')",
          "Subject": "【系统错误】审批邮件处理失败",
          "Body": "审批邮件处理发生错误，需要人工介入：\n\n发件人：@{variables('EmailData')['From']}\n邮件主题：@{variables('EmailData')['Subject']}\n错误信息：@{result('主处理逻辑')?['error']?['message']}\n\n请及时处理。",
          "Importance": "High"
        }
      }
    }
  },
  "runAfter": {
    "主处理逻辑": ["Failed", "TimedOut", "Cancelled"]
  }
}
```

## 10. 性能监控和优化

### 处理时间统计
```json
{
  "type": "SharePointCreateItem",
  "inputs": {
    "parameters": {
      "site": "@parameters('SharePointSiteUrl')",
      "list": "ProcessingMetrics",
      "item": {
        "Title": "@concat('EmailProcessing-', formatDateTime(utcNow(), 'yyyyMMdd-HHmmss'))",
        "ProcessingDate": "@utcNow()",
        "RequestID": "@variables('ProcessingResult')['RequestID']",
        "ReviewerEmail": "@variables('EmailData')['From']",
        "ProcessingDurationMs": "@sub(ticks(utcNow()), ticks(variables('ProcessingStartTime')))",
        "DecisionConfidence": "@variables('ProcessingResult')['Confidence']",
        "ValidationsPassed": "@length(filter(array('审核人验证', '申请验证', '分配验证', '决定识别'), item()))",
        "FinalStatus": "@if(variables('ProcessingResult')['IsValid'], 'Success', 'Failed')"
      }
    }
  }
}
```

### 流程优化建议
```json
{
  "PerformanceOptimization": {
    "CachingStrategy": {
      "ReviewerConfig": "缓存审核人配置30分钟",
      "RequestStatus": "实时查询，不缓存",
      "SystemParameters": "缓存配置参数1小时"
    },
    "ParallelProcessing": {
      "StatusUpdates": "并行更新个人状态和整体状态",
      "Notifications": "异步发送通知邮件",
      "Logging": "后台记录日志和统计"
    },
    "ErrorResilience": {
      "RetryPolicy": "指数退避，最多重试3次",
      "Timeout": "单个操作最多60秒",
      "FallbackActions": "失败时发送人工处理通知"
    }
  }
}
```