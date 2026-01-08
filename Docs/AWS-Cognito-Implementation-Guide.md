# AWS Cognito Implementation Documentation for Cruddur Application

## Overview
This documentation explains how AWS Cognito authentication is implemented in the Cruddur React application using AWS Amplify v6. AWS Cognito provides user authentication, registration, and password management services.

## 1. Configuration Setup

### Environment Variables (docker-compose.yml)
```yaml
frontend-react-js:
  environment:
    REACT_APP_AWS_PROJECT_REGION: "${AWS_REGION}"
    REACT_APP_AWS_COGNITO_IDENTITY_POOL_ID: "${REACT_APP_AWS_COGNITO_IDENTITY_POOL_ID}"
    REACT_APP_AWS_COGNITO_REGION: "${REACT_APP_AWS_COGNITO_REGION}"
    REACT_APP_AWS_USER_POOLS_ID: "${REACT_APP_AWS_USER_POOLS_ID}"
    REACT_APP_AWS_CLIENT_ID: "${REACT_APP_AWS_CLIENT_ID}"
```

**What this does:**
- Sets up environment variables that contain your AWS Cognito configuration
- These variables are loaded from your system environment into the Docker container
- The React app uses these to connect to your specific Cognito User Pool

### Amplify Configuration (App.js)
```javascript
import { Amplify } from 'aws-amplify';

Amplify.configure({
  Auth: {
    Cognito: {
      userPoolId: process.env.REACT_APP_AWS_USER_POOLS_ID,
      userPoolClientId: process.env.REACT_APP_AWS_CLIENT_ID,
      identityPoolId: process.env.REACT_APP_AWS_COGNITO_IDENTITY_POOL_ID,
    }
  }
});
```

**What this does:**
- Initializes AWS Amplify with your Cognito configuration
- Tells Amplify which User Pool and Client to use for authentication
- This runs once when the app starts, setting up the connection to AWS Cognito

## 2. User Registration (SignupPage.js)

### Import Statement
```javascript
import { signUp } from '@aws-amplify/auth';
```

### Registration Function
```javascript
const onsubmit = async (event) => {
  event.preventDefault();
  setCognitoErrors('')
  try {
    const { userId } = await signUp({
      username: email,
      password: password,
      options: {
        userAttributes: {
          name: name,
          email: email,
          preferred_username: username,
        },
        autoSignIn: true
      }
    });
    console.log(userId);
    window.location.href = `/confirm?email=${email}`
  } catch (error) {
    console.log(error);
    setCognitoErrors(error.message)
  }
  return false
}
```

**What this does:**
- Prevents the default form submission behavior
- Clears any previous error messages
- Calls AWS Cognito to create a new user account with:
  - Email as username
  - Password
  - Additional attributes (name, preferred_username)
- If successful, redirects to confirmation page
- If there's an error, displays the error message to the user

## 3. Email Confirmation (ConfirmationPage.js)

### Import Statements
```javascript
import { confirmSignUp, resendSignUpCode } from '@aws-amplify/auth';
```

### Confirmation Function
```javascript
const onsubmit = async (event) => {
  event.preventDefault();
  setErrors('')
  try {
    await confirmSignUp({ username: email, confirmationCode: code });
    window.location.href = "/"
  } catch (error) {
    setErrors(error.message)
  }
  return false
}
```

**What this does:**
- Takes the confirmation code sent to user's email
- Verifies the code with AWS Cognito
- If valid, activates the user account and redirects to home page
- If invalid, shows an error message

### Resend Code Function
```javascript
const resend_code = async (event) => {
  setCognitoErrors('')
  try {
    await resendSignUpCode({ username: email });
    console.log('code resent successfully');
    setCodeSent(true)
  } catch (err) {
    console.log(err)
    if (err.message === 'Username cannot be empty'){
      setCognitoErrors("You need to provide an email in order to send Resend Activiation Code")   
    } else if (err.message === "Username/client id combination not found."){
      setCognitoErrors("Email is invalid or cannot be found.")   
    }
  }
}
```

**What this does:**
- Requests AWS Cognito to send a new confirmation code
- Shows success message if code is sent
- Handles specific error cases with user-friendly messages

## 4. User Sign In (SigninPage.js)

### Import Statement
```javascript
import { signIn } from '@aws-amplify/auth';
```

