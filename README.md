```
import pytest
from datetime import datetime, timezone, timedelta
from unittest.mock import Mock, patch, MagicMock

# Mock TypeSerializer and TypeDeserializer
@pytest.fixture
def mock_type_serializer():
    with patch('botocore.dynamodb.types.TypeSerializer') as mock:
        serializer_instance = Mock()
        
        def mock_serialize(value):
            return {'S': str(value)} if isinstance(value, (str, int)) else {'M': value}
            
        serializer_instance.serialize = mock_serialize
        mock.return_value = serializer_instance
        yield serializer_instance

@pytest.fixture
def mock_type_deserializer():
    with patch('botocore.dynamodb.types.TypeDeserializer') as mock:
        deserializer_instance = Mock()
        
        def mock_deserialize(value):
            if 'S' in value:
                return value['S']
            elif 'M' in value:
                return value['M']
            return value
            
        deserializer_instance.deserialize = mock_deserialize
        mock.return_value = deserializer_instance
        yield deserializer_instance

# Update the sample_ddb_items fixture to use mock_type_serializer
@pytest.fixture
def sample_ddb_items(mock_type_serializer):
    return [
        {
            'cr_number': mock_type_serializer.serialize('CR1234'),
            'account_id': mock_type_serializer.serialize('123456789012'),
            'request_id': mock_type_serializer.serialize('req-001'),
            'cr_status': mock_type_serializer.serialize('New'),
            'request_status': mock_type_serializer.serialize('in_progress'),
            'account_entity': mock_type_serializer.serialize({'account': 'test'}),
            'request_body': mock_type_serializer.serialize({'role': 'test_role'})
        },
        {
            'cr_number': mock_type_serializer.serialize('CR1234'),
            'account_id': mock_type_serializer.serialize('123456789013'),
            'request_id': mock_type_serializer.serialize('req-002'),
            'cr_status': mock_type_serializer.serialize('New'),
            'request_status': mock_type_serializer.serialize('in_progress')
        }
    ]

# Update the prepare_in_progress_change_data test
@patch('botocore.dynamodb.types.TypeDeserializer')
def test_prepare_in_progress_change_data(mock_deserializer_class, sample_ddb_items, mock_type_deserializer):
    mock_deserializer_class.return_value = mock_type_deserializer
    
    result = prepare_in_progress_change_data(sample_ddb_items)
    assert 'CR1234' in result
    assert len(result['CR1234']) == 2
    assert result['CR1234'][0] == sample_ddb_items[0]

# Update the lambda handler test
@patch('botocore.dynamodb.types.TypeDeserializer')
@patch('iam_role_orchestrator_cr_implement.get_secret')
@patch('iam_role_orchestrator_cr_implement.SfnRequest')
def test_lambda_handler_with_available_changes(
    mock_sfn, 
    mock_get_secret, 
    mock_deserializer_class,
    sample_ddb_items, 
    mock_snow_api, 
    mock_ddb_client,
    mock_type_deserializer
):
    # Setup
    mock_deserializer_class.return_value = mock_type_deserializer
    mock_ddb_client.return_value.query_items_with_index.return_value = sample_ddb_items
    mock_get_secret.return_value = '{"sn_authentication_bearer_token": "token", "sn_client_id": "id", "sn_client_secret": "secret", "sn_change_x_request_id": "rid", "sn_x_custom": "custom"}'
    
    mock_cr = Mock(
        status='Implement',
        assigned_to='user1',
        end_date=(datetime.now(timezone.utc) + timedelta(days=1)).strftime('%Y-%m-%dT%H:%M:%S%z')
    )
    mock_snow_api.return_value.get_change.return_value = mock_cr
    
    with patch('iam_role_orchestrator_cr_implement.is_implemented_change', return_value=False):
        lambda_handler({}, {})
        
        # Verify SfnRequest was called
        mock_sfn.assert_called()

# Update the check_and_update_cr_info tests
class TestCheckAndUpdateCRInfo:
    @pytest.fixture
    def base_change(self, mock_type_serializer):
        return {
            'cr_number': mock_type_serializer.serialize('CR1234'),
            'account_id': mock_type_serializer.serialize('123456789012'),
            'request_id': mock_type_serializer.serialize('req-001'),
            'cr_status': mock_type_serializer.serialize('New')
        }

    @patch('botocore.dynamodb.types.TypeDeserializer')
    def test_new_cr(self, mock_deserializer_class, mock_snow_api, mock_ddb_client, base_change, future_date, mock_type_deserializer):
        mock_deserializer_class.return_value = mock_type_deserializer
        mock_cr = Mock(status='New', assigned_to='user1', end_date=future_date)
        mock_snow_api.get_change.return_value = mock_cr
        
        result = check_and_update_cr_info(mock_snow_api, mock_ddb_client, 'CR1234', [base_change])
        
        assert result == []
        mock_snow_api.assess_cr.assert_called_once_with('CR1234', 'user1')

    # Update other test methods similarly...
```
