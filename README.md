
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
```
import re
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.logging import correlation_id
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from botocore.exceptions import ClientError
import json
import os
import boto3
from pydantic import BaseModel, validator, root_validator
from typing import List, Optional
from boto3.dynamodb.types import TypeDeserializer
import requests
from datetime import datetime, timedelta

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

ssm = boto3.client('ssm')
dynamodb = boto3.client('dynamodb')

ALLOWED_AWS_REGION = ["us-east-1", "us-west-2"]  # Add your allowed regions here

class ChangeDetails(BaseModel):
    change_request_id: str
    url: str
    vpce_id: str
    region: str
    force_submit: bool = False

    @validator("change_request_id")
    def valid_change_request_id(cls, change_request_id: str) -> str:
        if not re.compile(r"^(change123).{2,}$").match(change_request_id):
            raise ValueError(f"{change_request_id} is not a valid change_request_id")
        return change_request_id

    @validator("region")
    def valid_region(cls, region: str) -> str:
        if region not in ALLOWED_AWS_REGION:
            raise ValueError(f"{region} is not a valid region in {ALLOWED_AWS_REGION}")
        return region

class RoleCreationRequest(BaseModel):
    account_id: str
    creation_type: str
    change_list: List[ChangeDetails]

class StatusUpdate(BaseModel):
    change_request_id: str
    account_id: str

    @root_validator(skip_on_failure=True)
    def check_valid(cls, values: dict) -> dict:
        if not values.get("change_request_id") and not values.get("account_id"):
            raise ValueError("This is required query parameter.")
        return values

class ChangeRequest(BaseModel):
    short_description: str
    description: str
    assignment_group: str
    start_date: Optional[str] = None
    end_date: Optional[str] = None
    implementation_plan: str = "Automated process will create the IAM role upon approval"
    test_plan: str = "Automated tests will be run to verify the role creation"
    rollback_plan: str = "Automated process will delete the IAM role if needed"
    justification: str = "Required for automated IAM role creation"

def deserialize_db_item(item: dict) -> dict:
    deserializer = TypeDeserializer()
    return {
        "change_request_id": deserializer.deserialize(item["change_request_id"]),
        "region": deserializer.deserialize(item["region"]),
        "creation_type": deserializer.deserialize(item["creation_type"]),
        "request_body": deserializer.deserialize(item["request_body"]),
        "trace_number": deserializer.deserialize(item["trace_number"]),
        "status_message": deserializer.deserialize(item["status_message"]),
    }

def get_request_status_by_change_request_id(change_request_id: str) -> dict:
    response = dynamodb.get_item(
        TableName=os.environ['DYNAMODB_TABLE_NAME'],
        Key={'change_request_id': {'S': change_request_id}}
    )
    return deserialize_db_item(response['Item'])

def get_request_status_by_account_id(account_id: str) -> list:
    response = dynamodb.query(
        TableName=os.environ['DYNAMODB_TABLE_NAME'],
        IndexName='account_id-index',
        KeyConditionExpression='account_id = :account_id',
        ExpressionAttributeValues={':account_id': {'S': account_id}}
    )
    return [deserialize_db_item(item) for item in response['Items']]

def get_snow_auth():
    snow_username = ssm.get_parameter(Name=os.environ['SNOW_USERNAME_PARAM'], WithDecryption=True)['Parameter']['Value']
    snow_password = ssm.get_parameter(Name=os.environ['SNOW_PASSWORD_PARAM'], WithDecryption=True)['Parameter']['Value']
    return snow_username, snow_password

def create_snow_change_request(change_request: ChangeRequest):
    snow_instance = os.environ['SNOW_INSTANCE']
    snow_username, snow_password = get_snow_auth()

    url = f"https://{snow_instance}.service-now.com/api/now/table/change_request"
    headers = {"Content-Type": "application/json"}
    
    payload = change_request.dict()
    payload["u_change_type"] = "normal"
    payload["u_change_class"] = "standard"
    
    if not payload.get("start_date"):
        payload["start_date"] = (datetime.now() + timedelta(hours=1)).strftime("%Y-%m-%d %H:%M:%S")
    if not payload.get("end_date"):
        payload["end_date"] = (datetime.now() + timedelta(hours=2)).strftime("%Y-%m-%d %H:%M:%S")

    response = requests.post(url, auth=(snow_username, snow_password), headers=headers, json=payload)
    response.raise_for_status()
    
    return response.json()["result"]["sys_id"]

@tracer.capture_method
def create_change_request(payload, account_entity):
    approval_group = account_entity.get("ApprovalGroup", "NotAvailable")
    
    if approval_group == "NotAvailable":
        logger.error("ApprovalGroup is not available for Prod account")
        return {"error": "ApprovalGroup not available"}, HTTPStatus.BAD_REQUEST.value
    
    try:
        change_request = ChangeRequest(
            short_description=f"Create IAM role {payload['custom_role_name']} in account {payload['account_id']}",
            description=f"Create IAM role {payload['custom_role_name']} in account {payload['account_id']}",
            assignment_group=approval_group
        )
        
        cr_sys_id = create_snow_change_request(change_request)
        logger.info(f"Change Request created successfully: {cr_sys_id}")
        return {"message": "Change Request created", "cr_sys_id": cr_sys_id}, HTTPStatus.ACCEPTED.value

    except requests.RequestException as e:
        logger.exception("Error occurred while creating Change Request")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value
    except Exception as e:
        logger.exception("Unexpected error occurred while creating Change Request")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value

@tracer.capture_method
def sfn_trigger_handler(payload):
    resp = {"request_id": logger.get_correlation_id()}
    try:
        logger.append_keys(account_id=payload["account_id"], role_name=payload["custom_role_name"])
        SfnRequest.validate_payload_params(payload)
        account_entity = utils.get_account(payload["account_id"])
        utils.raise_for_ineligible_request(payload, account_entity)
        
        # Check account placement
        placement = account_entity.get("Placement", "").lower()
        if placement == "prod":
            return create_change_request(payload, account_entity)
        elif placement in ["development", "pre-prod"]:
            SfnRequest(
                utils.decrypt_token(app.current_event.get_header_value(name="AuthorizationToken", case_sensitive=False)),
                payload,
                account_entity,
                logger.get_correlation_id()
            ).execute()
        else:
            raise ValueError(f"Invalid account placement: {placement}")

    except Exception as e:
        logger.exception("Failed to trigger step function")
        tracer.provider.current_subsegment().add_error_flag()
        tracer.capture_exception(e, traceback.extract_stack())
        resp["error"] = type(e).__name__
        if hasattr(e, "status_code"):
            return resp, e.status_code
        return resp, HTTPStatus.INTERNAL_SERVER_ERROR.value
    finally:
        logger.remove_keys(["account_id", "custom_role_name"])
    return resp, HTTPStatus.ACCEPTED.value

@app.post("/role/create")
def handle_custom_role():
    request_body = app.current_event.json_body
    account_id = request_body.get("account_id")
    custom_role_name = request_body.get("custom_role_name")
    payload = {
        "account_id": account_id,
        "build_type": "custom",
        "action": "apply",
        "custom_role_name": custom_role_name,
        "dry_run": True,
        "update_init_role": False
    }
    print(payload)
    return sfn_trigger_handler(payload)

@app.get("/status")
def get_request_status():
    try:
        data = StatusUpdate(**app.current_event.query_string_parameters)
        if data.change_request_id:
            result = get_request_status_by_change_request_id(data.change_request_id)
        elif data.account_id:
            result = get_request_status_by_account_id(data.account_id)
        else:
            return {"error": "Invalid request"}, HTTPStatus.BAD_REQUEST.value
        return result, HTTPStatus.OK.value
    except Exception as e:
        logger.exception("Error retrieving request status")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)
```

