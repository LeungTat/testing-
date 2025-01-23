```
import json
from datetime import datetime, timedelta, timezone
from unittest.mock import MagicMock, patch

import pytest
from boto3.dynamodb.conditions import Key
from tpamless_cr_status_poller import (get_available_crs, is_in_implement_state,
                                      lambda_handler, process, send_message,
                                      update_db_item, update_item_if_changed)


@pytest.fixture(autouse=True)
def mock_env_vars():
    with patch.dict('os.environ', {
        'SN_API_CREDS_SECRET_NAME': 'test-secret',
        'CR_INFORMATION_TABLE': 'test-table',
        'GITHUB_APP_QUEUE_URL': 'https://sqs.test.url'
    }):
        yield

@pytest.fixture
def mock_snow_change():
    change = MagicMock()
    change.status = -1  # IMPLEMENT state
    change.start_date = "2024-03-20T10:00:00+00:00"
    change.end_date = "2024-12-31T23:59:59+00:00"
    change.last_submission_date = "2024-03-19T15:00:00+00:00"
    change.category = "Normal"
    change.on_hold = "false"
    return change

@pytest.fixture
def mock_item():
    return {
        'change_number': 'CHG0012345',
        'owner': 'test-owner',
        'repo_name': 'test-repo',
        'release_id': '123',
        'pr_number': '456',
        'ttl': '1234567890',
        'stage': 'in_progress',
        'start_date': '2024-03-20T10:00:00+00:00',
        'end_date': '2024-12-31T23:59:59+00:00',
        'implement_date': '',
        'actual_last_submission_date': '2024-03-19T15:00:00+00:00'
    }

@pytest.fixture
def mock_context():
    context = MagicMock()
    context.aws_request_id = 'test-request-id'
    return context

class TestSendMessage:
    def test_send_message_success(self, mock_item):
        mock_sqs = MagicMock()
        send_message(mock_sqs, mock_item, 'cr_trigger')
        
        mock_sqs.send_message.assert_called_once()
        sent_payload = json.loads(mock_sqs.send_message.call_args[1]['MessageBody'])
        assert sent_payload['event_name'] == 'cr_trigger'
        assert 'ttl' not in sent_payload['event_payload']
        assert isinstance(sent_payload['event_payload']['release_id'], int)
        assert isinstance(sent_payload['event_payload']['pr_number'], int)
        assert sent_payload['event_payload']['owner'] == 'test-owner'
        assert sent_payload['event_payload']['repo'] == 'test-repo'

    def test_send_message_different_event(self, mock_item):
        mock_sqs = MagicMock()
        send_message(mock_sqs, mock_item, 'cr_check')
        
        sent_payload = json.loads(mock_sqs.send_message.call_args[1]['MessageBody'])
        assert sent_payload['event_name'] == 'cr_check'

class TestUpdateDbItem:
    def test_update_db_item_success(self):
        mock_table = MagicMock()
        
        update_db_item(
            mock_table,
            'CHG0012345',
            'implement',
            '2024-03-20T10:00:00+00:00',
            '2024-12-31T23:59:59+00:00',
            '2024-03-20T10:00:00+00:00',
            '2024-03-19T15:00:00+00:00'
        )
        
        mock_table.update_item.assert_called_once()
        update_args = mock_table.update_item.call_args[1]
        assert update_args['Key']['change_number'] == 'CHG0012345'
        assert ':s' in update_args['ExpressionAttributeValues']
        assert update_args['ExpressionAttributeValues'][':s'] == 'implement'

    def test_update_db_item_with_empty_values(self):
        mock_table = MagicMock()
        
        update_db_item(
            mock_table,
            'CHG0012345',
            '',
            '',
            '',
            '',
            ''
        )
        
        mock_table.update_item.assert_called_once()

class TestGetAvailableCRs:
    def test_get_available_crs_single_page(self):
        mock_table = MagicMock()
        mock_table.query.return_value = {
            'Items': [{'change_number': 'CHG0012345'}],
            'LastEvaluatedKey': None
        }
        
        results = list(get_available_crs(mock_table))
        assert len(results) == 1
        assert results[0] == [{'change_number': 'CHG0012345'}]
        
        mock_table.query.assert_called_once()
        query_args = mock_table.query.call_args[1]
        assert query_args['IndexName'] == 'stage-index'
        assert isinstance(query_args['KeyConditionExpression'], Key)

    def test_get_available_crs_multiple_pages(self):
        mock_table = MagicMock()
        mock_table.query.side_effect = [
            {
                'Items': [{'change_number': 'CHG0012345'}],
                'LastEvaluatedKey': {'key': 'value'}
            },
            {
                'Items': [{'change_number': 'CHG0012346'}],
                'LastEvaluatedKey': None
            }
        ]
        
        results = list(get_available_crs(mock_table))
        assert len(results) == 2
        assert mock_table.query.call_count == 2

class TestIsInImplementState:
    def test_is_in_implement_state_future(self):
        future_date = (datetime.now().astimezone(timezone.utc) + 
                      timedelta(days=1)).replace(microsecond=0).isoformat()
        assert is_in_implement_state(-1, future_date) is True

    def test_is_in_implement_state_past(self):
        past_date = (datetime.now().astimezone(timezone.utc) - 
                    timedelta(days=1)).replace(microsecond=0).isoformat()
        assert is_in_implement_state(-1, past_date) is False

    def test_is_in_implement_state_wrong_status(self):
        future_date = (datetime.now().astimezone(timezone.utc) + 
                      timedelta(days=1)).replace(microsecond=0).isoformat()
        assert is_in_implement_state(3, future_date) is False

class TestUpdateItemIfChanged:
    def test_update_item_if_changed_with_change(self):
        item = {'field': 'old_value'}
        assert update_item_if_changed(item, 'field', 'new_value') is True
        assert item['field'] == 'new_value'

    def test_update_item_if_changed_no_change(self):
        item = {'field': 'same_value'}
        assert update_item_if_changed(item, 'field', 'same_value') is False
        assert item['field'] == 'same_value'

class TestProcess:
    @patch('tpamless_cr_status_poller.send_message')
    def test_process_normal_implement(self, mock_send_message, mock_item, mock_snow_change):
        mock_table = MagicMock()
        mock_sqs = MagicMock()
        mock_snow_api = MagicMock()
        mock_snow_api.get_change.return_value = mock_snow_change
        
        process(mock_item, mock_table, mock_sqs, mock_snow_api)
        
        mock_snow_api.get_change.assert_called_once_with(mock_item['change_number'])
        assert mock_send_message.called

    @patch('tpamless_cr_status_poller.send_message')
    def test_process_on_hold(self, mock_send_message, mock_item, mock_snow_change):
        mock_snow_change.on_hold = "true"
        mock_table = MagicMock()
        mock_sqs = MagicMock()
        mock_snow_api = MagicMock()
        mock_snow_api.get_change.return_value = mock_snow_change
        
        process(mock_item, mock_table, mock_sqs, mock_snow_api)
        
        assert not mock_send_message.called

    @patch('tpamless_cr_status_poller.send_message')
    def test_process_past_end_date(self, mock_send_message, mock_item, mock_snow_change):
        mock_snow_change.end_date = "2023-01-01T00:00:00+00:00"
        mock_table = MagicMock()
        mock_sqs = MagicMock()
        mock_snow_api = MagicMock()
        mock_snow_api.get_change.return_value = mock_snow_change
        
        process(mock_item, mock_table, mock_sqs, mock_snow_api)
        
        assert mock_item['stage'] == 'close'

    @patch('tpamless_cr_status_poller.send_message')
    def test_process_with_error(self, mock_send_message, mock_item):
        mock_table = MagicMock()
        mock_sqs = MagicMock()
        mock_snow_api = MagicMock()
        mock_snow_api.get_change.side_effect = Exception("API error")
        
        process(mock_item, mock_table, mock_sqs, mock_snow_api)
        
        mock_snow_api.get_change.assert_called_once_with(mock_item['change_number'])
        assert not mock_send_message.called

    @patch('tpamless_cr_status_poller.send_message')
    def test_process_status_greater_than_implement(self, mock_send_message, mock_item, mock_snow_change):
        mock_snow_change.status = 1  # Status greater than IMPLEMENT (-1)
        mock_table = MagicMock()
        mock_sqs = MagicMock()
        mock_snow_api = MagicMock()
        mock_snow_api.get_change.return_value = mock_snow_change
        
        process(mock_item, mock_table, mock_sqs, mock_snow_api)
        
        assert mock_item['stage'] == 'close'

    @patch('tpamless_cr_status_poller.send_message')
    def test_process_status_smaller_than_implement(self, mock_send_message, mock_item, mock_snow_change):
        mock_snow_change.status = -2  # Status smaller than IMPLEMENT (-1)
        mock_table = MagicMock()
        mock_sqs = MagicMock()
        mock_snow_api = MagicMock()
        mock_snow_api.get_change.return_value = mock_snow_change
        
        process(mock_item, mock_table, mock_sqs, mock_snow_api)
        
        # Should call send_message with 'cr_check' event
        mock_send_message.assert_called_once()
        mock_send_message.assert_called_with(mock_sqs, mock_item, 'cr_check')
        assert mock_item['stage'] != 'close'  # Stage should not be changed to close

    @patch('tpamless_cr_status_poller.send_message')
    def test_process_status_is_in_implement(self, mock_send_message, mock_item, mock_snow_change):
        mock_snow_change.status = -1  # Status is IMPLEMENT
        mock_table = MagicMock()
        mock_sqs = MagicMock()
        mock_snow_api = MagicMock()
        mock_snow_api.get_change.return_value = mock_snow_change
        
        process(mock_item, mock_table, mock_sqs, mock_snow_api)
        
        # Should call send_message with 'cr_trigger' event
        mock_send_message.assert_called_once()
        mock_send_message.assert_called_with(mock_sqs, mock_item, 'cr_trigger')
        # Verify it's not called with 'cr_check'
        assert not any('cr_check' in call[0][2] for call in mock_send_message.call_args_list)
        assert mock_item['stage'] == 'in_progress'  # Stage should be in_progress

class TestLambdaHandler:
    @patch('tpamless_cr_status_poller.BotoClient')
    @patch('tpamless_cr_status_poller.SnowApi')
    def test_lambda_handler_success(self, mock_snow_api_class, mock_boto_client, mock_context):
        # Setup mocks
        mock_boto = MagicMock()
        mock_boto_client.return_value = mock_boto
        mock_boto.get_secret.return_value = json.dumps({"token": "test"})
        
        mock_table = MagicMock()
        mock_table.query.return_value = {
            'Items': [{'change_number': 'CHG0012345'}],
            'LastEvaluatedKey': None
        }
        
        mock_dynamodb = MagicMock()
        mock_dynamodb.Table.return_value = mock_table
        mock_boto.session.resource.return_value = mock_dynamodb
        
        # Execute
        lambda_handler({}, mock_context)
        
        # Verify
        mock_boto.get_secret.assert_called_once_with(secret_name='test-secret')
        mock_snow_api_class.assert_called_once()

    @patch('tpamless_cr_status_poller.BotoClient')
    def test_lambda_handler_secret_failure(self, mock_boto_client, mock_context):
        mock_boto = MagicMock()
        mock_boto_client.return_value = mock_boto
        mock_boto.get_secret.return_value = None
        
        result = lambda_handler({}, mock_context)
        
        assert result is None
        mock_boto.get_secret.assert_called_once_with(secret_name='test-secret')

``` 
