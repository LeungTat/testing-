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

    @patch('your_module.boto3.client')
    @patch('your_module.process_request')
    def test_lambda_handler(self, mock_process_request, mock_boto_client):
        mock_process_request.return_value = "Processing successful"
        event = {
            "body": json.dumps({"account_number": "12345", "region": "eu-west-1", "ipv4_subnet_mask": 25, "reference": "test_ref", "comments": "test_comments"}),
            "requestContext": {"authorizer": {"payload": ["12345"], "administrator_cidr_target_adlds_permission": 'true'}}
        }
        context = {}
        response = your_module.lambda_handler(event, context)
        expected_response = {
            "isBase64Encoded": False,
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps("Processing successful")
        }
        self.assertEqual(response, expected_response)


if __name__ == '__main__':
    unittest.main()
```

```
import requests

url = "https://graph.microsoft.com/v1.0/me/transitiveMemberOf/microsoft.graph.group"
headers = {"Authorization": "Bearer YOUR_ACCESS_TOKEN"}

all_groups = []
while url:
    response = requests.get(url, headers=headers)
    data = response.json()
    all_groups.extend(data['value'])
    url = data.get('@odata.nextLink', None)
```

```
def get_user_transitive_groups(access_token):
    url = "https://graph.microsoft.com/v1.0/me/transitiveMemberOf/microsoft.graph.group"
    headers = {
        "Authorization": f"Bearer {access_token}"
    }

    all_groups = []
    while url:
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            data = response.json()
            all_groups.extend(data['value'])
            url = data.get('@odata.nextLink', None)
        elif response.status_code == 401:
            raise Exception("Unauthorized: Invalid or expired access token")
        elif response.status_code == 403:
            raise Exception("Forbidden: Insufficient permissions to access the resource")
        else:
            raise Exception(f"Error: {response.status_code} - {response.text}")

    return all_groups
```
```
This Python code is an AWS Lambda function that uses the AWS Secrets Manager to securely store and retrieve secrets such as credentials for the Microsoft Online OAuth 2.0 API. The lambda_handler function accepts a request with a username and password, retrieves the stored secret (representing the client secret) from AWS Secrets Manager, and then uses these to make an authentication request to Microsoft's OAuth API to get a token. The result (an OAuth token or an error) is then returned in a structured response format.
```
```
This Python script is an AWS Lambda function that's designed to authenticate a user with Microsoft's Azure AD (Active Directory) OAuth service. Here's a breakdown of what it does:

get_secret(secret_name): This function retrieves a secret (in this case, client_secret) from AWS Secrets Manager. Secrets Manager is a service that helps you protect access to your applications, services, and IT resources. This function takes a secret's name as an input and returns the secret's value.

build_response(status_code, body, message=None): This function is a helper function that constructs an HTTP response with a specific status code, body, and an optional message. The response is in JSON format.

lambda_handler(event, context): This is the main function that AWS Lambda calls when the Lambda function is invoked. It attempts to authenticate a user with the Microsoft OAuth service by making a POST request to the token URL. The POST request includes the client ID, the client secret retrieved from AWS Secrets Manager, and the username and password from the event that triggered the function.

The function responds with a JSON object containing the response from the Microsoft OAuth service (which includes an access token if the authentication was successful) or an error message. The status code of the response is 200 if the authentication was successful, and 500 if an error occurred.

Note: This function is using the OAuth 2.0 Resource Owner Password Credentials Grant, which is not recommended for new applications because it includes the user's password in the request. New applications should use other OAuth 2.0 flows that don't require the app to handle the user's password.
```


```
import boto3
import json
import logging
import requests
import os
from botocore.exceptions import ClientError

logger = logging.getLogger()
logger.setLevel(logging.INFO)

secret_key_name = os.environ['SECRET_KEY_NAME']
client_id = os.environ['CLIENT_ID']
aws_region = os.environ['AWS_REGION']

def get_secret(secret_name):
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name=aws_region)

    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        logger.error("Client error retrieving secret {}, exception {}".format(secret_name, e.response['Error']['Code']))
        raise e
    else:
        if 'SecretString' in get_secret_value_response:
            secret = get_secret_value_response['SecretString']
            if secret: 
                return secret
            else:
                raise ValueError(f"Retrieved secret is empty for {secret_name}")
        else:
            raise ValueError('Retrieved value is not a valid JSON string')

def build_response(status_code, body, message=None):
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Credentials': 'true',
        'Content-Type': 'application/json',
    }
    response = {
        'statusCode': status_code,
        'body': json.dumps(body),
        'headers': headers,
    }
    if message is not None:
        response['body'] = json.dumps({"Message": message})
    return response

def lambda_handler(event, context):
    try:
        body = json.loads(event['body'])
        username = body.get('username')
        password = body.get('password')
        if not username or not password:
            return build_response(400, {}, "Missing 'username' or 'password'")
        client_secret = get_secret(secret_key_name)
        token_url = f"https://login.microsoftonline.com/organization/oauth2/v2.0/token"
        data = {
            'grant_type': 'password',
            'client_id': client_id,
            'client_secret': client_secret,
            'username': username,
            'password': password,
            'scope': "openid"
        }
        response = requests.post(token_url, data=data)
        response.raise_for_status()
        return build_response(200, response.json())
    except requests.exceptions.RequestException as e:
        logger.error('Request error: {}'.format(str(e)))
        return build_response(500, {}, "Failed to connect to Microsoft Online")
    except Exception as e:
        logger.error('Unknown error: {}'.format(str(e)))
        return build_response(500, {}, "Internal error")
```
```
def lambda_handler(event, context):
    try:
        body = json.loads(event['body'])
        username = body.get('username')
        password = body.get('password')

        if not username or not password:
            return build_response(400, {}, "Missing 'username' or 'password'")
        client_secret = get_secret(secret_key_name)
        
        token_url = f"https://login.microsoftonline.com/organization/oauth2/v2.0/token"
        data = {
            'grant_type': 'password',
            'client_id': client_id,
            'client_secret': client_secret,
            'username': username,
            'password': password,
            'scope': "openid"
        }
        response = requests.post(token_url, data=data)
        
        # if status_code is not 2xx, treat it as error
        if response.status_code < 200 or response.status_code >= 300:
            logger.error('Unexpected response from Microsoft Online: {}'.format(response.status_code))
            # if it is a client error (4xx), return status code 400
            if 400 <= response.status_code < 500:
                return build_response(400, {}, "Invalid username or password")
            # else if it is a server error (5xx), return status code 500
            else:
                return build_response(500, {}, "Microsoft Online server error")
                
        return build_response(200, response.json())
    except requests.exceptions.RequestException as e:
        logger.error('Request error: {}'.format(str(e)))
        return build_response(500, {}, "Failed to connect to Microsoft Online")
    except Exception as e:
        logger.error('Unknown error: {}'.format(str(e)))
        return build_response(500, {}, "Internal error")
```
```
import boto3
import json
import logging
import requests
import os
from botocore.exceptions import ClientError

logger = logging.getLogger()
logger.setLevel(logging.INFO)

secret_key_name = os.environ['SECRET_KEY_NAME']
client_id = os.environ['CLIENT_ID']
aws_region = os.environ['AWS_REGION']

def get_secret(secret_name):
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name=aws_region)

    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        logger.error("Client error retrieving secret {}, exception {}".format(secret_name, e.response['Error']['Code']))
        raise e
    else:
        if 'SecretString' in get_secret_value_response:
            secret = get_secret_value_response['SecretString']
            if secret: 
                return secret
            else:
                raise ValueError(f"Retrieved secret is empty for {secret_name}")
        else:
            raise ValueError('Retrieved value is not a valid JSON string')

def build_response(status_code, body, message=None):
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Credentials': 'true',
        'Content-Type': 'application/json',
    }
    response = {
        'statusCode': status_code,
        'body': json.dumps(body),
        'headers': headers,
    }
    if message is not None:
        response['body'] = json.dumps({"Message": message})
    return response

def lambda_handler(event, context):
    try:
        body = json.loads(event['body'])
        username = body.get('username')
        password = body.get('password')
        
        if not username or not password:
            return build_response(400, {}, "Missing 'username' or 'password'")
        client_secret = get_secret(secret_key_name)
        
        token_url = f"https://login.microsoftonline.com/organization/oauth2/v2.0/token"
        data = {
            'grant_type': 'password',
            'client_id': client_id,
            'client_secret': client_secret,
            'username': username,
            'password': password,
            'scope': "openid"
        }
        response = requests.post(token_url, data=data)

        if response.status_code >= 400:
            logger.error('Unexpected response from Microsoft Online: {}'.format(response.status_code))
            if response.status_code < 500:
                return build_response(400, {}, "Invalid username or password")
            else:
                return build_response(500, {}, "Microsoft Online server error")

        return build_response(200, response.json())
        
    except requests.exceptions.RequestException as e:
        logger.error('Request error: {}'.format(str(e)))
        return build_response(500, {}, "Failed to connect to Microsoft Online")
    except Exception as e:
        logger.error('Unknown error: {}'.format(str(e)))
        return build_response(500, {}, "Internal error")
```
```
openapi: 3.0.1
info:
  title: Lambda Function API
  description: A simple API for a Lambda function
  version: 1.0.0
servers:
  - url: https://<API-GATEWAY-URL>/
paths:
  /token:
    post:
      summary: Retrieve a token
      operationId: getToken
      requestBody:
        description: The user's credentials
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Credentials'
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  access_token:
                    type: string
                  token_type:
                    type: string
                  expires_in:
                    type: integer
                  scope:
                    type: string
                  id_token:
                    type: string
        '400':
          description: Invalid username or password
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: Microsoft Online server error or internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
components:
  schemas:
    Credentials:
      type: object
      properties:
        username:
          type: string
        password:
          type: string
      required:
        - username
        - password
    Error:
      type: object
      properties:
        Message:
          type: string
      required:
        - Message
```
```
from setuptools import setup, find_packages

setup(
    name='aws_auth_policy',
    version='0.0.1',
    packages=find_packages(),
    install_requires=[
        'requests',
        # Add any additional dependencies your function needs here
    ],
    classifiers=[
        'Development Status :: 3 - Alpha',
        'Intended Audience :: Developers',
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python :: 3.6',
        'Programming Language :: Python :: 3.7',
        'Programming Language :: Python :: 3.8',
        'Programming Language :: Python :: 3.9'
    ],
    keywords='aws, authorizer, lambda, policy'
)
```

```
import re
import requests

class HttpVerb:
    """HTTP verbs for API Gateway"""
    GET     = "GET"
    POST    = "POST"
    PUT     = "PUT"
    PATCH   = "PATCH"
    HEAD    = "HEAD"
    DELETE  = "DELETE"
    OPTIONS = "OPTIONS"
    ALL     = "*"

