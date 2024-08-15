
```
{
  "Comment": "A state machine to orchestrate IAM role creation and AD/DS group creation",
  "StartAt": "UpsertRequestRecord",
  "States": {
    "UpsertRequestRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Item": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          },
          "build_type": {
            "S.$": "$.build_body.build_type"
          },
          "action": {
            "S.$": "$.build_body.action"
          },
          "custom_role_name": {
            "S.$": "$.build_body.custom_role_name"
          },
          "dry_run": {
            "BOOL.$": "$.build_body.dry_run"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "correlation_id": {
            "S.$": "$.correlation_id"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "InsertRequestHistory"
    },
    "InsertRequestHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Item": {
          "request_id": {
            "S.$": "$.correlation_id"
          },
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          },
          "action": {
            "S.$": "$.build_body.action"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerBuild"
    },
    "TriggerBuild": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/build', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body.$": "$.build_body"
        }
      },
      "ResultPath": "$.build_trigger_results",
      "Next": "WaitForBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "GetBuildStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/build/{}', $.url, $.build_trigger_results.body.build_id)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.build_status_results",
      "Next": "CheckBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckBuildStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.build_status_results.body.can_retry",
          "BooleanEquals": true,
          "Next": "WaitForBuildStatus"
        },
        {
          "Variable": "$.build_status_results.body.build_status",
          "StringEquals": "SUCCEEDED",
          "Next": "TriggerADLDSCreation"
        }
      ],
      "Default": "GenerateBuildStatusErrorMessage"
    },
    "TriggerADLDSCreation": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/adlds-creation', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body": {
            "EIMID.$": "$.build_body.EIMID",
            "basic_entitlementName.$": "$.build_body.basic_entitlementName",
            "account_id.$": "$.build_body.account_id",
            "friendRequestId.$": "$.build_body.friendRequestId",
            "environment.$": "$.build_body.environment"
          }
        }
      },
      "ResultPath": "$.adlds_trigger_results",
      "Next": "WaitForADLDSStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForADLDSStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetADLDSStatus"
    },
    "GetADLDSStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/status/{}', $.url, $.adlds_trigger_results.body.correlationId)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.adlds_status_results",
      "Next": "CheckADLDSStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckADLDSStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.adlds_status_results.body.status",
          "StringEquals": "COMPLETED",
          "Next": "UpdateSuccessfulRecord"
        }
      ],
      "Default": "WaitForADLDSStatus"
    },
    "UpdateSuccessfulRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateSuccessfulHistory"
    },
    "UpdateSuccessfulHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "End": true
    },
    "GenerateBuildStatusErrorMessage": {
      "Type": "Pass",
      "Next": "UpdateFailedRecord",
      "Result": "Build status is not in [SUCCEEDED, IN_PROGRESS].",
      "ResultPath": "$.error_info"
    },
    "UpdateFailedRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#exe_status": "status",
          "#error_info": "error"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.error_info)"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateFailedHistory"
    },
    "UpdateFailedHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Next": "FailForBuild"
    },
    "FailForBuild": {
      "Type": "Fail",
      "Cause": "Error deploying IAM role",
      "Error": "See logs"
    }
  }
}
```

