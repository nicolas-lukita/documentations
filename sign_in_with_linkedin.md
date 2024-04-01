## Prerequisite

- LinkedIn Company Page

## Initial Setup

#### Create a LinkedIn App

1. Go to LinkedIn developer page https://www.linkedin.com/developers
2. Navigate to “My apps” and create an app connected to LinkedIn Company Page
3. Verify the app

#### Navigate to “Products” on the just created app and add “Sign in with LinkedIn” and “Share on LinkedIn”

#### Navigate to “Auth” and get client ID and client secret

#### Request access for “Sign in with LinkedIn” and “Share on LinkedIn” OAuth scopes and add redirect URL

## Server Side
<img width="452" alt="image" src="https://github.com/nicolas-lukita/documentations/assets/76612512/8a6dd520-cb5e-4d59-8788-e3bc7da75010">

#### Get Access Token

` POST https://www.linkedin.com/oauth/v2/accessToken`

| Parameter     | Type   | Description                                                                                |
| ------------- | ------ | ------------------------------------------------------------------------------------------ |
| grant_type    | String | authorization_code                                                                         |
| client_id     | String | Client ID from Initial Setup                                                               |
| client_secret | String | Client Secret from Initial Setup                                                           |
| redirect_uri  | url    | Redirected uri set on the Initial Setup                                                    |
| code          | String | Authorization code passed by client side (https://www.linkedin.com/oauth/v2/authorization) |

#### Successful response will contain: access_token, expires_in, and scope

#### Using the access_token and query parameters from previous step, we can get user data by requesting the following endpoints:

| Target        | Description           | Endpoint                                                                                   |
| ------------- | --------------------- | ------------------------------------------------------------------------------------------ |
| User Detail   | id firstName lastName | https://api.linkedin.com/v2/me                                                             |
| Email         | email address         | https://api.linkedin.com/v2/emailAddress?q=members&projection=(elements*(handle~))         |
| Display Image | profile picture       | https://api.linkedin.com/v2/me?projection=(profilePicture(displayImage~:playableStreams))  |
| redirect_uri  | url                   | Redirected uri set on the Initial Setup                                                    |
| code          | String                | Authorization code passed by client side (https://www.linkedin.com/oauth/v2/authorization) |

## Backend - Spring Boot

#### Store token and client id safely in application.yaml (can set up for other different environments: dev, local, prod)

```
apple:
client-id: #client id from Initial Setup step
client-secret: #client secret from Initial Setup step
redirect-url: https://nicolaslu.com
```

#### getAccessToken(authCode)

1. Function that handles post request to https://www.linkedin.com/oauth/v2/accessToken as the “Server Side” step above
2. The response will include access token which will be used for authenticating endpoints

#### getAuthEntity(authCode)

Helper function to generate HttpEntity with parameter and token from getAccessToken()

#### login(authCode)

1. Get user’s LinkedIn id and name by using user detail endpoint and HttpEntity from getAuthEntity()
2. Get user’s email address by using email endpoint and HttpEntity from getAuthEntity()
3. Check the user database and process accordingly
   1. User with LinkedIn Id exist: Generate token for login
   2. No user with LinkedIn Id but matching email: Link LinkedIn Id to the user and generate token for login
   3. No user found with matching LinkedIn Id nor email: Returns new UserInfo back to the client side with all user data fetched from endpoints
4. unlinkAccount(email)
   Unlink/remove the LinkedIn Id from the UserInfo
5. linkAccount(authCode, email)
   1. Get user LinkedIn Id and email from getAuthEntity()
   2. Check if other user with the LinkedIn Id already exist
   3. Check if current user already link with other LinkedIn Id
   4. If not, Link the LinkedIn Id to this user

## FrontEnd - Flutter

#### Install “LinkedIn login” package by running

`flutter pub add linkedin_login`

#### Get LinkedIn credentials by using package’s LinkedInAuthCodeWidget

```
LinkedInAuthCodeWidget(
useVirtualDisplay: true,
destroySession: false,
redirectUrl: envConfig.linkedInRedirectUrl,
clientId: envConfig.linkedInClientId,
onError: (final AuthorizationFailedAction e) {},
onGetAuthCode: \_linkedInLoginCallback,
)
```

1. Use the redirectUrl and clientId from previous steps
2. Set up callback functions for error and success

#### Handle login, link, and unlink events by passing the authCode received from previous step to the BE
