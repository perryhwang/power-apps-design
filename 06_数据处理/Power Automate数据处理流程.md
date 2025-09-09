# Power Automate 数据处理流程设计

## 流程概览

### 主流程：Excel数据导入处理
**触发器**: Power Apps触发 (PowerApps trigger)
**流程名称**: ProcessExcelDataImport

### 流程步骤详细设计

## 1. 触发器配置
```json
{
  "type": "PowerApps",
  "inputs": {
    "schema": {
      "type": "object",
      "properties": {
        "fileContent": {
          "type": "string",
          "title": "Excel文件内容"
        },
        "fileName": {
          "type": "string", 
          "title": "文件名"
        },
        "importedBy": {
          "type": "string",
          "title": "导入用户"
        }
      },
      "required": ["fileContent", "fileName", "importedBy"]
    }
  }
}
```

## 2. 解析Excel文件
```json
{
  "type": "ParseExcelFile",
  "runAfter": {},
  "inputs": {
    "source": "@triggerBody()['fileContent']",
    "tables": [{
      "name": "Sheet1"
    }]
  }
}
```

## 3. 初始化变量
```json
{
  "type": "InitializeVariable",
  "inputs": {
    "variables": [
      {
        "name": "ProcessedCount",
        "type": "integer",
        "value": 0
      },
      {
        "name": "ErrorCount", 
        "type": "integer",
        "value": 0
      },
      {
        "name": "ValidationErrors",
        "type": "array",
        "value": []
      },
      {
        "name": "ProcessingResults",
        "type": "array", 
        "value": []
      }
    ]
  }
}
```

## 4. 数据验证循环
```json
{
  "type": "ForEach",
  "foreach": "@body('ParseExcelFile')?['value']",
  "actions": {
    "验证必填字段": {
      "type": "Condition",
      "expression": {
        "and": [
          {
            "not": {
              "equals": ["@items('ForEach')?['申请人姓名']", ""]
            }
          },
          {
            "not": {
              "equals": ["@items('ForEach')?['申请人邮箱']", ""]
            }
          },
          {
            "not": {
              "equals": ["@items('ForEach')?['权限系统']", ""]
            }
          },
          {
            "not": {
              "equals": ["@items('ForEach')?['权限对象']", ""]
            }
          }
        ]
      },
      "actions": {
        "验证邮箱格式": {
          "type": "Condition", 
          "expression": {
            "contains": ["@items('ForEach')?['申请人邮箱']", "@"]
          },
          "actions": {
            "添加到处理队列": {
              "type": "AppendToArrayVariable",
              "inputs": {
                "name": "ProcessingResults",
                "value": {
                  "rowIndex": "@add(indexOf(body('ParseExcelFile')?['value'], items('ForEach')), 1)",
                  "applicant": "@items('ForEach')?['申请人姓名']",
                  "applicantEmail": "@items('ForEach')?['申请人邮箱']", 
                  "permissionSystem": "@items('ForEach')?['权限系统']",
                  "permissionObject": "@items('ForEach')?['权限对象']",
                  "reason": "@items('ForEach')?['申请原因']",
                  "expectedDuration": "@items('ForEach')?['预计使用期限']",
                  "status": "待验证"
                }
              }
            }
          },
          "else": {
            "actions": {
              "记录邮箱格式错误": {
                "type": "AppendToArrayVariable",
                "inputs": {
                  "name": "ValidationErrors",
                  "value": {
                    "rowIndex": "@add(indexOf(body('ParseExcelFile')?['value'], items('ForEach')), 1)",
                    "errorType": "格式错误",
                    "errorField": "申请人邮箱",
                    "errorContent": "@items('ForEach')?['申请人邮箱']",
                    "suggestion": "请输入正确的邮箱格式"
                  }
                }
              },
              "增加错误计数": {
                "type": "IncrementVariable",
                "inputs": {
                  "name": "ErrorCount",
                  "value": 1
                }
              }
            }
          }
        }
      },
      "else": {
        "actions": {
          "记录必填字段错误": {
            "type": "AppendToArrayVariable",
            "inputs": {
              "name": "ValidationErrors", 
              "value": {
                "rowIndex": "@add(indexOf(body('ParseExcelFile')?['value'], items('ForEach')), 1)",
                "errorType": "必填字段缺失",
                "errorField": "申请人姓名/邮箱/权限系统/权限对象",
                "errorContent": "存在空值",
                "suggestion": "请检查并填写所有必填字段"
              }
            }
          },
          "增加错误计数": {
            "type": "IncrementVariable",
            "inputs": {
              "name": "ErrorCount",
              "value": 1
            }
          }
        }
      }
    }
  }
}
```

