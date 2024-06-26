# Kong plugin | `x-custom-jwt`: How to craft and sign a custom JWT
1) Craft a custom JWT using the input Authentication properties
2) Load the private JWK from the plugin's configuration and convert it into a PEM format
3) Sign the JWT with the PEM string for building a JWS
4) Add the custom JWT to an HTTP Request Header backend API

The plugin `x-custom-jwt` doesn't check the validity of the input itself (neither checking of JWT signature & JWT expiration, nor user/password checking, nor checking Client TLS checking, nor api key checking). So it's **mandatory to use this plugin in conjunction with one of the Kong's security plugins**:
- [OIDC](https://docs.konghq.com/hub/kong-inc/openid-connect/)
- [JWT validation](https://docs.konghq.com/hub/kong-inc/jwt/)
- [Basic Authentication](https://docs.konghq.com/hub/kong-inc/basic-auth/)
- [Mutual TLS Authentication](https://docs.konghq.com/hub/kong-inc/mtls-auth/)
- [Key Authentication](https://docs.konghq.com/hub/kong-inc/key-auth/)

## `x-custom-jwt` plugin configuration reference
|FORM PARAMETER                 |DEFAULT          |DESCRIPTION                                                 |
|:------------------------------|:----------------|:-----------------------------------------------------------|
|config.apikey_header|apikey|The Http header name to get the `apiKey` for Key Authentication|
|config.bearer_clientid_claim|clientId|The claim name to extract the `clientId` (from a JWT) for Authentication Bearer|
|config.upstream_custom_jwt_header|X-Custom-Jwt|The Http header name where to drop the custom JWT in the upstream request. It overrides any existing value. If the value is `Authorization` the `Bearer` type is added in the value|
|config.downstream_custom_jwt_header|N/A|The Http header name where to drop the custom JWT in the downstream response. It overrides any existing value|
|config.expires_in|1800|Number of seconds for the `exp` (expiration) claim which are added to the current time|
|config.iss|https://kong-gateway:8443/x-custom-jwt|The `iss` (issuer) claim that identifies the Kong Gateway that issued the custom JWT|
|config.jku|https://kong-gateway:8443/x-custom-jwt/jwks|The `jku` (JWK set Url) that points to a Kong Gateway route for delivering the well-known location of JWKs|
|config.private_jwk|{"kty": "RSA","kid": "kong",...<***CHANGE_ME***>}|The JWK private key to sign the custom JWT. The format is JSON. RS256, RS512, SH256, SH512 algorithms are supported|
|config.verbose|false|Append to the Consumer a detailed message in case of error|

## High level algorithm of `x-custom-jwt` to craft and sign a custom JWT
```lua
-- Try to find one by one an Authentication given by the Consumer
find "Authorization: Bearer" header
if not "Authorization: Bearer" then
  find "Authorization: Basic" header
end
if not "Authorization: Bearer" and not "Authorization: Basic" then
  find "mTLS Client Certificate"
end
if not "Authorization: Bearer" and not "Authorization: Basic" and not "mTLS Client Certificate" then
  find "apikey" header
end

if not "Authorization: Bearer" and not "Authorization: Basic" and not "mTLS Client Certificate" and not "apikey" then
  -- The Consumer's request is blocked
  return "HTTP 401", "You are not authorized to access to this service"
end

-- If the Consumer sends a correct Authentication, we craft the 'x-custom-jwt' JWT
-- The 'client_id' claim has a value depending of the Consumer's Authentication
if "Authorization: Bearer" then
  -- Copy all the content of the AT (given by 'Authorization: Bearer')
  x-custom-jwt.payload = AT.payload
  x-custom-jwt.payload.client_id = x-custom-jwt.payload[plugin_conf.bearer_clientid_claim]
else if "Authorization: Basic" then
  -- Copy the username of the Basic Auth
  x-custom-jwt.payload.client_id = AT.payload.username
else if "mTLS Client Certificate" then
  -- Copy the entire subjectDN (all distinguished names) of the Client Certificate
  x-custom-jwt.payload.client_id = subjectDN
else if "apikey" then
  -- Copy the 'apikey' value
  x-custom-jwt.payload.client_id = apiKeyValue
end

-- If the 'x-custom-jwt.client_id' is not set
if not x-custom-jwt.payload.client_id then
  -- The Consumer's request is blocked
  return "HTTP 401", "You are not authorized to consume this service. Internal Error"
end

-- Header values for all Authentication methods
x-custom-jwt.header.typ = "JWT",
x-custom-jwt.header.alg = "<JWK.alg>", -- Got from the alg of private JWK
x-custom-jwt.header.kid = "<JWK.kid>", -- Got from the kid of private JWK
x-custom-jwt.header.jku = "<jku>" -- Got from the plugin Configuration

-- Common claims for all Authentication methods
x-custom-jwt.payload.iss = "<iss>" -- Got from the plugin Configuration
x-custom-jwt.payload.iat = "<current time>"
x-custom-jwt.payload.exp = x-custom-jwt.payload.iat + "<expires_in>"  -- Got from the plugin Configuration
x-custom-jwt.payload.aud = "<url>" -- the Backend_Api URL
x-custom-jwt.payload.jti = "<uuid>" -- Generation of a 'Universally unique identifier'

-- 'act.sub' claim
x-custom-jwt.payload.act.client_id = "<kong-consumer-custom-id>" or "<kong-consumer-id>" -- Set by security plugins (OIDC, Basic Auth, Key Authentication, Mutual TLS Auth, etc.)

-- Sign the JWT with a private JWK (set in the plugin configuration) for building a JWS 
jws_x_custom_jwt = jwt:sign (x-custom-jwt, private_jwk)

-- Add the custom JWT in an HTTP header to the request's headers before sending the request to the Backend API
kong.service.request.set_header(plugin_conf.upstream_custom_jwt_header, jws_x_custom_jwt)
```
## How to test the `x-custom-jwt` plugin with Kong Gateway
### Prerequisites 
**In this repo, there is the [konnect.yaml](./decK/konnect.yaml) decK file related to the prerequisites and following examples.** If you prefrer you can do `deck gateway sync` and avoid manual declaration as explained below.

1) Install the [Kong Gateway](https://docs.konghq.com/gateway/latest/install/)
2) Install the `x-custom-jwt` plugin by following the documentation, [here](https://docs.konghq.com/gateway/latest/plugin-development/file-structure/)
3) Install [http.ie](https://httpie.io/)
4) Prepare the RSA256 JWK for getting the Public and Private Keypair
- You can use the JWK keypair provided in this repo:
  - JWKS Public Keys (JSON Web Key Set) Public Key: [RS256-jwks-public.json](./test-keys/RS256-jwks-public.json)
  - JWK Private Key: [RS256-jwk-private.json](./test-keys/RS256-jwk-private.json)
- **Or** create your own JWK keypair: go for instance to the site https://mkjwk.org/ and configure the online tool with following values:
  - key Size: `2048`
  - Key Use: `Signature`
  - Algorithm: `RS256`
  - Key-ID (kid): `kong`
- Click on Generate, copy to clipboard the `Public and Private Keypair` (i.e. Private Key) and the `Public Key`
5) Create a Route to deliver the public JWKS
- The Route has the following properties:
  - name=`x-custom-jwt-jwks`
  - path=`/x-custom-jwt/jwks`
  - Click on `Save`
- Add the `Request Termination` plugin to the Route with:
  - `config.status_code`=`200`
  - `config.content_type`=`application/json`
  - `config.body`=copy/paste the content of [RS256-jwks-public.json](./test-keys/RS256-jwks-public.json) **Or**
  - `Config.Body`=**The `Public JWK Key` must be pasted from https://mkjwk.org/ and add `"keys": [` property for having a JWKS**. If needed, adapt the `kid` to a custom value. JWKS Structure:
    ```json
    {
      "keys": [
        {
          *****  CHANGE ME WITH THE PUBLIC JWK *****
        }
      ]
    }
    ```
  - Click on `Save`
- Add the `CORS` plugin to the Route with:
  - config.origins=`*`
  - Click on `Save`
6) Create a Gateway Service
- For `httpbin` service, add a Gateway Service with:
  - name=`httpbin`
  - URL=`http://httpbin.apim.eu/anything` 
  - Click on `Save`
- Add a Route to the Service with:
  - name=`httpbin` 
  - path=`/httpbin`
  - Click on `Save`
- Add `x-custom-jwt` plugin to the Service with:
  - config.bearer_clientid_claim=`clientId` or `** Replace with the right claim for having the proper Kong consumer reconciliation **`
  - config.iss=`<adapt the URL to your environment>` (example: https://kong-gateway:8443/x-custom-jwt)
  - config.jku=`<adapt the URL to your environment>` (example: https://kong-gateway:8443/x-custom-jwt/jwks)
  - config.private_jwk=copy/paste the content of [RS256-jwk-private.json](./test-keys/RS256-jwk-private.json) **Or**
  - config.private_jwk=paste the `Public and Private Keypair` from https://mkjwk.org/. If needed, adapt the `kid` to a custom value; the `kid` value must be the same as defined in `Prerequisites` heading (see the configuration of `Request Termination` plugin)
  - config.verbose=`true`
7) Create a Consumer with:
- Username=`contact@konghq.com`
- Custom Id=`contact@konghq.com-ID1`
8) Create a `contact@konghq.com` Client in your IdP Server for Example #1

### Example #1: "Authorization: Bearer" input
1) Open the `httpbin` Service
2) Create a new Route:
- name=`oidc`
- path=`/oidc`
3) Add `OpenId Connect` plugin to the Route with:
- config.client_id=`** Replace with your Client Id **`
- config.client_secret=`** Replace with your Client Secret **`
- config.issuer: `** Replace with your /.well-known/openid-configuration URL **`
- config.auth_methods: `client_credentials` + `introspection`
- config.consumer_claim = `clientId` or `** Replace with the right claim for having the proper Kong consumer reconciliation **`
4) Test
- `Request`:
  ```shell
  http -a contact@konghq.com:<**YOUR_PASSWORD**> kong-gateway:8000/oidc
  ```
  or
  ```shell
  http kong-gateway:8000/oidc Authorization:'Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJxOEVFR3YweE9FQkt3eFNJYVZDNGpHTWxVcF8yWURhS1pfMVdZNHV3b2lRIn0.eyJleHAiOjE3MTI2ODU3MTYsImlhdCI6MTcxMjY4NTQxNiwianRpIjoiYWI2MmQwNjUtNDYyNy00NDllLTk4ZDAtNTA0MGYwYjI4OTNhIiwiaXNzIjoiaHR0cHM6Ly9zc28uYXBpbS5ldTo4NDQzL2F1dGgvcmVhbG1zL0plcm9tZSIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiJjYzE2M2ZmNS1iZmMxLTRkNmYtYTFjMS02YjAzZTI5NWY2MmYiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJjb250YWN0QGtvbmdocS5jb20iLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIm9mZmxpbmVfYWNjZXNzIiwiZGVmYXVsdC1yb2xlcy1qZXJvbWUiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoib3BlbmlkIGVtYWlsIHByb2ZpbGUiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImNsaWVudElkIjoiY29udGFjdEBrb25naHEuY29tIiwiY2xpZW50SG9zdCI6Ijg4LjE3NS45LjE0NiIsInByZWZlcnJlZF91c2VybmFtZSI6InNlcnZpY2UtYWNjb3VudC1jb250YWN0QGtvbmdocS5jb20iLCJjbGllbnRBZGRyZXNzIjoiODguMTc1LjkuMTQ2In0.AE3wHHhElQWnuDCJO_XYSvBw7RND4ZB8FpgB9wKlSR5Zbr3XyFwTrbtOdC5A6DAkMdcZ5s-sWg1qDVefM6k2qVe-gj2kmFcMBBt8DQPD7YBKbHdJGaPxqCDrNOrmhMt6MC7EldHd0rJ4beF7i49q4eCyYuSLCpKeS-eTFw5L-s98uGoRxgEZEocaZl9Atu_ajB84HQBpQ31Z0ObKwrMME7TU4nyOWXFYs7ZcGlhamjC2dmDiVJkxKL3ochq6jbnfQAkwjq6EVrK_0KPdMOANcwoYi0gg2TqDq0b16CSA8zUYYf0qVxV69Dyl_3tcWnqvHs7kzelQMeNMziOl4_GZMg'
  ```
