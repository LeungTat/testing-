```
# testing-
import os
import json
import logging
import requests
from azure.identity import AuthorizationCodeCredential

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Set Azure AD App Registration settings
CLIENT_ID = os.environ["CLIENT_ID"]
CLIENT_SECRET = os.environ["CLIENT_SECRET"]
TENANT_ID = os.environ["TENANT_ID"]

# Set the Azure AD OpenID configuration URL
OPENID_CONFIG_URL = f"https://login.microsoftonline.com/{TENANT_ID}/v2.0/.well-known/openid-configuration"

# Get the OpenID configuration
openid_config = requests.get(OPENID_CONFIG_URL).json()

def lambda_handler(event, context):
    try:
        code = event["queryStringParameters"]["code"]
        redirect_uri = f"https://{event['headers']['Host']}{event['requestContext']['path']}"
        
        # Create Azure AD credential object
        credential = AuthorizationCodeCredential(TENANT_ID, CLIENT_ID, CLIENT_SECRET, code, redirect_uri)

        # Get the access token
        access_token = credential.get_token(openid_config["token_endpoint"])

        # Call the Azure AD UserInfo endpoint
        user_info = requests.get(
            openid_config["userinfo_endpoint"],
            headers={"Authorization": f"Bearer {access_token}"}
        ).json()

        return {
            "statusCode": 200,
            "body": json.dumps({"message": "Authenticated successfully", "user": user_info}),
            "headers": {"Content-Type": "application/json"}
        }

    except Exception as e:
        logger.error(f"Error: {e}")
        return {
            "statusCode": 400,
            "body": json.dumps({"message": "Authentication failed"}),
            "headers": {"Content-Type": "application/json"}
        }
        
        
```        


```
To get an authorization code from Azure AD using React, you can use the MSAL library. Here's a sample code snippet demonstrating how to do it:

Install MSAL:
bash
Copy code
npm install @azure/msal-browser
Create a configuration file msalConfig.js in the src folder:
javascript
Copy code
export const msalConfig = {
  auth: {
    clientId: 'your_client_id',
    authority: 'https://login.microsoftonline.com/your_tenant_id',
    redirectUri: 'http://localhost:3000',
    knownAuthorities: ['login.microsoftonline.com'],
  },
  cache: {
    cacheLocation: 'localStorage',
    storeAuthStateInCookie: false,
  },
};
Replace your_client_id and your_tenant_id with the actual values from your Azure AD app registration.

In your App.js, import the required modules and configure MSAL:
javascript
Copy code
import React, { useState, useEffect } from 'react';
import { PublicClientApplication, InteractionRequiredAuthError, InteractionType, EventType } from '@azure/msal-browser';
import { msalConfig } from './msalConfig';

const pca = new PublicClientApplication(msalConfig);
Create a function to initiate the login process and obtain the authorization code:
javascript
Copy code
const handleLogin = async () => {
  try {
    const authResult = await pca.loginPopup({
      scopes: ['openid', 'profile', 'email'],
      prompt: 'select_account',
      responseMode: 'query',
      codeChallengeMethod: 'S256',
    });

    if (authResult && authResult.code) {
      console.log('Authorization code:', authResult.code);
    }
  } catch (error) {
    if (error instanceof InteractionRequiredAuthError) {
      pca.loginRedirect({
        scopes: ['openid', 'profile', 'email'],
        prompt: 'select_account',
        responseMode: 'query',
        codeChallengeMethod: 'S256',
      });
    } else {
      console.error('Login failed:', error);
    }
  }
};
Finally, add a button to trigger the login process:
javascript
Copy code
function App() {
  return (
    <div className="App">
      <h1>Azure AD Authorization Code Example</h1>
      <button onClick={handleLogin}>Login with Azure AD</button>
    </div>
  );
}
Now, when you click the "Login with Azure AD" button, it will initiate the login process, and once completed, you'll receive an authorization code in the console. Remember that this code is only part of the OAuth2 flow and needs to be exchanged for an access token using a backend service.

Note: This example assumes you have a single-page React app. For server-rendered apps, the process may be different.
```


```
Yes, you can create the Azure AD login example in TypeScript. Here's how to do it:

Create a new TypeScript-based React app:
bash
Copy code
npx create-react-app azure-ad-login --template typescript
cd azure-ad-login
Install the necessary packages:
bash
Copy code
npm install @azure/msal-browser
Create a configuration file msalConfig.ts in the src folder:
typescript
Copy code
export const msalConfig = {
  auth: {
    clientId: 'your_client_id',
    authority: 'https://login.microsoftonline.com/your_tenant_id',
    redirectUri: 'http://localhost:3000',
    knownAuthorities: ['login.microsoftonline.com'],
  },
  cache: {
    cacheLocation: 'localStorage',
    storeAuthStateInCookie: false,
  },
};
Replace your_client_id and your_tenant_id with the actual values from your Azure AD app registration.

Modify the src/App.tsx file to implement the Azure AD login flow:
typescript
Copy code
import React, { useState, useEffect } from 'react';
import './App.css';
import { PublicClientApplication, InteractionRequiredAuthError, EventType } from '@azure/msal-browser';
import { msalConfig } from './msalConfig';

const pca = new PublicClientApplication(msalConfig);

const App: React.FC = () => {
  const handleLogin = async () => {
    try {
      const authResult = await pca.loginPopup({
        scopes: ['openid', 'profile', 'email'],
        prompt: 'select_account',
        responseMode: 'query',
        codeChallengeMethod: 'S256',
      });

      if (authResult && authResult.code) {
        console.log('Authorization code:', authResult.code);
      }
    } catch (error) {
      if (error instanceof InteractionRequiredAuthError) {
        pca.loginRedirect({
          scopes: ['openid', 'profile', 'email'],
          prompt: 'select_account',
          responseMode: 'query',
          codeChallengeMethod: 'S256',
        });
      } else {
        console.error('Login failed:', error);
      }
    }
  };

  return (
    <div className="App">
      <h1>Azure AD Authorization Code Example (TypeScript)</h1>
      <button onClick={handleLogin}>Login with Azure AD</button>
    </div>
  );
};

export default App;
Now you have a TypeScript-based React app that uses Azure AD for authentication and retrieves the authorization code using OAuth2. Remember that the authorization code is only part of the OAuth2 flow and needs to be exchanged for an access token using a backend service.
```
