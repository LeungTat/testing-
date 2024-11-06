```
import pytest
from datetime import datetime, timezone, timedelta
from unittest.mock import Mock, patch, MagicMock
from botocore.exceptions import ClientError

# Add mock for Boto3Client
@pytest.fixture
def mock_boto3_client():
    with patch('utils.aws.Boto3Client') as mock:
        mock_instance = Mock()
        mock_instance.get_client.return_value = Mock()
        mock.return_value = mock_instance
        yield mock

# Update the DynamoDB client mocks to use Boto3Client
@pytest.fixture
def mock_ddb_client(mock_boto3_client):
    with patch('utils.aws.IamRolesCRStatusDDBClient') as mock:
        mock_instance = Mock()
        # Mock the query methods
        mock_instance.query_items_with_index.return_value = []
        mock_instance.batch_write_items.return_value = None
        mock_instance.update_item.return_value = None
        mock.return_value = mock_instance
        yield mock_instance

@pytest.fixture
def mock_execution_ddb_client(mock_boto3_client):
    with patch('utils.aws.IamRolesExecutionStatusDDBClient') as mock:
        mock_instance = Mock()
        mock_instance.query_items_with_account_and_request_id.return_value = None
        mock.return_value = mock_instance
        yield mock_instance

# Test is_implemented_change with proper AWS mocks
def test_is_implemented_change(mock_execution_ddb_client):
    # Setup
    change = {
        'account_id': {'S': '123456789012'},
        'request_id': {'S': 'req-001'},
        'implementation_date': {'S': '2024-03-20T10:00:00Z'}
    }
    
    # Mock successful implementation
    mock_execution_ddb_client.query_items_with_account_and_request_id.return_value = {
        'status': {'S': 'COMPLETE'},
        'adLds_status': {'S': 'COMPLETE'}
    }
    
    result = is_implemented_change(change)
    assert result == '2024-03-20T10:00:00Z'
    
    # Verify the correct AWS method was called
    mock_execution_ddb_client.query_items_with_account_and_request_id.assert_called_once_with(
        '123456789012',
        'req-001'
    )

# Test lambda_handler with proper AWS mocks
@patch('iam_role_orchestrator_cr_implement.get_secret')
@patch('iam_role_orchestrator_cr_implement.SfnRequest')
def test_lambda_handler_with_available_changes(
    mock_sfn,
    mock_get_secret,
    mock_ddb_client,
    mock_snow_api,
    mock_boto3_client
):
    # Setup DynamoDB response
    mock_ddb_client.query_items_with_index.return_value = [{
        'cr_number': {'S': 'CR1234'},
        'account_id': {'S': '123456789012'},
        'request_id': {'S': 'req-001'},
        'cr_status': {'S': 'Implement'},
        'account_entity': {'M': {'account': {'S': 'test'}}},
        'request_body': {'M': {'role': {'S': 'test_role'}}}
    }]

    # Setup Snow API response
    mock_snow_api.return_value.get_change.return_value = Mock(
        status='Implement',
        assigned_to='user1',
        end_date=(datetime.now(timezone.utc) + timedelta(days=1)).strftime('%Y-%m-%dT%H:%M:%S%z')
    )

    # Setup secret response
    mock_get_secret.return_value = json.dumps({
        'sn_authentication_bearer_token': 'token',
        'sn_client_id': 'id',
        'sn_client_secret': 'secret',
        'sn_change_x_request_id': 'rid',
        'sn_x_custom': 'custom'
    })

    # Execute
    lambda_handler({}, {})

    # Verify
    mock_ddb_client.query_items_with_index.assert_called_once_with(
        index_name="request_status_index",
        conditions={
            "request_status": {
                "ComparisonOperator": "EQ",
                "AttributeValueList": [{"S": "in_progress"}]
            }
        },
        QueryFilter={
            "cr_number": {
                "ComparisonOperator": "NE",
                "AttributeValueList": [{"S": ""}]
            }
        }
    )
    mock_sfn.assert_called()
    mock_ddb_client.update_item.assert_called()

# Test check_and_update_cr_info with proper AWS mocks
def test_check_and_update_cr_info_implement_status(
    mock_ddb_client,
    mock_snow_api,
    mock_execution_ddb_client
):
    # Setup
    cr_number = 'CR1234'
    changes_data = [{
        'cr_number': {'S': cr_number},
        'account_id': {'S': '123456789012'},
        'request_id': {'S': 'req-001'},
        'cr_status': {'S': 'Implement'},
        'implementation_date': {'S': '2024-03-20T10:00:00Z'}
    }]

    # Mock Snow API response
    mock_snow_api.return_value.get_change.return_value = Mock(
        status='Implement',
        assigned_to='user1',
        end_date=(datetime.now(timezone.utc) + timedelta(days=1)).strftime('%Y-%m-%dT%H:%M:%S%z')
    )

    # Mock execution status check
    mock_execution_ddb_client.query_items_with_account_and_request_id.return_value = {
        'status': {'S': 'IN_PROGRESS'},
        'adLds_status': {'S': 'IN_PROGRESS'}
    }

    # Execute
    result = check_and_update_cr_info(mock_snow_api.return_value, mock_ddb_client, cr_number, changes_data)

    # Verify
    assert result == changes_data
    mock_ddb_client.batch_write_items.assert_called_once()
    expected_attributes = {
        'implementation_date': {'S': mock.ANY},  # We use mock.ANY because the timestamp will be current
        'status_message': {
            'M': {
                'status_code': {'S': 'schedule_cr'},
                'description': {'S': 'Waiting for implementing.'}
            }
        }
    }
    mock_ddb_client.batch_write_items.assert_called_with(
        put_items=changes_data,
        replace_attributes=expected_attributes
    )
```