- `Response`: expected value of `x-custom-jwt` plugin:
    * Base64 encoded:
    ```
    eyJqa3UiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dC9qd2tzIiwiYWxnIjoiUlMyNTYiLCJ0eXAiOiJKV1QiLCJraWQiOiJrb25nIn0.eyJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJkZWZhdWx0LXJvbGVzLWplcm9tZSIsInVtYV9hdXRob3JpemF0aW9uIl19LCJyZXNvdXJjZV9hY2Nlc3MiOnsiYWNjb3VudCI6eyJyb2xlcyI6WyJtYW5hZ2UtYWNjb3VudCIsIm1hbmFnZS1hY2NvdW50LWxpbmtzIiwidmlldy1wcm9maWxlIl19fSwic2NvcGUiOiJvcGVuaWQgZW1haWwgcHJvZmlsZSIsInN1YiI6ImNjMTYzZmY1LWJmYzEtNGQ2Zi1hMWMxLTZiMDNlMjk1ZjYyZiIsImF1ZCI6Imh0dHA6Ly9odHRwYmluLmFwaW0uZXUvYW55dGhpbmciLCJpYXQiOjE3MTI2ODU0MTYsImF6cCI6ImNvbnRhY3RAa29uZ2hxLmNvbSIsImNsaWVudEhvc3QiOiI4OC4xNzUuOS4xNDYiLCJjbGllbnRfaWQiOiJjb250YWN0QGtvbmdocS5jb20iLCJhY3IiOiIxIiwiYWN0Ijp7ImNsaWVudF9pZCI6ImNvbnRhY3RAa29uZ2hxLmNvbS1JRDEifSwicHJlZmVycmVkX3VzZXJuYW1lIjoic2VydmljZS1hY2NvdW50LWNvbnRhY3RAa29uZ2hxLmNvbSIsImNsaWVudEFkZHJlc3MiOiI4OC4xNzUuOS4xNDYiLCJpc3MiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dCIsImp0aSI6ImUyYTRhODBkLWNmOTgtNDU1Zi04YTZhLTQwY2Q2MTIzZGU1NSIsImNsaWVudElkIjoiY29udGFjdEBrb25naHEuY29tIiwiZXhwIjoxNzEyNjg3MjE2LCJ0eXAiOiJCZWFyZXIiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2V9.XnTIxPJ59YzpcQ8_Q-N_oyxQYwfJ-NQWY7iOXWmwehhyEotw0Z54r7n3zdm8GQXDHiqTkZFxtD7PnAwtphSRCWNnLe2lR0EV6nDMtElcHPViO-bs_ItFMKbLkKrzkWWNaUcK_oRAOMDB19QBvEaM0eoG6dE2o2a4KxQmtRxxDnB3RcTm5AABd7U9Iuqc6xrBoWBpWYdZakomWnJM78USvCB8sHAk4O2LdvzEbYElFuh-Z04sWE7OQVs_S4IF1xRTwaaeDKI2M3MVVvpIVaY-yUJ7tfXvxau4fc6fTEOGjqbv_oKqrKHHR8gcHJHsFppiKPpe2cn_S5xmiirHViBlaw
    ```
    * JSON decoded:
    ```json
    {
      "header": {
        "typ": "JWT",
        "alg": "RS256",
        "kid": "kong",
        "jku": "https://kong-gateway:8443/x-custom-jwt/jwks"
      },
      "payload": {
        "realm_access": {
          "roles": [
            "offline_access",
            "default-roles-jerome",
            "uma_authorization"
          ]
        },
        "resource_access": {
          "account": {
            "roles": [
              "manage-account",
              "manage-account-links",
              "view-profile"
            ]
          }
        },
        "scope": "openid email profile",
        "sub": "cc163ff5-bfc1-4d6f-a1c1-6b03e295f62f",
        "aud": "http://httpbin.apim.eu/anything",
        "iat": 1712685416,
        "azp": "contact@konghq.com",
        "clientHost": "88.175.9.146",
        "client_id": "contact@konghq.com",
        "acr": "1",
        "act": {
          "client_id": "contact@konghq.com-ID1"
        },
        "preferred_username": "service-account-contact@konghq.com",
        "clientAddress": "88.175.9.146",
        "iss": "https://kong-gateway:8443/x-custom-jwt",
        "jti": "e2a4a80d-cf98-455f-8a6a-40cd6123de55",
        "clientId": "contact@konghq.com",
        "exp": 1712687216,
        "typ": "Bearer",
        "email_verified": false
      },
      "signature": "xxxxx"
    }
    ```