class AuthPolicy:
    """Handles the creation of an AWS Auth policy"""
    version = "2012-10-17" 
    pathRegex = "^[\/.a-zA-Z0-9-\*:]+$"
    
    def __init__(self, principal, awsAccountId):
        self.awsAccountId = awsAccountId
        self.principalId = principal
        self.allowMethods = []
        self.denyMethods = []
        self.restApiId = "*"
        self.region = "*"
        self.stage = "*"

    def _addMethod(self, effect, verb, resource, conditions):
        """Adds a method to the internal lists of allowed or denied methods."""
        if verb != "*" and not hasattr(HttpVerb, verb):
            raise NameError("Invalid HTTP verb " + verb + ". Allowed verbs in HttpVerb class")

        resourcePattern = re.compile(self.pathRegex)
        if not resourcePattern.match(resource):
            raise NameError("Invalid resource path: " + resource + ". Path should match " + self.pathRegex)

        if resource[:1] == "/":
            resource = resource[1:]

        resourceArn = ("arn:aws:execute-api:" +
            self.region + ":" +
            self.awsAccountId + ":" +
            self.restApiId + "/" +
            self.stage + "/" +
            verb + "/" +
            resource)

        if effect.lower() == "allow":
            self.allowMethods.append({
                'resourceArn' : resourceArn,
                'conditions' : conditions
            })
        elif effect.lower() == "deny":
            self.denyMethods.append({
                'resourceArn' : resourceArn,
                'conditions' : conditions
            })

    def _getEmptyStatement(self, effect):
        """Returns an empty statement object prepopulated with the correct action and the
        desired effect."""
        statement = {
            'Action': 'execute-api:Invoke',
            'Effect': effect[:1].upper() + effect[1:].lower(),
            'Resource': []
        }
        return statement

    def _getStatementForEffect(self, effect, methods):
        """Generates a statement for a particular effect and methods"""
        statements = []
        if len(methods) > 0:
            statement = self._getEmptyStatement(effect)
            for curMethod in methods:
                if curMethod['conditions'] is None or len(curMethod['conditions']) == 0:
                    statement['Resource'].append(curMethod['resourceArn'])
                else:
                    conditionalStatement = self._getEmptyStatement(effect)
                    conditionalStatement['Resource'].append(curMethod['resourceArn'])
                    conditionalStatement['Condition'] = curMethod['conditions']
                    statements.append(conditionalStatement)
            statements.append(statement)
        return statements

    def allowAllMethods(self):
        """Allows all methods"""
        self._addMethod("Allow", HttpVerb.ALL, "*", [])

    def denyAllMethods(self):
        """Denies all methods"""
        self._addMethod("Deny", HttpVerb.ALL, "*", [])

    def allowMethod(self, verb, resource):
        """Allows a particular method"""
        self._addMethod("Allow", verb, resource, [])

    def denyMethod(self, verb, resource):
        """Denies a particular method"""
        self._addMethod("Deny", verb, resource, [])

    def allowMethodWithConditions(self, verb, resource, conditions):
        """Allows a particular method with conditions"""
        self._addMethod("Allow", verb, resource, conditions)

    def denyMethodWithConditions(self, verb, resource, conditions):
        """Denies a particular method with conditions"""
        self._addMethod("Deny", verb, resource, conditions)

    def build(self):
        """Builds the policy with added methods and conditions"""
        if ((self.allowMethods is None or len(self.allowMethods) == 0) and
            (self.denyMethods is None or len(self.denyMethods) == 0)):
            raise NameError("No statements defined for the policy")

        policy = {
            'principalId' : self.principalId,
            'policyDocument' : {
                'Version' : self.version,
                'Statement' : []
            }
        }

        policy['policyDocument']['Statement'].extend(self._getStatementForEffect("Allow", self.allowMethods))
        policy['policyDocument']['Statement'].extend(self._getStatementForEffect("Deny", self.denyMethods))

        return policy

def get_user_transitive_groups(access_token):
    """Function to get user transitive groups from Microsoft Graph API"""
    url = "https://graph.microsoft.com/v1.0/me/transitiveMemberOf/microsoft.graph.group"
    headers = {
        "Authorization": f"Bearer {access_token}"
    }
    all_groups = []
    while url:
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            data = response.json()
            all_groups.extend(data['value'])
            url = data.get('@odata.nextLink', None)
        elif response.status_code == 401:
            raise Exception("Unauthorized: Invalid or expired access token")
        elif response.status_code == 403:
            raise Exception("Forbidden: Insufficient permissions to access the resource")
        else:
            raise Exception(f"Error: {response.status_code} - {response.text}")
    return {"value":all_groups}
    
def filter_user_groups(groups, targetAccount):
    """Function to filter user groups"""
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 401:
        raise Exception("Unauthorized: Invalid or expired access token")
    elif response.status_code == 403:
Apologies for the incomplete response earlier. Here is the corrected and completed code:

```python
import re
import requests

class HttpVerb:
    """HTTP verbs for API Gateway"""
    GET     = "GET"
    POST    = "POST"
    PUT     = "PUT"
    PATCH   = "PATCH"
    HEAD    = "HEAD"
    DELETE  = "DELETE"
    OPTIONS = "OPTIONS"
    ALL     = "*"

class AuthPolicy:
    """Handles the creation of an AWS Auth policy"""
    version = "2012-10-17" 
    pathRegex = "^[\/.a-zA-Z0-9-\*:]+$"
    
    def __init__(self, principal, awsAccountId):
        self.awsAccountId = awsAccountId
        self.principalId = principal
        self.allowMethods = []
        self.denyMethods = []
        self.restApiId = "*"
        self.region = "*"
        self.stage = "*"

    def _addMethod(self, effect, verb, resource, conditions):
        # Implementation of _addMethod ...

    def _getEmptyStatement(self, effect):
        # Implementation of _getEmptyStatement ...

    def _getStatementForEffect(self, effect, methods):
        # Implementation of _getStatementForEffect ...

    def allowAllMethods(self):
        """Allows all methods"""
        self._addMethod("Allow", HttpVerb.ALL, "*", [])

    def denyAllMethods(self):
        """Denies all methods"""
        self._addMethod("Deny", HttpVerb.ALL, "*", [])

    def allowMethod(self, verb, resource):
        """Allows a particular method"""
        self._addMethod("Allow", verb, resource, [])

    def denyMethod(self, verb, resource):
        """Denies a particular method"""
        self._addMethod("Deny", verb, resource, [])

    def allowMethodWithConditions(self, verb, resource, conditions):
        """Allows a particular method with conditions"""
        self._addMethod("Allow", verb, resource, conditions)

    def denyMethodWithConditions(self, verb, resource, conditions):
        """Denies a particular method with conditions"""
        self._addMethod("Deny", verb, resource, conditions)

    def build(self):
        """Builds the policy with added methods and conditions"""
        # Implementation of build ...

def get_user_transitive_groups(access_token):
    """Function to get user transitive groups from Microsoft Graph API"""
    # Implementation of get_user_transitive_groups ...

def filter_user_groups(groups, targetAccount):
    """Function to filter user groups"""
    # Filtering based on the target account
    return [group for group in groups if targetAccount.lower() in group['displayName'].lower()]

def aws_account_filter(groups):
    """Function to filter AWS account IDs from the group names"""
    aws_account_ids = []
    for group in groups:
        if 'AWS-' in group['displayName']:
            aws_account_ids.append(group['displayName'].split('-')[1])
    return aws_account_ids
```

```
import unittest
from auth_policy import AuthPolicy, HttpVerb

class TestAuthPolicy(unittest.TestCase):

    def setUp(self):
        self.policy = AuthPolicy('principalId', 'awsAccountId')

    def test_allow_method(self):
        self.policy.allowMethod(HttpVerb.GET, 'resource')
        self.assertEqual(len(self.policy.allowMethods), 1)

    def test_deny_method(self):
        self.policy.denyMethod(HttpVerb.GET, 'resource')
        self.assertEqual(len(self.policy.denyMethods), 1)

    def test_allow_method_with_conditions(self):
        self.policy.allowMethodWithConditions(HttpVerb.GET, 'resource', {'ipAddress' : {'aws:SourceIp' : ['192.0.2.0/24']}})
        self.assertEqual(len(self.policy.allowMethods), 1)
        self.assertEqual(self.policy.allowMethods[0]['conditions'], {'ipAddress' : {'aws:SourceIp' : ['192.0.2.0/24']}})

    def test_deny_method_with_conditions(self):
        self.policy.denyMethodWithConditions(HttpVerb.GET, 'resource', {'ipAddress' : {'aws:SourceIp' : ['192.0.2.0/24']}})
        self.assertEqual(len(self.policy.denyMethods), 1)
        self.assertEqual(self.policy.denyMethods[0]['conditions'], {'ipAddress' : {'aws:SourceIp' : ['192.0.2.0/24']}})

    def test_allow_all_methods(self):
        self.policy.allowAllMethods()
        self.assertEqual(len(self.policy.allowMethods), 1)
        self.assertEqual(self.policy.allowMethods[0]['resourceArn'].split('/')[-1], '*')

    def test_deny_all_methods(self):
        self.policy.denyAllMethods()
        self.assertEqual(len(self.policy.denyMethods), 1)
        self.assertEqual(self.policy.denyMethods[0]['resourceArn'].split('/')[-1], '*')

    def test_build(self):
        self.policy.allowMethod(HttpVerb.GET, 'resource')
        policy = self.policy.build()
        self.assertEqual(policy['principalId'], 'principalId')
        self.assertEqual(len(policy['policyDocument']['Statement']), 1)

if __name__ == '__main__':
    unittest.main()

```
```
import unittest
from auth_policy import AuthPolicy, HttpVerb

class TestAuthPolicy(unittest.TestCase):

    def setUp(self):
        self.policy = AuthPolicy('principalId', 'awsAccountId')

    def test_invalid_verb(self):
        with self.assertRaises(NameError):
            self.policy.allowMethod('INVALID', 'resource')

    def test_invalid_resource_path(self):
        with self.assertRaises(NameError):
            self.policy.allowMethod(HttpVerb.GET, 'invalid resource path')

    def test_add_method_together(self):
        self.policy.allowMethod(HttpVerb.GET, 'resource1')
        self.policy.allowMethod(HttpVerb.POST, 'resource2')
        self.policy.denyMethod(HttpVerb.GET, 'resource3')
        self.policy.denyMethod(HttpVerb.POST, 'resource4')
        self.assertEqual(len(self.policy.allowMethods), 2)
        self.assertEqual(len(self.policy.denyMethods), 2)

    def test_no_statement_defined(self):
        with self.assertRaises(NameError):
            self.policy.build()

    def test_policy_document(self):
        self.policy.allowMethod(HttpVerb.GET, 'resource')
        policy = self.policy.build()
        self.assertEqual(policy['policyDocument']['Statement'][0]['Effect'], 'Allow')
        self.assertEqual(policy['policyDocument']['Statement'][0]['Resource'][0].split('/')[-1], 'resource')

    def test_build_with_all_methods(self):
        self.policy.allowAllMethods()
        self.policy.denyAllMethods()
        policy = self.policy.build()
        self.assertEqual(policy['policyDocument']['Statement'][0]['Effect'], 'Allow')
        self.assertEqual(policy['policyDocument']['Statement'][0]['Resource'][0].split('/')[-1], '*')
        self.assertEqual(policy['policyDocument']['Statement'][1]['Effect'], 'Deny')
        self.assertEqual(policy['policyDocument']['Statement'][1]['Resource'][0].split('/')[-1], '*')

if __name__ == '__main__':
    unittest.main()
```

```
import unittest
from unittest.mock import patch
from auth_policy import get_user_transitive_groups, filter_user_groups, aws_account_filter

class TestAuthPolicyHelpers(unittest.TestCase):
    
    @patch('auth_policy.requests.get')
    def test_get_user_transitive_groups(self, mock_get):
        mock_response = mock_get.return_value
        mock_response.status_code = 200
        mock_response.json.return_value = {'value': ['group1', 'group2']}
        
        result = get_user_transitive_groups('dummy_access_token')
        self.assertEqual(result, {'value': ['group1', 'group2']})

    @patch('auth_policy.requests.get')
    def test_get_user_transitive_groups_failure(self, mock_get):
        mock_response = mock_get.return_value
        mock_response.status_code = 401
        
        with self.assertRaises(Exception):
            get_user_transitive_groups('dummy_access_token')

    def test_filter_user_groups(self):
        groups = {'value': [{'displayName': 'group1'}, {'displayName': 'group2'}]}
        targetAccount = 'target'
        
        result = filter_user_groups(groups, targetAccount)
        self.assertEqual(result, ['group1', 'group2'])
        
    def test_aws_account_filter(self):
        groups = {'value': [{'displayName': 'AWS-1234-InfraDevops'}, {'displayName': 'AWS-5678-InfraDevops'}]}
        
        result = aws_account_filter(groups)
        self.assertEqual(result, ['1234', '5678'])


if __name__ == '__main__':
    unittest.main()
```

