# AWS Step Functions Authentication Workflow

## Overview
Built a complete user authentication workflow using AWS Step Functions with Cognito integration. The workflow handles user registration, login, and logout processes.

## Architecture Components

### 1. AWS Lambda Functions

#### CheckUserLambda
```python
import boto3

USER_POOL_ID = 'ap-south-1_Ox0knQelv'
cognito = boto3.client('cognito-idp')

def lambda_handler(event, context):
    username = event["username"]
    try:
        cognito.admin_get_user(UserPoolId=USER_POOL_ID, Username=username)
        event["userExists"] = True
    except cognito.exceptions.UserNotFoundException:
        event["userExists"] = False
    return event
```

#### SignupLambda
```python
import boto3

USER_POOL_ID = 'ap-south-1_Ox0knQelv'
CLIENT_ID = '5fktabakg95quea49h332e9ffs'

cognito = boto3.client('cognito-idp')

def lambda_handler(event, context):
    username = event["username"]
    password = event["password"]

    try:
        # Create the user in the app client
        cognito.sign_up(
            ClientId=CLIENT_ID,
            Username=username,
            Password=password,
            UserAttributes=[{"Name": "email", "Value": username}]
        )
        # Auto-confirm (admin)
        cognito.admin_confirm_sign_up(UserPoolId=USER_POOL_ID, Username=username)
        event["signup"] = "created_and_confirmed"
    except Exception as e:
        event["signupError"] = str(e)

    return event
```

#### LoginLambda
```python
import boto3
import hmac
import hashlib
import base64

USER_POOL_ID = 'ap-south-1_Ox0knQelv'
CLIENT_ID = '5fktabakg95quea49h332e9ffs'
CLIENT_SECRET = '1gdj4r9p0hjf7qvqkem3qiakqvfr7e12h3mfci915m87d5iveq0k'

cognito = boto3.client('cognito-idp')

def _secret_hash(username: str) -> str:
    msg = (username + CLIENT_ID).encode("utf-8")
    key = CLIENT_SECRET.encode("utf-8")
    dig = hmac.new(key, msg, hashlib.sha256).digest()
    return base64.b64encode(dig).decode()

def lambda_handler(event, context):
    username = event["username"]
    password = event["password"]

    try:
        resp = cognito.initiate_auth(
            AuthFlow="USER_PASSWORD_AUTH",
            ClientId=CLIENT_ID,
            AuthParameters={
                "USERNAME": username,
                "PASSWORD": password,
                "SECRET_HASH": _secret_hash(username),
            }
        )

        tokens = resp.get("AuthenticationResult", {})
        event["tokens"] = {
            "accessToken": tokens.get("AccessToken"),
            "idToken": tokens.get("IdToken"),
            "refreshToken": tokens.get("RefreshToken"),
            "expiresIn": tokens.get("ExpiresIn")
        }
    except Exception as e:
        event["loginError"] = str(e)

    return event
```

#### LogoutLambda
```python
import boto3

cognito = boto3.client('cognito-idp')

def lambda_handler(event, context):
    try:
        access_token = event["tokens"]["accessToken"]
        cognito.global_sign_out(AccessToken=access_token)
        event["logout"] = "success"
    except Exception as e:
        event["logoutError"] = str(e)
    return event
```

### 2. Step Functions State Machine