### Example #2: "Authorization: Basic" input
1) Open the `httpbin` Service
2) Create a new Route:
- name=`basicAuth`
- path=`/basicAuth`
3) Add `Basic Authentication` plugin to the Route (Leave default parameters)
4) Open the `contact@konghq.com` consumer, go on Credentials / Basic Authentication, click on a `+ New Basic Auth Credential` and put:
- username=`my-user`
- password=`My p@ssword!`
5) Click on save
6) Test
- `Request`:

  ```shell
  http -a 'my-user:My p@ssword!' kong-gateway:8000/basicAuth
  ```
  **If you imported the configuration with decK you'll have a 401 error.** It's due to hashing of passwords, so you have to redefine it: see step #4
- `Response`: expected value of `x-custom-jwt` plugin:
    * Base64 encoded:
    ```
    eyJhbGciOiJSUzI1NiIsImprdSI6Imh0dHBzOi8va29uZy1nYXRld2F5Ojg0NDMveC1jdXN0b20tand0L2p3a3MiLCJ0eXAiOiJKV1QiLCJraWQiOiJrb25nIn0.eyJjbGllbnRfaWQiOiJteS11c2VyIiwiaWF0IjoxNzEyNzY3MTM2LCJhY3QiOnsiY2xpZW50X2lkIjoiY29udGFjdEBrb25naHEuY29tLUlEMSJ9LCJqdGkiOiI5MjAyNjhmOC05MzFlLTRkMjYtODUyMi1jNmJhMGJhMjkzY2IiLCJleHAiOjE3MTI3Njg5MzYsImlzcyI6Imh0dHBzOi8va29uZy1nYXRld2F5Ojg0NDMveC1jdXN0b20tand0IiwiYXVkIjoiaHR0cDovL2h0dHBiaW4uYXBpbS5ldS9hbnl0aGluZyJ9.N0g0hkUCbFuaccJS32TQJI02wIbMwC1Qj8UnaVbYahokfCulGZkPP9rwmSy73PYJM2vab6PLoqeKQ7XqUIUtIMSuvNS4W6fcEO1ilVt_2LQyqYFR3NDIRLjVf3_LyGWcExsxceon-8LGfrZN817GlLG5XbHzIXZXPDsdiAca_nnZgFaWK7BChF4IOpym7clHD4c6Uh0XDLEkgLzinZLRGm-PTy4REKq7yF3V913aMrS-gMaSDJbpGk6TWGEKERKoyGxvN8y2vH0y-6TA-XVWUM8U3Vdg-wjczlEbMXmlvFdVc2hRAsrjgb19vph4LH2NkvauZdBsP7UBXhz2dKWNkw
    ```
    * JSON decoded:
    ```json
    {
      "header": {
        "typ": "JWT",
        "alg": "RS256",
        "kid": "kong",
        "jku": "https://kong-gateway:8443/x-custom-jwt/jwks"
      },
      "payload": {
        "client_id": "my-user",
        "iat": 1712767136,
        "act": {
          "client_id": "contact@konghq.com-ID1"
        },
        "jti": "920268f8-931e-4d26-8522-c6ba0ba293cb",
        "exp": 1712768936,
        "iss": "https://kong-gateway:8443/x-custom-jwt",
        "aud": "http://httpbin.apim.eu/anything"
      },
      "signature": "xxxxx"
    }
    ```