```
AWS Authorizer
Overview
AWS Authorizer is a Python library that assists with the creation of AWS IAM policies and facilitates user authentication and authorization against an Active Directory (AD).

Installation
The AWS Authorizer library can be installed via pip:

bash
Copy code
pip install aws_authorizer
Usage
Here is a sample usage of the AWS Authorizer library:

python
Copy code
from aws_authorizer import auth_policy, utils

# Create a new authorization policy
principal = "User"
awsAccountId = "123456789012"
policy = auth_policy.AuthPolicy(principal, awsAccountId)

# Allow a GET method on the "/pets" resource
policy.allowMethod(auth_policy.HttpVerb.GET, "/pets")

# Build the policy
result = policy.build()

# Fetch user's transitive groups from AD
access_token = "access_token_here"
user_groups = utils.get_user_transitive_groups(access_token)

# Filter the user's groups based on the target account
target_account = "target-account"
filtered_groups = utils.filter_user_groups(user_groups, target_account)

# Extract AWS account IDs from the user's groups
aws_account_ids = utils.aws_account_filter(user_groups)
API Reference
class auth_policy.AuthPolicy(principal: str, awsAccountId: str)
Creates a new authorization policy.

Parameters:

principal: A unique identifier for the end user.
awsAccountId: The AWS account id for which the policy will be generated.
class auth_policy.HttpVerb
This class encapsulates HTTP verbs allowed by AWS. It includes GET, POST, PUT, PATCH, HEAD, DELETE, OPTIONS, and ALL.

utils.get_user_transitive_groups(access_token: str) -> dict
Fetches the transitive groups of a user from the AD.

Parameters:

access_token: The access token used to authenticate against the AD.
utils.filter_user_groups(groups: list, targetAccount: str) -> bool
Filters the user's groups based on the target account.

Parameters:

groups: A list of groups to which the user belongs.
targetAccount: The target account against which to filter the user's groups.
utils.aws_account_filter(groups: list) -> list
Extracts the AWS account IDs from the user's groups.

Parameters:

groups: A list of groups to which the user belongs.
Contributing
Contributions to AWS Authorizer are welcome! To contribute, please fork the repository and submit a pull request. Make sure to write tests for your feature or bug fix, and that all tests pass before submitting the pull request.

License
AWS Authorizer is released under the MIT License.
```


```
import re
from typing import List, Optional, Dict, Union

class HttpVerb:
    GET     = "GET"
    POST    = "POST"
    PUT     = "PUT"
    PATCH   = "PATCH"
    HEAD    = "HEAD"
    DELETE  = "DELETE"
    OPTIONS = "OPTIONS"
    ALL     = "*"

class AuthPolicy:
    awsAccountId: str
    principalId: str
    version: str = "2012-10-17"
    pathRegex: str = "^[\/.a-zA-Z0-9-\*:]+$"
    allowMethods: List[Dict[str, Union[str, List[Dict[str, str]]]]]
    denyMethods: List[Dict[str, Union[str, List[Dict[str, str]]]]]
    restApiId: str = "*"
    region: str = "*"
    stage: str = "*"

    def __init__(self, principal: str, awsAccountId: str):
        self.awsAccountId = awsAccountId
        self.principalId = principal
        self.allowMethods = []
        self.denyMethods = []

    # ... Rest of your code remains unchanged ...

    def _addMethod(self, effect: str, verb: str, resource: str, conditions: List[Dict[str, str]]) -> None:
        # ... Rest of your code remains unchanged ...

    def _getEmptyStatement(self, effect: str) -> Dict[str, Union[str, List]]:
        # ... Rest of your code remains unchanged ...

    def _getStatementForEffect(self, effect: str, methods: List[Dict[str, Union[str, List[Dict[str, str]]]]]) -> List[Dict[str, Union[str, List]]]:
        # ... Rest of your code remains unchanged ...

    def allowAllMethods(self) -> None:
        # ... Rest of your code remains unchanged ...

    def denyAllMethods(self) -> None:
        # ... Rest of your code remains unchanged ...

    def allowMethod(self, verb: str, resource: str) -> None:
        # ... Rest of your code remains unchanged ...

    def denyMethod(self, verb: str, resource: str) -> None:
        # ... Rest of your code remains unchanged ...

    def allowMethodWithConditions(self, verb: str, resource: str, conditions: List[Dict[str, str]]) -> None:
        # ... Rest of your code remains unchanged ...

    def denyMethodWithConditions(self, verb: str, resource: str, conditions: List[Dict[str, str]]) -> None:
        # ... Rest of your code remains unchanged ...

    def build(self) -> Dict[str, Union[str, Dict[str, Union[str, List[Dict[str, Union[str, List]]]]]]]:
        # ... Rest of your code remains unchanged ...

```
```
from typing import Dict, List

class ADUtils:

    @staticmethod
    def get_user_transitive_groups(access_token: str) -> Dict[str, List[Dict]]:
        """Function to get user transitive groups from Active Directory."""
        # ... rest of your code remains unchanged ...

    @staticmethod
    def filter_user_groups(groups: Dict[str, List[Dict]], targetAccount: str) -> bool:
        """Function to filter user groups based on the target account."""
        # ... rest of your code remains unchanged ...

    @staticmethod
    def aws_account_filter(groups: Dict[str, List[Dict]]) -> List[str]:
        """Function to filter AWS account IDs from user groups."""
        # ... rest of your code remains unchanged ...
```

```
# Authorizer Library

This is a Python library to assist with the management of AWS Auth policies and other related functions.

## Classes

### 1. AuthPolicy

This class is used to manage AWS Auth policies. 

#### Attributes:

- `awsAccountId` (str): AWS Account ID.
- `principalId` (str): Principal ID.
- `version` (str): Policy version. Default is '2012-10-17'.
- `pathRegex` (str): Path regex. Default is '^[\/.a-zA-Z0-9-\*:]+$'.
- `allowMethods` (List[Dict[str, Union[str, List[Dict[str, str]]]]]): List of allowed methods.
- `denyMethods` (List[Dict[str, Union[str, List[Dict[str, str]]]]]): List of denied methods.
- `restApiId` (str): REST API ID. Default is '*'.
- `region` (str): Region. Default is '*'.
- `stage` (str): Stage. Default is '*'.

#### Methods:

- `build()` - Build the policy and return it.
- `_addMethod()` - Add a method to the internal lists of allowed or denied methods.
- `allowAllMethods()` - Allows all methods.
- `denyAllMethods()` - Denies all methods.
- `allowMethod()` - Allows a specific method.
- `denyMethod()` - Denies a specific method.

### 2. HttpVerb

This class contains HTTP Verbs constants. 

### 3. ADUtils

This class contains several utility methods for dealing with Azure Active Directory. 

#### Methods:

- `get_user_transitive_groups()` - Returns all the transitive groups a user belongs to.
- `filter_user_groups()` - Filters user groups based on the target account.
- `aws_account_filter()` - Filters the AWS account.

## Installation

```bash
pip install authorizer
```

```
from authorizer import AuthPolicy, HttpVerb, ADUtils

# Create a new AuthPolicy instance
policy = AuthPolicy(principal='example', awsAccountId='1234567890')

# Add methods to the policy
policy.allowMethod(HttpVerb.GET, '/users/*')

# Build the policy
result = policy.build()

# Print the result
print(result)
```
```
from typing import Dict, Any, Tuple
from authorizer import AuthPolicy

class RequestHandler:

    def __init__(self, event: Dict[str, Any]):
        self.event = event
        self.auth_method: str = None
        self.policy: AuthPolicy = None

    def _validate_event_type(self) -> None:
        if self.event['type'] != 'REQUEST':
            raise Exception('Unauthorized')

    def _prepare_auth_method(self) -> None:
        authorization_header = {
            k.lower(): v for k, v in self.event['headers'].items() if k.lower() == 'authorization'}
        self.auth_method = authorization_header.get('authorization')

        if not self.auth_method:
            raise Exception('Unauthorized - No Authorization header provided')

    def _prepare_policy(self) -> None:
        principalId: str = self.event['requestContext']['requestId']
        tmp: List[str] = self.event['methodArn'].split(':')
        apiGatewayArnTmp: List[str] = tmp[5].split('/')
        awsAccountId: str = tmp[4]

        self.policy = AuthPolicy(principalId, awsAccountId)
        self.policy.restApiId = apiGatewayArnTmp[0]
        self.policy.region = tmp[3]
        self.policy.stage = apiGatewayArnTmp[1]

    def handle(self) -> Tuple[AuthPolicy, str]:
        self._validate_event_type()
        self._prepare_auth_method()
        self._prepare_policy()

        return self.policy, self.auth_method
```
```
from typing import Dict, Any, Union
from authorizer import AuthPolicy, ADUtils
import json

class RequestHandler:

    def __init__(self, event: Dict[str, Any]):
        self.event = event
        self.auth_method: str = None
        self.policy: AuthPolicy = None
        self.payload: Union[str, None] = None
        self.ad_utils = ADUtils()

    # ... other methods remain unchanged ...

    def _handle_account_id_resource(self):
        verb = self.event["httpMethod"]
        resource = self.event["resource"]
        account_number = self.event['pathParameters']['account_number']
        access_token = self.event['headers']['authorizationToken']
        user_transitive_groups = self.ad_utils.get_user_transitive_groups(access_token)
        list_cidr_target_adlds_permission = self.ad_utils.filter_user_groups(user_transitive_groups, list_cidr_target_adlds)
        list_cidr_user_permission = self.ad_utils.filter_user_groups(user_transitive_groups, account_number) 
        if list_cidr_target_adlds_permission and list_cidr_user_permission:
            self.policy.allowMethod(verb, resource) 
        else:
            self.policy.denyAllMethods()

    def _handle_create_resource(self):
        verb = self.event["httpMethod"]
        path = self.event["path"]
        body = json.loads(self.event.get('body', '{}'))
        account_number = body.get('account_number')
        access_token = self.event['headers']['authorizationToken']
        user_transitive_groups = self.ad_utils.get_user_transitive_groups(access_token)
        create_cidr_target_adlds_permission = self.ad_utils.filter_user_groups(user_transitive_groups, create_cidr_target_adlds)
        self.payload = self.ad_utils.aws_account_filter(user_transitive_groups)
        if self.payload and create_cidr_target_adlds_permission:
            self.policy.allowMethod(verb, path)
        else:
            self.policy.denyAllMethods()

    def handle_request(self) -> Dict[str, Any]:
        self._validate_event_type()
        self._prepare_auth_method()
        self._prepare_policy()

        resource = self.event["resource"]
        if resource == "/account-id/{account-id}" and self.auth_method == "jwt":
            self._handle_account_id_resource()
        elif resource == "/create" and self.auth_method == "jwt":
            self._handle_create_resource()
        elif self.auth_method == "secretKey":
            self.policy.allowMethod(self.event["httpMethod"], self.event["path"])
        else:
            self.policy.denyAllMethods()
        
        authResponse = self.policy.build()
        if self.payload:
            context = {
                'payload': json.dumps(self.payload)
            }
            authResponse['context'] = context
        return authResponse
```

```
# Authorizer Library

This is a Python library to assist with the management of AWS Auth policies and other related functions.

## Classes

### 1. AuthPolicy

This class is used to manage AWS Auth policies. 

#### Attributes:

- `awsAccountId` (str): AWS Account ID.
- `principalId` (str): Principal ID.
- `version` (str): Policy version. Default is '2012-10-17'.
- `pathRegex` (str): Path regex. Default is '^[\/.a-zA-Z0-9-\*:]+$'.
- `allowMethods` (List[Dict[str, Union[str, List[Dict[str, str]]]]]): List of allowed methods.
- `denyMethods` (List[Dict[str, Union[str, List[Dict[str, str]]]]]): List of denied methods.
- `restApiId` (str): REST API ID. Default is '*'.
- `region` (str): Region. Default is '*'.
- `stage` (str): Stage. Default is '*'.

#### Methods:

- `build()` - Build the policy and return it.
- `_addMethod()` - Add a method to the internal lists of allowed or denied methods.
- `allowAllMethods()` - Allows all methods.
- `denyAllMethods()` - Denies all methods.
- `allowMethod()` - Allows a specific method.
- `denyMethod()` - Denies a specific method.

### 2. HttpVerb

This class contains HTTP Verbs constants. 

### 3. ADUtils

This class contains several utility methods for dealing with Azure Active Directory. 

#### Methods:

- `get_user_transitive_groups()` - Returns all the transitive groups a user belongs to.
- `filter_user_groups()` - Filters user groups based on the target account.
- `aws_account_filter()` - Filters the AWS account.

## Installation

