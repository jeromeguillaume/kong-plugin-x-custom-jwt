# Kong plugin | `x-custom-jwt`: craft a custom JWT and sign it for building a JWS
1) Craft a custom JWT called `x-custom-jwt`
2) Load the private JWK from the plugin's configuration and convert it into a PEM format
3) Sign the JWT with the PEM string for building a JWS (RS256 algorithm)
4) Add the 'x-custom-jwt' to an HTTP Request Header backend API

The plugin `x-custom-jwt` doesn't check the validity of the input itself (neither checking of JWT signature & JWT expiration, nor user/password checking, nor checking Client TLS checking, nor api key checking). So it's **mandatory to use this plugin in conjunction with Kong security plugins**:
- [OIDC](https://docs.konghq.com/hub/kong-inc/openid-connect/)
- [JWT validation](https://docs.konghq.com/hub/kong-inc/jwt/)
- [Basic authorization](https://docs.konghq.com/hub/kong-inc/basic-auth/)
- [mTLS](https://docs.konghq.com/hub/kong-inc/mtls-auth/)
- [Key Authentication - Encrypted](https://docs.konghq.com/hub/kong-inc/key-auth/)

## High level algorithm to craft `x-custom-jwt`
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
    return "HTTP 401", "You are not authorized to consume this service"
end

-- If the Consumer sends a correct Authentication, we craft the 'x-custom-jwt' JWT
-- The 'client_id' claim has a value depending of the Consumer's Authentication
if "Authorization: Bearer" then
    -- Copy all the content of the AT (given by 'Authorization: Bearer')
    x-custom-jwt.payload = AT.payload
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
x-custom-jwt.header.alg = "HS256",
x-custom-jwt.header.kid = "<JWK.kid>", -- Got from the kid of private JWK
x-custom-jwt.header.jku = "<jku>" -- Got from the plugin Configuration

-- Common claims for all Authentication methods
x-custom-jwt.payload.iss = "<iss>" -- Got from the plugin Configuration
x-custom-jwt.payload.iat = "<current time>"
x-custom-jwt.payload.exp = x-custom-jwt.payload.iat + "<expires_in>"  -- Got from the plugin Configuration
x-custom-jwt.payload.aud = "<url>" -- the Backend_Api URL
x-custom-jwt.payload.jti = "<uuid>" -- Generation of a 'Universally unique identifier'

-- 'act.sub' claim
find "X-Consumer-Custom-Id" header
if "X-Consumer-Custom-Id" then
    x-custom-jwt.payload.act.client_id = "<X-Consumer-Custom-Id>" -- Got from 'X-Consumer-Custom-Id' header which is set by security plugins (OIDC, Basic Auth, Key Authentication, Mutual TLS Auth, etc.)
end

-- Sign the JWT with a private JWK (set in the plugin configuration) for building a JWS 
jws_x_custom_jwt = jwt:sign (x-custom-jwt, private_jwk)

-- Add the 'x-custom-jwt' header to the request's headers before sending the request to the Backend API
kong.service.request.set_header("x-custom-jwt", jws_x_custom_jwt)
```
## How to test the `x-custom-jwt` plugin with Kong Gateway
### Prerequisites 

1) install the [Kong Gateway](https://docs.konghq.com/gateway/latest/install/)
2) Install [http.ie](https://httpie.io/)
3) Prepare the JWK for getting the Public and Private Keypair
- You can use the JWK keypair provided in this repo:
  - JWKS (JSON Web Key Set) Public Key: `./test-keys/jwks-public.json`
  - JWK Private Key: `./test-keys/jwk-private.json`
- **Or** create your own JWK keypair: go for instance to the site https://mkjwk.org/ and configure the online tool with following values:
  - key Size: `2048`
  - Key Use: `Signature`
  - Algorithm: `RS256`
  - Key-ID: `SHA-256`
- Click on Generate, copy to clipboard the `Public and Private Keypair` (i.e. Private Key) and the `Public Key`
4) Create a Route to deliver the public JWKS
- The Route has the following properties:
  - name=`x-custom-jwt-jwks`
  - path=`/x-custom-jwt/jwks`
  - Click on `Save`
- Add the `Request Termination` plugin to the Route with:
  - `config.status_code`=`200`
  - `config.content_type`=`application/json`
  - `config.body`=copy/paste the content of `./test-keys/jwks-public.json` **Or**
  - `Config.Body`=**The `Public JWK Key` must be pasted from https://mkjwk.org/ and add `"keys": [` property for having a JWKS** If needed, adapt the `kid` to a custom value. JWKS Structure:
    ```json
    {
      "keys": [
        {
          *****  CHANGE ME WITH THE PUBLIC JWKS *****
        }
      ]
    }
    ```
  - Click on `Save`
- Add the `CORS` plugin to the Route with:
  - config.origins=`*`
  - Click on `Save`
5) Create a Gateway Service
- For `httpbin` service, add a Gateway Service with:
  - name=`httpbin`
  - URL=`http://httpbin.apim.eu/anything` 
  - Click on `Save`
- Add a Route to the Service with:
  - name=`httpbin` 
  - path=`/httpbin`
  - Click on `Save`
- Add `x-custom-jwt` plugin to the Service with:
  - config.iss=`<adapt the URL to your environment>` (example: https://kong-gateway:8443/x-custom-jwt)
  - config.jku=`<adapt the URL to your environment>` (example: https://kong-gateway:8443/x-custom-jwt/jwks)
  - config.private_jwk=copy/paste the content of `./test-keys/jwk-private.json` **Or**
  - config.private_jwk=paste the `Public and Private Keypair` from https://mkjwk.org/. If needed, adapt the `kid` to a custom value; the `kid` value must be the same as defined in `Prerequisites` heading (see the configuration of `Request Termination` plugin)
6) Create a Consumer with:
- username=`contact@konghq.com`
- Custom ID=`contact@konghq.com-ID1`


### Example #1: "Authorization: Bearer" input
0) Let's use the `httpbin` route 
1) Test `Sample #1`
- `Request #1`:

  ```shell
  AT1=`cat ./sample-x-custom-jwt/1_input-access-token.txt` && http :8000/httpbin Authorization:' Bearer '$AT1
  ```
- `Response #1`: expected value of `x-custom-jwt` plugin:

  ```json
  {
    "header": {
      "kid":"kong",
      "jku": "http://localhost:8000/x-custom-jwt/jwks",
      "typ": "JWT",
      "alg": "RS256"
    },
    "payload": {
      "client_id": "ma9oycqlep",
      "act": {
        "client_id": "oauth-custom_id"
      },
      "aud": "http://httpbin.apim.eu/anything",
      "part_nr_ansp_person": "39444822",
      "iat": 1689239122,
      "pi.sri": "reaIoODhdakJoNacp3N0yQQU3Gw..rOmI",
      "sub": "L000001",
      "part_nr_org": "28021937",
      "exp": 1689240922,
      "scope": "openid profile email",
      "iss": "https://kong-gateway:8443/x-custom-jwt/v2",
      "jti": "ca2a1f74-1041-437d-b908-29743e3381f0"
    },
    "signature": "xxxxx"
  }
  ```
### Example #2: "Authorization: Basic" input
1) Open the Service (created above)
2) Create a new Route:
- name=`basicAuth`
- path=`/basicAuth`
3) Add `Basic Authentication` plugin to the Route (Leave default parameters)
4) Open the `contact@konghq.com` consumer, go on credentials, click on a `+ New Basic Auth Credential` and put:
- username=`my-auth`
- password=`My p@ssword!`
5) Click on save
6) Test
- `Request`:

  ```shell
  http -a 'my-auth:My p@ssword!' :8000/basicAuth
  ```