```
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.logging import correlation_id
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
import asyncio
from utils.aws import RemovalStatusDDBClient
from utils.omni import OmniClient
from utils.constants import CHANGE_REQUEST_STATUS
import json
from datetime import datetime, timedelta
from aws_lambda_powertools.utilities.parser import parse
from typing import Optional
from pydantic import BaseModel

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

@tracer.capture_method
async def create_change_request(payload, account_entity):
    approval_group = account_entity.get("ApprovalGroup", "NotAvailable")
    
    if approval_group == "NotAvailable":
        logger.error("ApprovalGroup is not available for Prod account")
        return {"error": "ApprovalGroup not available"}, HTTPStatus.BAD_REQUEST.value
    
    try:
        ddb_client = RemovalStatusDDBClient()

        # Check for existing in-progress requests
        existing_requests = ddb_client.query_items_with_index(
            index_name="account_id-index",
            key_condition_expression={
                "ComparisonOperator": "EQ",
                "AttributeValueList": [{"S": payload["account_id"]}]
            }
        )

        for request in existing_requests:
            request_body = json.loads(request.get("request_body", "{}"))
            if (request_body.get("custom_role_name") == payload["custom_role_name"] and
                request["status_message"]["S"] in [CHANGE_REQUEST_STATUS.CREATED.value, CHANGE_REQUEST_STATUS.IN_PROGRESS.value]):
                logger.info(f"Request already exists for role {payload['custom_role_name']} in account {payload['account_id']}. "
                            f"Correlation ID: {request['trace_number']['S']}")
                return {
                    "message": "Change Request already exists and is in progress",
                    "request_id": request["trace_number"]["S"],
                    "account_id": payload['account_id'],
                }, HTTPStatus.CONFLICT.value

        # Use logger.get_correlation_id() as the unique identifier
        request_id = logger.get_correlation_id()

        # Prepare the item for DynamoDB
        item = {
            'trace_number': {'S': request_id},
            'account_id': {'S': payload['account_id']},
            'creation_type': {'S': 'custom_role'},
            'request_body': {'S': json.dumps(payload)},
            'status_message': {'S': CHANGE_REQUEST_STATUS.CREATED.value},
            'created_at': {'S': datetime.now().isoformat()},
            'updated_at': {'S': datetime.now().isoformat()},
            'approval_group': {'S': approval_group}
        }
        
        # Record the change request in DynamoDB
        ddb_client.put_item(item)
        
        # Start asynchronous creation of ServiceNow change request
        asyncio.create_task(create_snow_change_request(request_id, payload, approval_group))
        
        return {
            "message": "Change Request is being created",
            "request_id": request_id,
            "account_id": payload['account_id'],
            "creation_type": "custom_role"
        }, HTTPStatus.ACCEPTED.value

    except Exception as e:
        logger.exception("Unexpected error occurred while initiating Change Request")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value

@tracer.capture_method
async def create_snow_change_request(request_id, payload, approval_group):
    try:
        ddb_client = RemovalStatusDDBClient()
        sn_secret = get_secret(SN_SECRET_NAME)
        if not sn_secret:
            raise ValueError("Failed to retrieve ServiceNow secret")

        sn_creds = json.loads(sn_secret)
        omni_client = OmniClient(
            sn_authorization=sn_creds.get('sn_authentication_bearer_token'),
            sn_client_id=sn_creds.get('sn_client_id'),
            sn_client_secret=sn_creds.get('sn_client_secret'),
            sn_change_requestid=sn_creds.get('sn_change_x_request_id')
        )

        data: Optional[CRData] = get_cr_data(ddb_client, approval_group, payload)
        if not data:
            raise ValueError("Failed to prepare CR data")

        omni_client.set_user(staff_id=data.staff_id, verify_config=True)
        
        cr_number = omni_client.create_cr(**data.dict())
        logger.info(f"Change Request created successfully: {cr_number}")

        # Update DynamoDB with the cr_number
        ddb_client.update_item(
            request_id,
            {
                'cr_number': {'S': cr_number},
                'cr_status': {'S': 'New'},
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_success'},
                    'description': {'S': f"<{cr_number}> CR created successfully."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

        # Assess the change request
        omni_client.assess_cr(cr_number)
        logger.info(f"Change Request {cr_number} assessed successfully")

        # Update DynamoDB after assessment
        ddb_client.update_item(
            request_id,
            {
                'cr_status': {'S': 'Assess'},
                'status_message': {'M': {
                    'status_code': {'S': 'assess_cr_success'},
                    'description': {'S': "The CR has been moved to assess."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

    except Exception as e:
        logger.exception(f"Error creating ServiceNow change request for request_id {request_id}")
        # Update DynamoDB with error status
        ddb_client.update_item(
            request_id,
            {
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_failed'},
                    'description': {'S': str(e)}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

@tracer.capture_method
def sfn_trigger_handler(payload):
    resp = {"request_id": logger.get_correlation_id()}
    try:
        logger.append_keys(account_id=payload["account_id"], role_name=payload["custom_role_name"])
        SfnRequest.validate_payload_params(payload)
        account_entity = utils.get_account(payload["account_id"])
        utils.raise_for_ineligible_request(payload, account_entity)
        
        # Check account placement
        placement = account_entity.get("Placement", "").lower()
        if placement == "prod":
            return create_change_request(payload, account_entity)
        elif placement in ["development", "pre-prod"]:
            SfnRequest(
                util.decrypt_token(app.current_event.get_header_value(name="AuthorizationToken", case_sensitive=False)),
                payload,
                account_entity,
                logger.get_correlation_id()
            ).execute()
        else:
            raise ValueError(f"Invalid account placement: {placement}")

    except Exception as e:
        logger.exception("Failed to trigger step function")
        tracer.provider.current_subsegment().add_error_flag()
        tracer.capture_exception(e, traceback.extract_stack())
        resp["error"] = type(e).__name__
        if hasattr(e, "status_code"):
            return resp, e.status_code
        return resp, HTTPStatus.INTERNAL_SERVER_ERROR.value
    finally:
        logger.remove_keys(["account_id", "custom_role_name"])
    return resp, HTTPStatus.ACCEPTED.value

@app.post("/role/create")
def handle_custom_role():
    request_body = app.current_event.json_body
    account_id = request_body.get("account_id")
    custom_role_name = request_body.get("custom_role_name")
    payload = {
        "account_id": account_id,
        "build_type": "custom",
        "action": "apply",
        "custom_role_name": custom_role_name,
        "dry_run": True,
        "update_init_role": False
    }
    print(payload)
    return sfn_trigger_handler(payload)

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)

class CRData(BaseModel):
    account_id: str = None
    schedule_date: str = None
    location: str = "Asia/Hong_Kong"
    staff_id: str = None
    approval_group: str
    email: str = None
    custom_role_name: str = None

    @root_validator(skip_on_failure=True)
    def check_model(cls, values: dict) -> dict:
        values['staff_id'], values['email'] = random.choice(list(CONTACT_MAPPING.items()))
        values['schedule_date'] = '2024-11-01 08:30:00'
        return values

@tracer.capture_method
def get_cr_data(ddb_client: RemovalStatusDDBClient, approval_group: str, payload: dict) -> Optional[CRData]:
    try:
        data: CRData = parse(
            CRData,
            {
                "approval_group": approval_group,
                "account_id": payload["account_id"],
                "custom_role_name": payload["custom_role_name"]
            }
        )
        return data
    except ValidationError as e:
        logger.error(e)
        return None
```
```
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.logging import correlation_id
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
import asyncio
from utils.aws import get_secret, RemovalStatusDDBClient
from utils.params import CONTACT_MAPPING, SidecarRemovalRequestBody
from utils.omni import OmniClient
from utils.constants import CHANGE_REQUEST_STATUS
from aws_lambda_powertools.utilities.parser import parse
import json
from datetime import datetime
import os
import random

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

SN_SECRET_NAME = os.environ.get('SN_SECRET_NAME')

class CRData(BaseModel):
    account_id: str = None
    schedule_date: str = None
    location: str = "Asia/Hong_Kong"
    staff_id: str = None
    approval_group: str
    email: str = None
    custom_role_name: str = None

    @root_validator(skip_on_failure=True)
    def check_model(cls, values: dict) -> dict:
        values['staff_id'], values['email'] = random.choice(list(CONTACT_MAPPING.items()))
        values['schedule_date'] = '2024-11-01 08:30:00'
        return values

@tracer.capture_method
def get_cr_data(ddb_client: RemovalStatusDDBClient, approval_group: str, payload: dict) -> Optional[CRData]:
    try:
        data: CRData = parse(
            CRData,
            {
                "approval_group": approval_group,
                "account_id": payload["account_id"],
                "custom_role_name": payload["custom_role_name"]
            }
        )
        return data
    except ValidationError as e:
        logger.error(e)
        return None

@tracer.capture_method
async def create_snow_change_request(request_id, payload, approval_group):
    try:
        ddb_client = RemovalStatusDDBClient()
        sn_secret = get_secret(SN_SECRET_NAME)
        if not sn_secret:
            raise ValueError("Failed to retrieve ServiceNow secret")

        sn_creds = json.loads(sn_secret)
        omni_client = OmniClient(
            sn_authorization=sn_creds.get('sn_authentication_bearer_token'),
            sn_client_id=sn_creds.get('sn_client_id'),
            sn_client_secret=sn_creds.get('sn_client_secret'),
            sn_change_requestid=sn_creds.get('sn_change_x_request_id')
        )

        data: Optional[CRData] = get_cr_data(ddb_client, approval_group, payload)
        if not data:
            raise ValueError("Failed to prepare CR data")

        omni_client.set_user(staff_id=data.staff_id, verify_config=True)
        
        cr_number = omni_client.create_cr(**data.dict())
        logger.info(f"Change Request created successfully: {cr_number}")

        # Update DynamoDB with the cr_number
        ddb_client.update_item(
            request_id,
            {
                'cr_number': {'S': cr_number},
                'cr_status': {'S': 'New'},
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_success'},
                    'description': {'S': f"<{cr_number}> CR created successfully."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

        # Assess the change request
        omni_client.assess_cr(cr_number)
        logger.info(f"Change Request {cr_number} assessed successfully")

        # Update DynamoDB after assessment
        ddb_client.update_item(
            request_id,
            {
                'cr_status': {'S': 'Assess'},
                'status_message': {'M': {
                    'status_code': {'S': 'assess_cr_success'},
                    'description': {'S': "The CR has been moved to assess."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

    except Exception as e:
        logger.exception(f"Error creating ServiceNow change request for request_id {request_id}")
        # Update DynamoDB with error status
        ddb_client.update_item(
            request_id,
            {
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_failed'},
                    'description': {'S': str(e)}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

@tracer.capture_method
async def create_change_request(payload, account_entity):
    approval_group = account_entity.get("ApprovalGroup", "NotAvailable")
    
    if approval_group == "NotAvailable":
        logger.error("ApprovalGroup is not available for Prod account")
        return {"error": "ApprovalGroup not available"}, HTTPStatus.BAD_REQUEST.value
    
    try:
        ddb_client = RemovalStatusDDBClient()

        # Check for existing in-progress requests
        existing_requests = ddb_client.query_items_with_index(
            index_name="account_id-index",
            key_condition_expression={
                "ComparisonOperator": "EQ",
                "AttributeValueList": [{"S": payload["account_id"]}]
            }
        )

        for request in existing_requests:
            request_body = json.loads(request.get("request_body", "{}"))
            if (request_body.get("custom_role_name") == payload["custom_role_name"] and
                request["status_message"]["S"] in [CHANGE_REQUEST_STATUS.CREATED.value, CHANGE_REQUEST_STATUS.IN_PROGRESS.value]):
                logger.info(f"Request already exists for role {payload['custom_role_name']} in account {payload['account_id']}. "
                            f"Correlation ID: {request['trace_number']['S']}")
                return {
                    "message": "Change Request already exists and is in progress",
                    "request_id": request["trace_number"]["S"],
                    "account_id": payload['account_id'],
                }, HTTPStatus.CONFLICT.value

        # Use logger.get_correlation_id() as the unique identifier
        request_id = logger.get_correlation_id()

        # Prepare the item for DynamoDB
        item = {
            'trace_number': {'S': request_id},
            'account_id': {'S': payload['account_id']},
            'creation_type': {'S': 'custom_role'},
            'request_body': {'S': json.dumps(payload)},
            'status_message': {'S': CHANGE_REQUEST_STATUS.CREATED.value},
            'created_at': {'S': datetime.now().isoformat()},
            'updated_at': {'S': datetime.now().isoformat()},
            'approval_group': {'S': approval_group}
        }
        
        # Record the change request in DynamoDB
        ddb_client.put_item(item)
        
        # Start asynchronous creation of ServiceNow change request
        asyncio.create_task(create_snow_change_request(request_id, payload, approval_group))
        
        return {
            "message": "Change Request is being created",
            "request_id": request_id,
            "account_id": payload['account_id'],
            "creation_type": "custom_role"
        }, HTTPStatus.ACCEPTED.value

    except Exception as e:
        logger.exception("Unexpected error occurred while initiating Change Request")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value

@tracer.capture_method
def sfn_trigger_handler(payload):
    resp = {"request_id": logger.get_correlation_id()}
    try:
        logger.append_keys(account_id=payload["account_id"], role_name=payload["custom_role_name"])
        SfnRequest.validate_payload_params(payload)
        account_entity = utils.get_account(payload["account_id"])
        utils.raise_for_ineligible_request(payload, account_entity)
        
        # Check account placement
        placement = account_entity.get("Placement", "").lower()
        if placement == "prod":
            return create_change_request(payload, account_entity)
        elif placement in ["development", "pre-prod"]:
            SfnRequest(
                util.decrypt_token(app.current_event.get_header_value(name="AuthorizationToken", case_sensitive=False)),
                payload,
                account_entity,
                logger.get_correlation_id()
            ).execute()
        else:
            raise ValueError(f"Invalid account placement: {placement}")

    except Exception as e:
        logger.exception("Failed to trigger step function")
        tracer.provider.current_subsegment().add_error_flag()
        tracer.capture_exception(e, traceback.extract_stack())
        resp["error"] = type(e).__name__
        if hasattr(e, "status_code"):
            return resp, e.status_code
        return resp, HTTPStatus.INTERNAL_SERVER_ERROR.value
    finally:
        logger.remove_keys(["account_id", "custom_role_name"])
    return resp, HTTPStatus.ACCEPTED.value

@app.post("/role/create")
def handle_custom_role():
    request_body = app.current_event.json_body
    account_id = request_body.get("account_id")
    custom_role_name = request_body.get("custom_role_name")
    payload = {
        "account_id": account_id,
        "build_type": "custom",
        "action": "apply",
        "custom_role_name": custom_role_name,
        "dry_run": True,
        "update_init_role": False
    }
    print(payload)
    return sfn_trigger_handler(payload)

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)
```

