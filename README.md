```
# Combining IAM Authentication with Custom Authorizer Logic
You're right - in API Gateway, you can typically only select one authorization type per method (either AWS_IAM or a Custom Authorizer, not both). However, there's an elegant solution to achieve both IAM-level security AND custom authorization logic:
Solution: Create a Custom Authorizer That Validates IAM Credentials
You can implement a custom authorizer that validates both:
AWS SigV4 signatures (IAM credentials)
Your own custom authorization rules (like checking for is_road=true)
Here's how to implement this hybrid approach:
Step 1: Create a Custom Authorizer Lambda That Validates SigV4


import json
import boto3
import re
import base64
import os
from aws_lambda_powertools import Logger
from botocore.auth import SigV4Auth
from botocore.awsrequest import AWSRequest
from botocore.credentials import Credentials

logger = Logger()

@logger.inject_lambda_context
def handler(event, context):
    logger.info("Processing authorizer request with IAM validation")
    
    try:
        # Extract the Authorization header
        auth_header = event.get('authorizationToken', '')
        
        # First: Validate IAM credentials (SigV4 signature)
        is_valid_iam = validate_sigv4_signature(event)
        if not is_valid_iam:
            logger.warning("Invalid IAM signature")
            return generate_deny_policy('user', event.get('methodArn', ''))
        
        # If IAM validation passed, now do custom authorization
        # For example, extract custom claims from headers or query parameters
        is_road = extract_is_road_parameter(event)
        
        # Generate policy with the custom context
        principal_id = get_caller_identity()  # Get the IAM identity that made the call
        
        # Return policy with custom context
        return generate_allow_policy(principal_id, event.get('methodArn', ''), {
            'is_road': str(is_road).lower(),
            'validatedBySigV4': 'true'
        })
        
    except Exception as e:
        logger.exception("Error in authorizer")
        return generate_deny_policy('user', event.get('methodArn', ''))

def validate_sigv4_signature(event):
    """
    Validate the SigV4 signature in the request
    """
    try:
        # Get necessary components from the request
        method = 'GET'  # Default method, adapt based on your API
        
        # Extract authorization header
        auth_header = event.get('authorizationToken', '')
        if not auth_header.startswith('AWS4-HMAC-SHA256'):
            # Try to get from headers section
            if 'headers' in event and 'Authorization' in event['headers']:
                auth_header = event['headers']['Authorization']
            
        # If still no auth header found, check multiValueHeaders
        if not auth_header.startswith('AWS4-HMAC-SHA256'):
            if 'multiValueHeaders' in event and 'Authorization' in event['multiValueHeaders']:
                auth_header = event['multiValueHeaders']['Authorization'][0]
        
        if not auth_header.startswith('AWS4-HMAC-SHA256'):
            logger.warning("No valid SigV4 Authorization header found")
            return False
        
        # Parse the SigV4 header to extract access key
        # Example: 'AWS4-HMAC-SHA256 Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request, ...'
        credential_match = re.search(r'Credential=([^/]+)/', auth_header)
        if not credential_match:
            logger.warning("Could not extract credential from Authorization header")
            return False
        
        access_key = credential_match.group(1)
        
        # Use AWS to validate the credentials
        # Note: This is a simplified approach - full SigV4 validation is complex
        sts = boto3.client('sts')
        try:
            # If credentials are invalid, this will throw an exception
            response = sts.get_caller_identity(
                # You can add additional validation parameters here
            )
            logger.info(f"STS validated caller identity: {response['Arn']}")
            return True
        except Exception as e:
            logger.warning(f"STS validation failed: {str(e)}")
            return False
            
    except Exception as e:
        logger.exception("Error validating SigV4 signature")
        return False

def extract_is_road_parameter(event):
    """
    Extract is_road parameter from query string, headers, or other sources
    """
    # Try to get from queryStringParameters
    if 'queryStringParameters' in event and event['queryStringParameters']:
        if 'is_road' in event['queryStringParameters']:
            return event['queryStringParameters']['is_road'].lower() == 'true'
    
    # Try to get from headers
    if 'headers' in event and event['headers']:
        if 'x-is-road' in event['headers']:
            return event['headers']['x-is-road'].lower() == 'true'
    
    # Default value
    return False

def get_caller_identity():
    """
    Get the caller's identity from STS
    """
    try:
        sts = boto3.client('sts')
        response = sts.get_caller_identity()
        return response['Arn']
    except Exception:
        return "unknown-user"

def generate_allow_policy(principal_id, resource, context=None):
    """Generate IAM policy document for Allow with custom context"""
    auth_response = {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Action': 'execute-api:Invoke',
                    'Effect': 'Allow',
                    'Resource': resource
                }
            ]
        }
    }
    
    # Add context if provided
    if context:
        auth_response['context'] = context
    
    return auth_response

def generate_deny_policy(principal_id, resource):
    """Generate IAM policy document for Deny"""
    return {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Action': 'execute-api:Invoke',
                    'Effect': 'Deny',
                    'Resource': resource
                }
            ]
        }
    }


Step 2: Configure API Gateway with the Custom Authorizer
Create a new authorizer in API Gateway console
Select "Lambda" authorizer type
Select the authorizer Lambda function you created
Set Identity Source to "Authorization" header
Set "Authorization Caching" to 0 (or a short TTL)
Attach this authorizer to your API methods


Step 3: Create a Lambda for Calling the API with SigV4



import boto3
import json
import os
import requests
from aws_requests_auth.aws_auth import AWSRequestsAuth
from aws_lambda_powertools import Logger, Tracer

logger = Logger()
tracer = Tracer()

@logger.inject_lambda_context
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    try:
        # API Gateway URL
        api_endpoint = os.environ['TARGET_API_ENDPOINT']
        region = os.environ['AWS_REGION']
        
        # Extract the API Gateway domain and path
        api_parts = api_endpoint.replace('https://', '').split('/', 1)
        api_host = api_parts[0]
        api_path = '/' + api_parts[1] if len(api_parts) > 1 else '/'
        
        # Get AWS credentials from the Lambda environment
        session = boto3.Session()
        credentials = session.get_credentials()
        
        # Create AWS SigV4 auth
        auth = AWSRequestsAuth(
            aws_access_key=credentials.access_key,
            aws_secret_access_key=credentials.secret_key,
            aws_token=credentials.token,
            aws_host=api_host,
            aws_region=region,
            aws_service='execute-api'
        )
        
        # Add the is_road parameter to the query string
        url = api_endpoint
        if '?' in url:
            url += '&is_road=true'
        else:
            url += '?is_road=true'
        
        # Alternatively, add as a header
        headers = {
            'Content-Type': 'application/json',
            'X-Is-Road': 'true'  # Custom header for is_road
        }
        
        logger.info(f"Calling API Gateway with SigV4 and is_road: {url}")
        
        # Make the request with SigV4 signing
        response = requests.get(
            url,
            auth=auth,
            headers=headers
        )
        
        # Process response
        if response.status_code >= 200 and response.status_code < 300:
            logger.info(f"API call successful: {response.status_code}")
            response_data = response.json()
            return {
                'statusCode': 200,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({
                    'message': 'API call successful',
                    'apiResponse': response_data
                })
            }
        else:
            logger.error(f"API call failed: {response.status_code}")
            logger.error(f"Response: {response.text}")
            return {
                'statusCode': response.status_code,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({
                    'message': 'API call failed',
                    'error': response.text
                })
            }
            
    except Exception as e:
        logger.exception("Error calling API Gateway")
        return {
            'statusCode': 500,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({
                'message': 'Internal error',
                'error': str(e)
            })
        }



Step 4: Implement the Target Lambda to Receive Both IAM and Custom Context




import json
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()
tracer = Tracer()
app = APIGatewayRestResolver()

@app.get("/resource")
@tracer.capture_method
def get_resource():
    # Get the current event
    event = app.current_event
    
    # Access the custom context from the authorizer
    try:
        # Log the full event for debugging
        logger.debug("Full event", extra={"event": event.raw_event})
        
        # Get authorizer context
        authorizer_context = {}
        if hasattr(event, 'request_context') and hasattr(event.request_context, 'authorizer'):
            authorizer_context = event.request_context.authorizer
        elif 'requestContext' in event.raw_event and 'authorizer' in event.raw_event['requestContext']:
            authorizer_context = event.raw_event['requestContext']['authorizer']
        
        # Get values with defaults
        is_road = authorizer_context.get('is_road', 'false')
        validated_by_sigv4 = authorizer_context.get('validatedBySigV4', 'false')
        
        logger.info("Authorizer context values", extra={
            "is_road": is_road,
            "validatedBySigV4": validated_by_sigv4
        })
        
        # Use the context values
        if is_road == 'true' and validated_by_sigv4 == 'true':
            response_message = 'This is a road user with IAM validation'
            # Add your road-specific logic here
        elif is_road == 'true':
            response_message = 'This is a road user without IAM validation'
        elif validated_by_sigv4 == 'true':
            response_message = 'This is an IAM validated user without road access'
        else:
            response_message = 'This is a regular user'
        
        # Return response
        return {
            "message": response_message,
            "isRoad": is_road,
            "validatedBySigV4": validated_by_sigv4
        }
        
    except Exception as e:
        logger.exception("Error processing request")
        return {
            "statusCode": 500,
            "body": json.dumps({
                "message": "Internal server error",
                "error": str(e)
            })
        }

@logger.inject_lambda_context
@tracer.capture_lambda_handler
def handler(event, context: LambdaContext):
    logger.info("Processing API request")
    logger.debug("Event received", extra={"event": event})
    
    # Add the event to the current context for the resolver to access
    return app.resolve(event, context)

Step 5: IAM Permissions
Add the following policy to the calling Lambda's execution role:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:region:account-id:api-id/stage/GET/resource"
        }
    ]
}