#### Complete State Machine Definition
```json
{
  "Comment": "User Authentication Workflow with Cognito",
  "StartAt": "checkuserlamda",
  "States": {
    "checkuserlamda": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": "{% $states.result.Payload %}",
      "Arguments": {
        "FunctionName": "CheckUserLambda",
        "Payload": "{% $states.input %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "Next": "IsNewUser"
    },
    "IsNewUser": {
      "Type": "Choice",
      "Choices": [
        {
          "Next": "SignupLambda",
          "Condition": "{% $userExists = false %}"
        }
      ],
      "Default": "LoginLambda"
    },
    "SignupLambda": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": "{% $states.result.Payload %}",
      "Arguments": {
        "FunctionName": "SignupLambda",
        "Payload": "{% $states.input %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "Next": "Wait"
    },
    "Wait": {
      "Type": "Wait",
      "Seconds": 120,
      "Next": "LogoutLambda"
    },
    "LogoutLambda": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": "{% $states.result.Payload %}",
      "Arguments": {
        "FunctionName": "LogoutLambda",
        "Payload": "{% $states.input %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "End": true
    },
    "LoginLambda": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": "{% $states.result.Payload %}",
      "Arguments": {
        "FunctionName": "LoginLambda",
        "Payload": "{% $states.input %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "Next": "WaitTill120"
    },
    "WaitTill120": {
      "Type": "Wait",
      "Seconds": 120,
      "Next": "LambdaLOGout"
    },
    "LambdaLOGout": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": "{% $states.result.Payload %}",
      "Arguments": {
        "FunctionName": "LogoutLambda",
        "Payload": "{% $states.input %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "End": true
    }
  },
  "QueryLanguage": "JSONata"
}
```

## Workflow Flow

### Expected Flow:
1. **Start** → CheckUserLambda
2. **CheckUserLambda** → IsNewUser (Choice State)
3. **IsNewUser** → SignupLambda (if user doesn't exist) OR LoginLambda (if user exists)
4. **SignupLambda** → Wait (120 seconds) → LogoutLambda → End
5. **LoginLambda** → WaitTill120 (120 seconds) → LambdaLOGout → End

### Input Format:
```json
{
  "username": "user@example.com",
  "password": "userpassword123"
}
```

### Expected Outputs:

#### CheckUserLambda Output:
```json
{
  "username": "user@example.com",
  "password": "userpassword123",
  "userExists": false
}
```

#### SignupLambda Output:
```json
{
  "username": "user@example.com",
  "password": "userpassword123",
  "signup": "created_and_confirmed"
}
```

#### LoginLambda Output:
```json
{
  "username": "user@example.com",
  "password": "userpassword123",
  "tokens": {
    "accessToken": "...",
    "idToken": "...",
    "refreshToken": "...",
    "expiresIn": 3600
  }
}
```

## Configuration Details

### AWS Cognito Configuration:
- **User Pool ID**: `ap-south-1_Ox0knQelv`
- **Client ID**: `5fktabakg95quea49h332e9ffs`
- **Client Secret**: `1gdj4r9p0hjf7qvqkem3qiakqvfr7e12h3mfci915m87d5iveq0k`
- **Region**: `ap-south-1`

### Step Functions Configuration:
- **Query Language**: JSONata
- **Retry Strategy**: Exponential backoff with jitter
- **Wait Duration**: 120 seconds (2 minutes)

## Issues Encountered

### Choice State Condition:
- **Problem**: Choice state condition `{% $userExists = false %}` not working correctly
- **Status**: Still routing to default (LoginLambda) instead of SignupLambda
- **Warning**: "Variable 'userExists' is possibly not defined"

### Alternative Conditions Tried:
- `{% $Payload.userExists = false %}`
- `{% $userExists == false %}`
- `{% $userExists === false %}`

## Current Status

✅ **Working Components**:
- All Lambda functions execute successfully
- State machine structure is correct
- Error handling and retries work
- Wait states function properly

⚠️ **Known Issues**:
- Choice state routing logic needs debugging
- Condition syntax may need adjustment

## Next Steps

1. **Debug Choice State**: Fix the condition syntax for proper routing
2. **Test Both Flows**: Ensure both signup and login paths work correctly
3. **Add Error Handling**: Implement better error handling for choice state
4. **Optimize Wait Times**: Adjust wait durations as needed

## Testing

### Test Cases:
1. **New User**: `userExists: false` → should go to SignupLambda
2. **Existing User**: `userExists: true` → should go to LoginLambda

### Test Input:
```json
{
  "username": "test@example.com",
  "password": "testpassword123"
}
```

---

*This documentation covers the complete implementation of the AWS Step Functions authentication workflow with Cognito integration.*