### Example #3: "mTLS Client Certificate" input
1) Create a CA Certificate: open Certificates page and click `New CA Certficate`
- Copy/paste the content of `./mTLS/ca.cert.pem` in the CA field
- Click on Create
2) Open the CA Certificate just created and copy/paste its `ID`
3) Open the `httpbin` Service
4) Create a new a Route:
- name=`mtlsAuth`
- path=`/mtlsAuth`
5) Add `Mutual TLS Authentication` plugin to the Route and add to config.ca_certificates the `ÌD` of the CA cert
6) Click on Install
7) Check the presence of `contact@konghq.com` consumer (linked with the client certificate)
8) Click on create
9) Test
- `Request`:

  ```shell
  http --verify=no --cert=./mTLS/01.pem --cert-key=./mTLS/client.key https://localhost:8443/mtlsAuth
  ```
- `Response`: expected value of `x-custom-jwt` plugin:
    * Base64 encoded:
    ```
    eyJqa3UiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dC9qd2tzIiwiYWxnIjoiUlMyNTYiLCJ0eXAiOiJKV1QiLCJraWQiOiJrb25nIn0.eyJhY3QiOnsiY2xpZW50X2lkIjoiY29udGFjdEBrb25naHEuY29tLUlEMSJ9LCJqdGkiOiIzOGNhMGY5MS1mZGJmLTQ5OTMtOWE4Ni0yZDUwYWFiMWY3MGIiLCJpc3MiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dCIsImNsaWVudF9pZCI6IkM9VVMsIFNUPUNhbGlmb3JuaWEsIE89S29uZyBJbmMuLCBPVT1Lb25uZWN0LCBDTj1rb25nLCBlbWFpbEFkZHJlc3M9Y29udGFjdEBrb25naHEuY29tIiwiZXhwIjoxNzEyNjg3MjkzLCJpYXQiOjE3MTI2ODU0OTMsImF1ZCI6Imh0dHA6Ly9odHRwYmluLmFwaW0uZXUvYW55dGhpbmcifQ.EzLDNu05GJRlYptJlAMDppxGw_bsIsQDZTpqeATvlacDWKilQyx2wSIwwdzzuouCoJw2odl_bSN-8Le6AhVrP3Ao0Ak3XL9LTrIbuKbJa3EYZGelOit7KcT51VpJfRWewXOPgKuDY7zPCw9oHisnV-gCZKR_IfY3-7oj4ILHczIpBt6JB8D2FstCGYwTLI5CV9JWlIU1QZpCQMord48tAvcyjYRL3-1qafds9H6Ko-w6UUlIPR5YB0pKstctVzb1-zotE4rbVqYK0zorPlBEobxOBNlZ3ATqwckx-onOvvN6IZEy6J3yqWLFI9K0iSymvgaWu2w2ERIZCJCAqPNIwQ
    ```
    * JSON decoded:
    ```json
    {
      "header": {
        "typ": "JWT",
        "alg": "RS256",
        "kid": "kong",
        "jku": "https://kong-gateway:8443/x-custom-jwt/jwks"
      },
      "payload": {
        "act": {
          "client_id": "contact@konghq.com-ID1"
        },
        "jti": "38ca0f91-fdbf-4993-9a86-2d50aab1f70b",
        "iss": "https://kong-gateway:8443/x-custom-jwt",
        "client_id": "C=US, ST=California, O=Kong Inc., OU=Konnect, CN=kong, emailAddress=contact@konghq.com",
        "exp": 1712687293,
        "iat": 1712685493,
        "aud": "http://httpbin.apim.eu/anything"
      },
      "signature": "xxxxx"
    }
    ```
