```
import json
from datetime import datetime, timezone
from unittest.mock import MagicMock, patch

import pytest
from tpamless_cr_status_poller import (get_available_crs, is_in_implement_state,
                                      process, send_message, update_db_item,
                                      update_item_if_changed)


@pytest.fixture
def mock_snow_change():
    change = MagicMock()
    change.status = 4  # IMPLEMENT state
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

def test_send_message(mock_item):
    mock_sqs = MagicMock()
    
    send_message(mock_sqs, mock_item, 'cr_trigger')
    
    mock_sqs.send_message.assert_called_once()
    sent_payload = json.loads(mock_sqs.send_message.call_args[1]['MessageBody'])
    assert sent_payload['event_name'] == 'cr_trigger'
    assert 'ttl' not in sent_payload['event_payload']
    assert isinstance(sent_payload['event_payload']['release_id'], int)
    assert isinstance(sent_payload['event_payload']['pr_number'], int)

def test_update_db_item():
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

def test_get_available_crs():
    mock_table = MagicMock()
    mock_table.query.return_value = {
        'Items': [{'change_number': 'CHG0012345'}],
        'LastEvaluatedKey': None
    }
    
    results = list(get_available_crs(mock_table))
    assert len(results) == 1
    assert results[0] == [{'change_number': 'CHG0012345'}]

def test_get_available_crs_with_error():
    mock_table = MagicMock()
    mock_table.query.side_effect = Exception("DynamoDB error")
    
    results = list(get_available_crs(mock_table))
    assert len(results) == 1
    assert results[0] == []

def test_is_in_implement_state():
    future_date = (datetime.now().astimezone(timezone.utc)
                  .replace(microsecond=0).isoformat())
    
    # Test implement state with future date
    assert is_in_implement_state(4, future_date) is True
    
    # Test implement state with past date
    past_date = "2023-01-01T00:00:00+00:00"
    assert is_in_implement_state(4, past_date) is False
    
    # Test non-implement state
    assert is_in_implement_state(3, future_date) is False

def test_update_item_if_changed():
    item = {'field': 'old_value'}
    
    # Test when value changes
    assert update_item_if_changed(item, 'field', 'new_value') is True
    assert item['field'] == 'new_value'
    
    # Test when value doesn't change
    assert update_item_if_changed(item, 'field', 'new_value') is False

@patch('tpamless_cr_status_poller.send_message')
def test_process(mock_send_message, mock_item, mock_snow_change):
    mock_table = MagicMock()
    mock_sqs = MagicMock()
    mock_snow_api = MagicMock()
    mock_snow_api.get_change.return_value = mock_snow_change
    
    # Test normal processing
    process(mock_item, mock_table, mock_sqs, mock_snow_api)
    
    mock_snow_api.get_change.assert_called_once_with(mock_item['change_number'])
    assert mock_send_message.called

@patch('tpamless_cr_status_poller.send_message')
def test_process_with_error(mock_send_message, mock_item):
    mock_table = MagicMock()
    mock_sqs = MagicMock()
    mock_snow_api = MagicMock()
    mock_snow_api.get_change.side_effect = Exception("API error")
    
    # Should not raise exception
    process(mock_item, mock_table, mock_sqs, mock_snow_api)
    
    mock_snow_api.get_change.assert_called_once_with(mock_item['change_number'])
    assert not mock_send_message.called
```
