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



```
Here is an example of how you can manually implement the authorization code flow in your React app:

First, install the crypto-js package to help generate the code challenge and code verifier:
bash
Copy code
npm install crypto-js
Add a utility function to create the code challenge and code verifier in a new file pkce.ts:
typescript
Copy code
import { Base64 } from 'crypto-js';

export function generateRandomString(length: number): string {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length));
  }
  return result;
}

export function generateCodeVerifier(): string {
  return generateRandomString(128);
}

export function generateCodeChallenge(codeVerifier: string): string {
  const codeChallenge = Base64.stringify(codeVerifier);
  return codeChallenge;
}
Modify your App.tsx file:
typescript
Copy code
import React, { useState, useEffect } from 'react';
import './App.css';
import { generateCodeVerifier, generateCodeChallenge } from './pkce';

const clientId = 'your_client_id';
const tenantId = 'your_tenant_id';
const redirectUri = 'http://localhost:3000';

const App: React.FC = () => {
  const [authCode, setAuthCode] = useState('');

  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const code = params.get('code');
    if (code) {
      setAuthCode(code);
    }
  }, []);

  const handleLogin = () => {
    const codeVerifier = generateCodeVerifier();
    const codeChallenge = generateCodeChallenge(codeVerifier);
    sessionStorage.setItem('code_verifier', codeVerifier);

    const authUrl = `https://login.microsoftonline.com/${tenantId}/oauth2/v2.0/authorize?client_id=${clientId}&response_type=code&redirect_uri=${encodeURIComponent(
      redirectUri
    )}&response_mode=query&scope=openid%20profile%20email&state=12345&code_challenge=${codeChallenge}&code_challenge_method=S256`;

    window.location.href = authUrl;
  };

  return (
    <div className="App">
      <h1>Azure AD Authorization Code Example (Manual)</h1>
      {authCode ? (
        <div>
          <p>Authorization code: {authCode}</p>
        </div>
      ) : (
        <button onClick={handleLogin}>Login with Azure AD</button>
      )}
    </div>
  );
};

export default App;
This implementation manually initiates the authorization code flow, and when the user is redirected back to the app, it will display the authorization code on the screen. Please note that this example does not handle the token exchange or token validation. You will need to do that on your backend service.




Apologies for the oversight. I made an error in the import statement. Instead of using Base64 from crypto-js, you should import the sha256 function and use it to generate the code challenge.

Here's the corrected pkce.ts:

typescript
Copy code
import { sha256 } from 'crypto-js';

export function generateRandomString(length: number): string {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length));
  }
  return result;
}

export function generateCodeVerifier(): string {
  return generateRandomString(128);
}

export function generateCodeChallenge(codeVerifier: string): string {
  const hashedCodeVerifier = sha256(codeVerifier);
  const base64UrlEncodedHash = btoa(hashedCodeVerifier.toString())
    .replace('+', '-')
    .replace('/', '_')
    .replace(/=+$/, '');
  return base64UrlEncodedHash;
}
The generateCodeChallenge function now imports the sha256 function from crypto-js and uses it to hash the code verifier. The hashed code verifier is then Base64Url encoded to create the code challenge.


 apologize for the oversight. The issue is caused by the way the code challenge is generated. The generateCodeChallenge function needs to be updated to correctly hash and encode the code verifier. Here's the corrected pkce.ts:

typescript
Copy code
import { sha256 } from 'crypto-js';
import { enc } from 'crypto-js';

export function generateRandomString(length: number): string {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length));
  }
  return result;
}

export function generateCodeVerifier(): string {
  return generateRandomString(128);
}

export function generateCodeChallenge(codeVerifier: string): string {
  const hashedCodeVerifier = sha256(codeVerifier);
  const base64UrlEncodedHash = enc.Base64.stringify(hashedCodeVerifier)
    .replace('+', '-')
    .replace('/', '_')
    .replace(/=+$/, '');
  return base64UrlEncodedHash;
}
Now, the generateCodeChallenge function imports both sha256 and enc from crypto-js. It hashes the code verifier using sha256 and then encodes it using enc.Base64. The resulting string is then modified to be URL-safe by replacing characters and removing trailing equal signs.

This should resolve the "invalid size of Code_Challenge parameter" error.


```
```
To create an AWS Lambda function for Azure AD OpenID login using Python, you'll need to first set up an Azure AD App Registration and then create a Python Lambda function to handle the authentication.

Azure AD App Registration:
Follow these steps to set up an App Registration in the Azure portal:

1.1. Sign in to the Azure portal (https://portal.azure.com/)

1.2. Go to "Azure Active Directory" > "App registrations" > "New registration"

1.3. Provide a name for your application, and under "Supported account types," select the appropriate option for your requirements.

1.4. For "Redirect URI," choose "Web" and enter the following: https://<YOUR_API_GATEWAY_URL>/callback, replacing <YOUR_API_GATEWAY_URL> with the appropriate API Gateway URL.

1.5. Click "Register" to create the application.

1.6. Note the "Application (client) ID" and "Directory (tenant) ID" from the "Overview" tab.

1.7. Go to "Certificates & secrets" > "New client secret" to create a new secret. Note the secret value, as it will not be displayed again.

1.8. Go to "Manifest" and set "accessTokenAcceptedVersion": 2, then save the changes.

Create a Python Lambda function:
2.1. Set up a new Python project in your local environment. You can use virtualenv or pipenv to manage dependencies. Install the required packages:

bash
Copy code
pip install requests
2.2. Create a file named lambda_function.py and add the following code:

python
Copy code
import os
import json
import logging
import requests

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
        
        # Prepare token request data
        data = {
            "grant_type": "authorization_code",
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
            "code": code,
            "redirect_uri": redirect_uri,
            "scope": "openid"
        }

        # Request the access token
        response = requests.post(openid_config["token_endpoint"], data=data)
        response.raise_for_status()

        # Parse the access token
        token_data = response.json()
        access_token = token_data["access_token"]

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
import { sha256 } from 'crypto-js';
import { enc } from 'crypto-js';

export function generateRandomString(length: number): string {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length));
  }
  return result;
}

export function generateCodeVerifier(): string {
  return generateRandomString(128);
}

export function generateCodeChallenge(codeVerifier: string): string {
  const hashedCodeVerifier = sha256(codeVerifier);
  const base64UrlEncodedHash = base64UrlEncode(hashedCodeVerifier);
  return base64UrlEncodedHash;
}

function base64UrlEncode(value: string): string {
  const base64 = enc.Base64.stringify(value);
  const base64Url = base64.replace('+', '-').replace('/', '_').replace(/=+$/, '');
  return base64Url;
}
```


```
import { sha256 } from 'crypto-js';
import { enc } from 'crypto-js';

export function generateRandomString(length: number): string {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length));
  }
  return result;
}

export function generateCodeVerifier(): string {
  return generateRandomString(128);
}

export function generateCodeChallenge(codeVerifier: string): string {
  const hashedCodeVerifier = sha256(codeVerifier);
  const base64UrlEncodedHash = base64UrlEncode(enc.Base64.stringify(hashedCodeVerifier));
  return base64UrlEncodedHash;
}

function base64UrlEncode(value: string): string {
  const base64Url = value.replace('+', '-').replace('/', '_').replace(/=+$/, '');
  return base64Url;
}
```


```
Yes, it is possible to set up a React application to get the authorization code, and then use a backend server to exchange the authorization code for an access token. Here's a high-level overview of the process:

Create a React application and configure it to use the PKCE flow for Azure AD authentication. I provided an example of this in the previous responses.

Set up a backend server to handle the authorization code exchange. This server can be implemented using a framework like Express.js (Node.js), Flask (Python), or any other backend framework of your choice. Here, I'll provide an example using Express.js.

When the React application receives the authorization code, make an API call to the backend server with the authorization code and code verifier. The backend server will then exchange the authorization code for an access token and return it to the React application.

Let's create a simple Express.js backend server for this purpose:

Install Node.js and create a new directory for your server:
bash
Copy code
mkdir backend-server
cd backend-server
Initialize your project and install the required packages:
bash
Copy code
npm init -y
npm install express axios
Create a new file named app.js:
javascript
Copy code
const express = require('express');
const axios = require('axios');

const app = express();
const port = 3001;

const clientId = 'your_client_id';
const clientSecret = 'your_client_secret';
const tenantId = 'your_tenant_id';
const redirectUri = 'http://localhost:3000';

app.use(express.json());

app.post('/exchange', async (req, res) => {
  const code = req.body.code;
  const codeVerifier = req.body.code_verifier;

  if (!code || !codeVerifier) {
    return res.status(400).send('Missing code or code_verifier');
  }

  try {
    const tokenEndpoint = `https://login.microsoftonline.com/${tenantId}/oauth2/v2.0/token`;
    const requestBody = {
      client_id: clientId,
      client_secret: clientSecret,
      grant_type: 'authorization_code',
      code,
      redirect_uri: redirectUri,
      code_verifier: codeVerifier,
    };

    const response = await axios.post(tokenEndpoint, new URLSearchParams(requestBody));
    const accessToken = response.data.access_token;
    res.send({ access_token: accessToken });
  } catch (error) {
    res.status(500).send(`Error exchanging authorization code for access token: ${error.message}`);
  }
});

