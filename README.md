```
import pytest
from unittest.mock import patch, MagicMock
from http import HTTPStatus
from iam_role_orchestrator_trigger import app, handle_custom_role, handle_custom_role_road, get_request_status, get_request_status_by_account_id

# Mock environment variables
@pytest.fixture(autouse=True)
def mock_env_vars(monkeypatch):
    monkeypatch.setenv("CR_CREATION_SQS_QUEUE_URL", "test-queue-url")
    monkeypatch.setenv("AWS_ROAD_ONBOARDING_REQUEST_SYNC_ROLE", "arn:aws:iam::123456789012:role/test-role")

# Test cases for /role/create endpoint
def test_handle_custom_role_success():
    with patch('iam_role_orchestrator_trigger.sfn_trigger_handler') as mock_handler:
        mock_handler.return_value = ({"message": "Success"}, HTTPStatus.ACCEPTED.value)
        
        # Mock the current event
        app.current_event = MagicMock()
        app.current_event.raw_event = {"requestContext": {"authorizer": {"is_road": False}}}
        app.current_event.json_body = {
            "account_id": "123456789012",
            "custom_role_name": "test-role"
        }
        
        response, status_code = handle_custom_role()
        
        assert status_code == HTTPStatus.ACCEPTED.value
        assert response["message"] == "Success"
        mock_handler.assert_called_once()

def test_handle_custom_role_missing_parameters():
    app.current_event = MagicMock()
    app.current_event.raw_event = {"requestContext": {"authorizer": {"is_road": False}}}
    app.current_event.json_body = {}  # Missing required parameters
    
    response, status_code = handle_custom_role()
    
    assert status_code == HTTPStatus.BAD_REQUEST.value
    assert "error" in response

# Test cases for /road/role/create endpoint
def test_handle_custom_role_road_success():
    with patch('iam_role_orchestrator_trigger.sfn_trigger_handler') as mock_handler:
        mock_handler.return_value = ({"message": "Success"}, HTTPStatus.ACCEPTED.value)
        
        # Mock the current event with valid ROAD role
        app.current_event = MagicMock()
        app.current_event.raw_event = {
            "requestContext": {
                "identity": {
                    "userArn": "arn:aws:iam::123456789012:role/test-role"
                }
            }
        }
        app.current_event.json_body = {
            "account_id": "123456789012",
            "custom_role_name": "test-role"
        }
        
        response, status_code = handle_custom_role_road()
        
        assert status_code == HTTPStatus.ACCEPTED.value
        assert response["message"] == "Success"
        mock_handler.assert_called_once()

def test_handle_custom_role_road_unauthorized():
    app.current_event = MagicMock()
    app.current_event.raw_event = {
        "requestContext": {
            "identity": {
                "userArn": "arn:aws:iam::123456789012:role/invalid-role"
            }
        }
    }
    app.current_event.json_body = {
        "account_id": "123456789012",
        "custom_role_name": "test-role"
    }
    
    response, status_code = handle_custom_role_road()
    
    assert status_code == HTTPStatus.UNAUTHORIZED.value
    assert response["error"] == "Unauthorized"

# Test cases for /status/<request_id> endpoint
def test_get_request_status_success():
    with patch('iam_role_orchestrator_trigger.IamRolesExecutionStatusDDBClient') as mock_ddb:
        mock_ddb.return_value.query_items_with_index.return_value = [{
            "custom_role_name": {"S": "test-role"},
            "account_id": {"S": "123456789012"},
            "status": {"S": "COMPLETED"},
            "adIds_status": {"S": "SUCCESS"},
            "start_at": {"S": "2024-01-01T00:00:00Z"}
        }]
        
        response, status_code = get_request_status("test-request-id")
        
        assert status_code == HTTPStatus.OK.value
        assert response["custom_role_name"] == "test-role"
        assert response["account_id"] == "123456789012"
        assert response["iam_role_status"] == "COMPLETED"

def test_get_request_status_not_found():
    with patch('iam_role_orchestrator_trigger.IamRolesExecutionStatusDDBClient') as mock_ddb:
        mock_ddb.return_value.query_items_with_index.return_value = []
        
        response, status_code = get_request_status("non-existent-id")
        
        assert status_code == HTTPStatus.NOT_FOUND.value
        assert response["message"] == "Request not found"

# Test cases for /status/account/<account_id> endpoint
def test_get_request_status_by_account_id_success():
    with patch('iam_role_orchestrator_trigger.IamRolesExecutionStatusDDBClient') as mock_ddb:
        mock_ddb.return_value.query_items_with_account_id.return_value = [
            {
                "custom_role_name": {"S": "test-role-1"},
                "account_id": {"S": "123456789012"},
                "status": {"S": "COMPLETED"},
                "adIds_status": {"S": "SUCCESS"},
                "start_at": {"S": "2024-01-01T00:00:00Z"}
            },
            {
                "custom_role_name": {"S": "test-role-2"},
                "account_id": {"S": "123456789012"},
                "status": {"S": "IN_PROGRESS"},
                "adIds_status": {"S": "PENDING"},
                "start_at": {"S": "2024-01-02T00:00:00Z"}
            }
        ]
        
        response, status_code = get_request_status_by_account_id("123456789012")
        
        assert status_code == HTTPStatus.OK.value
        assert len(response["items"]) == 2
        assert response["items"][0]["custom_role_name"] == "test-role-1"
        assert response["items"][1]["custom_role_name"] == "test-role-2"

def test_get_request_status_by_account_id_not_found():
    with patch('iam_role_orchestrator_trigger.IamRolesExecutionStatusDDBClient') as mock_ddb:
        mock_ddb.return_value.query_items_with_account_id.return_value = []
        
        response, status_code = get_request_status_by_account_id("non-existent-account")
        
        assert status_code == HTTPStatus.NOT_FOUND.value
        assert response["message"] == "Request not found"
```