- `Response`: expected value of `x-custom-jwt` plugin:

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
        "client_id": "my-auth-username-ID"
      },
      "jti": "08e1f9e0-7cb7-4fb3-9d9a-1de487af3a03",
      "iss": "https://kong-gateway:8443/x-custom-jwt",
      "aud": "http://httpbin.apim.eu/anything",
      "iat": 1712585199,
      "exp": 1712586999,
      "client_id": "012345AZERTY!"
    },
    "signature": "xxxxx"
  }
  ```
### Example #3: "mTLS Client Certificate" input
1) Create a CA Certificate: open Certificates page and click `New CA Certficate`
- Copy/paste the content of `./mTLS/ca.cert.pem` in the CA field
- Click on Create
2) Open the CA Certificate just created and copy/paste its `ID`
3) Open the Service (created above)
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
    eyJraWQiOiJrb25nIiwidHlwIjoiSldUIiwiYWxnIjoiUlMyNTYiLCJqa3UiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dC9qd2tzIn0.eyJhY3QiOnsiY2xpZW50X2lkIjoiY29udGFjdEBrb25naHEuY29tLUlEMSJ9LCJqdGkiOiI4OTBkNWFiMy1mOTViLTQ0OWMtODFiYS04ZWJlNjA3NDhlNGYiLCJpc3MiOiJodHRwczovL2tvbmctZ2F0ZXdheTo4NDQzL3gtY3VzdG9tLWp3dCIsImF1ZCI6Imh0dHA6Ly9odHRwYmluLmFwaW0uZXUvYW55dGhpbmciLCJpYXQiOjE3MTI1OTYyMTEsImV4cCI6MTcxMjU5ODAxMSwiY2xpZW50X2lkIjoiQz1VUywgU1Q9Q2FsaWZvcm5pYSwgTz1Lb25nIEluYy4sIE9VPUtvbm5lY3QsIENOPWtvbmcsIGVtYWlsQWRkcmVzcz1jb250YWN0QGtvbmdocS5jb20ifQ.hdvVaMro_dnaeEaraPCGDmVtqF-Ju8HautMLIgezXPzM6lYEfuN44ZFOO4-pAKBy6i7zcuI6HXnuOqHjwcK8pcggi9yic7XL2AJxwDXyyXlt6DDgixul4fFZnNQ2AQRZCTEfNSbVBk3MFNZHbv4bhwOmEn8GCt9xjK-fEQ6bcTIcphJ7xm-jnRQQd7IqJxoMj4nkM_HyN30zt0N3ynVNWcEZPl1v64PyWqAr6Fw6izL7sN0y4dSmeyufs36AYA0e6JznmR5CnnQrWyb1arwvuNx2t89BwMFEzThWk8SfvmY4r5fy1KyOou9p39V3UdMBNz6oGiLqd7wdYVEU2OH74g
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
        "jti": "890d5ab3-f95b-449c-81ba-8ebe60748e4f",
        "iss": "https://kong-gateway:8443/x-custom-jwt",
        "aud": "http://httpbin.apim.eu/anything",
        "iat": 1712596211,
        "exp": 1712598011,
        "client_id": "C=US, ST=California, O=Kong Inc., OU=Konnect, CN=kong, emailAddress=contact@konghq.com"
      },
      "signature": "xxxxx"
    }
    ```
### Example #4: "Key Authentication" input
1) Open the Service (created above)
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
  http :8000/apiKey apikey:'012345AZERTY!'
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

## Check the JWS with https://jwt.io
1) Open https://jwt.io
2) Copy/paste the `x-custom-jwt` header value
- If everything works correctly the jwt.io sends a `Signature Verified` message
- The public key is downloaded automatically through the route `x-custom-jwt-jwks` and the `Request Termination` plugin. If it's not the case, open the Browser Developer Tools and see the network tab and console tab. The classic issue is getting the JWKS by using the self-signed Kong certificate (on 8443); it's easy to fix the issue by opening a tab, going on the `jku` request (i.e. https://kong-gateway:8443/x-custom-jwt/jwks), clicking on Advanced and by clicking on 'Proceed to'
- There is a known limitation on jwt.io with `"use": "enc"` the match isn't done correctly and the JWK is not loaded automatically: we simply have to copy/paste the public JWK directly in the `VERIFY SIGNATURE` of the web page. With `"use": "sig"` there is no restriction
![Alt text](/images/1-JWT.io.jpg "jwt.io")