## 5. 重复数据检查
```json
{
  "type": "ForEach",
  "foreach": "@variables('ProcessingResults')",
  "actions": {
    "查询现有申请": {
      "type": "SharePointGetItems",
      "inputs": {
        "site": "@parameters('SharePointSite')",
        "list": "PermissionRequests",
        "query": {
          "filter": "ApplicantEmail eq '@{items('ForEach_2')?['applicantEmail']}' and PermissionSystem eq '@{items('ForEach_2')?['permissionSystem']}' and PermissionObject eq '@{items('ForEach_2')?['permissionObject']}'"
        }
      }
    },
    "检查重复": {
      "type": "Condition",
      "expression": {
        "greater": ["@length(body('查询现有申请')?['value'])", 0]
      },
      "actions": {
        "标记重复": {
          "type": "AppendToArrayVariable",
          "inputs": {
            "name": "ValidationErrors",
            "value": {
              "rowIndex": "@items('ForEach_2')?['rowIndex']",
              "errorType": "重复申请",
              "errorField": "申请人邮箱+权限系统+权限对象",
              "errorContent": "已存在相同的权限申请",
              "suggestion": "请检查是否为重复申请，如需要请联系管理员"
            }
          }
        }
      }
    }
  }
}
```

## 6. 生成申请编号
```json
{
  "type": "ForEach",
  "foreach": "@variables('ProcessingResults')",
  "actions": {
    "生成申请编号": {
      "type": "Compose",
      "inputs": "@concat('REQ', formatDateTime(utcNow(), 'yyyyMMdd'), padLeft(string(add(indexOf(variables('ProcessingResults'), items('ForEach_3')), 1)), 4, '0'))"
    },
    "更新处理结果": {
      "type": "SetVariable",
      "inputs": {
        "name": "ProcessingResults",
        "value": "@setProperty(items('ForEach_3'), 'requestID', outputs('生成申请编号'))"
      }
    }
  }
}
```

## 7. 批量创建SharePoint记录
```json
{
  "type": "ForEach",
  "foreach": "@variables('ProcessingResults')",
  "actions": {
    "创建权限申请记录": {
      "type": "SharePointCreateItem",
      "inputs": {
        "site": "@parameters('SharePointSite')",
        "list": "PermissionRequests",
        "item": {
          "Title": "@items('ForEach_4')?['requestID']",
          "RequestID": "@items('ForEach_4')?['requestID']",
          "Applicant": "@items('ForEach_4')?['applicant']",
          "ApplicantEmail": "@items('ForEach_4')?['applicantEmail']",
          "PermissionSystem": "@items('ForEach_4')?['permissionSystem']",
          "PermissionObject": "@items('ForEach_4')?['permissionObject']",
          "Reason": "@items('ForEach_4')?['reason']",
          "ExpectedDuration": "@items('ForEach_4')?['expectedDuration']",
          "Status": "待审批",
          "RequestDate": "@utcNow()",
          "ImportedBy": "@triggerBody()['importedBy']"
        }
      },
      "runAfter": {},
      "runtimeConfiguration": {
        "staticResult": {
          "staticResultOptions": "Disabled"
        }
      }
    },
    "增加处理计数": {
      "type": "IncrementVariable",
      "inputs": {
        "name": "ProcessedCount",
        "value": 1
      }
    }
  },
  "runAfter": {
    "生成申请编号循环": ["Succeeded"]
  }
}
```

## 8. 发送导入结果通知
```json
{
  "type": "SendEmail",
  "inputs": {
    "to": "@triggerBody()['importedBy']",
    "subject": "权限申请数据导入完成",
    "body": {
      "content": "<h3>数据导入完成</h3><p>导入文件: @{triggerBody()['fileName']}</p><p>成功导入: @{variables('ProcessedCount')} 条</p><p>错误记录: @{variables('ErrorCount')} 条</p>@{if(greater(variables('ErrorCount'), 0), concat('<h4>错误详情:</h4><ul>', join(variables('ValidationErrors'), '</li><li>'), '</ul>'), '')}",
      "contentType": "Html"
    }
  }
}
```

