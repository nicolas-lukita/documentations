## Prerequisites
- Apple Developer Program membership

## Initial Setup
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
  d. Add Return URLs 
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
client_id = 'app-id-identifier' #com.synpulse.pulse8app
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

- team_id can be found on top right of Apple Developer account or inside App ID
- client_id is the app’s bundle identifier
- key_id is the private key identifier

#### 5. Save the file and run ruby client_secret.rb to generate JWT token
#### 6. Enable SIWA capabilities on XCode client side project’s runner
 
## Server Side
si![siwafc](https://github.com/nicolas-lukita/documentations/assets/76612512/ec302c78-707d-44e9-a1ba-e365d1d964ec)

#### 1. Store token and client id safely

```
apple:
  client-id: com.synpulse.pulse8app #app bundle identifier
  client-secret: #JWT token generated from initial setup step 5
  redirect-url: https://synpulse.com
```

#### 2. Set up post request to https://appleid.apple.com/auth/token with these query parameters
![siwaqp](https://github.com/nicolas-lukita/documentations/assets/76612512/d1be9fef-f339-4d60-99db-3ec87722bd65)
  - client_id is the app bundle identifier
  - client_secret is the JWT token we generated previously
  - grant_type set to “authorization_code”
  - redirect_uri is the redirect url
  - code is the authorization code passed from Client Side
#### 3. Decode the response’s id_token payload. The payload will contains the user’s Apple Id (“sub”) and email/private relay email (“email”) 
<img width="376" alt="siwajwtres" src="https://github.com/nicolas-lukita/documentations/assets/76612512/287df48c-5b12-423a-87aa-2b064577eba2">

#### 4. Use the Apple Id and the email for processing the user login.

## Backend - Spring Boot
This part is about the Pulse8 app backend Apple authentication service flow.
#### 1. Store token and client id safely in application.yaml (can set up for other different environments: dev, local, prod)

```
apple:
  client-id: com.synpulse.pulse8app #app bundle identifier
  client-secret: #JWT token generated from initial setup step 5
  redirect-url: https://synpulse.com
```
#### 2. getAccessToken(authCode)
  1. Function that handles post request to  `https://appleid.apple.com/auth/token` with the same query parameters as step 2 on Server Side above
![siwaqp](https://github.com/nicolas-lukita/documentations/assets/76612512/488ab9a3-2947-4af4-998d-537c4edf4a18)

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
##Frontend - Flutter
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
                  redirectUri: Uri.parse("https://synpulse.com")));
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