app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
Replace your_client_id, your_client_secret, and your_tenant_id with the appropriate values from your Azure AD app registration.

Start the server:
bash
Copy code
node app.js
Now, in your React application, you can send a POST request to the backend server to exchange the authorization code for an access token:

Install axios in your React application:
bash
Copy code
npm install axios
Update the App.tsx file to send a request to the backend server:
typescript
Copy code
// ... other imports
import axios from 'axios';

// ...

// Add this function inside the App component
const exchangeCodeForToken = async (code: string, codeVerifier: string) => {
  try {
    const response = await axios.post('http://localhost:3001/exchange', {
      code,
      code_verifier: codeVerifier,
    });
    const accessToken = response.data.access_token;
    console.log('Access token:', accessToken);
  } catch (error) {
    console.error('Failed to exchange authorization code for



anthonykslai@gmail.com
can you complete the sample react app ?
Sure! Here's a complete React application that works with the Express.js backend server to exchange the authorization code for an access token.

First, make sure you have the axios package installed in your React application:
bash
Copy code
npm install axios
Update your App.tsx file with the following code:
typescript
Copy code
import React, { useState, useEffect } from 'react';
import './App.css';
import { generateCodeVerifier, generateCodeChallenge } from './pkce';
import axios from 'axios';

const clientId = 'your_client_id';
const tenantId = 'your_tenant_id';
const redirectUri = 'http://localhost:3000';

const App: React.FC = () => {
  const [authCode, setAuthCode] = useState('');

  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const code = params.get('code');
    if (code) {
      setAuthCode(code);

      const codeVerifier = sessionStorage.getItem('code_verifier');
      if (codeVerifier) {
        exchangeCodeForToken(code, codeVerifier);
      }
    }
  }, []);

  const handleLogin = () => {
    const codeVerifier = generateCodeVerifier();
    const codeChallenge = generateCodeChallenge(codeVerifier);
    sessionStorage.setItem('code_verifier', codeVerifier);

    const authUrl = `https://login.microsoftonline.com/${tenantId}/oauth2/v2.0/authorize?client_id=${clientId}&response_type=code&redirect_uri=${encodeURIComponent(
      redirectUri
    )}&response_mode=query&scope=openid%20profile%20email&state=12345&code_challenge=${codeChallenge}&code_challenge_method=S256`;

    window.location.href = authUrl;
  };

  const exchangeCodeForToken = async (code: string, codeVerifier: string) => {
    try {
      const response = await axios.post('http://localhost:3001/exchange', {
        code,
        code_verifier: codeVerifier,
      });
      const accessToken = response.data.access_token;
      console.log('Access token:', accessToken);
    } catch (error) {
      console.error('Failed to exchange authorization code for access token:', error);
    }
  };

  return (
    <div className="App">
      <h1>Azure AD Authorization Code Example</h1>
      {authCode ? (
        <div>
          <p>Authorization code: {authCode}</p>
        </div>
      ) : (
        <button onClick={handleLogin}>Login with Azure AD</button>
      )}
    </div>
  );
};

export default App;
Make sure to replace your_client_id and your_tenant_id with the appropriate values from your Azure AD app registration.

In the same folder as App.tsx, create a new file named pkce.ts with the following content:
typescript
Copy code
import { sha256 } from 'crypto-js';
import { enc } from 'crypto-js';

export function generateRandomString(length: number): string {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length));
  }
  return result;
}

export function generateCodeVerifier(): string {
  return generateRandomString(128);
}

export function generateCodeChallenge(codeVerifier: string): string {
  const hashedCodeVerifier = sha256(codeVerifier);
  const base64UrlEncodedHash = base64UrlEncode(enc.Base64.stringify(hashedCodeVerifier));
  return base64UrlEncodedHash;
}

function base64UrlEncode(value: string): string {



anthonykslai@gmail.com
please complete the code
Apologies for the incomplete code. Here's the complete pkce.ts file:

typescript
Copy code
import { sha256 } from 'crypto-js';
import { enc } from 'crypto-js';

export function generateRandomString(length: number): string {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length));
  }
  return result;
}

export function generateCodeVerifier(): string {
  return generateRandomString(128);
}

export function generateCodeChallenge(codeVerifier: string): string {
  const hashedCodeVerifier = sha256(codeVerifier);
  const base64UrlEncodedHash = base64UrlEncode(enc.Base64.stringify(hashedCodeVerifier));
  return base64UrlEncodedHash;
}

function base64UrlEncode(value: string): string {
  const base64Url = value.replace('+', '-').replace('/', '_').replace(/=+$/, '');
  return base64Url;
}
With these files, you now have a complete React application that retrieves the authorization code and sends it to the Express.js backend server to exchange it for an access token.
```

```
Apologies for the confusion. It seems that the issue is due to the fact that the code_verifier is being stored in the session storage, but when the page is redirected, the storage might be cleared. Let's modify the code to store the code_verifier in the local storage to maintain its value throughout the process.

Update the handleLogin function in your App.tsx file as follows:

typescript
Copy code
const handleLogin = () => {
  const codeVerifier = generateCodeVerifier();
  const codeChallenge = generateCodeChallenge(codeVerifier);
  localStorage.setItem('code_verifier', codeVerifier); // Change this line

  const authUrl = `https://login.microsoftonline.com/${tenantId}/oauth2/v2.0/authorize?client_id=${clientId}&response_type=code&redirect_uri=${encodeURIComponent(
    redirectUri
  )}&response_mode=query&scope=openid%20profile%20email&state=12345&code_challenge=${codeChallenge}&code_challenge_method=S256`;

  window.location.href = authUrl;
};
Also, update the useEffect function to retrieve the code_verifier from the local storage:

typescript
Copy code
useEffect(() => {
  const params = new URLSearchParams(window.location.search);
  const code = params.get('code');
  if (code) {
    setAuthCode(code);

    const codeVerifier = localStorage.getItem('code_verifier'); // Change this line
    if (codeVerifier) {
      exchangeCodeForToken(code, codeVerifier);
    }
  }
}, []);
Now, the code_verifier will be stored in the local storage, and it should persist even when the page is redirected. This should resolve the "code_verifier does not match the code_challenge" error.
```


```
Certainly! Here's a Python AWS Lambda function that can exchange the authorization code for an access token using Azure AD:

First, make sure you have the boto3, requests, and python-dotenv packages installed:
bash
Copy code
pip install boto3 requests python-dotenv
Create a new file named lambda_function.py with the following content:
python
Copy code
import os
import json
import boto3
import requests
from botocore.exceptions import ClientError
from dotenv import load_dotenv

# Load environment variables from the .env file
load_dotenv()

# Set up your Azure AD app details
CLIENT_ID = os.environ['CLIENT_ID']
CLIENT_SECRET = os.environ['CLIENT_SECRET']
TENANT_ID = os.environ['TENANT_ID']
REDIRECT_URI = os.environ['REDIRECT_URI']


def lambda_handler(event, context):
    # Extract the code and code_verifier from the event
    code = event['code']
    code_verifier = event['code_verifier']

    # Prepare the token request data
    token_url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
    headers = {
        "Content-Type": "application/x-www-form-urlencoded",
    }
    data = {
        "grant_type": "authorization_code",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "redirect_uri": REDIRECT_URI,
        "code": code,
        "code_verifier": code_verifier,
    }

    # Send the token request
    response = requests.post(token_url, headers=headers, data=data)

    if response.status_code == 200:
        # Successfully retrieved the access token
        access_token = response.json()['access_token']
        return {
            'statusCode': 200,
            'body': json.dumps({
                'access_token': access_token,
            }),
        }
    else:
        # Failed to retrieve the access token
        return {
            'statusCode': response.status_code,
            'body': json.dumps({
                'error': response.json(),
            }),
        }
Create a .env file in the same folder as lambda_function.py and add the following environment variables:
makefile
Copy code
CLIENT_ID=your_client_id
CLIENT_SECRET=your_client_secret
TENANT_ID=your_tenant_id
REDIRECT_URI=http://localhost:3000
Make sure to replace your_client_id, your_client_secret, and your_tenant_id with the appropriate values from your Azure AD app registration. Also, update the REDIRECT_URI if necessary.

You can deploy this Lambda function using the AWS Management Console, AWS CLI, or any other deployment tool that supports AWS Lambda. Don't forget to include the requests and python-dotenv packages in your deployment package.

To test the Lambda function, you can invoke it with an event containing the code and code_verifier:

json
Copy code
{
  "code": "your_authorization_code",
  "code_verifier": "your_code_verifier"
}
Replace your_authorization_code and your_code_verifier with the actual values obtained from the frontend.

After deploying and testing the Lambda function, you should be able to exchange the authorization code for an access token using Azure AD.
```


```
To create an AWS Lambda function that calls the Microsoft Graph API to get the group information of a user, follow these steps:

Make sure you have the required Python packages installed:
bash
Copy code
pip install boto3 requests python-dotenv
Create a new file named lambda_function.py with the following content:
python
Copy code
import os
import json
import boto3
import requests
from botocore.exceptions import ClientError
from dotenv import load_dotenv

# Load environment variables from the .env file
load_dotenv()

# Set up your Azure AD app details
CLIENT_ID = os.environ['CLIENT_ID']
TENANT_ID = os.environ['TENANT_ID']