### Example #4: "Key Authentication" input
1) Open the `httpbin` Service
2) Create a new a Route:
- name=`apiKey`
- path=`/apiKey`
3) Add `Key Auth` plugin to the Route with:
- config.key_names=`apikey`
4) Open the `contact@konghq.com` consumer, click on Credentials / Key Authentication and click on `+ New Key Auth Credential`, put:
- Key=`012345AZERTY!`
5) Click on `Create`
6) Test
- `Request`:

  ```shell
  http kong-gateway:8000/apiKey apikey:'012345AZERTY!'
  ```
- `Response`: expected value of `x-custom-jwt` plugin:

    * Base64 encoded:
    ```
    eyJraWQiOiJrb25nIiwidHlwIjoiSldUIiwiYWxnIjoiUlMyNTYiLCJqa3UiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dC9qd2tzIn0.eyJhY3QiOnsiY2xpZW50X2lkIjoiY29udGFjdEBrb25naHEuY29tLUlEMSJ9LCJqdGkiOiJkNzMxNWU5YS1hYjJjLTRlOGMtYWRhOS0zYjEyYzNiNGYzZTQiLCJpc3MiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dCIsImF1ZCI6Imh0dHA6Ly9odHRwYmluLmFwaW0uZXUvYW55dGhpbmciLCJpYXQiOjE3MTI1OTY0OTMsImV4cCI6MTcxMjU5ODI5MywiY2xpZW50X2lkIjoiMDEyMzQ1QVpFUlRZISJ9.GxM-20uKCkkN06IVSLAyR97QsR2mpMXnaIZzvyuD_cQo5ETIw6Axkb0X8rmNtPONa27okdPB_xVV8XOHC2QSeF4p8h7LZzgZKUg1_7Ixjw4A0Xs5CrRk58aSxFP1EjBGGR7jL896sqtTjz2coJZ7q0ZTqcTG0VDvMCoxmVYa4G5XDm-zOABkFf-Cp4oWxMkFxF3b6m22rjQeI25_5NxJaNAJM6VFVBcmXF9wJTDiOie11eKScuYNRgoICp_XDgPpqLWET4DIPYYWCw_ZFG9vlckXBteTVdEZxvxLVvVtxcrANeDRN3RR0XcSByh5pOIa-2rsa7cUGEyGDVeS4pwIIQ
    ```
    * JSON decoded:
    ```json
    {
      "header": {
        "typ": "JWT",
        "alg": "RS256",
        "kid": "kong",
        "jku": "https://kong-gateway:8443/x-custom-jwt/jwks"
      },
      "payload": {
        "act": {
          "client_id": "contact@konghq.com-ID1"
        },
        "jti": "d7315e9a-ab2c-4e8c-ada9-3b12c3b4f3e4",
        "iss": "https://kong-gateway:8443/x-custom-jwt",
        "aud": "http://httpbin.apim.eu/anything",
        "iat": 1712596493,
        "exp": 1712598293,
        "client_id": "012345AZERTY!"
      },
      "signature": "xxxxx"
    }
    ```