## 9. 返回处理结果给Power Apps
```json
{
  "type": "Response",
  "inputs": {
    "statusCode": 200,
    "body": {
      "success": true,
      "processedCount": "@variables('ProcessedCount')",
      "errorCount": "@variables('ErrorCount')",
      "validationErrors": "@variables('ValidationErrors')",
      "message": "数据导入处理完成"
    }
  }
}
```

## 子流程：智能审核人匹配

### 流程名称: AssignReviewersFlow
**触发器**: HTTP请求触发

### 子流程步骤

## 1. 获取权限系统配置
```json
{
  "type": "SharePointGetItems",
  "inputs": {
    "site": "@parameters('SharePointSite')",
    "list": "ReviewerConfig",
    "query": {
      "filter": "IsActive eq true"
    }
  }
}
```

## 2. 匹配审核人逻辑
```json
{
  "type": "ForEach",
  "foreach": "@triggerBody()['requests']",
  "actions": {
    "查找匹配审核人": {
      "type": "Filter",
      "inputs": {
        "from": "@body('获取权限系统配置')?['value']",
        "where": "@contains(item()['SystemScope'], items('ForEach')?['permissionSystem'])"
      }
    },
    "分配审核人": {
      "type": "Condition",
      "expression": {
        "greater": ["@length(body('查找匹配审核人'))", 0]
      },
      "actions": {
        "更新申请记录": {
          "type": "SharePointUpdateItem",
          "inputs": {
            "site": "@parameters('SharePointSite')",
            "list": "PermissionRequests",
            "id": "@items('ForEach')?['id']",
            "item": {
              "AssignedReviewers": "@join(select(body('查找匹配审核人'), item()['ReviewerEmail']), ';')",
              "Status": "已分配审核人"
            }
          }
        }
      },
      "else": {
        "actions": {
          "标记无审核人": {
            "type": "SharePointUpdateItem", 
            "inputs": {
              "site": "@parameters('SharePointSite')",
              "list": "PermissionRequests",
              "id": "@items('ForEach')?['id']",
              "item": {
                "Status": "无匹配审核人",
                "Comments": "未找到负责该权限系统的审核人"
              }
            }
          }
        }
      }
    }
  }
}
```

## 错误处理机制

### 全局错误处理
```json
{
  "type": "Scope",
  "actions": {
    "主要处理逻辑": "..."
  },
  "runAfter": {},
  "trackedProperties": {
    "flowName": "ProcessExcelDataImport",
    "importedBy": "@triggerBody()['importedBy']"
  },
  "runtimeConfiguration": {
    "paginationPolicy": {
      "minimumItemCount": 100000
    }
  }
}
```

### 错误恢复处理
```json
{
  "type": "Scope",
  "actions": {
    "发送错误通知": {
      "type": "SendEmail",
      "inputs": {
        "to": "@triggerBody()['importedBy']",
        "subject": "数据导入过程发生错误",
        "body": {
          "content": "<h3>导入失败</h3><p>导入文件: @{triggerBody()['fileName']}</p><p>错误信息: @{result('Scope')[0]['error']['message']}</p><p>请检查数据格式后重新导入。</p>",
          "contentType": "Html"
        }
      }
    }
  },
  "runAfter": {
    "Scope": ["Failed", "TimedOut"]
  }
}
```

## 性能优化配置

### 并发控制
```json
{
  "concurrency": {
    "repetitions": 1
  },
  "runtimeConfiguration": {
    "concurrency": {
      "repetitions": 50
    }
  }
}
```

### 超时设置
```json
{
  "timeout": "PT10M",
  "retry": {
    "count": 3,
    "interval": "PT1M",
    "type": "exponential"
  }
}
```

## 日志记录

### 操作日志
```json
{
  "type": "SharePointCreateItem",
  "inputs": {
    "site": "@parameters('SharePointSite')",
    "list": "ImportLogs",
    "item": {
      "Title": "@concat('Import_', formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'))",
      "ImportedBy": "@triggerBody()['importedBy']",
      "FileName": "@triggerBody()['fileName']",
      "ProcessedCount": "@variables('ProcessedCount')",
      "ErrorCount": "@variables('ErrorCount')",
      "ImportDate": "@utcNow()",
      "Status": "@if(equals(variables('ErrorCount'), 0), 'Success', 'Partial Success')"
    }
  }
}