def get_user_groups(access_token, user_id):
    graph_url = f"https://graph.microsoft.com/v1.0/users/{user_id}/memberOf"
    headers = {
        "Authorization": f"Bearer {access_token}"
    }

    response = requests.get(graph_url, headers=headers)
    return response.json()

def lambda_handler(event, context):
    access_token = event['access_token']
    user_id = event['user_id']

    user_groups = get_user_groups(access_token, user_id)

    return {
        'statusCode': 200,
        'body': json.dumps(user_groups)
    }
Create a .env file in the same folder as lambda_function.py and add the following environment variables:
makefile
Copy code
CLIENT_ID=your_client_id
TENANT_ID=your_tenant_id
Replace your_client_id and your_tenant_id with the appropriate values from your Azure AD app registration.

Deploy the Lambda function to AWS using the AWS Management Console, AWS CLI, or any other deployment tool that supports AWS Lambda. Make sure to include the requests and python-dotenv packages in your deployment package.

To test the Lambda function, you can invoke it with an event containing the access_token and user_id:

json
Copy code
{
  "access_token": "your_access_token",
  "user_id": "your_user_id"
}
Replace your_access_token and your_user_id with the actual values obtained from the frontend or another source.

After deploying and testing the Lambda function, you should be able to use it to call the Microsoft Graph API and retrieve the group information of a user with a valid access token.
```



```
To create an AWS Lambda function in Python that calls the Microsoft Graph API to get the transitive group memberships of the signed-in user and filters the user groups, follow these steps:

Make sure you have the required Python packages installed:
bash
Copy code
pip install boto3 requests python-dotenv
Create a new file named lambda_function.py with the following content:
python
Copy code
import os
import json
import boto3
import requests
from botocore.exceptions import ClientError
from dotenv import load_dotenv

# Load environment variables from the .env file
load_dotenv()

# Set up your Azure AD app details
CLIENT_ID = os.environ['CLIENT_ID']
TENANT_ID = os.environ['TENANT_ID']

def get_user_transitive_groups(access_token):
    graph_url = "https://graph.microsoft.com/v1.0/me/transitiveMemberOf/microsoft.graph.group?$count=true"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "ConsistencyLevel": "eventual"
    }

    response = requests.get(graph_url, headers=headers)
    return response.json()

def filter_user_groups(groups):
    user_groups = [group for group in groups['value'] if group['@odata.type'] == "#microsoft.graph.group"]
    return user_groups

def lambda_handler(event, context):
    access_token = event['access_token']

    user_transitive_groups = get_user_transitive_groups(access_token)
    user_groups = filter_user_groups(user_transitive_groups)

    return {
        'statusCode': 200,
        'body': json.dumps(user_groups)
    }
Create a .env file in the same folder as lambda_function.py and add the following environment variables:
makefile
Copy code
CLIENT_ID=your_client_id
TENANT_ID=your_tenant_id
Replace your_client_id and your_tenant_id with the appropriate values from your Azure AD app registration.

Deploy the Lambda function to AWS using the AWS Management Console, AWS CLI, or any other deployment tool that supports AWS Lambda. Make sure to include the requests and python-dotenv packages in your deployment package.

To test the Lambda function, you can invoke it with an event containing the access_token:

json
Copy code
{
  "access_token": "your_access_token"
}
Replace your_access_token with the actual value obtained from the frontend or another source.

After deploying and testing the Lambda function, you should be able to use it to call the Microsoft Graph API and retrieve the filtered user groups with a valid access token.
```

```
Please create a aws lambda code for azure ad login using username and password with python 

To create an AWS Lambda function for Azure AD login using a username and password in Python, you will need to use the Resource Owner Password Credentials (ROPC) flow. However, this flow is not recommended for most applications because it requires handling user credentials directly, which can lead to security vulnerabilities.

Microsoft strongly advises against using the ROPC flow, and it's not supported for use with multi-factor authentication or Microsoft personal accounts. It is highly recommended to use other OAuth2 flows, such as the Authorization Code flow or the Implicit flow.

If you understand the risks and limitations and still want to proceed with the ROPC flow, here's a simple example of how to implement it using an AWS Lambda function in Python:

Install the required Python packages:
bash
Copy code
pip install boto3 requests
Create a new file named lambda_function.py with the following content:
python
Copy code
import os
import json
import boto3
import requests
from botocore.exceptions import ClientError

# Set up your Azure AD app details
CLIENT_ID = os.environ['CLIENT_ID']
TENANT_ID = os.environ['TENANT_ID']
CLIENT_SECRET = os.environ['CLIENT_SECRET']

def get_access_token(username, password):
    token_url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/token"
    data = {
        "grant_type": "password",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "resource": "https://graph.microsoft.com",
        "username": username,
        "password": password,
        "scope": "openid"
    }

    response = requests.post(token_url, data=data)
    return response.json()

def lambda_handler(event, context):
    username = event['username']
    password = event['password']

    token_response = get_access_token(username, password)

    return {
        'statusCode': 200,
        'body': json.dumps(token_response)
    }
Set the required environment variables for the Lambda function:
CLIENT_ID: Your Azure AD app client ID
TENANT_ID: Your Azure AD tenant ID
CLIENT_SECRET: Your Azure AD app client secret
Deploy the Lambda function to AWS using the AWS Management Console, AWS CLI, or any other deployment tool that supports AWS Lambda. Make sure to include the requests package in your deployment package.

To test the Lambda function, you can invoke it with an event containing the username and password:

json
Copy code
{
  "username": "your_username",
  "password": "your_password"
}
Replace your_username and your_password with actual values.

Keep in mind that using the ROPC flow is not recommended, and you should consider alternative authentication flows for better security and user experience.





anthonykslai@gmail.com
Please help to write a test case for it 
To write a test case for the Lambda function, you can use Python's built-in unittest library. You will also need to use the unittest.mock library to mock the requests.post method to avoid making actual HTTP requests during the test.

Install the required Python packages for testing:
bash
Copy code
pip install unittest requests-mock
Create a new file named test_lambda_function.py with the following content:
python
Copy code
import unittest
import json
from unittest.mock import patch
import requests_mock
import lambda_function

class TestLambdaFunction(unittest.TestCase):

    @patch("requests.post")
    def test_get_access_token(self, mock_post):
        mock_response = {
            "access_token": "mock_access_token",
            "token_type": "Bearer",
            "expires_in": 3600
        }

        mock_post.return_value.json.return_value = mock_response

        result = lambda_function.get_access_token("test_username", "test_password")

        self.assertEqual(result, mock_response)

    def test_lambda_handler(self):
        with requests_mock.Mocker() as m:
            mock_response = {
                "access_token": "mock_access_token",
                "token_type": "Bearer",
                "expires_in": 3600
            }

            m.post("https://login.microsoftonline.com/{}/oauth2/token".format(lambda_function.TENANT_ID), json=mock_response)

            event = {
                "username": "test_username",
                "password": "test_password"
            }

            response = lambda_function.lambda_handler(event, None)

            self.assertEqual(response["statusCode"], 200)
            self.assertEqual(json.loads(response["body"]), mock_response)

if __name__ == "__main__":
    unittest.main()
This script defines two test cases:

test_get_access_token tests the get_access_token function. It uses the unittest.mock.patch decorator to mock the requests.post method and provide a fake response.
test_lambda_handler tests the lambda_handler function. It uses the requests_mock library to mock the HTTP request made by the get_access_token function during the test.
To run the test cases, execute the following command:

bash
Copy code
python test_lambda_function.py
This command will run the tests and display the results. If the tests pass, you should see output similar to the following:

markdown
Copy code
..
----------------------------------------------------------------------
Ran 2 tests in 0.006s

OK
```

```
import os
import json
import logging
import requests

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
        
        # Prepare token request data
        data = {
            "grant_type": "authorization_code",
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
            "code": code,
            "redirect_uri": redirect_uri,
            "scope": "openid"
        }

        # Request the access token
        response = requests.post(openid_config["token_endpoint"], data=data)
        response.raise_for_status()

        # Parse the access token
        token_data = response.json()
        access_token = token_data["access_token"]

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
To check if the get_user_transitive_groups function has any error response and raise an error message, you can update the function to handle different status codes returned by the Microsoft Graph API.

Update the get_user_transitive_groups function in the authorizer.py file as follows:

python
Copy code
def get_user_transitive_groups(access_token):
    graph_url = "https://graph.microsoft.com/v1.0/me/transitiveMemberOf"
    headers = {
        "Authorization": f"Bearer {access_token}"
    }

    response = requests.get(graph_url, headers=headers)
    
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 401:
        raise Exception("Unauthorized: Invalid or expired access token")
    elif response.status_code == 403:
        raise Exception("Forbidden: Insufficient permissions to access the resource")
    else:
        raise Exception(f"Error: {response.status_code} - {response.text}")
In this updated version of the function, we check the status code of the response. If it's 200, we return the JSON data. If it's 401, we raise an "Unauthorized" exception. If it's 403, we raise a "Forbidden" exception. For other status codes, we raise a generic "Error" exception with the status code and response text.

Next, update the lambda_handler function to handle the exceptions raised by the get_user_transitive_groups function:

python
Copy code
def lambda_handler(event, context):
    access_token = event['authorizationToken']
    
    try:
        # Replace 'your_secret_key' with your actual secret key
        decoded_token = decode(access_token, 'your_secret_key', algorithms=['HS256'])
    except exceptions.InvalidTokenError:
        return {
            "statusCode": 401,
            "body": "Invalid JWT"
        }

    try:
        user_transitive_groups = get_user_transitive_groups(access_token)
    except Exception as e:
        return {
            "statusCode": 400,
            "body": str(e)
        }

    group_name = "your_group_name" # Replace with the name of the group to check
    user_in_group = is_user_in_group(user_transitive_groups, group_name)

    principal_id = "user"

    if user_in_group:
        return generate_policy(principal_id, 'Allow', event['methodArn'])
    else:
        return generate_policy(principal_id, 'Deny', event['methodArn'])
In this updated version of the lambda_handler function, we use a try-except block to catch exceptions raised by the get_user_transitive_groups function. If an exception is caught, we return a 400 status code with the error message from the exception.

Now, if there's an error response from the Microsoft Graph API, the Lambda authorizer will return an appropriate error message.
```

```
import json
import unittest
from unittest.mock import Mock, patch
import lambda_function

class TestLambdaFunction(unittest.TestCase):

    @patch("requests.post")
    @patch("requests.get")
    def test_successful_authentication(self, mock_get, mock_post):
        # Mock the token endpoint response
        token_response = Mock()
        token_response.json.return_value = {
            "access_token": "sample_access_token",
            "token_type": "Bearer",
            "expires_in": 3600
        }
        mock_post.return_value = token_response
        
        # Mock the OpenID configuration response
        openid_config_response = Mock()
        openid_config_response.json.return_value = {
            "issuer": "https://login.microsoftonline.com/{tenantid}/v2.0",
            "authorization_endpoint": "https://login.microsoftonline.com/{tenantid}/oauth2/v2.0/authorize",
            "token_endpoint": "https://login.microsoftonline.com/{tenantid}/oauth2/v2.0/token",
            "userinfo_endpoint": "https://graph.microsoft.com/oidc/userinfo",
            "jwks_uri": "https://login.microsoftonline.com/{tenantid}/discovery/v2.0/keys"
        }
        mock_get.return_value = openid_config_response

        # Mock the event object received by the Lambda function
        event = {
            "queryStringParameters": {"code": "sample_authorization_code"},
            "headers": {"Host": "sample-api-gateway-url"},
            "requestContext": {"path": "/callback"}
        }

        response = lambda_function.lambda_handler(event, None)

        self.assertEqual(response["statusCode"], 200)
        self.assertEqual(json.loads(response["body"]), {"access_token": "sample_access_token"})
        mock_post.assert_called_once()
        mock_get.assert_called_once()

if __name__ == "__main__":
    unittest.main()
    ```

```

import pyautogui
import time

设置移动的时间间隔和总时间
interval = 1  # 移动的时间间隔（秒）
total_time = 10  # 移动的总时间（秒）

获取当前鼠标位置
start_x, start_y = pyautogui.position()

计算移动的目标位置
end_x = start_x + 100  # 在x轴方向移动100个像素
end_y = start_y + 100  # 在y轴方向移动100个像素

计算移动的次数
num_steps = int(total_time / interval)

开始移动鼠标
for i in range(num_steps):
    # 计算每步移动的增量
    delta_x = (end_x - start_x) / num_steps
    delta_y = (end_y - start_y) / num_steps

    # 计算每步移动的目标位置
    target_x = start_x + i * delta_x
    target_y = start_y + i * delta_y

    # 移动鼠标到目标位置
    pyautogui.moveTo(target_x, target_y)
    
    # 等待一段时间
    time.sleep(interval)

移动完成后，将鼠标移动回起始位置
pyautogui.moveTo(start_x, start_y)

```


```
import pyautogui
import time

# 设置移动的时间间隔
interval = 1  # 移动的时间间隔（秒）

# 获取屏幕的宽度和高度
screen_width, screen_height = pyautogui.size()

# 设置初始位置
x = 0
y = 0

# 开始移动鼠标
while True:
    # 移动鼠标到指定位置
    pyautogui.moveTo(x, y, duration=interval)

    # 更新坐标
    x += 10
    y += 10

    # 判断是否达到屏幕边界
    if x >= screen_width:
        x = 0
    if y >= screen_height:
        y = 0

    # 等待一段时间
    time.sleep(interval)
    ```


```
import axios from 'axios';

const url = 'https://api.example.com/endpoint'; // Replace with the actual API endpoint URL

// Optional: Define headers, parameters, and payload
const headers = {
  Authorization: 'Bearer <your_access_token>',
  'Content-Type': 'application/json'
};

const params = {
  param1: 'value1',
  param2: 'value2'
};

const payload = {
  key1: 'value1',
  key2: 'value2'
};

// Make the API request
async function makeRequest(): Promise<void> {
  try {
    const response = await axios.get(url, {
      headers: headers,
      params: params,
      data: payload
    });
    // For other HTTP methods, replace 'get' with 'post', 'put', 'delete', etc.

    // API request was successful
    const data = response.data; // Extract the JSON data from the response
    console.log(data);
  } catch (error) {
    // API request failed
    console.error('An error occurred:', error.message);
  }
}

makeRequest();
```

```

import requests

url = "https://api.example.com/endpoint"  # Replace with the actual API endpoint URL

# Optional: Define headers, parameters, and payload
headers = {
    "Authorization": "Bearer <your_access_token>",
    "Content-Type": "application/json"
}

params = {
    "param1": "value1",
    "param2": "value2"
}

payload = {
    "key1": "value1",
    "key2": "value2"
}

# Make the API request
try:
    response = requests.get(url, headers=headers, params=params, json=payload)
    # For other HTTP methods, replace 'get' with 'post', 'put', 'delete', etc.

    # Check the response status code
    if response.status_code == requests.codes.ok:
        # API request was successful
        data = response.json()  # Extract the JSON data from the response
        print(data)
    else:
        # API request failed
        print("Request failed with status code:", response.status_code)

except requests.exceptions.RequestException as e:
    print("An error occurred:", e)
    
    ```
    
    ```
    const jwtToken = 'your_jwt_token';  // Replace with your actual JWT

const decodedToken = JSON.parse(atob(jwtToken.split('.')[1]));
console.log(decodedToken);
```


```
import unittest
from unittest.mock import patch, MagicMock
from botocore.exceptions import ClientError
import your_module  # Replace this with the actual module name where your lambda_handler function is defined

class TestLambdaHandler(unittest.TestCase):
    @patch('os.environ', return_value={'SECRET_KEY_NAME': 'test_secret_key_name', 'CLIENT_ID': 'test_client_id', 'AWS_REGION': 'test_region'})
    @patch('your_module.boto3.session.Session')
    @patch('your_module.requests.post')
    def test_lambda_handler(self, requests_post_mock, boto_session_mock, os_environ_mock):
        # Arrange
        mock_event = {
            'body': '{"username": "test_user", "password": "test_password"}'
        }
        mock_context = {}

        # Mocking boto3 secret manager response
        mock_secrets_client = MagicMock()
        boto_session_mock.return_value.client.return_value = mock_secrets_client
        mock_secrets_client.get_secret_value.return_value = {'SecretString': 'test_secret'}

        # Mocking requests post response
        mock_response = MagicMock()
        mock_response.json.return_value = {'access_token': 'test_access_token'}
        requests_post_mock.return_value = mock_response

        expected_output = {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Credentials': 'true',
                'Content-Type': 'application/json',
            },
            'body': '{"access_token": "test_access_token"}'
        }

        # Act
        actual_output = your_module.lambda_handler(mock_event, mock_context)

        # Assert
        self.assertEqual(actual_output, expected_output)

    @patch('os.environ', return_value={'SECRET_KEY_NAME': 'test_secret_key_name', 'CLIENT_ID': 'test_client_id', 'AWS_REGION': 'test_region'})
    @patch('your_module.boto3.session.Session')
    @patch('your_module.requests.post')
    def test_lambda_handler_error(self, requests_post_mock, boto_session_mock, os_environ_mock):
        # Arrange
        mock_event = {
            'body': '{"username": "test_user", "password": "test_password"}'
        }
        mock_context = {}

        # Mocking boto3 secret manager response
        mock_secrets_client = MagicMock()
        boto_session_mock.return_value.client.return_value = mock_secrets_client
        mock_secrets_client.get_secret_value.return_value = {'SecretString': 'test_secret'}

        # Mocking requests post response to throw an exception
        requests_post_mock.side_effect = Exception("Test Exception")

        expected_output = {
            'statusCode': 500,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Credentials': 'true',
                'Content-Type': 'application/json',
            },
            'body': '{}'
        }

        # Act
        actual_output = your_module.lambda_handler(mock_event, mock_context)

        # Assert
        self.assertEqual(actual_output, expected_output)

    @patch('os.environ', return_value={'SECRET_KEY_NAME': 'test_secret_key_name', 'CLIENT_ID': 'test_client_id', 'AWS_REGION': 'test_region'})
    @patch('your_module.boto3.session.Session')
    def test_lambda_handler_client_error(self, boto_session_mock, os_environ_mock):
        # Arrange
        mock_event = {
            'body': '{"username": "test_user", "password": "test_password"}'
        }
        mock_context = {}
        mock_secrets_client = MagicMock()
        boto_session_mock.return_value.client.return_value = mock_secrets_client
        mock_secrets_client.get_secret_value.side_effect = ClientError({"Error": {"Code": "SecretNotFound"}}, "get_secret_value")

        expected_output = {
            'statusCode': 500,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Credentials': 'true',
                'Content-Type': 'application/json',
            },
            'body': '{}'
        }

        # Act
        actual_output = your_module.lambda_handler(mock_event, mock_context)

        # Assert
        self.assertEqual(actual_output, expected_output)

    @patch('os.environ', return_value={'SECRET_KEY_NAME': 'test_secret_key_name', 'CLIENT_ID': 'test_client_id', 'AWS_REGION': 'test_region'})
    def test_lambda_handler_bad_request(self, os_environ_mock):
        # Arrange
        mock_event = {
            'body': 'This is not valid JSON'
        }
        mock_context = {}

        expected_output = {
            'statusCode': 500,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Credentials': 'true',
                'Content-Type': 'application/json',
            },
            'body': '{}'
        }

        # Act
        actual_output = your_module.lambda_handler(mock_event, mock_context)

        # Assert
        self.assertEqual(actual_output, expected_output)

    @patch('os.environ', return_value={'SECRET_KEY_NAME': 'test_secret_key_name', 'CLIENT_ID': 'test_client_id', 'AWS_REGION': 'test_region'})
    def test_lambda_handler_missing_credentials(self, os_environ_mock):
        # Arrange
        mock_event = {
            'body': '{"username": "test_user"}'  # Missing password
        }
        mock_context = {}

        expected_output = {
            'statusCode': 500,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Credentials': 'true',
                'Content-Type': 'application/json',
            },
            'body': '{}'
        }

        # Act
        actual_output = your_module.lambda_handler(mock_event, mock_context)

        # Assert
        self.assertEqual(actual_output, expected_output)