```
{
  "Comment": "A state machine to orchestrate IAM role creation and AD/DS group creation",
  "StartAt": "UpsertRequestRecord",
  "States": {
    "UpsertRequestRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Item": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          },
          "build_type": {
            "S.$": "$.build_body.build_type"
          },
          "action": {
            "S.$": "$.build_body.action"
          },
          "custom_role_name": {
            "S.$": "$.build_body.custom_role_name"
          },
          "dry_run": {
            "BOOL.$": "$.build_body.dry_run"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "correlation_id": {
            "S.$": "$.correlation_id"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "InsertRequestHistory"
    },
    "InsertRequestHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Item": {
          "request_id": {
            "S.$": "$.correlation_id"
          },
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          },
          "action": {
            "S.$": "$.build_body.action"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerBuild"
    },
    "TriggerBuild": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/build', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body.$": "$.build_body"
        }
      },
      "ResultPath": "$.build_trigger_results",
      "Next": "WaitForBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForBuildStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetBuildStatus"
    },
    "GetBuildStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/build/{}', $.url, $.build_trigger_results.body.build_id)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.build_status_results",
      "Next": "CheckBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckBuildStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.build_status_results.body.can_retry",
          "BooleanEquals": true,
          "Next": "WaitForBuildStatus"
        },
        {
          "Variable": "$.build_status_results.body.build_status",
          "StringEquals": "SUCCEEDED",
          "Next": "UpdateSuccessfulRecord"
        }
      ],
      "Default": "GenerateBuildStatusErrorMessage"
    },
    "UpdateSuccessfulRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateSuccessfulHistory"
    },
    "UpdateSuccessfulHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerADLDSCreation"
    },
    "TriggerADLDSCreation": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/adlds-creation', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body": {
            "EIMID.$": "$.build_body.EIMID",
            "basic_entitlementName.$": "$.build_body.basic_entitlementName",
            "account_id.$": "$.build_body.account_id",
            "friendRequestId.$": "$.build_body.friendRequestId",
            "environment.$": "$.build_body.environment"
          }
        }
      },
      "ResultPath": "$.adlds_trigger_results",
      "Next": "UpdateADLDSRecord",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "UpdateADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #eimid = :eimid, #adlds_status = :adlds_status, #adlds_start = :adlds_start, #adlds_exec_id = :adlds_exec_id",
        "ExpressionAttributeNames": {
          "#eimid": "eimid",
          "#adlds_status": "adlds_status",
          "#adlds_start": "adlds_start_at",
          "#adlds_exec_id": "adlds_execution_id"
        },
        "ExpressionAttributeValues": {
          ":eimid": {
            "S.$": "$.build_body.EIMID"
          },
          ":adlds_status": {
            "S": "IN_PROGRESS"
          },
          ":adlds_start": {
            "S.$": "$$.State.EnteredTime"
          },
          ":adlds_exec_id": {
            "S.$": "$.adlds_trigger_results.body.correlationId"
          }
        }
      },
      "ResultPath": null,
      "Next": "WaitForADLDSStatus",
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForADLDSStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetADLDSStatus"
    },
    "GetADLDSStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/status/{}', $.url, $.adlds_trigger_results.body.correlationId)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.adlds_status_results",
      "Next": "CheckADLDSStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckADLDSStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.adlds_status_results.body.status",
          "StringEquals": "COMPLETED",
          "Next": "UpdateSuccessfulADLDSRecord"
        }
      ],
      "Default": "WaitForADLDSStatus"
    },
    "UpdateSuccessfulADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #adlds_status = :adlds_status",
        "ExpressionAttributeNames": {
          "#adlds_status": "adlds_status"
        },
        "ExpressionAttributeValues": {
          ":adlds_status": {
            "S": "COMPLETED"
          }
        }
      },
      "ResultPath": null,
      "End": true,
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "GenerateBuildStatusErrorMessage": {
      "Type": "Pass",
      "Next": "UpdateFailedRecord",
      "Result": "Build status is not in [SUCCEEDED, IN_PROGRESS].",
      "ResultPath": "$.error_info"
    },
    "UpdateFailedRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#exe_status": "status",
          "#error_info": "error"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.error_info)"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateFailedHistory"
    },
    "UpdateFailedHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Next": "FailForBuild"
    },
    "FailForBuild": {
      "Type": "Fail",
      "Cause": "Error deploying IAM role",
      "Error": "See logs"
    }
  }
}
```
```
{
  "Comment": "A state machine to orchestrate IAM role creation and AD/DS group creation",
  "StartAt": "UpsertRequestRecord",
  "States": {
    "UpsertRequestRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Item": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          },
          "build_type": {
            "S.$": "$.build_body.build_type"
          },
          "action": {
            "S.$": "$.build_body.action"
          },
          "custom_role_name": {
            "S.$": "$.build_body.custom_role_name"
          },
          "dry_run": {
            "BOOL.$": "$.build_body.dry_run"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "correlation_id": {
            "S.$": "$.correlation_id"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "InsertRequestHistory"
    },
    "InsertRequestHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Item": {
          "request_id": {
            "S.$": "$.correlation_id"
          },
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          },
          "action": {
            "S.$": "$.build_body.action"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerBuild"
    },
    "TriggerBuild": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/build', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body.$": "$.build_body"
        }
      },
      "ResultPath": "$.build_trigger_results",
      "Next": "WaitForBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForBuildStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetBuildStatus"
    },
    "GetBuildStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/build/{}', $.url, $.build_trigger_results.body.build_id)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.build_status_results",
      "Next": "CheckBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckBuildStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.build_status_results.body.can_retry",
          "BooleanEquals": true,
          "Next": "WaitForBuildStatus"
        },
        {
          "Variable": "$.build_status_results.body.build_status",
          "StringEquals": "SUCCEEDED",
          "Next": "UpdateSuccessfulRecord"
        }
      ],
      "Default": "GenerateBuildStatusErrorMessage"
    },
    "UpdateSuccessfulRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateSuccessfulHistory"
    },
    "UpdateSuccessfulHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerADLDSCreation"
    },
    "TriggerADLDSCreation": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/adlds-creation', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body": {
            "EIMID.$": "$.build_body.EIMID",
            "basic_entitlementName.$": "$.build_body.basic_entitlementName",
            "account_id.$": "$.build_body.account_id",
            "friendRequestId.$": "$.build_body.friendRequestId",
            "environment.$": "$.build_body.environment"
          }
        }
      },
      "ResultPath": "$.adlds_trigger_results",
      "Next": "UpdateADLDSRecord",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedADLDSRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "UpdateADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #eimid = :eimid, #adlds_status = :adlds_status, #adlds_start = :adlds_start, #adlds_exec_id = :adlds_exec_id",
        "ExpressionAttributeNames": {
          "#eimid": "eimid",
          "#adlds_status": "adlds_status",
          "#adlds_start": "adlds_start_at",
          "#adlds_exec_id": "adlds_execution_id"
        },
        "ExpressionAttributeValues": {
          ":eimid": {
            "S.$": "$.build_body.EIMID"
          },
          ":adlds_status": {
            "S": "IN_PROGRESS"
          },
          ":adlds_start": {
            "S.$": "$$.State.EnteredTime"
          },
          ":adlds_exec_id": {
            "S.$": "$.adlds_trigger_results.body.correlationId"
          }
        }
      },
      "ResultPath": null,
      "Next": "WaitForADLDSStatus",
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForADLDSStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetADLDSStatus"
    },
    "GetADLDSStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/status/{}', $.url, $.adlds_trigger_results.body.correlationId)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.adlds_status_results",
      "Next": "CheckADLDSStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedADLDSRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckADLDSStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.adlds_status_results.body.status",
          "StringEquals": "COMPLETED",
          "Next": "UpdateSuccessfulADLDSRecord"
        }
      ],
      "Default": "WaitForADLDSStatus"
    },
    "UpdateSuccessfulADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #adlds_status = :adlds_status",
        "ExpressionAttributeNames": {
          "#adlds_status": "adlds_status"
        },
        "ExpressionAttributeValues": {
          ":adlds_status": {
            "S": "COMPLETED"
          }
        }
      },
      "ResultPath": null,
      "End": true,
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "GenerateBuildStatusErrorMessage": {
      "Type": "Pass",
      "Next": "UpdateFailedRecord",
      "Result": "Build status is not in [SUCCEEDED, IN_PROGRESS].",
      "ResultPath": "$.error_info"
    },
    "UpdateFailedRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#exe_status": "status",
          "#error_info": "error"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.error_info)"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateFailedHistory"
    },
    "UpdateFailedHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Next": "FailForBuild"
    },
    "FailForBuild": {
      "Type": "Fail",
      "Cause": "Error deploying IAM role",
      "Error": "See logs"
    },
    "UpdateFailedADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.build_body.account_id"
          },
          "resource": {
            "S.$": "$.ddb_composite_sort_key"
          }
        },
        "UpdateExpression": "SET #adlds_status = :adlds_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#adlds_status": "adlds_status",
          "#error_info": "adlds_error"
        },
        "ExpressionAttributeValues": {
          ":adlds_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.error_info)"
          }
        }
      },
      "ResultPath": null,
      "Next": "FailForADLDS",
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "FailForADLDS": {
      "Type": "Fail",
      "Cause": "Error creating ADLDS group",
      "Error": "See adlds_error in DynamoDB record for details"
    }
  }
}
```
```
{
  "Comment": "A state machine to orchestrate IAM role creation and AD/DS group creation",
  "StartAt": "UpsertRequestRecord",
  "States": {
    "UpsertRequestRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Item": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          },
          "build_type": {
            "S.$": "$.iam_details.build_type"
          },
          "action": {
            "S.$": "$.iam_details.action"
          },
          "custom_role_name": {
            "S.$": "$.iam_details.custom_role_name"
          },
          "dry_run": {
            "BOOL.$": "$.iam_details.dry_run"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "correlation_id": {
            "S.$": "$.correlation_id"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "InsertRequestHistory"
    },
    "InsertRequestHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Item": {
          "request_id": {
            "S.$": "$.correlation_id"
          },
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          },
          "action": {
            "S.$": "$.iam_details.action"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerBuild"
    },
    "TriggerBuild": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/build', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body.$": "$.iam_details"
        }
      },
      "ResultPath": "$.build_trigger_results",
      "Next": "WaitForBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForBuildStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetBuildStatus"
    },
    "GetBuildStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/build/{}', $.url, $.build_trigger_results.body.build_id)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.build_status_results",
      "Next": "CheckBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckBuildStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.build_status_results.body.can_retry",
          "BooleanEquals": true,
          "Next": "WaitForBuildStatus"
        },
        {
          "Variable": "$.build_status_results.body.build_status",
          "StringEquals": "SUCCEEDED",
          "Next": "UpdateSuccessfulRecord"
        }
      ],
      "Default": "GenerateBuildStatusErrorMessage"
    },
    "UpdateSuccessfulRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateSuccessfulHistory"
    },
    "UpdateSuccessfulHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerADLDSCreation"
    },
    "TriggerADLDSCreation": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/adlds-creation', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body": {
            "EIMID.$": "$.adlds_details.EIMID",
            "basic_entitlementName.$": "$.adlds_details.basic_entitlementName",
            "account_id.$": "$.account_id",
            "friendRequestId.$": "$.adlds_details.friendRequestId",
            "environment.$": "$.adlds_details.environment"
          }
        }
      },
      "ResultPath": "$.adlds_trigger_results",
      "Next": "UpdateADLDSRecord",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedADLDSRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "UpdateADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #eimid = :eimid, #adlds_status = :adlds_status, #adlds_start = :adlds_start, #adlds_exec_id = :adlds_exec_id",
        "ExpressionAttributeNames": {
          "#eimid": "eimid",
          "#adlds_status": "adlds_status",
          "#adlds_start": "adlds_start_at",
          "#adlds_exec_id": "adlds_execution_id"
        },
        "ExpressionAttributeValues": {
          ":eimid": {
            "S.$": "$.adlds_details.EIMID"
          },
          ":adlds_status": {
            "S": "IN_PROGRESS"
          },
          ":adlds_start": {
            "S.$": "$$.State.EnteredTime"
          },
          ":adlds_exec_id": {
            "S.$": "$.adlds_trigger_results.body.correlationId"
          }
        }
      },
      "ResultPath": null,
      "Next": "WaitForADLDSStatus",
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForADLDSStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetADLDSStatus"
    },
    "GetADLDSStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/status/{}', $.url, $.adlds_trigger_results.body.correlationId)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.adlds_status_results",
      "Next": "CheckADLDSStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedADLDSRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckADLDSStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.adlds_status_results.body.status",
          "StringEquals": "COMPLETED",
          "Next": "UpdateSuccessfulADLDSRecord"
        },
        {
          "Variable": "$.adlds_status_results.body.status",
          "StringEquals": "FAILED",
          "Next": "UpdateFailedADLDSRecord"
        }
      ],
      "Default": "WaitForADLDSStatus"
    },
    "UpdateSuccessfulADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #adlds_status = :adlds_status",
        "ExpressionAttributeNames": {
          "#adlds_status": "adlds_status"
        },
        "ExpressionAttributeValues": {
          ":adlds_status": {
            "S": "COMPLETED"
          }
        }
      },
      "ResultPath": null,
      "End": true,
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "GenerateBuildStatusErrorMessage": {
      "Type": "Pass",
      "Next": "UpdateFailedRecord",
      "Result": "Build status is not in [SUCCEEDED, IN_PROGRESS].",
      "ResultPath": "$.error_info"
    },
    "UpdateFailedRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#exe_status": "status",
          "#error_info": "error"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.error_info)"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateFailedHistory"
    },
    "UpdateFailedHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Next": "FailForBuild"
    },
    "FailForBuild": {
      "Type": "Fail",
      "Cause": "Error deploying IAM role",
      "Error": "See logs"
    },
    "UpdateFailedADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #adlds_status = :adlds_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#adlds_status": "adlds_status",
          "#error_info": "adlds_error"
        },
        "ExpressionAttributeValues": {
          ":adlds_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.adlds_status_results.body.error)"
          }
        }
      },
      "ResultPath": null,
      "Next": "FailForADLDS",
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "FailForADLDS": {
      "Type": "Fail",
      "Cause": "Error creating ADLDS group",
      "Error": "See adlds_error in DynamoDB record for details"
    }
  }
}

```
```
{
  "Comment": "A state machine to orchestrate IAM role creation and AD/DS group creation",
  "StartAt": "UpsertRequestRecord",
  "States": {
    "UpsertRequestRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Item": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          },
          "build_type": {
            "S.$": "$.iam_details.build_type"
          },
          "action": {
            "S.$": "$.iam_details.action"
          },
          "custom_role_name": {
            "S.$": "$.iam_details.custom_role_name"
          },
          "dry_run": {
            "BOOL.$": "$.iam_details.dry_run"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "correlation_id": {
            "S.$": "$.correlation_id"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "InsertRequestHistory"
    },
    "InsertRequestHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Item": {
          "request_id": {
            "S.$": "$.correlation_id"
          },
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          },
          "action": {
            "S.$": "$.iam_details.action"
          },
          "job_id": {
            "S.$": "$$.Execution.Id"
          },
          "start_at": {
            "S.$": "$$.Execution.StartTime"
          },
          "status": {
            "S": "IN_PROGRESS"
          },
          "trace_header": {
            "S.$": "$.trace_header"
          }
        }
      },
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerBuild"
    },
    "TriggerBuild": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/build', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body.$": "$.iam_details"
        }
      },
      "ResultPath": "$.build_trigger_results",
      "Next": "WaitForBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForBuildStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetBuildStatus"
    },
    "GetBuildStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/build/{}', $.url, $.build_trigger_results.body.build_id)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.build_status_results",
      "Next": "CheckBuildStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckBuildStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.build_status_results.body.can_retry",
          "BooleanEquals": true,
          "Next": "WaitForBuildStatus"
        },
        {
          "Variable": "$.build_status_results.body.build_status",
          "StringEquals": "SUCCEEDED",
          "Next": "UpdateSuccessfulRecord"
        }
      ],
      "Default": "GenerateBuildStatusErrorMessage"
    },
    "UpdateSuccessfulRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateSuccessfulHistory"
    },
    "UpdateSuccessfulHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "COMPLETE"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "TriggerADLDSCreation"
    },
    "TriggerADLDSCreation": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "POST",
          "url.$": "States.Format('{}/adlds-creation', $.url)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          },
          "body": {
            "EIMID.$": "$.adlds_details.EIMID",
            "basic_entitlementName.$": "$.adlds_details.basic_entitlementName",
            "account_id.$": "$.account_id",
            "friendRequestId.$": "$.adlds_details.friendRequestId",
            "environment.$": "$.adlds_details.environment"
          }
        }
      },
      "ResultPath": "$.adlds_trigger_results",
      "Next": "UpdateADLDSRecord",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedADLDSRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "UpdateADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #eimid = :eimid, #adlds_status = :adlds_status, #adlds_start = :adlds_start, #adlds_exec_id = :adlds_exec_id",
        "ExpressionAttributeNames": {
          "#eimid": "eimid",
          "#adlds_status": "adlds_status",
          "#adlds_start": "adlds_start_at",
          "#adlds_exec_id": "adlds_execution_id"
        },
        "ExpressionAttributeValues": {
          ":eimid": {
            "S.$": "$.adlds_details.EIMID"
          },
          ":adlds_status": {
            "S": "IN_PROGRESS"
          },
          ":adlds_start": {
            "S.$": "$$.State.EnteredTime"
          },
          ":adlds_exec_id": {
            "S.$": "$.adlds_trigger_results.body.correlationId"
          }
        }
      },
      "ResultPath": null,
      "Next": "WaitForADLDSStatus",
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "WaitForADLDSStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetADLDSStatus"
    },
    "GetADLDSStatus": {
      "Type": "Task",
      "Resource": "${module.lambda_function.iam_role_orchestrator_proxy.function_arn}",
      "Parameters": {
        "Payload": {
          "method": "GET",
          "url.$": "States.Format('{}/status/{}', $.url, $.adlds_trigger_results.body.correlationId)",
          "headers": {
            "Authorization": "jwt",
            "Encrypted-Authorization.$": "$.encrypted_jwt",
            "Content-Type": "application/json",
            "Referer": "${local.api_alb_fqdn}/proxy",
            "x-amzn-RequestId.$": "$.correlation_id"
          }
        }
      },
      "ResultPath": "$.adlds_status_results",
      "Next": "CheckADLDSStatus",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "UpdateFailedADLDSRecord",
          "ResultPath": "$.error_info"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": [
            "CustomNonRetryableError"
          ],
          "MaxAttempts": 0
        },
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "CheckADLDSStatus": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.adlds_status_results.body.status",
          "StringEquals": "COMPLETED",
          "Next": "UpdateSuccessfulADLDSRecord"
        },
        {
          "Variable": "$.adlds_status_results.body.status",
          "StringEquals": "FAILED",
          "Next": "UpdateFailedADLDSRecord"
        }
      ],
      "Default": "GenerateADLDSStatusErrorMessage"
    },
    "GenerateADLDSStatusErrorMessage": {
      "Type": "Pass",
      "Parameters": {
        "error_info": {
          "error_message.$": "States.Format('Unexpected ADLDS status: {}. Error message: {}', $.adlds_status_results.body.status, $.adlds_status_results.body.message)"
        }
      },
      "ResultPath": "$.error_info",
      "Next": "UpdateFailedADLDSRecord"
    },
    "UpdateSuccessfulADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #adlds_status = :adlds_status",
        "ExpressionAttributeNames": {
          "#adlds_status": "adlds_status"
        },
        "ExpressionAttributeValues": {
          ":adlds_status": {
            "S": "COMPLETED"
          }
        }
      },
      "ResultPath": null,
      "End": true,
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "GenerateBuildStatusErrorMessage": {
      "Type": "Pass",
      "Next": "UpdateFailedRecord",
      "Result": "Build status is not in [SUCCEEDED, IN_PROGRESS].",
      "ResultPath": "$.error_info"
    },
    "UpdateFailedRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#exe_status": "status",
          "#error_info": "error"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.error_info)"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "ResultPath": null,
      "Next": "UpdateFailedHistory"
    },
    "UpdateFailedHistory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_request_history.id}",
        "Key": {
          "request_id": {
            "S.$": "$.correlation_id"
          }
        },
        "UpdateExpression": "SET #exe_status = :new_status",
        "ExpressionAttributeNames": {
          "#exe_status": "status"
        },
        "ExpressionAttributeValues": {
          ":new_status": {
            "S": "FAILED"
          }
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Next": "FailForBuild"
    },
    "FailForBuild": {
      "Type": "Fail",
      "Cause": "Error deploying IAM role",
      "Error": "See logs"
    },
    "UpdateFailedADLDSRecord": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "${aws_dynamodb_table.iam_orchestrator_execution_status.id}",
        "Key": {
          "account_id": {
            "S.$": "$.account_id"
          },
          "resource": {
            "S.$": "$.resource"
          }
        },
        "UpdateExpression": "SET #adlds_status = :adlds_status, #error_info = :error_clause",
        "ExpressionAttributeNames": {
          "#adlds_status": "adlds_status",
          "#error_info": "adlds_error"
        },
        "ExpressionAttributeValues": {
          ":adlds_status": {
            "S": "FAILED"
          },
          ":error_clause": {
            "S.$": "States.JsonToString($.error_info)"
          }
        }
      },
      "ResultPath": null,
      "Next": "FailForADLDS",
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "IntervalSeconds": 3,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ]
    },
    "FailForADLDS": {
      "Type": "Fail",
      "Cause": "Error in ADLDS group creation process",
      "Error.$": "$.error_info.error_message"
    }
  }
}
```
