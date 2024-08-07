
```
{
  "Comment": "A state machine to orchestrate AD/DS group (not yet implemented) and IAM role creation",
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
          "Next": "UpdateSuccessfulRecord"
        }
      ],
      "Default": "GenerateBuildStatusErrorMessage"
    },
    "WaitForBuildStatus": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "GetBuildStatus"
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
    }
  }
}


```
