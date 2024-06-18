```
import re

class ParamChecker:
    """
    Check parameters
    """
    def __init__(self, event_params):
        self.required_params = [
            "ein_id",
            "basic_entitlement_name",
            "account_id",
            "friendlyRequestId",
            "environment",
        ]
        self.event_params = event_params

    def check_params_exist(self):
        for param in self.required_params:
            if param not in self.event_params:
                return {"success": False, "message": f"Missing parameters {param}"}
        return {"success": True, "message": "all params valid"}

    def check_params_valid(self):
        # Check EIN ID against a regex to make sure it is the correct structure
        if re.match(r"^[0-9]{6,8}$", self.event_params["ein_id"]) is None:
            return {
                "success": False,
                "message": "EIM ID must contain 6-8 digits numbers.",
            }
        # Check if basic_entitlement_name does not contain any special characters
        if re.match(r"^[a-zA-Z0-9]+$", self.event_params["basic_entitlement_name"]) is None:
            return {
                "success": False,
                "message": "Entitlement Name must not contain special characters.",
            }
        if re.match(r"^[0-9]{12}$", self.event_params["account_id"]) is None:
            return {
                "success": False,
                "message": "Account ID must contain 12 digits numbers.",
            }
        # Environment MUST be 'dev' 'preprod' 'prod'
        if self.event_params["environment"] not in ["innovation", "dev", "preprod", "prod"]:
            return {
                "success": False,
                "message": "Environment must be 'innovation', 'dev', 'preprod' or 'prod'.",
            }
        return {"success": True, "message": "all params valid"}

    def check_params(self):
        result = self.check_params_exist()
        if result["success"] is False:
            return result

        result = self.check_params_valid()
        return result
```