```bash
pip install authorizer
Usage
python
Copy code
from authorizer import AuthPolicy, HttpVerb, ADUtils

# Create a new AuthPolicy instance
policy = AuthPolicy(principal='example', awsAccountId='1234567890')

# Add methods to the policy
policy.allowMethod(HttpVerb.GET, '/users/*')

# Build the policy
result = policy.build()

# Print the result
print(result)
Testing
This library includes a set of unit tests. To run the tests, use the following command:

bash
Copy code
python -m unittest test_auth_policy.py
arduino
Copy code
Please modify this template to fit your specific needs and circumstances.
```
```
from typing import Dict, Any

def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    try:
        handler = RequestHandler(event)
        return handler.handle_request()
    except Exception as e:
        logger.error(e)
        handler.policy.denyAllMethods()
        authResponse = handler.policy.build()
        return authResponse
```
```
from typing import Dict, Any
from authorizer import AuthPolicy

class EventProcessor:

    def __init__(self, event: Dict[str, Any]):
        if event['type'] != 'REQUEST':
            raise Exception('Unauthorized')

        principalId = event['requestContext']['requestId']
        tmp = event['methodArn'].split(':')
        apiGatewayArnTmp = tmp[5].split('/')
        awsAccountId = tmp[4]
        verb = event["httpMethod"]
        path = event["path"]
        resource = event["resource"]
        
        policy = AuthPolicy(principalId, awsAccountId)
        policy.restApiId = apiGatewayArnTmp[0]
        policy.region = tmp[3]
        policy.stage = apiGatewayArnTmp[1]

        authorization_header = {
            k.lower(): v for k, v in event['headers'].items() if k.lower() == 'authorization'}
        auth_method = authorization_header['authorization']

        self.policy = policy
        self.verb = verb
        self.path = path
        self.resource = resource
        self.auth_method = auth_method

    def get_policy(self) -> AuthPolicy:
        return self.policy

    def get_verb(self) -> str:
        return self.verb

    def get_path(self) -> str:
        return self.path

    def get_resource(self) -> str:
        return self.resource

    def get_auth_method(self) -> str:
        return self.auth_method
```
```
def lambda_handler(event, context):
    processor = EventProcessor(event)
    policy = processor.get_policy()
    verb = processor.get_verb()
    path = processor.get_path()
    resource = processor.get_resource()
    auth_method = processor.get_auth_method()
    ```
```
Yes, you can refactor the decision logic based on the resource and the auth method using the "Dispatch Table" pattern. Dispatch tables are dictionaries that map a key to a function, allowing you to replace if/else statements with dictionary lookups.

Here's a refactor of the handle_request method using a dispatch table:

python
Copy code
from typing import Dict, Any, Union
from authorizer import AuthPolicy, ADUtils
import json

class RequestHandler:

    def __init__(self, event: Dict[str, Any]):
        self.event = event
        self.auth_method: str = None
        self.policy: AuthPolicy = None
        self.payload: Union[str, None] = None
        self.ad_utils = ADUtils()
        self.resource_map = {
            ("/account-id/{account-id}", "jwt"): self._handle_account_id_resource,
            ("/create", "jwt"): self._handle_create_resource,
            (None, "secretKey"): self._allow_all
        }

    # ... other methods remain unchanged ...

    def _handle_account_id_resource(self):
        # code goes here

    def _handle_create_resource(self):
        # code goes here

    def _allow_all(self):
        self.policy.allowMethod(self.event["httpMethod"], self.event["path"])

    def handle_request(self) -> Dict[str, Any]:
        self._validate_event_type()
        self._prepare_auth_method()
        self._prepare_policy()

        key = (self.event["resource"], self.auth_method)
        handler = self.resource_map.get(key, self.policy.denyAllMethods)
        handler()

        authResponse = self.policy.build()
        if self.payload:
            context = {
                'payload': json.dumps(self.payload)
            }
            authResponse['context'] = context
        return authResponse
        ```
```
from typing import Dict, Any
from authorizer import AuthPolicy

class EventProcessor:

    def __init__(self, event: Dict[str, Any]) -> None:
        if event['type'] != 'REQUEST':
            raise Exception('Unauthorized')

        principalId: str = event['requestContext']['requestId']
        tmp: List[str] = event['methodArn'].split(':')
        apiGatewayArnTmp: List[str] = tmp[5].split('/')
        awsAccountId: str = tmp[4]
        verb: str = event["httpMethod"]
        path: str = event["path"]
        resource: str = event["resource"]
        
        policy: AuthPolicy = AuthPolicy(principalId, awsAccountId)
        policy.restApiId = apiGatewayArnTmp[0]
        policy.region = tmp[3]
        policy.stage = apiGatewayArnTmp[1]

        authorization_header: Dict[str, str] = {
            k.lower(): v for k, v in event['headers'].items() if k.lower() == 'authorization'}
        auth_method: str = authorization_header['authorization']

        self.policy: AuthPolicy = policy
        self.verb: str = verb
        self.path: str = path
        self.resource: str = resource
        self.auth_method: str = auth_method

    def get_policy(self) -> AuthPolicy:
        return self.policy

    def get_verb(self) -> str:
        return self.verb

    def get_path(self) -> str:
        return self.path

    def get_resource(self) -> str:
        return self.resource

    def get_auth_method(self) -> str:
        return self.auth_method
```
```
from typing import Dict, Any, Optional
from aws_event_processor import EventProcessor

def lambda_handler(event: Dict[str, Any], context: Optional[Any]) -> Dict[str, Any]:
    processor: EventProcessor = EventProcessor(event)
    policy: AuthPolicy = processor.get_policy()
    verb: str = processor.get_verb()
    path: str = processor.get_path()
    resource: str = processor.get_resource()
    auth_method: str = processor.get_auth_method()
    # ... remaining lambda function code ...
```
```
import json
from typing import Dict, Any, Callable, Optional

class PolicyDispatcher:
    def __init__(self, policy, event):
        self.policy = policy
        self.event = event
        self.account_number = None
        self.access_token = None
        self.user_transitive_groups = None
        self.dispatch_table = {
            ("/account-id/{account-id}", "GET", "jwt"): self.process_account_id,
            ("/create", "POST", "jwt"): self.process_create,
            # You can add more tuples and associated methods as needed
        }

    def process_account_id(self):
        self.account_number = self.event['pathParameters']['account_number']
        self.access_token = self.event['headers']['authorizationToken']
        self.user_transitive_groups = get_user_transitive_groups(self.access_token)
        list_cidr_target_adlds_permission = filter_user_groups(self.user_transitive_groups, list_cidr_target_adlds)
        list_cidr_user_permission = filter_user_groups(self.user_transitive_groups, self.account_number) 
        if list_cidr_target_adlds_permission and list_cidr_user_permission:
            self.policy.allowMethod(self.event["httpMethod"], self.event["resource"]) 
        else:
            self.policy.denyAllMethods()

    def process_create(self):
        body = json.loads(self.event.get('body', '{}'))
        self.account_number = body.get('account_number')
        self.access_token = self.event['headers']['authorizationToken']
        self.user_transitive_groups = get_user_transitive_groups(self.access_token)
        create_cidr_target_adlds_permission = filter_user_groups(self.user_transitive_groups, create_cidr_target_adlds)
        payload = aws_account_filter(self.user_transitive_groups)
        if payload and create_cidr_target_adlds_permission:
            self.policy.allowMethod(self.event["httpMethod"], self.event["path"])
        else:
            self.policy.denyAllMethods()

    def dispatch(self, resource: str, verb: str, auth_method: str):
        function: Optional[Callable[[], None]] = self.dispatch_table.get((resource, verb, auth_method))
        if function:
            function()
        elif auth_method == "secretKey":
            self.policy.allowMethod(verb, self.event["path"])
        else:
            self.policy.denyAllMethods()

```
```
def lambda_handler(event, context):
    try:
        processor = EventProcessor(event)
        policy = processor.get_policy()
        dispatcher = PolicyDispatcher(policy, event)
        dispatcher.dispatch(processor.get_resource(), processor.get_verb(), processor.get_auth_method())

        authResponse = policy.build()
        if 'payload' in locals():
            context = {
                'payload': json.dumps(payload)
            }
            authResponse['context'] = context
        return authResponse
    except Exception as e:
        logger.error(e)
        policy.denyAllMethods()
        authResponse = policy.build()
        return authResponse
```