if __name__ == '__main__':
    unittest.main()
```


```
@patch.dict(os.environ, {'SECRET_KEY_NAME': 'test_secret_key_name', 'CLIENT_ID': 'test_client_id', 'AWS_REGION': 'test_region'})

```
```
import boto3
import json
import logging
import requests
import os
from botocore.exceptions import ClientError
from http import HTTPStatus

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def get_secret(client, secret_name):
    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        logger.error(f"Client error retrieving secret {secret_name}, exception {e.response['Error']['Code']}")
        raise e
    else:
        if 'SecretString' in get_secret_value_response:
            return get_secret_value_response['SecretString']
        else:
            raise ValueError(f"Unexpected response from AWS Secrets Manager: {get_secret_value_response}")

def build_response(status_code, body, message=None):
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Credentials': 'true',
        'Content-Type': 'application/json',
    }
    response = {
        'statusCode': status_code,
        'body': body,
        'headers': headers,
    }
    if message is not None:
        response['body']["Message"] = message
    return response

def lambda_handler(event, context):
    try:
        body = json.loads(event.get('body', '{}'))  # default to an empty dict if 'body' is not in event
        username = body.get('username')
        password = body.get('password')

        if not username or not password:
            logger.error('Missing username or password in request body')
            return build_response(HTTPStatus.BAD_REQUEST, {}, 'Missing username or password')

        session = boto3.session.Session()
        secretsmanager_client = session.client(service_name='secretsmanager', region_name=os.environ['AWS_REGION'])
        client_secret = get_secret(secretsmanager_client, os.environ['SECRET_KEY_NAME'])

        token_url = "https://login.microsoftonline.com/organization/oauth2/v2.0/token"
        data = {
            'grant_type': 'password',
            'client_id': os.environ['CLIENT_ID'],
            'client_secret': client_secret,
            'username': username,
            'password': password,
            'scope': "openid"
        }

        response = requests.post(token_url, data=data)
        return build_response(HTTPStatus.OK, json.dumps(response.json()))
    except Exception as e:
        logger.error('Unknown error: {}'.format(str(e)))
        return build_response(HTTPStatus.INTERNAL_SERVER_ERROR, {}, "Internal error")
```

```
import unittest
import your_module
import json

