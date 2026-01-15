# Authentication Reference

Detailed setup for each Zapier authentication scheme.

## OAuth v2 (Recommended)

Best user experience—familiar popup flow, no manual credential copying.

### Prerequisites
- Register Zapier as OAuth client in your app
- Obtain Client ID and Client Secret
- Configure redirect URI: `https://zapier.com/dashboard/auth/oauth/return/App{ID}CLIAPI/`

### Configuration Steps

1. **Add Input Fields** (optional): For custom domains or workspace selection
   
2. **OAuth v2 Endpoint Configuration**:
   - **Client ID**: From your app's developer settings
   - **Client Secret**: From your app's developer settings
   - **Authorization URL**: Where users log in (e.g., `https://app.example.com/oauth/authorize`)
   - **Scope**: Space-separated permissions (e.g., `read write`)
   - **Access Token Request URL**: Token exchange endpoint (e.g., `https://app.example.com/oauth/token`)
   
3. **Refresh Token Request** (if supported):
   - Enable "Automatically Refresh Token"
   - Configure refresh endpoint

### Token Access in Code Mode
```javascript
const options = {
  url: 'https://api.example.com/resource',
  headers: {
    'Authorization': `Bearer ${bundle.authData.access_token}`
  }
};
```

### Test Request
Configure endpoint that validates the token (typically `/me` or `/users/me`).

---

## API Key

Users copy an API key from your app's settings.

### Configuration Steps

1. **Add Input Fields**:
   - Key: `api_key`
   - Label: "API Key"  
   - Type: Password (masks input)
   - Required: Yes
   - Help Text: "Find your API key in Settings → API"

2. **Configure Test Request**:
   - URL: Your test endpoint
   - Add key via Headers or URL Params

### Header Authentication
```
Header: Authorization
Value: Bearer {{bundle.authData.api_key}}
```

Or custom header:
```
Header: X-API-Key
Value: {{bundle.authData.api_key}}
```

### Query Parameter Authentication
```
URL: https://api.example.com/me?api_key={{bundle.authData.api_key}}
```

### Code Mode Example
```javascript
const options = {
  url: 'https://api.example.com/me',
  headers: {
    'X-API-Key': bundle.authData.api_key
  }
};
return z.request(options).then(response => response.data);
```

---

## Session Authentication

Exchange credentials for a session token, optionally with refresh.

### Configuration Steps

1. **Add Input Fields**:
   ```
   username (String, Required)
   password (Password, Required)
   ```

2. **Configure Token Exchange Request**:
   - URL: Your login/token endpoint
   - Method: POST
   - Body:
     ```json
     {
       "username": "{{bundle.authData.username}}",
       "password": "{{bundle.authData.password}}"
     }
     ```

3. **Store Token**: Map response field to `sessionToken`

4. **Use in Requests**:
   ```
   Header: Authorization
   Value: Bearer {{bundle.authData.sessionToken}}
   ```

### Code Mode Token Exchange
```javascript
const options = {
  url: 'https://api.example.com/auth/token',
  method: 'POST',
  body: {
    username: bundle.authData.username,
    password: bundle.authData.password
  }
};

return z.request(options).then(response => {
  return {
    sessionToken: response.data.token,
    // Store other fields as needed
    userId: response.data.user_id
  };
});
```

---

## Basic Authentication

Credentials sent with every request (least secure option).

### Configuration Steps

1. **Add Input Fields**:
   ```
   username (String, Required)
   password (Password, Required)
   ```

2. **Configure Test Request**:
   Zapier automatically encodes credentials as Base64 `Authorization: Basic` header.

### Code Mode (if needed)
```javascript
const credentials = Buffer.from(
  `${bundle.authData.username}:${bundle.authData.password}`
).toString('base64');

const options = {
  url: 'https://api.example.com/me',
  headers: {
    'Authorization': `Basic ${credentials}`
  }
};
```

---

## Connection Labels

Display meaningful account identifier in Zapier's connection list.

### In Authentication Settings
Add Connection Label template:
```
{{bundle.authData.email}}
```
or
```
{{bundle.authData.username}} ({{bundle.authData.workspace}})
```

### Dynamic Labels from Test Response
If test request returns user info:
```
{{bundle.inputData.email}}
```
Use `inputData` for fields from the test response, `authData` for stored credentials.

---

## Computed Fields

Store additional data from authentication for later use.

### Use Cases
- Store user ID from OAuth response
- Save workspace/tenant identifier
- Cache account-level settings

### In OAuth Redirect Response
Map response fields to computed fields:
```
Field Key: user_id
Response Path: $.user.id
```

Access later: `{{bundle.authData.user_id}}`

---

## Troubleshooting Authentication

### 401 Unauthorized
- Token expired or invalid
- Wrong header format
- Missing required scopes

### 403 Forbidden  
- Valid token but insufficient permissions
- Account-level restrictions

### 400 Bad Request
- Malformed token exchange request
- Invalid client credentials
- Wrong grant type

### Connection Label Shows "undefined"
- Test request not returning expected fields
- Wrong field path in label template
- Field name mismatch