```
import json
from typing import Dict, Any, Callable, Optional, List, Union
# other imports ...

class PolicyDispatcher:
    def __init__(self, policy: "AuthPolicy", event: Dict[str, Any]):
        self.policy = policy
        self.event = event
        self.account_number: Optional[str] = None
        self.access_token: Optional[str] = None
        self.user_transitive_groups: Optional[List[str]] = None
        self.dispatch_table: Dict[Tuple[str, str, str], Callable[[], None]] = {
            ("/account-id/{account-id}", "GET", "jwt"): self.process_account_id,
            ("/create", "POST", "jwt"): self.process_create,
            # You can add more tuples and associated methods as needed
        }

    def process_account_id(self) -> None:
        self.account_number = self.event['pathParameters']['account_number']
        self.access_token = self.event['headers']['authorizationToken']
        self.user_transitive_groups = get_user_transitive_groups(self.access_token)
        list_cidr_target_adlds_permission = filter_user_groups(self.user_transitive_groups, list_cidr_target_adlds)
        list_cidr_user_permission = filter_user_groups(self.user_transitive_groups, self.account_number) 
        if list_cidr_target_adlds_permission and list_cidr_user_permission:
            self.policy.allowMethod(self.event["httpMethod"], self.event["resource"]) 
        else:
            self.policy.denyAllMethods()

    def process_create(self) -> None:
        body: Dict[str, Any] = json.loads(self.event.get('body', '{}'))
        self.account_number = body.get('account_number')
        self.access_token = self.event['headers']['authorizationToken']
        self.user_transitive_groups = get_user_transitive_groups(self.access_token)
        create_cidr_target_adlds_permission = filter_user_groups(self.user_transitive_groups, create_cidr_target_adlds)
        payload = aws_account_filter(self.user_transitive_groups)
        if payload and create_cidr_target_adlds_permission:
            self.policy.allowMethod(self.event["httpMethod"], self.event["path"])
        else:
            self.policy.denyAllMethods()

    def dispatch(self, resource: str, verb: str, auth_method: str) -> None:
        function: Optional[Callable[[], None]] = self.dispatch_table.get((resource, verb, auth_method))
        if function:
            function()
        elif auth_method == "secretKey":
            self.policy.allowMethod(verb, self.event["path"])
        else:
            self.policy.denyAllMethods()

def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Union[str, List[str], Dict[str, Any]]]:
    try:
        processor: EventProcessor = EventProcessor(event)
        policy: "AuthPolicy" = processor.get_policy()
        dispatcher: PolicyDispatcher = PolicyDispatcher(policy, event)
        dispatcher.dispatch(processor.get_resource(), processor.get_verb(), processor.get_auth_method())

        authResponse: Dict[str, Union[str, List[str], Dict[str, Any]]] = policy.build()
        if 'payload' in locals():
            context: Dict[str, str] = {
                'payload': json.dumps(payload)
            }
            authResponse['context'] = context
        return authResponse
    except Exception as e:
        logger.error(e)
        policy.denyAllMethods()
        authResponse = policy.build()
        return authResponse
```
```
EventProcessor Class Documentation
Overview
The EventProcessor class is designed to process AWS Lambda events, specifically those of type 'REQUEST'. It extracts and manages various details from the incoming event, such as the HTTP verb, path, resource, and authorization method.

Attributes
policy: An instance of the AuthPolicy class containing information about the event's policy.
verb: A string representing the HTTP verb (e.g., GET, POST) of the request.
path: A string containing the path of the request.
resource: A string describing the resource accessed by the request.
auth_method: A string representing the authorization method used in the request.
Methods
__init__(self, event: Dict[str, Any]) -> None
Constructor method. Initializes an EventProcessor object based on the provided event.

Parameters:

event: A dictionary containing details of the AWS Lambda event.
Raises:

Exception: If the event type is not 'REQUEST'.
get_policy(self) -> AuthPolicy
Returns the policy related to the event.

Returns:

An AuthPolicy object.
get_verb(self) -> str
Returns the HTTP verb of the event.

Returns:

A string representing the HTTP verb.
get_path(self) -> str
Returns the path of the event.

Returns:

A string representing the path.
get_resource(self) -> str
Returns the resource accessed in the event.

Returns:

A string representing the resource.
get_auth_method(self) -> str
Returns the authorization method used in the event.

Returns:

A string representing the authorization method.
```
```
import unittest
from unittest.mock import patch, Mock

# Assuming the original class and methods are in a module named `authorizer`
from authorizer import PolicyDispatcher, EventProcessor

class TestPolicyDispatcher(unittest.TestCase):

    def setUp(self):
        self.sample_event = {
            "type": "REQUEST",
            "requestContext": {
                "requestId": "sample_id"
            },
            "methodArn": "arn:aws:execute-api:us-west-1:123456789012:s0meId/prod/GET/",
            "httpMethod": "GET",
            "path": "/sample-path",
            "resource": "/sample-resource",
            "headers": {
                "authorization": "jwt"
            }
        }

    @patch("authorizer.get_user_transitive_groups")
    @patch("authorizer.filter_user_groups")
    def test_process_account_id(self, mock_filter_user_groups, mock_get_user_transitive_groups):
        # Mock the external calls
        mock_get_user_transitive_groups.return_value = ["group1", "group2"]
        mock_filter_user_groups.return_value = True

        processor = EventProcessor(self.sample_event)
        policy = Mock()  # Create a mock policy
        dispatcher = PolicyDispatcher(policy, self.sample_event)
        dispatcher.process_account_id()

        policy.allowMethod.assert_called_once()  # Assert that allowMethod was called once

    @patch("authorizer.get_user_transitive_groups")
    @patch("authorizer.filter_user_groups")
    @patch("authorizer.aws_account_filter")
    def test_process_create(self, mock_aws_account_filter, mock_filter_user_groups, mock_get_user_transitive_groups):
        # Mock the external calls
        mock_get_user_transitive_groups.return_value = ["group1", "group2"]
        mock_filter_user_groups.return_value = True
        mock_aws_account_filter.return_value = True

        self.sample_event['httpMethod'] = 'POST'
        self.sample_event['path'] = '/create'
        self.sample_event['body'] = json.dumps({"account_number": "1234567890"})

        processor = EventProcessor(self.sample_event)
        policy = Mock()  # Create a mock policy
        dispatcher = PolicyDispatcher(policy, self.sample_event)
        dispatcher.process_create()

        policy.allowMethod.assert_called_once()  # Assert that allowMethod was called once

if __name__ == '__main__':
    unittest.main()

```
```
import unittest
from your_module_where_EventProcessor_is import EventProcessor

class TestEventProcessor(unittest.TestCase):

    def test_event_processor_normal_case(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {'requestId': 'some_request_id'},
            'methodArn': 'some:arn:info:with:accountId',
            'httpMethod': 'GET',
            'path': '/some/path',
            'resource': '/some/resource',
            'headers': {'authorization': 'some_authorization_token'}
        }

        processor = EventProcessor(event)

        self.assertEqual(processor.get_policy().principalId, 'some_request_id')
        self.assertEqual(processor.get_verb(), 'GET')
        self.assertEqual(processor.get_path(), '/some/path')
        self.assertEqual(processor.get_resource(), '/some/resource')
        self.assertEqual(processor.get_auth_method(), 'some_authorization_token')

    def test_event_processor_unauthorized_type(self):
        event = {
            'type': 'SOME_OTHER_TYPE'
            # other fields omitted for brevity
        }

        with self.assertRaises(Exception):
            EventProcessor(event)

    def test_event_processor_missing_authorization(self):
        event = {
            'type': 'REQUEST',
            'requestContext': {'requestId': 'some_request_id'},
            'methodArn': 'some:arn:info:with:accountId',
            'httpMethod': 'GET',
            'path': '/some/path',
            'resource': '/some/resource',
            # No headers field in this event
        }

        processor = EventProcessor(event)

        self.assertEqual(processor.get_auth_method(), 'unknown')

if __name__ == '__main__':
    unittest.main()
```
```
import json
import pytest
from unittest.mock import patch, Mock
from your_lambda_file import lambda_handler  # Replace 'your_lambda_file' with the actual file name

@pytest.fixture
def mock_env_variables(monkeypatch):
    monkeypatch.setenv('endpoint_cidr_blocks', 'some_value')
    monkeypatch.setenv('cmp_relay_function_arn', 'some_value')
    monkeypatch.setenv('account_dispatch_function_arn', 'some_value')

@pytest.fixture
def mock_event():
    return {
        "body": json.dumps({"account_number": "1234", "region": "eu-west-1", "ipv4_subnet_mask": 25}),
        "requestContext": {
            "authorizer": {
                "payload": ["1234"],
                "administrator_cidr_target_adlds_permission": "false"
            }
        }
    }

@pytest.fixture
def mock_context():
    return Mock()

@patch('your_lambda_file.boto3.client')
def test_lambda_handler_success(mock_boto_client, mock_event, mock_context, mock_env_variables):
    mock_lambda = Mock()
    mock_lambda.invoke.return_value = {'Payload': Mock(read=lambda: json.dumps({"some_key": "some_value"}).encode("utf-8"))}
    mock_boto_client.return_value = mock_lambda

    response = lambda_handler(mock_event, mock_context)

    assert response["statusCode"] == 200
    assert "This CIDR range" in json.loads(response["body"])

@patch('your_lambda_file.boto3.client')
def test_lambda_handler_invalid_trigger(mock_boto_client, mock_event, mock_context, mock_env_variables):
    mock_event['body'] = None

    response = lambda_handler(mock_event, mock_context)

    assert response["statusCode"] == 500
    assert "Invalid trigger" in json.loads(response["body"])

@patch('your_lambda_file.boto3.client')
def test_lambda_handler_not_allowed_action(mock_boto_client, mock_event, mock_context, mock_env_variables):
    mock_event['requestContext']['authorizer']['payload'] = []

    response = lambda_handler(mock_event, mock_context)

    assert response["statusCode"] == 400
    assert "You are not allowed perform this action" in json.loads(response["body"])

@patch('your_lambda_file.boto3.client')
def test_lambda_handler_invalid_region(mock_boto_client, mock_event, mock_context, mock_env_variables):
    mock_event['body'] = json.dumps({"account_number": "1234", "region": "invalid-region", "ipv4_subnet_mask": 25})
    
    response = lambda_handler(mock_event, mock_context)

    assert response["statusCode"] == 400
    assert "This is not allowed AWS region" in json.loads(response["body"])

@patch('your_lambda_file.boto3.client')
def test_lambda_handler_invalid_subnet_mask(mock_boto_client, mock_event, mock_context, mock_env_variables):
    mock_event['body'] = json.dumps({"account_number": "1234", "region": "eu-west-1", "ipv4_subnet_mask": 23})
    
    response = lambda_handler(mock_event, mock_context)

    assert response["statusCode"] == 400
    assert "Request subnet mask is larger than 24" in json.loads(response["body"])

@patch('your_lambda_file.boto3.client')
def test_lambda_handler_account_live(mock_boto_client, mock_event, mock_context, mock_env_variables):
    mock_lambda = Mock()
    mock_lambda.invoke.return_value = {'Payload': Mock(read=lambda: json.dumps({"id": "5678", "is_live": "1"}).encode("utf-8"))}
    mock_boto_client.return_value = mock_lambda

    response = lambda_handler(mock_event, mock_context)

    assert response["statusCode"] == 400
    assert "This Account has already gone live" in json.loads(response["body"])

@patch('your_lambda_file.boto3.client')
def test_lambda_handler_unexpected_lambda_response(mock_boto_client, mock_event, mock_context, mock_env_variables):
    mock_lambda = Mock()
    mock_lambda.invoke.return_value = {'Payload': Mock(read=lambda: "not a json string".encode("utf-8"))}
    mock_boto_client.return_value = mock_lambda
    
    response = lambda_handler(mock_event, mock_context)
    
    assert response["statusCode"] == 400
    assert "This is not a valid request" in json.loads(response["body"])
```
```
from unittest.mock import patch, MagicMock, Mock
import unittest
import json
from your_module import lambda_handler  # Replace 'your_module' with the actual module name

class TestLambdaFunction(unittest.TestCase):

    @patch("your_module.input_validation")
    @patch("your_module.cmp_cidr_reciever")
    @patch("your_module.cmp_cidr_creator")
    @patch("your_module.account_number_dispatcher")
    @patch("boto3.client")
    def test_lambda_handler_one_cidr_per_region(self, mock_boto_client, mock_account_number_dispatcher, mock_cmp_cidr_creator, mock_cmp_cidr_reciever, mock_input_validation):
        # Setup mock for boto3 client
        mock_lambda = MagicMock()
        mock_boto_client.return_value = mock_lambda
        mock_lambda.invoke.return_value = {'Payload': MagicMock(read=lambda: json.dumps({"some_key": "some_value"}).encode("utf-8"))}

        # Mock the return values of internal functions
        mock_account_number_dispatcher.return_value = json.dumps({"id": "some_id", "is_live": "0"})
        mock_cmp_cidr_reciever.return_value = [{"region": "us-east-1"}]  # The region already exists
        mock_input_validation.return_value = True  # Input validation passes

        # Prepare the event payload
        event = {
            "body": json.dumps({
                "account_number": "1234567890",
                "region": "us-east-1",
                "ipv4_subnet_mask": 25
            }),
            "requestContext": {
                "authorizer": {
                    "payload": ["1234567890"],
                    "administrator_cidr_target_adlds_permission": "false"
                }
            }
        }

        context = MagicMock()

        # Invoke the lambda function and check the response
        response = lambda_handler(event, context)

        self.assertEqual(response["statusCode"], 400)
        expected_body = "Only one CIDR is allowed per region"
        self.assertEqual(json.loads(response["body"]), expected_body)

    # ... (your other test cases)

if __name__ == '__main__':
    unittest.main()
```
```
import unittest
from unittest.mock import patch, Mock, MagicMock
import json
from your_module import lambda_hander  # Replace 'your_module' with the actual module name

class TestLambdaHandler(unittest.TestCase):

    @patch('your_module.boto3.client')
    def test_lambda_handler_success(self, mock_boto_client):
        mock_lambda = Mock()
        mock_lambda.invoke.return_value = {'Payload': Mock(read=lambda: json.dumps({"body": "some_value"}).encode("utf-8"))}
        mock_boto_client.return_value = mock_lambda
        
        event = {
            "pathParameters": {"account_number": "123456"},
        }
        context = MagicMock()

        response = lambda_hander(event, context)

        self.assertEqual(response["statusCode"], 200)
        self.assertEqual(json.loads(response["body"]), {"body": "some_value"})

    @patch('your_module.boto3.client')
    def test_lambda_handler_cidr_not_exist(self, mock_boto_client):
        mock_lambda = Mock()
        mock_lambda.invoke.return_value = {'Payload': Mock(read=lambda: json.dumps({"result": None}).encode("utf-8"))}
        mock_boto_client.return_value = mock_lambda

        event = {
            "pathParameters": {"account_number": "123456"},
        }
        context = MagicMock()

        response = lambda_hander(event, context)

        self.assertEqual(response["statusCode"], 400)
        self.assertEqual(json.loads(response["body"]), "This CIDR ID does not exist in CMP")

    def test_lambda_handler_invalid_trigger(self):
        event = {}
        context = MagicMock()

        response = lambda_hander(event, context)

        self.assertEqual(response["statusCode"], 500)
        self.assertEqual(json.loads(response["body"]), "Invalid trigger")

if __name__ == '__main__':
    unittest.main()

```


```
Sure, let's restructure your code to make the Test stage in the codepipeline_ci module conditional based on the var.enviroment.

First, let's define a local variable to decide whether to include the Test stage:
hcl
Copy code
locals {
  include_test_stage = contains(["dev", "pre-prod"], var.enviroment) ? true : false
}
Then, adjust the module codepipeline_ci to conditionally include the Test stage using the local variable:
Here's your updated code:

hcl
Copy code
locals {
  include_test_stage = contains(["dev", "pre-prod"], var.enviroment) ? true : false
}

module "codepipeline_ci" {
  count = var.enviroment == "prod" ? 0 : 1
  source = "../modules/codepipeline"
  name = "${var.pipeline_name}-ci"
  source_provider = "S3"
  source_config_s3 ={
    S3Bucket = local.sources_bucket_name
    S3ObjectKey = var.pipline_ci_source
    PollForSourceChanges = true
  }
  artificats_bucket_name = local.artifacts_bucket_name
  
  free_stages = local.include_test_stage ? [
    {
      name = "Test"
      actions = [
        {
          name = var.pipline_stage_settings.lint_unit_test.name
          category = var.pipline_stage_settings.lint_unit_test.category
          owner = "AWS"
          provider = "CodeBuild"
          namespace = var.pipline_stage_settings.lint_unit_test.namespace
          input_artifacts = var.pipline_stage_settings.lint_unit_test.input_artifacts
          output_artifacts = var.pipline_stage_settings.lint_unit_test.output_artifacts
          version = "1"
          configuration = merge(
            {
              ProjectName = module.codebuild_lint_unit_test.codebuild_project_name
            },
            local.formatted_pipline_stages_extra_config.lint_unit_test
          )
        }
      ]
    }
  ] : []
}
In the adjusted code, I introduced the locals block to determine whether the Test stage should be included. Then, in the codepipeline_ci module, I used the include_test_stage local variable to conditionally set the free_stages attribute.

Remember to test these changes thoroughly before applying them in your environment.
```