```
existing_requests = ddb_client.query_items_with_index(
            index_name="account_id-index",
            key_condition_expression="account_id = :account_id",
            expression_attribute_values={
                ":account_id": {"S": payload["account_id"]}
            }
        )
```
```
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.logging import correlation_id
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
import asyncio
from utils.aws import get_secret, RemovalStatusDDBClient
from utils.params import CONTACT_MAPPING, SidecarRemovalRequestBody
from utils.omni import OmniClient
from utils.constants import CHANGE_REQUEST_STATUS
from aws_lambda_powertools.utilities.parser import parse, ValidationError
from pydantic import BaseModel, root_validator
import json
from datetime import datetime
import os
import random
from typing import Optional
import threading

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

SN_SECRET_NAME = os.environ.get('SN_SECRET_NAME')

class CRData(BaseModel):
    account_id: str = None
    schedule_date: str = None
    location: str = "Asia/Hong_Kong"
    staff_id: str = None
    approval_group: str
    email: str = None
    custom_role_name: str = None

    @root_validator(skip_on_failure=True)
    def check_model(cls, values: dict) -> dict:
        values['staff_id'], values['email'] = random.choice(list(CONTACT_MAPPING.items()))
        values['schedule_date'] = '2024-11-01 08:30:00'
        return values

@tracer.capture_method
def get_cr_data(ddb_client: RemovalStatusDDBClient, approval_group: str, payload: dict) -> Optional[CRData]:
    try:
        data: CRData = parse(
            CRData,
            {
                "approval_group": approval_group,
                "account_id": payload["account_id"],
                "custom_role_name": payload["custom_role_name"]
            }
        )
        return data
    except ValidationError as e:
        logger.error(e)
        return None

def run_async_function(coroutine, *args, **kwargs):
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop.run_until_complete(coroutine(*args, **kwargs))
    loop.close()

@tracer.capture_method
def create_change_request(payload, account_entity):
    approval_group = account_entity.get("ApprovalGroup", "NotAvailable")
    
    if approval_group == "NotAvailable":
        logger.error("ApprovalGroup is not available for Prod account")
        return {"error": "ApprovalGroup not available"}, HTTPStatus.BAD_REQUEST.value
    
    try:
        ddb_client = RemovalStatusDDBClient()

        # Check for existing in-progress requests
        existing_requests = ddb_client.query_items_with_index(
            index_name="account_id-index",
            key_condition_expression="account_id = :account_id",
            expression_attribute_values={
                ":account_id": {"S": payload["account_id"]}
            }
        )

        for request in existing_requests:
            request_body = json.loads(request.get("request_body", "{}"))
            if (request_body.get("custom_role_name") == payload["custom_role_name"] and
                request["status_message"]["S"] in ["CREATED", "IN_PROGRESS"]):
                logger.info(f"Request already exists for role {payload['custom_role_name']} in account {payload['account_id']}. "
                            f"Correlation ID: {request['trace_number']['S']}")
                return {
                    "message": "Change Request already exists and is in progress",
                    "request_id": request["trace_number"]["S"],
                    "account_id": payload['account_id'],
                }, HTTPStatus.CONFLICT.value

        # Use logger.get_correlation_id() as the unique identifier
        request_id = logger.get_correlation_id()

        # Prepare the item for DynamoDB
        item = {
            'trace_number': {'S': request_id},
            'account_id': {'S': payload['account_id']},
            'creation_type': {'S': 'custom_role'},
            'request_body': {'S': json.dumps(payload)},
            'status_message': {'S': "CREATED"},
            'created_at': {'S': datetime.now().isoformat()},
            'updated_at': {'S': datetime.now().isoformat()},
            'approval_group': {'S': approval_group}
        }
        
        # Record the change request in DynamoDB
        ddb_client.put_item(item)
        
        # Start a new thread for ServiceNow change request creation
        thread = threading.Thread(target=run_async_function, args=(create_snow_change_request, request_id, payload, approval_group))
        thread.start()
        
        return {
            "message": "Change Request is being created",
            "request_id": request_id,
            "account_id": payload['account_id'],
            "creation_type": "custom_role"
        }, HTTPStatus.ACCEPTED.value

    except Exception as e:
        logger.exception("Unexpected error occurred while initiating Change Request")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value

@tracer.capture_method
async def create_snow_change_request(request_id, payload, approval_group):
    try:
        ddb_client = RemovalStatusDDBClient()
        sn_secret = get_secret(SN_SECRET_NAME)
        if not sn_secret:
            raise ValueError("Failed to retrieve ServiceNow secret")

        sn_creds = json.loads(sn_secret)
        omni_client = OmniClient(
            sn_authorization=sn_creds.get('sn_authentication_bearer_token'),
            sn_client_id=sn_creds.get('sn_client_id'),
            sn_client_secret=sn_creds.get('sn_client_secret'),
            sn_change_requestid=sn_creds.get('sn_change_x_request_id')
        )

        data = get_cr_data(ddb_client, approval_group, payload)
        if not data:
            raise ValueError("Failed to prepare CR data")

        omni_client.set_user(staff_id=data.staff_id, verify_config=True)
        
        cr_number = await omni_client.create_cr(**data.dict())
        logger.info(f"Change Request created successfully: {cr_number}")

        # Update DynamoDB with the cr_number
        ddb_client.update_item(
            request_id,
            {
                'cr_number': {'S': cr_number},
                'cr_status': {'S': 'New'},
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_success'},
                    'description': {'S': f"<{cr_number}> CR created successfully."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

        # Assess the change request
        await omni_client.assess_cr(cr_number)
        logger.info(f"Change Request {cr_number} assessed successfully")

        # Update DynamoDB after assessment
        ddb_client.update_item(
            request_id,
            {
                'cr_status': {'S': 'Assess'},
                'status_message': {'M': {
                    'status_code': {'S': 'assess_cr_success'},
                    'description': {'S': "The CR has been moved to assess."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

    except Exception as e:
        logger.exception(f"Error creating ServiceNow change request for request_id {request_id}")
        # Update DynamoDB with error status
        ddb_client.update_item(
            request_id,
            {
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_failed'},
                    'description': {'S': str(e)}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

@tracer.capture_method
def sfn_trigger_handler(payload):
    resp = {"request_id": logger.get_correlation_id()}
    try:
        logger.append_keys(account_id=payload["account_id"], role_name=payload["custom_role_name"])
        SfnRequest.validate_payload_params(payload)
        account_entity = utils.get_account(payload["account_id"])
        utils.raise_for_ineligible_request(payload, account_entity)
        
        # Check account placement
        placement = account_entity.get("Placement", "").lower()
        if placement == "prod":
            return create_change_request(payload, account_entity)
        elif placement in ["development", "pre-prod"]:
            SfnRequest(
                util.decrypt_token(app.current_event.get_header_value(name="AuthorizationToken", case_sensitive=False)),
                payload,
                account_entity,
                logger.get_correlation_id()
            ).execute()
        else:
            raise ValueError(f"Invalid account placement: {placement}")

    except Exception as e:
        logger.exception("Failed to trigger step function")
        tracer.provider.current_subsegment().add_error_flag()
        tracer.capture_exception(e, traceback.extract_stack())
        resp["error"] = type(e).__name__
        if hasattr(e, "status_code"):
            return resp, e.status_code
        return resp, HTTPStatus.INTERNAL_SERVER_ERROR.value
    finally:
        logger.remove_keys(["account_id", "custom_role_name"])
    return resp, HTTPStatus.ACCEPTED.value

@app.post("/role/create")
def handle_custom_role():
    request_body = app.current_event.json_body
    account_id = request_body.get("account_id")
    custom_role_name = request_body.get("custom_role_name")
    payload = {
        "account_id": account_id,
        "build_type": "custom",
        "action": "apply",
        "custom_role_name": custom_role_name,
        "dry_run": True,
        "update_init_role": False
    }
    print(payload)
    return sfn_trigger_handler(payload)

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)
```