### Sign In Function
```javascript
const onsubmit = async (event) => {
  setErrors('')
  event.preventDefault();
  try {
    const { isSignedIn } = await signIn({ username: email, password });
    if (isSignedIn) {
      window.location.href = "/"
    }
  } catch (error) {
    if (error.name === 'UserNotConfirmedException') {
      window.location.href = "/confirm"
    }
    setErrors(error.message)
  }
  return false
}
```

**What this does:**
- Attempts to sign in the user with email and password
- If successful, redirects to home page
- If user hasn't confirmed their email, redirects to confirmation page
- Shows error messages for invalid credentials

## 5. Password Recovery (RecoverPage.js)

### Import Statements
```javascript
import { resetPassword, confirmResetPassword } from '@aws-amplify/auth';
```

### Send Recovery Code
```javascript
const onsubmit_send_code = async (event) => {
  event.preventDefault();
  setCognitoErrors('')
  try {
    await resetPassword({ username });
    setFormState('confirm_code');
  } catch (err) {
    setCognitoErrors(err.message);
  }
  return false
}
```

**What this does:**
- Requests AWS Cognito to send a password reset code to user's email
- Changes the form to show the code input fields
- Handles errors if email doesn't exist

### Confirm New Password
```javascript
const onsubmit_confirm_code = async (event) => {
  event.preventDefault();
  setCognitoErrors('')
  if (password === passwordAgain){
    try {
      await confirmResetPassword({ username, confirmationCode: code, newPassword: password });
      setFormState('success');
    } catch (err) {
      setCognitoErrors(err.message);
    }
  } else {
    setCognitoErrors('Passwords do not match')
  }
  return false
}
```

**What this does:**
- Verifies that both password fields match
- Confirms the password reset with the code from email
- Sets the new password in AWS Cognito
- Shows success message when complete

## 6. Authentication Check (HomeFeedPage.js)

### Import Statement
```javascript
import { getCurrentUser } from '@aws-amplify/auth';
```

### Check Authentication Status
```javascript
const checkAuth = async () => {
  try {
    const user = await getCurrentUser();
    console.log('user', user);
    setUser({
      display_name: user.signInDetails?.loginId || user.username,
      handle: user.username
    });
  } catch (err) {
    console.log('Not authenticated:', err);
  }
};
```

**What this does:**
- Checks if a user is currently signed in
- If signed in, gets user information and stores it in component state
- If not signed in, logs the error (user sees login options)
- This runs when the home page loads

## 7. Sign Out (ProfileInfo.js)

### Import Statement
```javascript
import { signOut as amplifySignOut } from '@aws-amplify/auth';
```

### Sign Out Function
```javascript
const signOut = async () => {
  try {
    await amplifySignOut({ global: true });
    window.location.href = "/"
  } catch (error) {
    console.log('error signing out: ', error);
  }
}
```

**What this does:**
- Signs the user out from AWS Cognito
- `global: true` means sign out from all devices
- Redirects to home page after sign out
- Handles any errors during sign out

## Key Concepts Explained

### State Management
Each component uses React's `useState` to manage:
- Form input values (email, password, etc.)
- Error messages
- Loading states
- User information

### Error Handling
All AWS Cognito operations are wrapped in try-catch blocks to:
- Display user-friendly error messages
- Handle specific error types (like unconfirmed users)
- Prevent app crashes

### Navigation
The app uses `window.location.href` to redirect users:
- After successful registration → confirmation page
- After successful sign in → home page
- After sign out → home page

### Security Features
- Passwords are never stored locally
- All authentication happens through AWS Cognito
- User sessions are managed by AWS
- Email verification is required for new accounts

## Implementation Summary

This implementation provides a complete authentication system with:

1. **User Registration** - New users can create accounts
2. **Email Verification** - Users must confirm their email addresses
3. **User Sign In** - Authenticated access to the application
4. **Password Recovery** - Users can reset forgotten passwords
5. **Session Management** - Automatic handling of user sessions
6. **Sign Out** - Secure logout functionality

All authentication operations are handled securely through AWS Cognito's infrastructure, ensuring industry-standard security practices are followed.

## File Structure

```
frontend-react-js/src/
├── App.js                    # Amplify configuration
├── pages/
│   ├── SignupPage.js        # User registration
│   ├── ConfirmationPage.js  # Email confirmation
│   ├── SigninPage.js        # User sign in
│   ├── RecoverPage.js       # Password recovery
│   └── HomeFeedPage.js      # Authentication check
└── components/
    └── ProfileInfo.js       # Sign out functionality
```

This documentation covers the complete AWS Cognito implementation in the Cruddur application using AWS Amplify v6.