## Check the JWS with the JWT plugin and build the `/introspect` Route
Use the [JWT plugin](https://docs.konghq.com/hub/kong-inc/jwt/) to check the JWS. The JWT plugin loads the credentials from the Consumer entity depending of algorithm:
  - For RSA (RS256, RS512): the Consumer credential uses the PEM format (derived from the Public JWK)
  - For HMAC (SH256, SH512): the Consumer credential uses the Symetric Key (see `k` value in the JWK)

Let's create an `/introspect` Route with the [JWT](https://docs.konghq.com/hub/kong-inc/jwt/) plugin and [Request Termination](https://docs.konghq.com/hub/kong-inc/request-termination/) plugin:
1) Get the Credential
- For RSA:
  - Get the content of [RS256-public.pem](./test-keys/RS256-public.pem) **or**
  - Build a PEM file from JWK public key by using the [convert-RSA-JWK-to-PEM.js](test-keys/convert-RSA-JWK-to-PEM.js)  node.js tool:
  ```
  node convert-RSA-JWK-to-PEM.js RS256-jwk-public.json
  ```
  - Put the content in a `.pem` file by including `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----`
- For HMAC:
  - Get the `k` content of [SH256-jwk-symetric-key.json](./test-keys/SH256-jwk-symetric-key.json)