```
def run_async_function(coroutine, *args, **kwargs):
    asyncio.run(coroutine(*args, **kwargs))
```

```
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.logging import correlation_id
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
import asyncio
from utils.aws import get_secret, RemovalStatusDDBClient
from utils.params import CONTACT_MAPPING, SidecarRemovalRequestBody
from utils.omni import OmniClient
from utils.constants import CHANGE_REQUEST_STATUS
from aws_lambda_powertools.utilities.parser import parse, ValidationError
from pydantic import BaseModel, root_validator
import json
from datetime import datetime
import os
import random
from typing import Optional

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

SN_SECRET_NAME = os.environ.get('SN_SECRET_NAME')

class CRData(BaseModel):
    account_id: str = None
    schedule_date: str = None
    location: str = "Asia/Hong_Kong"
    staff_id: str = None
    approval_group: str
    email: str = None
    custom_role_name: str = None

    @root_validator(skip_on_failure=True)
    def check_model(cls, values: dict) -> dict:
        values['staff_id'], values['email'] = random.choice(list(CONTACT_MAPPING.items()))
        values['schedule_date'] = '2024-11-01 08:30:00'
        return values

@tracer.capture_method
def get_cr_data(ddb_client: RemovalStatusDDBClient, approval_group: str, payload: dict) -> Optional[CRData]:
    try:
        data: CRData = parse(
            CRData,
            {
                "approval_group": approval_group,
                "account_id": payload["account_id"],
                "custom_role_name": payload["custom_role_name"]
            }
        )
        return data
    except ValidationError as e:
        logger.error(e)
        return None

def run_async_function(coroutine, *args, **kwargs):
    asyncio.run(coroutine(*args, **kwargs))

@tracer.capture_method
def create_change_request(payload, account_entity):
    approval_group = account_entity.get("ApprovalGroup", "NotAvailable")
    
    if approval_group == "NotAvailable":
        logger.error("ApprovalGroup is not available for Prod account")
        return {"error": "ApprovalGroup not available"}, HTTPStatus.BAD_REQUEST.value
    
    try:
        ddb_client = RemovalStatusDDBClient()

        # Check for existing in-progress requests
        existing_requests = ddb_client.query_items_with_index(
            index_name="account_id-index",
            key_condition_expression="account_id = :account_id",
            expression_attribute_values={
                ":account_id": {"S": payload["account_id"]}
            }
        )

        for request in existing_requests:
            request_body = json.loads(request.get("request_body", "{}"))
            if (request_body.get("custom_role_name") == payload["custom_role_name"] and
                request["status_message"]["S"] in ["CREATED", "IN_PROGRESS"]):
                logger.info(f"Request already exists for role {payload['custom_role_name']} in account {payload['account_id']}. "
                            f"Correlation ID: {request['trace_number']['S']}")
                return {
                    "message": "Change Request already exists and is in progress",
                    "request_id": request["trace_number"]["S"],
                    "account_id": payload['account_id'],
                }, HTTPStatus.CONFLICT.value

        # Use logger.get_correlation_id() as the unique identifier
        request_id = logger.get_correlation_id()

        # Prepare the item for DynamoDB
        item = {
            'trace_number': {'S': request_id},
            'account_id': {'S': payload['account_id']},
            'creation_type': {'S': 'custom_role'},
            'request_body': {'S': json.dumps(payload)},
            'status_message': {'S': "CREATED"},
            'created_at': {'S': datetime.now().isoformat()},
            'updated_at': {'S': datetime.now().isoformat()},
            'approval_group': {'S': approval_group}
        }
        
        # Record the change request in DynamoDB
        ddb_client.put_item(item)
        
        # Start a new thread for ServiceNow change request creation
        thread = threading.Thread(target=run_async_function, args=(create_snow_change_request, request_id, payload, approval_group))
        thread.start()
        
        return {
            "message": "Change Request is being created",
            "request_id": request_id,
            "account_id": payload['account_id'],
            "creation_type": "custom_role"
        }, HTTPStatus.ACCEPTED.value

    except Exception as e:
        logger.exception("Unexpected error occurred while initiating Change Request")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value

@tracer.capture_method
async def create_snow_change_request(request_id, payload, approval_group):
    try:
        ddb_client = RemovalStatusDDBClient()
        sn_secret = get_secret(SN_SECRET_NAME)
        if not sn_secret:
            raise ValueError("Failed to retrieve ServiceNow secret")

        sn_creds = json.loads(sn_secret)
        omni_client = OmniClient(
            sn_authorization=sn_creds.get('sn_authentication_bearer_token'),
            sn_client_id=sn_creds.get('sn_client_id'),
            sn_client_secret=sn_creds.get('sn_client_secret'),
            sn_change_requestid=sn_creds.get('sn_change_x_request_id')
        )

        data = get_cr_data(ddb_client, approval_group, payload)
        if not data:
            raise ValueError("Failed to prepare CR data")

        omni_client.set_user(staff_id=data.staff_id, verify_config=True)
        
        cr_number = await omni_client.create_cr(**data.dict())
        logger.info(f"Change Request created successfully: {cr_number}")

        # Update DynamoDB with the cr_number
        ddb_client.update_item(
            request_id,
            {
                'cr_number': {'S': cr_number},
                'cr_status': {'S': 'New'},
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_success'},
                    'description': {'S': f"<{cr_number}> CR created successfully."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

        # Assess the change request
        await omni_client.assess_cr(cr_number)
        logger.info(f"Change Request {cr_number} assessed successfully")

        # Update DynamoDB after assessment
        ddb_client.update_item(
            request_id,
            {
                'cr_status': {'S': 'Assess'},
                'status_message': {'M': {
                    'status_code': {'S': 'assess_cr_success'},
                    'description': {'S': "The CR has been moved to assess."}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

    except Exception as e:
        logger.exception(f"Error creating ServiceNow change request for request_id {request_id}")
        # Update DynamoDB with error status
        ddb_client.update_item(
            request_id,
            {
                'status_message': {'M': {
                    'status_code': {'S': 'create_cr_failed'},
                    'description': {'S': str(e)}
                }},
                'updated_at': {'S': datetime.now().isoformat()}
            }
        )

@tracer.capture_method
def sfn_trigger_handler(payload):
    resp = {"request_id": logger.get_correlation_id()}
    try:
        logger.append_keys(account_id=payload["account_id"], role_name=payload["custom_role_name"])
        SfnRequest.validate_payload_params(payload)
        account_entity = utils.get_account(payload["account_id"])
        utils.raise_for_ineligible_request(payload, account_entity)
        
        # Check account placement
        placement = account_entity.get("Placement", "").lower()
        if placement == "prod":
            return create_change_request(payload, account_entity)
        elif placement in ["development", "pre-prod"]:
            SfnRequest(
                util.decrypt_token(app.current_event.get_header_value(name="AuthorizationToken", case_sensitive=False)),
                payload,
                account_entity,
                logger.get_correlation_id()
            ).execute()
        else:
            raise ValueError(f"Invalid account placement: {placement}")

    except Exception as e:
        logger.exception("Failed to trigger step function")
        tracer.provider.current_subsegment().add_error_flag()
        tracer.capture_exception(e, traceback.extract_stack())
        resp["error"] = type(e).__name__
        if hasattr(e, "status_code"):
            return resp, e.status_code
        return resp, HTTPStatus.INTERNAL_SERVER_ERROR.value
    finally:
        logger.remove_keys(["account_id", "custom_role_name"])
    return resp, HTTPStatus.ACCEPTED.value

@app.post("/role/create")
def handle_custom_role():
    request_body = app.current_event.json_body
    account_id = request_body.get("account_id")
    custom_role_name = request_body.get("custom_role_name")
    payload = {
        "account_id": account_id,
        "build_type": "custom",
        "action": "apply",
        "custom_role_name": custom_role_name,
        "dry_run": True,
        "update_init_role": False
    }
    print(payload)
    return sfn_trigger_handler(payload)

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)
```
```
import os
import re
from datetime import datetime
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.logging import correlation_id
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.parser import BaseModel, ValidationError, parse, validator
from utils.aws import DDBClient, get_existed_request_item, send_message_to_sqs
from utils.params import VALID_SNOW_LOCATION, ALLOWED_HSBC_REGION

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

STATUS_TABLE = os.environ.get('STATUS_TABLE')
CR_CREATION_SQS_QUEUE_URL = os.environ.get('CR_CREATION_SQS_QUEUE_URL')

class RemovalRequester(BaseModel):
    request_id: str
    account_id: str
    custom_role_name: str
    schedule_date: str
    location: str
    staff_id: str
    approval_group: str

    @validator('request_id')
    def set_request_id(cls, v):
        return logger.get_correlation_id()

    @validator('schedule_date')
    def validate_schedule_date(cls, schedule_date: str) -> str:
        try:
            return str(datetime.strptime(schedule_date, "%Y-%m-%d %H:%M:%S"))
        except ValueError:
            raise ValueError(f"{schedule_date} is not a validate schedule_date with 'YYYY-MM-DD HH:MM:SS'")

    @validator('location')
    def validate_location(cls, location: str) -> str:
        if location not in VALID_SNOW_LOCATION:
            raise ValueError(f"{location} is not a valid location in {VALID_SNOW_LOCATION}")
        return location

    @validator('staff_id')
    def validate_staff_id(cls, staff_id: str) -> str:
        if not staff_id.isnumeric():
            raise ValueError(f"{staff_id} is not a numeric string")
        if len(staff_id) != 8:
            raise ValueError(f"{staff_id} does not have a valid length with 8")
        return staff_id

@tracer.capture_method
def create_change_request(payload, account_entity):
    approval_group = account_entity.get("ApprovalGroup", "NotAvailable")
    
    if approval_group == "NotAvailable":
        logger.error("ApprovalGroup is not available for Prod account")
        return {"error": "ApprovalGroup not available"}, HTTPStatus.BAD_REQUEST.value
    
    try:
        ddb_client = DDBClient(table_name=STATUS_TABLE)
        
        # Check if there's an existing request with the same custom_role_name and in_progress status
        existing_items = get_existed_request_item(
            ddb_client=ddb_client,
            account_id=payload["account_id"],
            custom_role_name=payload["custom_role_name"],
            request_status="in_progress"
        )
        
        if existing_items:
            logger.warning(f"There is already an in-progress request for account {payload['account_id']} and custom role {payload['custom_role_name']}")
            return {"error": "An in-progress request already exists for this custom role"}, HTTPStatus.BAD_REQUEST.value

        data = RemovalRequester(
            account_id=payload["account_id"],
            custom_role_name=payload["custom_role_name"],
            schedule_date=datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            location=account_entity.get("Location", ""),
            staff_id=payload.get("staff_id", ""),
            approval_group=approval_group
        )
        
        item = {
            "request_id": {"S": data.request_id},
            "account_id": {"S": data.account_id},
            "cr_number": {"S": ""},
            "cr_status": {"S": "New"},
            "implementation_date": {"S": ""},
            "request_status": {"S": "in_progress"},
            "request_body": {"M": {
                "custom_role_name": {"S": data.custom_role_name},
                "account_id": {"S": data.account_id},
                "account_entity": {"M": {
                    key: {"S": str(value)} for key, value in account_entity.items()
                }}
            }},
            "status_message": {"M": {
                "status_code": {"S": "create_request"},
                "description": {"S": "Create a request successfully."}
            }}
        }
        ddb_client.put_item(item=item)
        
        message_id = send_message_to_sqs(
            queue_url=CR_CREATION_SQS_QUEUE_URL,
            message={
                "request_id": data.request_id,
                "schedule_date": data.schedule_date,
                "location": data.location,
                "staff_id": data.staff_id,
                "approval_group": data.approval_group
            }
        )
        
        if message_id:
            logger.info(f"Sent a message to sidecar_removal_cr_creation_queue. Message ID: {message_id}")
            return {"message": f"Change Request created: {data.request_id}", "approval_group": approval_group}, HTTPStatus.ACCEPTED.value
    except Exception as e:
        logger.exception("Failed to create Change Request")
        return {"error": str(e)}, HTTPStatus.INTERNAL_SERVER_ERROR.value

@tracer.capture_method
def sfn_trigger_handler(payload):
    resp = {"request_id": logger.get_correlation_id()}
    try:
        logger.append_keys(account_id=payload["account_id"], role_name=payload["custom_role_name"])
        SfnRequest.validate_payload_params(payload)
        account_entity = utils.get_account(payload["account_id"])
        utils.raise_for_ineligible_request(payload, account_entity)
        
        # Check account placement
        placement = account_entity.get("Placement", "").lower()
        if placement == "prod":
            return create_change_request(payload, account_entity)
        elif placement in ["development", "pre-prod"]:
            SfnRequest(
                util.decrypt_token(app.current_event.get_header_value(name="AuthorizationToken", case_sensitive=False)),
                payload,
                account_entity,
                logger.get_correlation_id()
            ).execute()
        else:
            raise ValueError(f"Invalid account placement: {placement}")

    except Exception as e:
        logger.exception("Failed to trigger step function")
        tracer.provider.current_subsegment().add_error_flag()
        tracer.capture_exception(e, traceback.extract_stack())
        resp["error"] = type(e).__name__
        if hasattr(e, "status_code"):
            return resp, e.status_code
        return resp, HTTPStatus.INTERNAL_SERVER_ERROR.value
    finally:
        logger.remove_keys(["account_id", "custom_role_name"])
    return resp, HTTPStatus.ACCEPTED.value

@app.post("/role/create")
def handle_custom_role():
    request_body = app.current_event.json_body
    account_id = request_body.get("account_id")
    custom_role_name = request_body.get("custom_role_name")
    payload = {
        "account_id": account_id,
        "build_type": "custom",
        "action": "apply",
        "custom_role_name": custom_role_name,
        "dry_run": True,
        "update_init_role": False
    }
    print(payload)
    return sfn_trigger_handler(payload)

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)
```
