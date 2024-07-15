
```
import logging
import os

from authorizer_library.event_processor import EventProcessor
from authorizer_library.utils import AuthUtils

logger = logging.getLogger()
logger.setLevel(logging.INFO)

ADLDS_PATH = {"/status/{status}/executionId", "/adlds-creation"}
target_adlds = {"adlds_creation_group": os.environ["ADLDS_CREATION_TARGET_ADLDS"]}

class PolicyDispatcher:
    def __init__(self, self, policy, event):
        self.policy = policy
        self.event = event
        self.access_token = None
        self.user_transitive_groups = None
        self.dispatch_table = {
            (ADLDS_PATH["status"], "GET", "secretkey"): self.process_query_request,
            (ADLDS_PATH["status"], "GET", None): self.process_query_request,
            (ADLDS_PATH["creation"], "POST", "secretkey"): self.process_query_request,
            (ADLDS_PATH["creation"], "POST", None): self.process_query_request,
        }

    def process_query_request(self):
        auth_method = self.event["headers"].get("Authorization", None)
        if self.event["httpMethod"] == "GET" and self.event["resource"] == "/status/{executionId}":
            self.policy.allowMethod("GET", self.event["path"])
        elif auth_method == "secretkey":
            self.policy.allowMethod(self.event["httpMethod"], self.event["path"])
        else:
            self.policy.denyAllMethods()

    def process_creation_request(self):
        user_transitive_groups = AuthUtils.get_user_transitive_groups(self.access_token)
        has_target_group = AuthUtils.filter_user_groups(user_transitive_groups, target_adlds["adlds_creation_group"])
        auth_method = self.event["headers"].get("Authorization", None)
        if auth_method == "secretkey":
            logger.info(f"Using {auth_method}")
            self.policy.allowMethod(self.event["httpMethod"], self.event["path"])
        elif has_target_group:
            logger.info(f"Group matched: {target_adlds['adlds_creation_group']}")
            self.policy.allowMethod(self.event["httpMethod"], self.event["path"])
        else:
            self.policy.denyAllMethods()

    def dispatch(self, self, resource, verb, auth_method):
        function = self.dispatch_table.get((resource, verb, auth_method))
        if function:
            function()
        else:
            self.policy.denyAllMethods()

    def lambda_handler(self, event, context):
        processor = EventProcessor(event)
        policy = processor.get_policy()
        logger.info("New ADLDS Automation Event")
        logger.info(f"Resource: {processor.get_resource()}, Path: {processor.get_path()}, Verb: {processor.get_verb()}, Method: {processor.get_auth_method()}")  # noqa: E501

        try:
            dispatcher = PolicyDispatcher(policy, event)
            dispatcher.dispatch(processor.get_resource(), processor.get_verb(), processor.get_auth_method())
            authResponse = dispatcher.policy.build()
        except Exception as e:
            logger.error(e)
            self.policy.denyAllMethods()
            authResponse = self.policy.build()
        return authResponse

```