2) Attach the JWT credential to the Consumer
- For Kong EE: use Kong Manager:
  - Open the `contact@konghq.com` consumer, click on Credentials / JWT and click on `+ New JWT Credential`, put:
    - For RSA (RS256):
      - algorithm=`RS256`
      - key=`my-user`
      - rsa_public_key=copy/paste the PEM content from step #1 ([RS256-public.pem](./test-keys/RS256-public.pem))
      - click on Save
    - For HMAC (SH256):
      - algorithm=`SH256`
      - key=`my-user`
      - secret=copy/paste the `k` content of [SH256-jwk-symetric-key.json](./test-keys/SH256-jwk-symetric-key.json)
      - click on Save
- For Konnect: use API:
  - For RSA (RS256):
    ```sh
    http https://us.api.konghq.com/konnect-api/api/runtime_groups/<**YOUR_RUNTIME_GROUP_ID**>/consumers/<**ID_OF_CONTACT@KONGHQ.COM_CONSUMER_ID**>/jwt Authorization:'Bearer kpat_<**YOUR_PAT**>' algorithm=RS256 key=my-user rsa_public_key=@RS256-public.pem
    ```
  - For HMAC (SH256):
    ```sh
    http https://us.api.konghq.com/konnect-api/api/runtime_groups/<**YOUR_RUNTIME_GROUP_ID**>/consumers/<**ID_OF_CONTACT@KONGHQ.COM_CONSUMER_ID**>/jwt Authorization:'Bearer kpat_<**YOUR_PAT**>' algorithm=HS256 key=my-user secret="<**SECRET_FROM_STEP_ABOVE**>"
    ```

3) Create the `/introspect` Route
- Create a new a Route (with **no Gateway Service**)
  - name=`Introspect`
  - path=`/introspect`
  - click on Save
- Add the `JWT` plugin to the Route with:
  - config.claims_to_verify=`exp`
  - config.key_claim_name=`client_id`
  - click on Save
- Add the `Request Termination` plugin to the Route with:
  - config.status_code=`200`
  - config.body=`{"message": "Ok"}`
  - config.content_type=`application/json`

4) Get a new `x-custom-jwt` with the Basic Authorization
- `Request`:
  ```shell
  http -a 'my-user:My p@ssword!' kong-gateway:8000/basicAuth
  ```
