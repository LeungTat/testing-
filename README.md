
```
import utils
import traceback
from http import HTTPStatus
from sfn_request import SfnRequest
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.logging import correlation_id
from aws_lambda_powertools.event_handler import APIGatewayRestResolver

logger = Logger(location="%(module)s.%(funcName)s:%(lineno)d")
tracer = Tracer()
app = APIGatewayRestResolver(strip_prefixes=["/adfs-role-orch"])

@tracer.capture_method
def create_change_request(payload, account_entity):
    approval_group = account_entity.get("ApprovalGroup", "NotAvailable")
    
    if approval_group == "NotAvailable":
        logger.error("ApprovalGroup is not available for Prod account")
        return {"error": "ApprovalGroup not available"}, HTTPStatus.BAD_REQUEST.value
    
    # Add your CR creation logic here
    logger.info(f"Creating Change Request for Prod account. Approval Group: {approval_group}")
    # Return appropriate response
    return {"message": "Change Request created", "approval_group": approval_group}, HTTPStatus.ACCEPTED.value

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
