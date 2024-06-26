## Prerequisites
- Apple Developer Program membership

## Apple Setup
#### 1. Register App ID
1. Register the App ID in https://developer.apple.com/account/resources/identifiers/list/bundleId
2. Set the app’s description and bundle identifier
3. Enable “Sign in with Apple” capabilities
<img width="975" alt="img" src="https://github.com/nicolas-lukita/documentations/assets/76612512/00319a57-5057-41b4-8942-f45795d365da">

#### 2. Create the Service ID
1. Create Service ID in https://developer.apple.com/account/resources/identifiers/list/serviceId
2. Set the description and identifier
3. After the service ID is created, enable “Sign in with Apple”
  a. Link the Service ID to the App ID
  b. Add Domains and Subdomains of the website that will use the SIWA (Sign in with Apple).
  c. This section still needs to be filled even if there is no intention to use SIWA on any website.
  d. Add Return URLs (Callback URL for Android/web)
  e. Save the changes
#### 3. Create Key
1. Create key in https://developer.apple.com/account/resources/authkeys/list
2. Enable SIWA for the key and link it to the App ID
   <img width="935" alt="2" src="https://github.com/nicolas-lukita/documentations/assets/76612512/e39d87a8-5efc-406d-886a-a10f485e4943">

#### 4. Generate Client Secret
1. Download the key from previous step
2. Rename the (key-name).p8 to key.txt
3. On the terminal, go to directory where the key is located and install JWT Gem (sudo gem install jwt)
4. Create a file named client_secret.rb and open it in a text editor
5. Code: 

```
require 'jwt'
key_file = 'key.txt'
team_id = 'team-id'
client_id = 'app-id-identifier' #com.nicolaslu.appname
key_id = 'key-id'
ecdsa_key = OpenSSL::PKey::EC.new IO.read key_file
headers = {
'kid' => key_id
}
claims = {
    'iss' => team_id,
    'iat' => Time.now.to_i,
    'exp' => Time.now.to_i + 86400*180,
    'aud' => 'https://appleid.apple.com',
    'sub' => client_id,
}
token = JWT.encode claims, ecdsa_key, 'ES256', headers
puts token
```

| Variable   | Description                                                           |
|------------|-----------------------------------------------------------------------|
| team_id    | Can be found on top right of Apple Developer account or inside App ID |
| client_id  | The app’s bundle identifier                                           |
| key_id     | The private key identifier                                            |

#### 5. Save the file and run ruby client_secret.rb to generate JWT token
#### 6. Enable SIWA capabilities on XCode client side project’s runner