```
If you want to run the Test stage in the dev environment only and the Scan stage in the prod environment only, you can adjust the codepipline_cd module to conditionally add stages based on the var.enviroment value.

Here's how:

Define two local variables, one for including the Test stage and the other for including the Scan stage:
hcl
Copy code
locals {
  include_test_stage = var.enviroment == "dev" ? true : false
  include_scan_stage = var.enviroment == "prod" ? true : false
}
Modify the codepipline_cd module to conditionally include the Test and Scan stages:
hcl
Copy code
module "codepipline_cd" {
  # ... (rest of the module code)

  free_stages = concat(
    local.include_test_stage ? [{
      name    = "Test"
      actions = [
        {
          # ... (rest of the Test action details)
        }
      ]
    }] : [],

    local.include_scan_stage ? [{
      name    = "Scan"
      actions = [
        {
          # ... (rest of the Scan action details)
        }
      ]
    }] : []
  )
}
```

```
"Scheduling conflicts with other changes for this service have been reviewed and are not an issue. Enabling the Service Control Policy (SCP) on the AWS master account for the project team is an independent action. This means it operates separately from other configurations or changes within the service. Implementing the SCP won't interfere with, nor be impacted by, any other ongoing or planned modifications for this service. Thus, the team can proceed with this action confidently, knowing it won't inadvertently disrupt other service changes."
```


```
"Scheduling conflicts with other changes for this service have been reviewed and are not an issue. Enabling the Service Control Policy (SCP) on the AWS master account for the project team is an independent action. This means it operates separately from other configurations or changes within the service. Hence, the team may advance with this action, assured in the knowledge that it shall not inadvertently impede or conflict with other service modifications."
```


```
import os
import re
from string import Template

def extract_placeholders_from_template(template_path):
    # Read the template
    with open(template_path, 'r') as file:
        template_content = file.read()
    
    # Use regex to find all placeholders of the format ${variable_name}
    placeholders = re.findall(r'\$\{(\w+)\}', template_content)
    
    return placeholders

def populate_tfvars_from_template(template_path, environment_data):
    # Read the template
    with open(template_path, 'r') as file:
        template_content = file.read()
    
    # Use the Template class to substitute placeholders with actual values
    template = Template(template_content)
    populated_content = template.safe_substitute(environment_data)
    
    return populated_content

def lambda_handler(event, context):
    # Extracting the environment details from the event body
    environment = event.get('environment', {})
    
    # Directory where we'll store populated tfvars
    directory = '/tmp/builds/base/backend_config/'

    # Ensure the directory exists
    if not os.path.exists(directory):
        os.makedirs(directory)

    # For each environment, extract placeholders, populate the template, and write to a file
    for env_name in ['dev', 'preprod', 'prod']:
        template_path = os.path.join(os.environ['LAMBDA_TASK_ROOT'], f'{env_name}_template.tfvars')
        
        # Extract placeholders
        placeholders = extract_placeholders_from_template(template_path)
        
        # Check if we have data for all placeholders
        missing_data = [p for p in placeholders if p not in environment.get(env_name, {})]
        if missing_data:
            return {
                'statusCode': 400,
                'body': f"Missing data for placeholders: {', '.join(missing_data)} in {env_name} environment",
                'headers': {
                    'Content-Type': 'text/plain'
                }
            }
        
        populated_content = populate_tfvars_from_template(template_path, environment.get(env_name, {}))
        
        with open(os.path.join(directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(populated_content)

    # Return a success message with the paths to the generated files
    return {
        'statusCode': 200,
        'body': f"TFVars files generated at {directory}",
        'headers': {
            'Content-Type': 'text/plain'
        }
    }
```

```
import json
import os

def generate_tfvars_for_env(environment_data):
    account_id = environment_data.get('account_id', '')
    tfvars_dict = {
        'region': 'eu-west-1',
        'bucket': f"{account_id}-tf-state",
        'key': 'release-orchestration/base-infra/terraform.tfstate'
    }
    return json.dumps(tfvars_dict, indent=4)

def lambda_handler(event, context):
    # Extracting the environment details from the event body
    environment = event.get('environment', {})
    
    # Directory where we'll store populated tfvars
    directory = '/tmp/builds/base/backend_config/'

    # Ensure the directory exists
    if not os.path.exists(directory):
        os.makedirs(directory)

    # For each environment, generate the tfvars and write to a file
    account_ids = {}  # To store account_ids for each environment
    for env_name in ['dev', 'preprod', 'prod']:
        content = generate_tfvars_for_env(environment.get(env_name, {}))
        
        # Extract and store the account_id for the environment
        account_ids[env_name] = environment.get(env_name, {}).get('account_id', 'N/A')
        
        with open(os.path.join(directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(content)

    # Return a success message with the paths to the generated files and the account_ids
    return {
        'statusCode': 200,
        'body': {
            'message': f"TFVars files generated at {directory}",
            'account_ids': account_ids
        },
        'headers': {
            'Content-Type': 'application/json'
        }
    }
```

```
environment = "{{ environment }}"
vpc_id = "{{ vpc_id }}"
subnet_ids = {{ subnet_ids | tojson }}
build_svc_role_policy_arns = {{ build_svc_role_policy_arns | tojson }}
build_settings = {{ build_settings | tojson }}
pipeline_name = "{{ pipeline_name }}"
pipeline_ci_source = "{{ pipeline_ci_source }}"
pipeline_cd_source = "{{ pipeline_cd_source }}"
pipeline_stages_settings = {{ pipeline_stages_settings | tojson }}

```

```
import hcl2
import json

# Assume `request_body` is a string containing the JSON request body
request_body = '''
{
    "eim_id" : "9681879",
    // ... (rest of your JSON data) ...
}
'''

# Parse the JSON data
data = json.loads(request_body)

# Read the template.tfvars file
with open('template.tfvars', 'r') as f:
    template_content = f.read()

# Replace placeholders with actual values
updated_content = template_content.replace('{{ vpc_id }}', data['environment']['dev']['vpc_id'])
updated_content = updated_content.replace('{{ subnet_ids }}', ', '.join(data['environment']['dev']['non_routable_subnets']))
updated_content = updated_content.replace('{{ pipeline_name }}', data['pipline_name'])
updated_content = updated_content.replace('{{ pipeline_ci_source }}', f'{data["github_repo_url"].rstrip("/")}/source-ci.zip')
updated_content = updated_content.replace('{{ pipeline_cd_source }}', f'{data["github_repo_url"].rstrip("/")}/source-cd.zip')

# Parse the updated content to ensure it's valid HCL
parsed_hcl = hcl2.loads(updated_content)

# Write the updated content to a new tfvars file
with open('updated.tfvars', 'w') as f:
    f.write(updated_content)
```

```
import json

# Assume data is the JSON payload you received
data = {
    "eim_id": "9681879",
    "environment": {
        "dev": {
            "account_id": "123465789",
            "vpc_id": "vpc-123456789",
            "non_routable_subnets": [
                "subnet-123456789",
                "subnet-123456789",
                "subnet-123456789"
            ]
        },
        "preprod": {
            "account_id": "123465789",
            "vpc_id": "vpc-123456789",
            "non_routable_subnets": [
                "subnet-123456789",
                "subnet-123456789",
                "subnet-123456789"
            ]
        },
        "prod": {
            "account_id": "123465789",
            "vpc_id": "vpc-123456789",
            "non_routable_subnets": [
                "subnet-123456789",
                "subnet-123456789",
                "subnet-123456789"
            ]
        }
    },
    "github_repo_url": "http://alm-github.systems.uk.hsbc/GCS-AWS-Microservices/account-metadata/",
    "pipline_name": "account_metata"
}

# Split the URL by '/'
url_parts = data["github_repo_url"].split('/')

# Get the last three parts and join them with '/'
desired_string = '/'.join(url_parts[-3:])

# Now append the desired suffix
pipeline_ci_source = f'{desired_string}source-ci.zip'
pipeline_cd_source = f'{desired_string}source-cd.zip'

print(pipeline_ci_source)  # Outputs: GCS-AWS-Microservices/account-metadata/source-ci.zip
print(pipeline_cd_source)  # Outputs: GCS-AWS-Microservices/account-metadata/source-cd.zip
```
```
import json

# Assume data is the JSON payload you received
data = {
    "eim_id": "9681879",
    # ... other data ...
    "github_repo_url": "http://alm-github.systems.uk.hsbc/GCS-AWS-Microservices/account-metadata",
    "pipline_name": "account_metata"
}

# Split the URL by '/'
url_parts = data["github_repo_url"].split('/')

# Get the last three parts and join them with '/'
# Since there's no trailing slash, url_parts[-1] will be 'account-metadata' and not an empty string
desired_string = '/'.join(url_parts[-3:])

# Ensure there's a trailing slash
if not desired_string.endswith('/'):
    desired_string += '/'

# Now append the desired suffix
pipeline_ci_source = f'{desired_string}source-ci.zip'
pipeline_cd_source = f'{desired_string}source-cd.zip'

print(pipeline_ci_source)  # Outputs: GCS-AWS-Microservices/account-metadata/source-ci.zip
print(pipeline_cd_source)  # Outputs: GCS-AWS-Microservices/account-metadata/source-cd.zip
```

```
# Assume data is the JSON payload you received
data = {
    "eim_id": "9681879",
    # ... other data ...
    "github_repo_url": "http://alm-github.systems.uk.hsbc/GCS-AWS-Microservices/account-metadata",
    "pipline_name": "account_metata"
}

# Split the URL by '/'
url_parts = data["github_repo_url"].split('/')

# Get the last two parts, replace slashes with hyphens, and join them with a hyphen
formatted_string = '-'.join(url_parts[-2:])

# Now you have the formatted string
print(formatted_string)  # Outputs: GCS-AWS-Microservices-account-metadata
```

```
environment = "dev"
vpc_id = "{{ vpc_id }}"
subnet_ids = ["{{ subnet_id_1 }}", "{{ subnet_id_2 }}", "{{ subnet_id_3 }}"]
build_svc_role_policy_arns = ["{{ build_svc_role_policy_arn }}"]
build_settings = {
    lint_unit_test = {
        name        = "lint-unit-test"
        description = "Linting and unit test"
        buildspec   = "buildspec/lint_unit_test.yaml"
        image       = "7257134640.dkr.ecr.eu-west-1.amazonaws.com/scanned/release/gcs/jenkins-aws01/terraform-awscli-codebuild:multi"
    },
    # ... other build settings ...
}

pipeline_name = "{{ pipeline_name }}"
pipeline_ci_source = "{{ github_repo_formatted }}/source-ci.zip"
pipeline_cd_source = "{{ github_repo_formatted }}/source-cd.zip"

pipeline_stages_settings = {
  "lint_unit_test" = {
    name = "LintAndUnitTest"
    category = "Test"
    namespace = "LintAndUnitTestVariables"
    input_artifacts = ["SourceArtifact"]
    output_artifacts = ["LintAndUnitTestArtifact"]
    extra_configuration = {}
  },
  # ... other pipeline stages settings ...
}

```
import os
import hcl2
import json

def populate_tfvars_from_template(template_path, environment_data):
    # Read the template using python-hcl2
    with open(template_path, 'r') as file:
        template_content = hcl2.load(file)

    # Update the template content with values from environment_data
    account_id = environment_data.get('account_id', '')
    template_content['bucket'] = f"{account_id}-tf-state"
    
    return json.dumps(template_content, indent=4)