class TestLambdaHandler(unittest.TestCase):

    def test_GET_request_authorized_with_jwt(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'GET',
            'path': '/account-id/{account-id}',
            'resource': '/account-id/{account-id}',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            },
            'pathParameters': {
                'account_number': 'valid_account_number'
            }
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertNotIn('Deny', str(response))
        
    def test_POST_request_authorized_with_jwt(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'POST',
            'path': '/create',
            'resource': '/create',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            },
            'body': json.dumps({
                'account_number': 'valid_account_number'
            })
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertNotIn('Deny', str(response))

    def test_request_authorized_with_secretKey(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'POST',
            'path': '/path',
            'resource': '/resource',
            'headers': {
                'authorization': 'secretKey'
            },
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertNotIn('Deny', str(response))

    def test_request_unauthorized_method(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'PUT',
            'path': '/path',
            'resource': '/resource',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            }
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertIn('Deny', str(response))

    def test_request_invalid_type(self):
        event = {
            'type': 'INVALID',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'GET',
            'path': '/account-id/{account-id}',
            'resource': '/account-id/{account-id}',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            },
            'pathParameters': {
                'account_number': 'valid_account_number'
            }
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertIn('Deny', str(response))

if __name__ == '__main__':
    unittest.main()
 ```
    
    
```
import unittest
from your_module import get_user_transitive_groups

class TestGetUserTransitiveGroups(unittest.TestCase):

    def test_valid_user(self):
        username = "valid_user"
        expected_groups = ['group1', 'group2', 'group3']
        actual_groups = get_user_transitive_groups(username)
        self.assertEqual(actual_groups, expected_groups)

    def test_user_no_groups(self):
        username = "user_no_groups"
        expected_groups = []
        actual_groups = get_user_transitive_groups(username)
        self.assertEqual(actual_groups, expected_groups)

    def test_non_existent_user(self):
        username = "non_existent_user"
        with self.assertRaises(UserDoesNotExistError):
            get_user_transitive_groups(username)

    def test_invalid_input_type(self):
        username = 123456 # this should raise an error since username should be a string
        with self.assertRaises(TypeError):
            get_user_transitive_groups(username)

if __name__ == '__main__':
    unittest.main()
 ```   

```
import unittest
from your_module import filter_user_groups, aws_account_filter

class TestFilterUserGroups(unittest.TestCase):

    def test_valid_groups(self):
        groups = [{'displayName': 'group1'}, {'displayName': 'group2'}]
        target_account = 'targetAccount'
        expected_filtered_groups = ['group1', 'group2']
        actual_filtered_groups = filter_user_groups(groups, target_account)
        self.assertEqual(actual_filtered_groups, expected_filtered_groups)

    def test_no_groups(self):
        groups = []
        target_account = 'targetAccount'
        expected_filtered_groups = []
        actual_filtered_groups = filter_user_groups(groups, target_account)
        self.assertEqual(actual_filtered_groups, expected_filtered_groups)

class TestAwsAccountFilter(unittest.TestCase):

    def test_valid_groups(self):
        groups = {'value': [{'displayName': 'AWS-1234567890-InfraDevops'}, {'displayName': 'AWS-9876543210-InfraDevops'}]}
        expected_accounts = ['1234567890', '9876543210']
        actual_accounts = aws_account_filter(groups)
        self.assertEqual(actual_accounts, expected_accounts)

    def test_no_groups(self):
        groups = {'value': []}
        expected_accounts = []
        actual_accounts = aws_account_filter(groups)
        self.assertEqual(actual_accounts, expected_accounts)

    def test_no_matching_groups(self):
        groups = {'value': [{'displayName': 'non-matching-group1'}, {'displayName': 'non-matching-group2'}]}
        expected_accounts = []
        actual_accounts = aws_account_filter(groups)
        self.assertEqual(actual_accounts, expected_accounts)

if __name__ == '__main__':
    unittest.main()
```    


```

import unittest
import your_module
import json
from unittest.mock import patch
import os

class TestLambdaHandler(unittest.TestCase):

    @patch.dict(os.environ, {
        'REGION': 'us-west-2',
        'LIST_CIDR_TARGET_ADLDS': 'some-value',
        'CREATE_CIDR_TARGET_ADLDS': 'some-value'
    })
    def test_GET_request_authorized_with_jwt(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'GET',
            'path': '/account-id/{account-id}',
            'resource': '/account-id/{account-id}',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            },
            'pathParameters': {
                'account_number': 'valid_account_number'
            }
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertNotIn('Deny', str(response))

    @patch.dict(os.environ, {
        'REGION': 'us-west-2',
        'LIST_CIDR_TARGET_ADLDS': 'some-value',
        'CREATE_CIDR_TARGET_ADLDS': 'some-value'
    })
    def test_POST_request_authorized_with_jwt(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'POST',
            'path': '/create',
            'resource': '/create',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            },
            'body': json.dumps({
                'account_number': 'valid_account_number'
            })
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertNotIn('Deny', str(response))

    @patch.dict(os.environ, {
        'REGION': 'us-west-2',
        'LIST_CIDR_TARGET_ADLDS': 'some-value',
        'CREATE_CIDR_TARGET_ADLDS': 'some-value'
    })
    def test_request_authorized_with_secretKey(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'POST',
            'path': '/path',
            'resource': '/resource',
            'headers': {
                'authorization': 'secretKey'
            },
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertNotIn('Deny', str(response))

    @patch.dict(os.environ, {
        'REGION': 'us-west-2',
        'LIST_CIDR_TARGET_ADLDS': 'some-value',
        'CREATE_CIDR_TARGET_ADLDS': 'some-value'
    })
    def test_request_unauthorized_method(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'PUT',
            'path': '/path',
            'resource': '/resource',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            }
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertIn('Deny', str(response))

    @patch.dict(os.environ, {
        'REGION': 'us-west-2',
        'LIST_CIDR_TARGET_ADLDS': 'some-value',
        'CREATE_CIDR_TARGET_ADLDS': 'some-value'
    })
    def test_request_invalid_type(self):
        event = {
            'type': 'INVALID',
            'requestContext': {
                'requestId': '123'
            },
            'methodArn': 'arn:aws:execute-api:us-west-2:1234567890:apiId/stage/method/resourcePath',
            'httpMethod': 'GET',
            'path': '/account-id/{account-id}',
            'resource': '/account-id/{account-id}',
            'headers': {
                'authorization': 'jwt',
                'authorizationToken': 'valid_token'
            },
            'pathParameters': {
                'account_number': 'valid_account_number'
            }
        }

        context = {}
        response = your_module.lambda_handler(event, context)
        self.assertIn('Deny', str(response))

if __name__ == '__main__':
    unittest.main()

```



```

Certainly! The provided code is a Lambda function that handles authorization and access control for an API Gateway. It uses an IAM policy to determine whether a request should be allowed or denied based on various conditions, including the user's groups and the requested resource and verb. The function integrates with the Microsoft Graph API to retrieve group information and applies filtering based on specific criteria. It supports both JWT and secret key authorization methods. The function builds the IAM policy and returns it as the response, allowing or denying access to the API based on the defined rules.
```


```
Sure! The provided code appears to be a Lambda function that handles authorization and access control for an API Gateway. It uses the AWS Identity and Access Management (IAM) policy to determine whether a request should be allowed or denied based on various conditions.

Here's a breakdown of the code:

The code imports necessary modules, including logging, os, re, json, and requests, and sets up logging configurations.

The HttpVerb class defines constants for different HTTP methods (GET, POST, PUT, etc.) that will be used later.

The AuthPolicy class represents an IAM policy and provides methods to build the policy based on allowed and denied methods.

The get_user_transitive_groups function makes a request to the Microsoft Graph API to retrieve the groups that the user is a member of. It requires an access token for authentication and returns the response in JSON format.

The filter_user_groups function takes the response from get_user_transitive_groups and filters the groups based on certain criteria. It returns the filtered groups in JSON format.

The aws_account_filter function takes the filtered groups and extracts AWS account IDs from their display names using regular expressions. It returns a list of unique AWS account IDs.

The lambda_handler function is the entry point for the Lambda function. It receives an event and context object.

The function checks the type of the event. If it's not a "REQUEST" type, it raises an exception indicating unauthorized access.

The function extracts various information from the event, such as the principal ID, AWS account ID, HTTP verb, path, and resource.

It initializes an instance of the AuthPolicy class with the principal ID and AWS account ID.

Depending on the resource, verb, and authorization method (JWT or secret key), the function determines whether to allow or deny access.

For the "/account-id/{account-id}" resource and GET verb with JWT authorization, it retrieves the user's transitive groups, filters them based on list_cidr_target_adlds and list_cidr_user_permission, and allows or denies access accordingly.

For the "/create" resource and POST verb with JWT authorization, it retrieves the user's transitive groups, filters them based on create_cidr_target_adlds_permission, and allows or denies access. It also calls the aws_account_filter function to extract AWS account IDs for further processing.

If the authorization method is "secretKey", it allows access.

If none of the conditions match, it denies access.

The function builds the IAM policy based on the allowed and denied methods and returns the policy as the response.

If there is an exception during the execution, it logs the error, denies all methods, and returns the policy with the denied statement.

Overall, this code provides a mechanism to authorize API requests based on the user's groups and specific conditions. It integrates with Microsoft Graph API to retrieve group information and implements authorization logic using IAM policies.
```


```
import unittest
from unittest.mock import patch, MagicMock
import boto3
from botocore.stub import Stubber
import json
import os
import your_module  # replace this with the name of your python file

class TestLambdaHandler(unittest.TestCase):

    @patch('your_module.boto3.client')
    def test_lambda_handler_success(self, mock_boto_client):
        os.environ['endpoint_cidr_blocks'] = "mock_endpoint"
        os.environ['cmp_relay_function_arn'] = "mock_arn"

        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda

        expected_response = {
            "result": [
                {
                    "ipv4_subnet_address": "0.0.0.0"
                }
            ]
        }
        
        # Define the mocked response
        mock_lambda.invoke.return_value = {
            'Payload': MagicMock(read=MagicMock(return_value=json.dumps(expected_response).encode('utf-8')))
        }
        
        event = {
            'pathParmeters': {
                'id': "1234"
            }
        }

        response = your_module.lambda_handler(event, None)
        self.assertEqual(json.loads(response['body']), "Your CIDR IP is 0.0.0.0")
    
    @patch('your_module.boto3.client')
    def test_lambda_handler_no_cidr(self, mock_boto_client):
        os.environ['endpoint_cidr_blocks'] = "mock_endpoint"
        os.environ['cmp_relay_function_arn'] = "mock_arn"

        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda

        expected_response = {
            "result": []
        }
        
        mock_lambda.invoke.return_value = {
            'Payload': MagicMock(read=MagicMock(return_value=json.dumps(expected_response).encode('utf-8')))
        }
        
        event = {
            'pathParmeters': {
                'id': "1234"
            }
        }

        response = your_module.lambda_handler(event, None)
        self.assertEqual(json.loads(response['body']), "This CIDR ID does not exist in CMP")
    
    def test_lambda_handler_invalid_trigger(self):
        response = your_module.lambda_handler({}, None)
        self.assertEqual(json.loads(response['body']), "Invalid trigger")


if __name__ == '__main__':
    unittest.main()

```


```
class TestLambdaHandler(unittest.TestCase):

    @patch('your_module.boto3.client')
    def test_lambda_handler_success(self, mock_boto_client):
        os.environ['endpoint_cidr_blocks'] = "mock_endpoint"
        os.environ['cmp_relay_function_arn'] = "mock_arn"

        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda

        expected_response = {
            "result": [
                {
                    "ipv4_subnet_address": "0.0.0.0"
                }
            ]
        }
        
        mock_lambda.invoke.return_value = {
            'Payload': MagicMock(read=MagicMock(return_value=json.dumps(expected_response).encode('utf-8')))
        }
        
        event = {
            'pathParameters': {  # Corrected here
                'id': "1234"
            }
        }

        response = your_module.lambda_handler(event, None)
        self.assertEqual(json.loads(response['body']), "Your CIDR IP is 0.0.0.0")

```

```
1. Start
2. Execute lambda_handler
   - Check if the event type is 'REQUEST', if not raise an exception
   - Extract necessary information from the event (like principalId, awsAccountId, verb, path, resource, etc.)
3. Check the authorization method
   - If 'jwt':
     - If resource is "/account-id/{account-id}" and verb is "GET":
       - Extract the account number and access token
       - Call the get_user_transitive_groups function with the access token
       - Filter user's groups for list_cidr_target_adlds and account number
       - If user has permissions for both, allow the method; else deny all methods
     - If resource is "/create" and verb is "POST":
       - Extract the account number and access token from the body
       - Call the get_user_transitive_groups function with the access token
       - Filter user's groups for create_cidr_target_adlds
       - Filter AWS account ids from the user's groups
       - If user has permissions and there are valid AWS account ids, allow the method; else deny all methods
   - If 'secretKey':
     - Allow the method
   - If neither 'jwt' nor 'secretKey':
     - Deny all methods
4. Build the policy using policy.build() method
5. If there's a payload, add it to the context
6. Return the policy
7. Exception handling
   - If an exception occurs at any point, log the error, deny all methods, build and return the policy
8. End
```



```
import unittest
import json
from unittest.mock import patch, MagicMock
import os
import boto3
from your_module import account_number_dispatcher, cmp_cidr_receiver, lambda_hander  # replace 'your_module' with the actual name of your module

class TestLambdaFunction(unittest.TestCase):

    @patch("boto3.client")
    def test_account_number_dispatcher(self, mock_boto_client):
        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda
        mock_lambda.invoke.return_value = {
            "Payload": MagicMock(read=MagicMock(
                return_value=json.dumps({"body": "mock_response"}).encode("utf-8")
            ))
        }
        account_number = "1234567890"
        response = account_number_dispatcher(account_number)
        self.assertEqual(response, "mock_response")

    @patch("boto3.client")
    def test_cmp_cidr_receiver(self, mock_boto_client):
        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda
        mock_lambda.invoke.return_value = {
            "Payload": MagicMock(read=MagicMock(
                return_value=json.dumps({"result": [{"create_by": "mock", "connected": "mock", "account": "mock", "other": "data"}]}).encode("utf-8")
            ))
        }
        method = "GET"
        endpoint = "mock_endpoint"
        id = "mock_id"
        entries = {"create_by", "connected", "account"}
        response = cmp_cidr_receiver(method, endpoint, id, entries)
        self.assertEqual(response, {"other": "data"})
    
    @patch("your_module.account_number_dispatcher")  # replace 'your_module' with the actual name of your module
    @patch("your_module.cmp_cidr_receiver")  # replace 'your_module' with the actual name of your module
    def test_lambda_handler(self, mock_cmp_cidr_receiver, mock_account_number_dispatcher):
        event = {
            "pathParameters": {
                "account_number": "1234567890"
            }
        }
        context = {}
        mock_account_number_dispatcher.return_value = "mock_account_id"
        mock_cmp_cidr_receiver.return_value = {"other": "data"}
        response = lambda_hander(event, context)
        self.assertEqual(json.loads(response["body"]), {"other": "data"})


if __name__ == "__main__":
    unittest.main()
```


```
import unittest
import json
from unittest.mock import patch, MagicMock
import os
import boto3
from your_module import cmp_account_meta_receiver, lambda_handler  # replace 'your_module' with your module name

class TestLambdaFunction(unittest.TestCase):

    @patch("boto3.client")
    def test_cmp_account_meta_receiver(self, mock_boto_client):
        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda
        mock_lambda.invoke.return_value = {
            "Payload": MagicMock(read=MagicMock(
                return_value=json.dumps({"result": [{"id": "mock_id", "other": "data"}]}).encode("utf-8")
            ))
        }
        method = "GET"
        endpoint = "mock_endpoint"
        account_number = "1234567890"
        response = cmp_account_meta_receiver(method, endpoint, account_number)
        self.assertEqual(response, {"id": "mock_id", "other": "data"})
    
    @patch("your_module.cmp_account_meta_receiver")  # replace 'your_module' with your module name
    def test_lambda_handler(self, mock_cmp_account_meta_receiver):
        event = {
            "pathParameters": {
                "account_number": "1234567890"
            }
        }
        context = {}
        
        # case when cmp_account_meta_receiver returns a dictionary
        mock_cmp_account_meta_receiver.return_value = {"id": "mock_id", "other": "data"}
        response = lambda_handler(event, context)
        self.assertEqual(json.loads(response["body"]), "mock_id")
        
        # case when cmp_account_meta_receiver returns None
        mock_cmp_account_meta_receiver.return_value = None
        response = lambda_handler(event, context)
        self.assertEqual(response["body"], "\"This Account does not exist in CMP\"")
        
        # case when pathParameters is not given in the event
        event = {}
        response = lambda_handler(event, context)
        self.assertEqual(response["body"], "\"Invalid trigger\"")

if __name__ == "__main__":
    unittest.main()
```
```
import unittest
import json
from unittest.mock import patch, MagicMock
import os
import boto3
from botocore.exceptions import BotoCoreError, ClientError
from your_module import cmp_account_meta_receiver, lambda_handler  # replace 'your_module' with your module name

class TestLambdaFunction(unittest.TestCase):

    @patch("boto3.client")
    def test_cmp_account_meta_receiver(self, mock_boto_client):
        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda

        # test a successful invocation
        mock_lambda.invoke.return_value = {
            "Payload": MagicMock(read=MagicMock(
                return_value=json.dumps({"result": [{"id": "mock_id", "other": "data"}]}).encode("utf-8")
            ))
        }
        method = "GET"
        endpoint = "mock_endpoint"
        account_number = "1234567890"
        response = cmp_account_meta_receiver(method, endpoint, account_number)
        self.assertEqual(response, {"id": "mock_id", "other": "data"})

        # test when the lambda function returns no result
        mock_lambda.invoke.return_value = {
            "Payload": MagicMock(read=MagicMock(
                return_value=json.dumps({"result": []}).encode("utf-8")
            ))
        }
        response = cmp_account_meta_receiver(method, endpoint, account_number)
        self.assertIsNone(response)

        # test when the lambda function raises an exception
        mock_lambda.invoke.side_effect = BotoCoreError
        with self.assertRaises(BotoCoreError):
            response = cmp_account_meta_receiver(method, endpoint, account_number)

    @patch("your_module.cmp_account_meta_receiver")  # replace 'your_module' with your module name
    def test_lambda_handler(self, mock_cmp_account_meta_receiver):
        event = {
            "pathParameters": {
                "account_number": "1234567890"
            }
        }
        context = {}

        # case when cmp_account_meta_receiver returns a dictionary
        mock_cmp_account_meta_receiver.return_value = {"id": "mock_id", "other": "data"}
        response = lambda_handler(event, context)
        self.assertEqual(json.loads(response["body"]), "mock_id")
        self.assertEqual(response["statusCode"], 200)
        self.assertEqual(response["headers"]["Content-Type"], "application/json")
        self.assertFalse(response["isBase64Encoded"])

        # case when cmp_account_meta_receiver returns None
        mock_cmp_account_meta_receiver.return_value = None
        response = lambda_handler(event, context)
        self.assertEqual(response["body"], "\"This Account does not exist in CMP\"")

        # case when pathParameters is not given in the event
        event = {}
        response = lambda_handler(event, context)
        self.assertEqual(response["body"], "\"Invalid trigger\"")

if __name__ == "__main__":
    unittest.main()
```
```
import unittest
from unittest.mock import patch, MagicMock
from your_module import lambda_handler  # replace 'your_module' with your module name

class TestLambdaFunction(unittest.TestCase):

    @patch("your_module.cmp_account_meta_receiver")  # replace 'your_module' with your module name
    def test_lambda_handler(self, mock_cmp_account_meta_receiver):
        event = {
            "pathParameters": {
                "account_number": "1234567890"
            }
        }
        context = MagicMock()  # Use MagicMock instead of a plain dictionary

        # case when cmp_account_meta_receiver returns a dictionary
        mock_cmp_account_meta_receiver.return_value = {"id": "mock_id", "other": "data"}
        response = lambda_handler(event, context)
        self.assertEqual(json.loads(response["body"]), "mock_id")
        
        # case when cmp_account_meta_receiver returns None
        mock_cmp_account_meta_receiver.return_value = None
        response = lambda_handler(event, context)
        self.assertEqual(response["body"], "\"This Account does not exist in CMP\"")
        
        # case when pathParameters is not given in the event
        event = {}
        response = lambda_handler(event, context)
        self.assertEqual(response["body"], "\"Invalid trigger\"")

if __name__ == '__main__':
    unittest.main()
```

```
import unittest
from unittest.mock import patch, MagicMock
from your_module import lambda_handler  # replace 'your_module' with your actual module name

class TestLambdaFunction(unittest.TestCase):

    @patch("your_module.cmp_cidr_creator")
    @patch("your_module.account_number_dispatcher")
    @patch("boto3.client")
    def test_lambda_handler(self, mock_boto_client, mock_account_number_dispatcher, mock_cmp_cidr_creator):
        # Create a MagicMock for the 'lambda' client
        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda

        event = {
            "body": json.dumps({"account_number": "1234567890", "region": "us-east-1", "ipv4_subnet_mask": 25}),
            "requestContext": {
                "authorizer": {
                    "payload": ["1234567890"],
                    "administrator_cidr_target_adlds_permission": False
                }
            }
        }

        context = MagicMock()  # Use MagicMock instead of a plain dictionary

        # case when account_number_dispatcher and cmp_cidr_creator returns a valid result
        mock_account_number_dispatcher.return_value = "mock_id"
        mock_cmp_cidr_creator.return_value = {"ipv4_subnet_address": "192.168.0.0", "created": "2023-06-30"}

        response = lambda_handler(event, context)
        expected_response = "This CIDR range 192.168.0.0 is assigned to 1234567890 in us-east-1 at 2023-06-30"
        self.assertEqual(json.loads(response["body"]), expected_response)
        
        # case when input validation fails
        event["body"] = json.dumps({"account_number": "1234567890", "region": "invalid-region", "ipv4_subnet_mask": 25})
        response = lambda_handler(event, context)
        self.assertEqual(json.loads(response["body"]), "Not all required parameters were provided or there was bad data in the request.")

        # case when user is not allowed to perform action
        event["requestContext"]["authorizer"]["payload"] = ["0987654321"]
        response = lambda_handler(event, context)
        self.assertEqual(json.loads(response["body"]), "You are not allowed perform this action")

        # case when body is not given in the event
        event = {"requestContext": {"authorizer": {"payload": ["1234567890"]}}}
        response = lambda_handler(event, context)
        self.assertEqual(json.loads(response["body"]), "Invalid trigger")

if __name__ == '__main__':
    unittest.main()

```

```
import json
import os
import boto3
import logging

ENDPOINT_CIDR = os.environ.get("endpoint_cidr_blocks")
CMP_RELAY_FUNCTION_ARN = os.environ.get("cmp_relay_function_arn")
ACCOUNT_DISPATCH_FUNCTION_ARN = os.environ.get("account_dispatch_function_arn")

def account_number_dispatcher(account_number):
    resp_steam = boto3.client('lambda').invoke(
        FunctionName=ACCOUNT_DISPATCH_FUNCTION_ARN,
        InvocationType='RequestResponse',
        LogType='None',
        Payload=json.dumps({"pathParameters":{
            "account_number":account_number
        }
            
        }), 
    )['Payload']
    resp = json.loads(resp_steam.read().decode("utf-8"))
    body_resp = resp.get("body")

    return body_resp

def input_validation(region, ipv4_subnet_mask):
    if not (region and ipv4_subnet_mask):
        logging.error("An Unspecified error occurred")
        return False
    
    allowed_regions = [
        "eu-west-1",
        "eu-west-2",
        "us-east-1",
        "us-east-2",
        "ap-east-1",
        "ap-southeast-1",
        "ap-south-1",
    ]

    if region not in allowed_regions:
        print("This is not allowed AWS region in HSBC - stopping")
        return False
    if ipv4_subnet_mask < 24:
        print("Request subnet mask is larger than 24, please get approval from architect")
        return False
    elif ipv4_subnet_mask > 26:
        print("Request subnet mask is out of range")
        return False 
    print("This is allowed AWS region in HSBC - proceeding")
    return True

def cmp_cidr_creator(method, endpoint, account_id, region, ipv4_subnet_mask, reference=None, comments=None):

    resp_stream =boto3.client('lambda').invoke(
        FunctionName=CMP_RELAY_FUNCTION_ARN,
        InvocationType='RequestResponse',
        LogType='None',
        Payload=json.dumps({
            "method": method,
            "endpoint": endpoint,
            "body": {
                "account": account_id,
                "region": region,
                "ipv4_subnet_mask": ipv4_subnet_mask,
                "reference": reference,
                "comments": comments
            }
        })
    )['Payload']
    resp = json.load(resp_stream.read().decode("utf-8"))
    return resp

def process_request(body, context):
    account_number = body.get("account_number")
    account_data = account_number_dispatcher(account_number)
    account_id = json.load(account_data).get('id')
    is_live = json.load(account_data).get('is_live')
    if is_live is "1":
        return "This Account has already go live , it does not allow to preform action"
    region = body.get("region")
    ipv4_subnet_mask = body.get("ipv4_subnet_mask")
    reference = body.get("reference")
    comments = body.get("comments")
    if input_validation(region, ipv4_subnet_mask) is True:
        res = cmp_cidr_creator(method, ENDPOINT_CIDR, account_id, region, ipv4_subnet_mask, reference, comments)
        if res is None:
            return "This is not valid request"
        else:
            return "This CIDR range {} is assigned to {} in {} at {}".format(
                res['ipv4_subnet_address'], account_number, region, res['created'])
    else:
        return "Not all required parameters were provided or there was bad data in the request."

def lambda_handler(event, context):

    method = "POST"
    res = None

    if event.get("body") is not None:
        body = json.loads(event.get("body"))
        account_list = event['requestContext'].get('authorizer','{}').get('payload','[]')
        adminstrator_cidr_target_adlds_permission = event['requestContext'].get(
            'authorizer', '{}').get('administrator_cidr_target_adlds_permission')
        if str(account_number) not in account_list and adminstrator_cidr_target_adlds_permission != true:
            res = "You are not allowed perform this action"
        else:
            res = process_request(body, context)
    else:
        res = "Invalid trigger"

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(res)
    }     

```

```
import json
import os
import boto3
import logging

ENDPOINT_CIDR = os.environ.get("endpoint_cidr_blocks")
CMP_RELAY_FUNCTION_ARN = os.environ.get("cmp_relay_function_arn")
ACCOUNT_DISPATCH_FUNCTION_ARN = os.environ.get("account_dispatch_function_arn")

def account_number_dispatcher(account_number):
    resp_stream = boto3.client('lambda').invoke(
        FunctionName=ACCOUNT_DISPATCH_FUNCTION_ARN,
        InvocationType='RequestResponse',
        LogType='None',
        Payload=json.dumps({"pathParameters":{
            "account_number":account_number
        }
        }), 
    )['Payload']
    resp = json.loads(resp_stream.read().decode("utf-8"))
    return resp

def input_validation(region, ipv4_subnet_mask):
    if not (region and ipv4_subnet_mask):
        logging.error("An Unspecified error occurred")
        return False
    
    allowed_regions = [
        "eu-west-1",
        "eu-west-2",
        "us-east-1",
        "us-east-2",
        "ap-east-1",
        "ap-southeast-1",
        "ap-south-1",
    ]

    if region not in allowed_regions:
        print("This is not allowed AWS region in HSBC - stopping")
        return False
    if ipv4_subnet_mask < 24:
        print("Request subnet mask is larger than 24, please get approval from architect")
        return False
    elif ipv4_subnet_mask > 26:
        print("Request subnet mask is out of range")
        return False 
    print("This is allowed AWS region in HSBC - proceeding")
    return True

def cmp_cidr_creator(method, endpoint, account_id, region, ipv4_subnet_mask, reference=None, comments=None):
    resp_stream =boto3.client('lambda').invoke(
        FunctionName=CMP_RELAY_FUNCTION_ARN,
        InvocationType='RequestResponse',
        LogType='None',
        Payload=json.dumps({
            "method": method,
            "endpoint": endpoint,
            "body": {
                "account": account_id,
                "region": region,
                "ipv4_subnet_mask": ipv4_subnet_mask,
                "reference": reference,
                "comments": comments
            }
        })
    )['Payload']
    resp = json.load(resp_stream)
    return resp

def process_request(account_number, body, context):
    account_data = account_number_dispatcher(account_number)
    account_id = account_data.get('id')
    is_live = account_data.get('is_live')
    if is_live is "1":
        return "This Account has already go live , it does not allow to preform action"
    region = body.get("region")
    ipv4_subnet_mask = body.get("ipv4_subnet_mask")
    reference = body.get("reference")
    comments = body.get("comments")
    if input_validation(region, ipv4_subnet_mask) is True:
        res = cmp_cidr_creator("POST", ENDPOINT_CIDR, account_id, region, ipv4_subnet_mask, reference, comments)
        if res is None:
            return "This is not valid request"
        else:
            return "This CIDR range {} is assigned to {} in {} at {}".format(
                res['ipv4_subnet_address'], account_number, region, res['created'])
    else:
        return "Not all required parameters were provided or there was bad data in the request."

def lambda_handler(event, context):
    res = None
    if event.get("body") is not None:
        body = json.loads(event.get("body"))
        account_number = body.get("account_number")
        account_list = event['requestContext'].get('authorizer','{}').get('payload','[]')
        adminstrator_cidr_target_adlds_permission = event['requestContext'].get(
            'authorizer', '{}').get('administrator_cidr_target_adlds_permission')
        if str(account_number) not in account_list and adminstrator_cidr_target_adlds_permission != true:
            res = "You are not allowed perform this action"
        else:
            res = process_request(account_number, body, context)
    else:
        res = "Invalid trigger"

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(res)
    }     