## Configure Apple Private Relay Communication
When users created their account for the first time using SIWA and choose to hide their email address, Apple will provide a private relay email instead which can be used to relay email to user.
1. Go to Services on Apple developer page (https://developer.apple.com/account/resources/services)
2. Configure “Sign in with Apple for Email Communication”
<img width="1037" alt="image" src="https://github.com/nicolas-lukita/documentations/assets/76612512/df7c7beb-8311-45f4-a7cc-2babe4926bde">

3. Add email sources using domains or email address
<img width="598" alt="image" src="https://github.com/nicolas-lukita/documentations/assets/76612512/e2ef7783-fc7f-4165-89c8-60c289cfe1d3">

4. Check and reverify SPF if the email sources didn’t passed the SPF check
<img width="939" alt="image" src="https://github.com/nicolas-lukita/documentations/assets/76612512/b5e873d4-4227-4340-9448-beb77b52030d">

## Server Side
<img width="858" alt="image" src="https://github.com/nicolas-lukita/documentations/assets/76612512/00a0448f-c7e3-4b40-b810-caa2c049cede">

#### 1. Store token and client id safely

```
apple:
  client-id: com.nicolaslu.appname #app bundle identifier
  client-secret: #JWT token generated from initial setup step 5
  redirect-url: https://nicolaslu.com
```

#### 2. Set up post request to https://appleid.apple.com/auth/token with these query parameters
![siwaqp](https://github.com/nicolas-lukita/documentations/assets/76612512/52a7623a-de75-438c-b4c5-d16d79c0e963)
| Parameter     | Description                                    |
|---------------|------------------------------------------------|
| client_id     | The app bundle identifier for iOS, app service identifer for Android/web                      |
| client_secret | The JWT token we generated previously          |
| grant_type    | Set to “authorization_code”                    |
| redirect_uri  | The redirect url                               |
| code          | The authorization code passed from Client Side |
#### 3. Decode the response’s id_token payload. The payload will contains the user’s Apple Id (“sub”) and email/private relay email (“email”) 
<img width="376" alt="siwajwtres" src="https://github.com/nicolas-lukita/documentations/assets/76612512/680698e6-8756-43d6-9674-113edb55fd7e">

#### 4. Use the Apple Id and the email for processing the user login.

## Android
To enable SIWA for android, set up a callback endpoint to relay the credential back to the android app from webview.
In this post endpoint, redirect the user to:
```
intent://callback?$requestBody#Intent;package=$packageIdentifier;scheme=signinwithapple;end
```
Where `requestBody` is the post request body and `packageIdentifier` is the android app identifier.

**Use the app `service id` instead of bundle id as the client_id for android/web**

## Backend - Spring Boot
#### 1. Store token and client id safely in application.yaml (can set up for other different environments: dev, local, prod)

```
apple:
  client-id: com.nicolaslu.appname #app bundle identifier
  client-secret: #JWT token generated from initial setup step 5
  redirect-url: https://nicolaslu.com
```
#### 2. getAccessToken(authCode)
  1. Function that handles post request to  `https://appleid.apple.com/auth/token` with the same query parameters as step 2 on Server Side above
![siwaqp](https://github.com/nicolas-lukita/documentations/assets/76612512/adbdecf1-b02f-40d4-9366-ab770a5f1eb8)

  2. The response will include identity token which contains user’s Apple Id and email

#### 3. login(authCode)
  1. Get user’s email and Apple Id by running getAccessToken() and decode the JWT token received
  2. Check the user database and process accordingly
    a. User with Apple Id exist: Generate token for login
    b. No user with Apple Id but matching email: Link Apple Id to the user and generate token for login
    c. No user found with matching Apple Id nor email: Returns new UserInfo back to the client side with email and Apple Id (Continue register flow in Client side)
#### 4. unlinkAccount(email)
Unlink/remove the Apple Id from the UserInfo
#### 5. linkAccount(authCode, email)
  1. Get user Apple Id and email from getAuthToken
  2. Check if other user with the Apple Id already exist
  3. Check if current user already link with other Apple Id
  4. If not, Link the Apple Id to this user
#### 6. callback(requestBody)
Request body received would be in form of string: `code=...&id_token=...`
```
@PostMapping
    fun redirectPost(@RequestBody requestBody: String): ResponseEntity<String> {
        try {
            println("Received request body: $requestBody")

            val packageIdentifier = ANDROID_APP_IDENTIFIER
            val redirectUrl = "intent://callback?$requestBody#Intent;package=$packageIdentifier;scheme=signinwithapple;end"

            println("Redirecting to: $redirectUrl")
            return ResponseEntity.status(HttpStatus.FOUND).header("Location", redirectUrl).build()
        } catch (e: Exception) {
            println("Error processing request: ${e.message}")
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Error processing request: ${e.message}")
        }
    }
```
## Frontend - Flutter
#### 1. Install “Sign in with Apple” package by running
`flutter pub add sign_in_with_apple`
#### 2. Create an async function to get Apple credentials by using the package

```
Future<AuthorizationCredentialAppleID> getAppleCredential() async {
    try {
      final AuthorizationCredentialAppleID credential =
          await SignInWithApple.getAppleIDCredential(
              scopes: [
            AppleIDAuthorizationScopes.email,
            AppleIDAuthorizationScopes.fullName,
          ],
              webAuthenticationOptions: WebAuthenticationOptions(
                  clientId: "${envConfig.appIdentifier}-service",
                  redirectUri: Uri.parse("https://nicolaslu.com")));
      return credential;
    } on PlatformException catch (e) {
      throw Exception('Failed to get Apple ID Credential: ${e.message}');
    }
  }
```
  1. Set the scopes of credentials we need from the client (email and fullName)
  2. [Optional] Set up webAuthenticationOptions for using SIWA on web
  3. The returned credentials will be:
    a. authorizationCode (Will be passed to BE)
    b. email (if user allows, will returned user’s email ; null other wise)
    c. familyName 
    d. givenName 
    e. identityToken (Same token we got on Server Side step 2)
    f. userIdentifier (Apple Id)
    g. state
  4. Email, FamilyName, and GivenName will only be retrieved if user allows it on login process and this values will only showed ONCE on the first time user use SIWA on the app. (Might be best to cache the data)
#### 3. Handle login, link, unlink events by using the credentials received from Step 3 and pass the required data to the BE

## Note: 
When in development, some data received from Apple can only be received once at the first time user login using SIWA. 
To refresh this, follow these steps:
 Settings > Sign in & Security > Sign in with Apple > remove the relevant app > Log out > Log in