def lambda_handler(event, context):
    # Extracting the environment details from the event body
    environment = event.get('environment', {})

    # Directory where we'll store populated tfvars
    directory = '/tmp/builds/base/backend_config/'

    # Ensure the directory exists
    if not os.path.exists(directory):
        os.makedirs(directory)

    # For each environment, populate the template and write to a file
    for env_name in ['dev', 'preprod', 'prod']:
        template_path = os.path.join(os.environ['LAMBDA_TASK_ROOT'], f'{env_name}_template.tfvars')
        populated_content = populate_tfvars_from_template(template_path, environment.get(env_name, {}))
        
        with open(os.path.join(directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(populated_content)

    # Return a success message with the paths to the generated files
    return {
        'statusCode': 200,
        'body': f"TFVars files generated at {directory}",
        'headers': {
            'Content-Type': 'text/plain'
        }
    }
```

```
import json
import os

def generate_tfvars_for_env(environment_data):
    account_id = environment_data.get('account_id', '')
    tfvars_dict = {
        'region': 'eu-west-1',
        'bucket': f"{account_id}-tf-state",
        'key': 'release-orchestration/base-infra/terraform.tfstate'
    }
    return json.dumps(tfvars_dict, indent=4)

def lambda_handler(event, context):
    # Extracting the environment details from the event body
    environment = event.get('environment', {})
    
    # Directory where we'll store populated tfvars
    directory = '/tmp/builds/base/backend_config/'

    # Ensure the directory exists
    if not os.path.exists(directory):
        os.makedirs(directory)

    # For each environment, generate the tfvars and write to a file
    for env_name in ['dev', 'preprod', 'prod']:
        content = generate_tfvars_for_env(environment.get(env_name, {}))
        
        with open(os.path.join(directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(content)

    # Return a success message with the paths to the generated files
    return {
        'statusCode': 200,
        'body': f"TFVars files generated at {directory}",
        'headers': {
            'Content-Type': 'text/plain'
        }
    }
```

```
import json
import os

def generate_tfvars_for_env(environment_data):
    account_id = environment_data.get('account_id', '')
    tfvars_dict = {
        'region': 'eu-west-1',
        'bucket': f"{account_id}-tf-state",
        'key': 'release-orchestration/base-infra/terraform.tfstate'
    }
    return json.dumps(tfvars_dict, indent=4)

def lambda_handler(event, context):
    # Extracting the environment details from the event body
    environment = event.get('environment', {})
    
    # Directory where we'll store populated tfvars
    directory = '/tmp/builds/base/backend_config/'

    # Ensure the directory exists
    if not os.path.exists(directory):
        os.makedirs(directory)

    # For each environment, generate the tfvars and write to a file
    account_ids = {}  # To store account_ids for each environment
    for env_name in ['dev', 'preprod', 'prod']:
        content = generate_tfvars_for_env(environment.get(env_name, {}))
        
        # Extract and store the account_id for the environment
        account_ids[env_name] = environment.get(env_name, {}).get('account_id', 'N/A')
        
        with open(os.path.join(directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(content)

    # Return a success message with the paths to the generated files and the account_ids
    return {
        'statusCode': 200,
        'body': {
            'message': f"TFVars files generated at {directory}",
            'account_ids': account_ids
        },
        'headers': {
            'Content-Type': 'application/json'
        }
    }
```
```
region = "eu-west-1"
bucket = "${account_id}-tf-state"
key = "release-orchestration/base-infra/terraform.tfstate"


import os
from string import Template

def populate_tfvars_from_template(template_path, environment_data):
    # Read the template
    with open(template_path, 'r') as file:
        template_content = file.read()
    
    # Use the Template class to substitute placeholders with actual values
    template = Template(template_content)
    populated_content = template.substitute(environment_data)
    
    return populated_content

def lambda_handler(event, context):
    # Extracting the environment details from the event body
    environment = event.get('environment', {})
    
    # Directory where we'll store populated tfvars
    directory = '/tmp/builds/base/backend_config/'

    # Ensure the directory exists
    if not os.path.exists(directory):
        os.makedirs(directory)

    # For each environment, populate the template and write to a file
    for env_name in ['dev', 'preprod', 'prod']:
        template_path = os.path.join(os.environ['LAMBDA_TASK_ROOT'], f'{env_name}_template.tfvars')
        populated_content = populate_tfvars_from_template(template_path, environment.get(env_name, {}))
        
        with open(os.path.join(directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(populated_content)

    # Return a success message with the paths to the generated files
    return {
        'statusCode': 200,
        'body': f"TFVars files generated at {directory}",
        'headers': {
            'Content-Type': 'text/plain'
        }
    }
```
```
class DataTransformer:
    def __init__(self, input_data):
        self.input_data = input_data
        
    def transform(self):
        # Ensure github_repo_url does not have a trailing slash for consistent formatting
        github_repo_url = self.input_data['github_repo_url'].rstrip('/')
        pipline_name = self.input_data['pipline_name']
        
        # Prepare the base structure of output_data
        output_data = {
            'eim_id': self.input_data['eim_id'],
            'environment': {}
        }
        
        # Process each environment
        for env, details in self.input_data['environment'].items():
            # Extract and restructure subnet data
            subnet_ids = details['non_routable_subnets']
            subnet_data = {f'subnet_id_{i+1}': subnet_id for i, subnet_id in enumerate(subnet_ids)}
            
            # Prepare the data for this environment
            env_data = {
                'account_id': details['account_id'],
                'vpc_id': details['vpc_id'],
                'pipeline_name': pipline_name,
                'pipeline_ci_source': f'{github_repo_url}/source-ci.zip',
                'pipeline_cd_source': f'{github_repo_url}/source-cd.zip',
                **subnet_data  # Merge subnet_data into env_data
            }
            
            # Add this environment's data to the output_data
            output_data['environment'][env] = env_data
        
        return output_data

# Usage:
input_data = {
    # ... (as provided)
}

transformer = DataTransformer(input_data)
output_data = transformer.transform()
print(output_data)
```
```
def transform_data(input_data):
    # Ensure github_repo_url does not have a trailing slash for consistent formatting
    github_repo_url = input_data['github_repo_url'].rstrip('/')
    pipline_name = input_data['pipline_name']
    
    # Prepare the base structure of output_data
    output_data = {
        'eim_id': input_data['eim_id'],
        'environment': {}
    }
    
    # Process each environment
    for env, details in input_data['environment'].items():
        # Extract and restructure subnet data
        subnet_ids = details['non_routable_subnets']
        subnet_data = {f'subnet_id_{i+1}': subnet_id for i, subnet_id in enumerate(subnet_ids)}
        
        # Prepare the data for this environment
        env_data = {
            'account_id': details['account_id'],
            'vpc_id': details['vpc_id'],
            'pipeline_name': pipline_name,
            'pipeline_ci_source': f'{github_repo_url}/source-ci.zip',
            'pipeline_cd_source': f'{github_repo_url}/source-cd.zip',
            **subnet_data  # Merge subnet_data into env_data
        }
        
        # Add this environment's data to the output_data
        output_data['environment'][env] = env_data
    
    return output_data

# Usage:
input_data = {
    # ... (as provided)
}
```
```
class DataTransformer:
    def __init__(self, input_data):
        self.input_data = input_data
        
    def transform(self):
        # Ensure github_repo_url does not have a trailing slash for consistent formatting
        github_repo_url = self.input_data['github_repo_url'].rstrip('/')
        pipline_name = self.input_data['pipline_name']
        
        # Prepare the base structure of output_data
        output_data = {
            'eim_id': self.input_data['eim_id'],
            'environment': {}
        }
        
        # Process each environment
        for env, details in self.input_data['environment'].items():
            # Extract and restructure subnet data
            subnet_ids = details['non_routable_subnets']
            subnet_data = {f'subnet_id_{i+1}': subnet_id for i, subnet_id in enumerate(subnet_ids)}
            
            # Prepare the data for this environment
            env_data = {
                'account_id': details['account_id'],
                'vpc_id': details['vpc_id'],
                'pipeline_name': pipline_name,
                'pipeline_ci_source': f'{github_repo_url}/source-ci.zip',
                'pipeline_cd_source': f'{github_repo_url}/source-cd.zip',
                **subnet_data  # Merge subnet_data into env_data
            }
            
            # Add this environment's data to the output_data
            output_data['environment'][env] = env_data
        
        return output_data

# Usage:
input_data = {
    # ... (as provided)
}

transformer = DataTransformer(input_data)
output_data = transformer.transform()
print(output_data)
```

```
from typing import Dict, List, Any

class DataTransformer:
    def __init__(self, input_data: Dict[str, Any]) -> None:
        self.input_data = input_data
        
    def transform(self) -> Dict[str, Any]:
        # Ensure github_repo_url does not have a trailing slash for consistent formatting
        github_repo_url = self.input_data['github_repo_url'].rstrip('/')
        pipline_name = self.input_data['pipline_name']
        
        # Prepare the base structure of output_data
        output_data = {
            'eim_id': self.input_data['eim_id'],
            'environment': {}
        }
        
        # Process each environment
        for env, details in self.input_data['environment'].items():
            # Extract and restructure subnet data
            subnet_ids = details['non_routable_subnets']
            subnet_data = {f'subnet_id_{i+1}': subnet_id for i, subnet_id in enumerate(subnet_ids)}
            
            # Prepare the data for this environment
            env_data = {
                'account_id': details['account_id'],
                'vpc_id': details['vpc_id'],
                'pipeline_name': pipline_name,
                'pipeline_ci_source': f'{github_repo_url}/source-ci.zip',
                'pipeline_cd_source': f'{github_repo_url}/source-cd.zip',
                **subnet_data  # Merge subnet_data into env_data
            }
            
            # Add this environment's data to the output_data
            output_data['environment'][env] = env_data
        
        return output_data

# Usage:
input_data = {
    # ... (as provided)
}

transformer = DataTransformer(input_data)
output_data = transformer.transform()
print(output_data)
```
```
from typing import List, Dict, Any

class TerraformProcessor:

    def __init__(self, tf, working_directory: str, input_data: Dict[str, Any]):
        self.tf = tf
        self.working_directory = working_directory
        self.input_data = input_data
        self.transformer = DataTransformer(self.input_data)

    def extract_placeholders_from_template(self, template_path: str) -> List[str]:
        # Logic for extracting placeholders from template goes here
        pass

    def populate_tfvars_from_template(self, template_path: str, data: Dict[str, Any]) -> str:
        # Logic for populating tfvars from template goes here
        pass

    def process_backend_config(self, env_name: str) -> None:
        backend_config_directory = os.path.join(self.working_directory, 'backend_config/')
        template_path = os.path.join(backend_config_directory, 'template.tfvars')
        placeholders = self.extract_placeholders_from_template(template_path)
        output_data = self.transformer.transform()
        missing_data = [p for p in placeholders if p not in output_data['environment'].get(env_name, {})]
        if missing_data:
            print('some variable missing')
        populated_content = self.populate_tfvars_from_template(template_path, output_data['environment'].get(env_name, {}))
        with open(os.path.join(backend_config_directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(populated_content)

    def process_env_vars(self, env_name: str) -> None:
        env_vars_directory = os.path.join(self.working_directory, 'env_vars/')
        template_path = os.path.join(env_vars_directory, f'template.{env_name}.tfvars')
        placeholders = self.extract_placeholders_from_template(template_path)
        output_data = self.transformer.transform()
        missing_data = [p for p in placeholders if p not in output_data['environment'].get(env_name, {})]
        if missing_data:
            print('some variable missing')
        populated_content = self.populate_tfvars_from_template(template_path, output_data['environment'].get(env_name, {}))
        with open(os.path.join(env_vars_directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(populated_content)

    def process_environments(self, env_list: List[str], process_env_vars: bool = True) -> None:
        for env_name in env_list:
            self.process_backend_config(env_name)
            if process_env_vars:
                self.process_env_vars(env_name)

# Using the class
environments = ['dev', 'preprod', 'prod']

with TerraformRunner('./builds/base', target_creds) as tf:
    processor = TerraformProcessor(tf, tf.working_dir, input_data)
    processor.process_environments(environments, process_env_vars=False)

with TerraformRunner('./builds/pipeline', target_creds) as tf:
    processor = TerraformProcessor(tf, tf.working_dir, input_data)
    processor.process_environments(environments)
```
```
def process_environments(tf, working_directory, input_data, env_list, process_env_vars=True):
    for env_name in env_list:
        backend_config_directory = os.path.join(working_directory, 'backend_config/')
        template_path = os.path.join(backend_config_directory, 'template.tfvars')
        placeholders = extract_placeholders_from_template(template_path)
        transformer = DataTransformer(input_data)
        output_data = transformer.transform()
        missing_data = [p for p in placeholders if p not in output_data['environment'].get(env_name, {})]
        if missing_data:
            print('some variable missing')
        populated_content = populate_tfvars_from_template(template_path, output_data['environment'].get(env_name, {}))
        with open(os.path.join(backend_config_directory, f'{env_name}.tfvars'), 'w') as f:
            f.write(populated_content)

    if process_env_vars:
        for env_name in env_list:
            env_vars_directory = os.path.join(working_directory, 'env_vars/')
            template_path = os.path.join(env_vars_directory, f'template.{env_name}.tfvars')
            placeholders = extract_placeholders_from_template(template_path)
            transformer = DataTransformer(input_data)
            output_data = transformer.transform()
            missing_data = [p for p in placeholders if p not in output_data['environment'].get(env_name, {})]
            if missing_data:
                print('some variable missing')
            populated_content = populate_tfvars_from_template(template_path, output_data['environment'].get(env_name, {}))
            with open(os.path.join(env_vars_directory, f'{env_name}.tfvars'), 'w') as f:
                f.write(populated_content)

# Using the function
environments = ['dev', 'preprod', 'prod']

with TerraformRunner('./builds/base', target_creds) as tf:
    process_environments(tf, tf.working_dir, input_data, environments, process_env_vars=False)

with TerraformRunner('./builds/pipeline', target_creds) as tf:
    process_environments(tf, tf.working_dir, input_data, environments)
```
```
import json
import boto3

def lambda_handler(event: Dict, context: Any) -> None:
    # Assuming you've set up the boto3 SQS client at the top of your script
    sqs = boto3.client('sqs')
    sqs_queue_url = 'YOUR_SQS_QUEUE_URL'  # Replace this with your SQS Queue URL

    PREV_ENVS = {'dev': None, 'preprod': 'dev', 'prod': 'preprod'}
    NEXT_ENVS = {'dev': 'preprod', 'preprod': 'prod', 'prod': None}

    # Extract payload
    if event.get('Records'):
        payload = json.loads(event['Records'][0]['body'])
        current_env = payload['current']
        current_env_config = payload['environments'][current_env]
        account_id = current_env_config['account_id']
        
        # ... Your existing processing code ...

        try:
            # ... Your Terraform processing code ...

            # If processing completes without errors, send success message
            success_payload = payload.copy()
            success_payload['current'] = NEXT_ENVS[current_env]
            success_payload['action'] = 'apply'
            sqs.send_message(QueueUrl=sqs_queue_url, MessageBody=json.dumps(success_payload))

        except Exception as e:
            # If there's an error, send failure message
            failure_payload = payload.copy()
            failure_payload['current'] = current_env
            failure_payload['action'] = 'rollback'
            sqs.send_message(QueueUrl=sqs_queue_url, MessageBody=json.dumps(failure_payload))
            print(f"Error occurred: {str(e)}")

        finally:
            # Optional: Logging or any cleanup activities can be added here
            print(os.getcwd())
            print(os.listdir('/tmp'))
            return
```

```
def lambda_handler(event: Dict, context: Any) -> None:
    # Extracting the SQS message payload.
    payload = json.loads(event["Records"][0]["body"])
    # Identify the source SQS queue based on the event source ARN.
    source_queue_arn = event["Records"][0]["eventSourceARN"]
    
    current_env = payload["current"]
    action = payload["action"]
    
    NEXT_ENVS = {
        'dev': 'preprod',
        'preprod': 'prod',
        'prod': None
    }
    
    PREV_ENVS = {
        'dev': None,
        'preprod': 'dev',
        'prod': 'preprod'
    }
    
    sqs_client = Boto3Client('sqs')  # Adjust according to your needs.
    
    # Depending on the ARN, determine if the message came from DLQ or normal queue.
    if "deadletterqueue-name" in source_queue_arn:  # Replace 'deadletterqueue-name' with your DLQ name.
        queue_type = "dlq"
    else:
        queue_type = "normal"
    
    if queue_type == "dlq" or action == "rollback":
        # If the message is from the DLQ or if action is rollback, perform rollback flow.
        next_action = "rollback"
        next_env = PREV_ENVS.get(current_env)
        
        if next_env:  # Ensuring we don't send any SQS message after 'dev' rollback.
            sqs_client.send_message(
                QueueUrl="YOUR_NORMAL_QUEUE_URL",  # Replace with your normal SQS Queue URL.
                MessageBody=json.dumps({
                    "current": next_env,
                    "action": next_action
                })
            )
    else:
        # For apply flow
        next_action = "apply"
        next_env = NEXT_ENVS.get(current_env)
        
        if next_env:  # Ensuring we don't send any SQS message after 'prod' apply.
            sqs_client.send_message(
                QueueUrl="YOUR_NORMAL_QUEUE_URL",  # Replace with your normal SQS Queue URL.
                MessageBody=json.dumps({
                    "current": next_env,
                    "action": next_action
                })
            )
```

resource "aws_sfn_state_machine" "example_state_machine" {
  name     = "example_state_machine"
  role_arn = aws_iam_role.step_function_role.arn

  definition = <<EOF
{
  "Comment": "A state machine to automate pipeline deployments across environments with retry and rollback.",
  "StartAt": "DeployDev",
  "States": {
    "DeployDev": {
      "Type": "Task",
      "Resource": "${aws_lambda_function.deploy_dev.arn}",
      "Next": "DeployPreProd",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException", "Lambda.SdkClientException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "RollbackFlow"
        }
      ]
    },
    "DeployPreProd": {
      "Type": "Task",
      "Resource": "${aws_lambda_function.deploy_preprod.arn}",
      "Next": "DeployProd"
    },
    "DeployProd": {
      "Type": "Task",
      "Resource": "${aws_lambda_function.deploy_prod.arn}",
      "End": true
    },
    "RollbackFlow": {
      "Type": "Task",
      "Resource": "${aws_lambda_function.rollback_lambda.arn}",
      "End": true
    }
  }
}
EOF

  depends_on = [
    aws_iam_role_policy.step_function_policy
  ]
}

```
```
import boto3
import json
import os
from typing import Any, Dict
from boto3.dynamodb.conditions import Key
from botocore.exceptions import ClientError
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.utilities.data_classes import event_source, APIGatewayProxyEvent
from aws_lambda_powertools.utilities.parser import parse, ValidationError
from aws_lambda_powertools.utilities.typing import LambdaContext

# Environment variables
CURRENT_REGION = os.getenv('AWS_REGION')
APP_TABLE_NAME = os.getenv('APP_TABLE_NAME', 'release_orchestration_app')
STEP_FUNCTION_ARN = os.getenv('STEP_FUNCTION_ARN')  # Add this to your environment variables

# AWS Lambda Powertools
tracer = Tracer()
logger = Logger()

# Models for input and response
class UpdateRequest(BaseModel):
    github_repo_url: str

class LambdaProxyResponse(BaseModel):
    statusCode: int
    body: str
    isBase64Encoded: Optional[bool] = False
    headers: Optional[Dict] = None

class SuccessResponse(LambdaProxyResponse):
    statusCode: Optional[int] = 200
    body: Optional[str] = 'Success'

class FailedResponse(LambdaProxyResponse):
    pass

# DynamoDB Base Class
class DDBBaseClass:
    def __init__(self, table: str, region: str, session: boto3.session.Session):
        self.dynamodb = session.resource('dynamodb', region_name=region)
        self.table = self.dynamodb.Table(table)

    def query_by_github_repo_url(self, github_repo_url: str) -> Any:
        try:
            resp = self.table.query(
                IndexName='GithubRepoUrlIndex',  # Assuming you have a GSI for github_repo_url
                KeyConditionExpression=Key('github_repo_url').eq(github_repo_url)
            )
            return resp['Items']
        except ClientError as e:
            logger.error(f'Error querying item: {e}')
            raise

# Lambda handler
@logger.inject_lambda_context(clear_state=True)
@event_source(data_class=APIGatewayProxyEvent)
def lambda_handler(event: APIGatewayProxyEvent, context: LambdaContext) -> Dict:
    logger.debug(f'event: {event}')
    logger.debug(f'context: {context}')

    try:
        data: UpdateRequest = parse(model=UpdateRequest, event=event.json_body)
        logger.info('Validated user input successfully')
    except ValidationError as e:
        logger.info(f'User input validation error: {str(e)}, {event}')
        return FailedResponse(statusCode=400, body='Invalid Request').dict()

    session = boto3.session.Session()
    app = DDBBaseClass(table=APP_TABLE_NAME, region=CURRENT_REGION, session=session)

    try:
        items = app.query_by_github_repo_url(github_repo_url=data.github_repo_url)
        if not items:
            return FailedResponse(statusCode=404, body='Pipeline not found').dict()
    except Exception as e:
        logger.error(f'Error querying pipeline: "{str(e)}"')
        return FailedResponse(statusCode=500, body='Internal Error').dict()

    step_function_client = session.client('stepfunctions')
    try:
        response = step_function_client.start_execution(
            stateMachineArn=STEP_FUNCTION_ARN,
            input=json.dumps(items[0])  # Using the first item found as input
        )
        logger.info(f'Step Function triggered successfully: {response}')
    except ClientError as e:
        logger.error(f'Error triggering Step Function: {e}')
        return FailedResponse(statusCode=500, body='Internal Error').dict()

    return SuccessResponse(body='Pipeline update process initiated').dict()
```

```
{
  "Comment": "A State Machine to orchestrate pipeline updates across environments using app_worker",
  "StartAt": "UpdateDevEnvironment",
  "States": {
    "UpdateDevEnvironment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:app_worker",
      "InputPath": "$",
      "ResultPath": "$.devResult",
      "Next": "UpdatePreprodEnvironment",
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ]
    },
    "UpdatePreprodEnvironment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:app_worker",
      "Parameters": {
        "eim_id.$": "$.eim_id",
        "environments.$": "$.environments",
        "github_repo_url.$": "$.github_repo_url",
        "pipeline_name.$": "$.pipeline_name",
        "current": "preprod",
        "action": "apply"
      },
      "InputPath": "$",
      "ResultPath": "$.preprodResult",
      "Next": "UpdateProdEnvironment",
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ]
    },
    "UpdateProdEnvironment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:app_worker",
      "Parameters": {
        "eim_id.$": "$.eim_id",
        "environments.$": "$.environments",
        "github_repo_url.$": "$.github_repo_url",
        "pipeline_name.$": "$.pipeline_name",
        "current": "prod",
        "action": "apply"
      },
      "InputPath": "$",
      "ResultPath": "$.prodResult",
      "End": true,
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ]
    }
  }
}
```

```
class TestCidrManagerDelete(unittest.TestCase):

    @patch("boto3.client")
    def test_delete_cidr_id(self, mocker_boto3_client):
        # ... existing test ...

    @patch("boto3.client")
    def test_invoke_payload(self, mocker_boto3_client):
        # Test if the correct payload is sent to AWS Lambda
        method = "DELETE"
        endpoint = "cidr_blocks"
        cidr_id = "1"
        expected_payload = json.dumps({"method": method, "endpoint": f"{endpoint}/{cidr_id}"})

        cidr_manager_delete.cidr_blocks_remover(method, endpoint, cidr_id)
        args, kwargs = mocker_boto3_client.return_value.invoke.call_args
        actual_payload = kwargs['Payload']
        self.assertEqual(expected_payload, actual_payload)

    @patch.dict(os.environ, {'CMP_RELAY_FUNCTION_ARN': ''})
    def test_missing_function_arn(self):
        # Test behavior when the CMP_RELAY_FUNCTION_ARN is not set
        with self.assertRaises(KeyError):
            cidr_manager_delete.cidr_blocks_remover("DELETE", "cidr_blocks", "1")

    @patch("boto3.client")
    def test_non_200_response(self, mocker_boto3_client):
        # Test handling of non-200 response from the relay function
        mocker_boto3_client.return_value.invoke.return_value = {
            "StatusCode": 400,
            "Payload": json.dumps({"error": "Bad Request"})
        }
        response = cidr_manager_delete.cidr_blocks_remover("DELETE", "cidr_blocks", "1")
        self.assertIn('error', response)

    @patch("boto3.client")
    def test_boto3_invoke_exception(self, mocker_boto3_client):
        # Test exception handling for boto3 invocation
        mocker_boto3_client.return_value.invoke.side_effect = Exception("AWS Error")
        with self.assertRaises(Exception) as context:
            cidr_manager_delete.cidr_blocks_remover("DELETE", "cidr_blocks", "1")
        self.assertTrue('AWS Error' in str(context.exception))

```