```
```
import unittest
from unittest.mock import patch, Mock
import json
import your_module  # replace with your actual module name

class TestLambdaHandler(unittest.TestCase):
    
    @patch('your_module.boto3.client')
    def test_account_number_dispatcher(self, mock_boto_client):
        mock_response = Mock()
        mock_response.read.return_value = json.dumps({"body": "body_content"}).encode()
        mock_boto_client.return_value.invoke.return_value = {"Payload": mock_response}
        response = your_module.account_number_dispatcher('12345')
        self.assertEqual(response, json.loads('{"body": "body_content"}'))

    def test_input_validation(self):
        self.assertTrue(your_module.input_validation("eu-west-1", 25))
        self.assertFalse(your_module.input_validation("eu-west-1", 23))
        self.assertFalse(your_module.input_validation("invalid-region", 25))
        self.assertFalse(your_module.input_validation("eu-west-1", 27))

    @patch('your_module.boto3.client')
    def test_cmp_cidr_creator(self, mock_boto_client):
        mock_response = Mock()
        mock_response.read.return_value = json.dumps({"ipv4_subnet_address": "192.168.0.0", "created": "2023-06-27"}).encode()
        mock_boto_client.return_value.invoke.return_value = {"Payload": mock_response}
        response = your_module.cmp_cidr_creator("POST", "endpoint", "account_id", "region", "ipv4_subnet_mask")
        self.assertEqual(response, {"ipv4_subnet_address": "192.168.0.0", "created": "2023-06-27"})

    @patch('your_module.boto3.client')
    def test_process_request(self, mock_boto_client):
        mock_response = Mock()
        mock_response.read.return_value = json.dumps({"id": "id_content", "is_live": "0"}).encode()
        mock_boto_client.return_value.invoke.return_value = {"Payload": mock_response}

        account_number = '12345'
        body = {"region": "eu-west-1", "ipv4_subnet_mask": 25, "reference": "test_ref", "comments": "test_comments"}
        context = {}

        with patch('your_module.input_validation', return_value=True), patch('your_module.cmp_cidr_creator', return_value={"ipv4_subnet_address": "192.168.0.0", "created": "2023-06-27"}):
            response = your_module.process_request(account_number, body, context)
            self.assertEqual(response, "This CIDR range 192.168.0.0 is assigned to 12345 in eu-west-1 at 2023-06-27")

    # TODO: You can add more test cases to cover more scenarios. For example, you can test lambda_handler with different event data.


if __name__ == '__main__':
    unittest.main()
```