- `Response`: copy/paste the `x-custom-jwt` value
  * Base64 encoded:
    ```
    eyJ0eXAiOiJKV1QiLCJqa3UiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dC9qd2tzIiwia2lkIjoia29uZyIsImFsZyI6IlJTMjU2In0.eyJjbGllbnRfaWQiOiJteS11c2VyIiwiaXNzIjoiaHR0cHM6Ly9rb25nLWdhdGV3YXk6ODQ0My94LWN1c3RvbS1qd3QiLCJhdWQiOiJodHRwOi8vaHR0cGJpbi5hcGltLmV1L2FueXRoaW5nIiwiaWF0IjoxNzEzMzc0MTU2LCJqdGkiOiIwYjFjNTdmNy01NjBlLTRjYmYtODAzMi1lMTdlMGU4MjIzYWYiLCJleHAiOjE3MTMzNzU5NTYsImFjdCI6eyJjbGllbnRfaWQiOiJjb250YWN0QGtvbmdocS5jb20tSUQxIn19.UoaBk5PPl2GkwKOGClD0k7bfjpvp_7e9-n4oNhRZHyv-Om1w2qriTN-ehCBlOIrECY4drFgsU3NE0EGelmhwmRcKz3K9u79fL0R1IgfZ2sI_sEh2SVau7lquWFFV-B-WWe1uY53CVBOF1XJBmUCEHxDLBr1zCVA-izXq7Xfr9-ls3Hb1j2B09GdhXSXroxIniu4pcWZHN4GP-nl5rmxo1eASvs5JJo7HD3G4ivczEy_ujtEtVUGQ23zBLdEzUxr9p91gjTeao9l7uQwhCjSuLEXcfHuU8A2OcgIS1r3rbpRSoyCEcSCg7al2FjsGeaGUzvXGe4xssTOGBwSVXtxApw
    ```
5) Check the `x-custom-jwt` with `/introspect` route
- `Request`:
  ```shell
  http kong-gateway:8000/introspect Authorization:'Bearer eyJ0eXAiOiJKV1QiLCJqa3UiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dC9qd2tzIiwia2lkIjoia29uZyIsImFsZyI6IlJTMjU2In0.eyJjbGllbnRfaWQiOiJteS11c2VyIiwiaXNzIjoiaHR0cHM6Ly9rb25nLWdhdGV3YXk6ODQ0My94LWN1c3RvbS1qd3QiLCJhdWQiOiJodHRwOi8vaHR0cGJpbi5hcGltLmV1L2FueXRoaW5nIiwiaWF0IjoxNzEzMzc0MTU2LCJqdGkiOiIwYjFjNTdmNy01NjBlLTRjYmYtODAzMi1lMTdlMGU4MjIzYWYiLCJleHAiOjE3MTMzNzU5NTYsImFjdCI6eyJjbGllbnRfaWQiOiJjb250YWN0QGtvbmdocS5jb20tSUQxIn19.UoaBk5PPl2GkwKOGClD0k7bfjpvp_7e9-n4oNhRZHyv-Om1w2qriTN-ehCBlOIrECY4drFgsU3NE0EGelmhwmRcKz3K9u79fL0R1IgfZ2sI_sEh2SVau7lquWFFV-B-WWe1uY53CVBOF1XJBmUCEHxDLBr1zCVA-izXq7Xfr9-ls3Hb1j2B09GdhXSXroxIniu4pcWZHN4GP-nl5rmxo1eASvs5JJo7HD3G4ivczEy_ujtEtVUGQ23zBLdEzUxr9p91gjTeao9l7uQwhCjSuLEXcfHuU8A2OcgIS1r3rbpRSoyCEcSCg7al2FjsGeaGUzvXGe4xssTOGBwSVXtxApw'
  ```
- `Response`:
  - The JWT is valid:
    ```json
    "HTTP/1.1 200 OK"
    
    {
      "message": "Ok"
    }
    ```
  - The JWT has an Invalid signature:
    ```json
    "HTTP/1.1 401 Unauthorized"
    {
      "message": "Invalid signature"
    }
    ```
  - The JWT has expired:
    ```json
    "HTTP/1.1 401 Unauthorized"
    {
      "exp": "token expired"
    }
    ```
  - The JWT does not have a common `client_id` value in the Consumer JWT credentials and in `x-custom-jwt`:
    ```json
    "HTTP/1.1 401 Unauthorized"
    {
      "message": "No credentials found for given 'client_id'"
    }
    ```

## Check the JWS with https://jwt.io
1) Open https://jwt.io
2) Copy/paste the `x-custom-jwt` header value
- If everything works correctly the jwt.io sends a `Signature Verified` message
- The public key is downloaded automatically through the `x-custom-jwt-jwks` route and the `Request Termination` plugin. If it's not the case, open the Browser Developer Tools and see the network tab and console tab. The classic issue is getting the JWKS by using the self-signed Kong certificate (on 8443); it's easy to fix the issue by opening a new tab, going on the `jku` request (i.e. https://kong-gateway:8443/x-custom-jwt/jwks), clicking on Advanced and by clicking on `Proceed to`
- There is a known limitation on jwt.io with `"use": "enc"` the match isn't done correctly and the JWK is not loaded automatically: we simply have to copy/paste the public JWK directly in the `VERIFY SIGNATURE` of the web page. With `"use": "sig"` there is no restriction
![Alt text](/images/1-JWT.io.jpg "jwt.io")