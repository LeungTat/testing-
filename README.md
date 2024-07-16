
```
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools.logging import Logger, Tracer
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.event_handler import APIGatewayRestResolver

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

# Define the allowed Lambda ARN
allowed_lambda_arn = os.environ["ALLOWED_LAMBDA_ARN"]

def is_request_from_allowed_lambda(event):
    source_arn = event.get("requestContext", {}).get("identity", {}).get("sourceArn", "")
    return source_arn == allowed_lambda_arn

@tracer.capture_method
def sfn_trigger_handler(payload, event):
    if not is_request_from_allowed_lambda(event):
        return {"message": "Forbidden"}, HTTPStatus.FORBIDDEN

    resp = {"request_id": logger.get_correlation_id()}
    try:
        logger.append_keys(account_id=payload['account_id'], role_name=payload['custom_role_name'])
        SfnRequest.validate_payload_params(payload)
        account_entity = utils.get_account_entity(payload['account_id'])
        utils.raise_for_ineligible_request(payload, account_entity)
        SfnRequest(
            utils.encrypt_token(app.current_event.get_header_value(name='AuthorizationToken', case_sensitive=False)),
            payload,
            account_entity,
            logger.get_correlation_id()
        ).execute()
    except Exception as e:
        logger.exception('Failed to trigger step function')
        tracer.provider.current_subsegment().add_error_flag()
        tracer.provider.current_subsegment().add_exception(e, traceback.extract_stack())
        resp['error'] = type(e).__name__
        if hasattr(e, 'status_code'):
            return resp, e.status_code
        return resp, HTTPStatus.INTERNAL_SERVER_ERROR.value
    finally:
        logger.remove_keys(['account_id', 'custom_role_name'])
    return resp, HTTPStatus.ACCEPTED.value

@app.post('/role/create')
def handle_custom_role():
    request_body = app.current_event.json_body
    account_id = request_body.get('account_id')
    custom_role_name = request_body.get('custom_role_name')

    payload = {
        'account_id': account_id,
        'build_type': 'custom',
        'action': 'apply',
        'custom_role_name': custom_role_name,
        'dry_run': False,
        'update_init_role': False
    }
    return sfn_trigger_handler(payload, app.current_event)

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)



```
