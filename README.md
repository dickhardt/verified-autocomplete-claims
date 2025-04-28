# verified-autocomplete-claim
Verified Autocomplete Claim - a browser mediated verified claim release

> This document currently focusses on providing a verified email claim, but the protocol can be extended to other claims

## Verified Email

Verifying control of an email address is a frequent activity on the web today and is used both to prove the user has provided a valid email address, and as a means of authenticating the user when returning to an application. 

Verification is performed by either:

1) sending the user a link they click on or a verification code. This requires the user to switch from the application they are using to their email address and having to wait for the email arrive, and then perform the verification action. This friction often causes drop off in users completing the task.
2) the user logs in with a social login provider such as Apple or Google that provide a verified email address. This requires the application to have set up a relationship with each social provider, and the user to be using one of those services and wanting to share the additional profile information that is also provided in the OpenID Connect flow.

Verified Autocomplete Claims enables an application to obtain a verified email address from any Issuer the user wishes to use without any prior registration by the application, and to only share the verified email address improving the privacy aspects of the interaction.

The protocol aligns with the issuer->holder->verifier pattern where the holder is the browser and the verifier is the website requesting a verified email address. Having the browser mediate the issuance and presentation blinds the identity of the verifier from the issuer, improving user privacy. The issuer can be any service that a DNS record for the email domain delegates as being authoritative for the email domain.

The protocol also builds on the Issuer having an existing Passkey for the user for authentication by allowing the Issuer to respond with a WebAuthN challenge on issuance that the browser can then start a WebAuthN browser user experience without loading a page from the Issuer, allowing seamless authentication and authorization user experience.


## Key Concepts


- VAC-JWT: A JWT signed by the Issuer containing the claim and nonce provided by the RP. The VAC-JWT is requested of the Issuer by the browser at the user's request, and provided to the web page that made the request aka Replying Party (RP) aka verifier.

- Issuer: a service that exposes a `vca_issuance_endpoint` that is called by the browser to obtain an VAC-JWT, and a `sd_jwt_uri` that contains the public keys used to verify the VAC-JWT. The Issuer is identified by its domain, an eTLD+1 (eg `issuer.example`). THe hostname in all URLs from the Issuer's metadata MUST end with the Issuer's domain. This identifier is what binds the VAC-JWT, the DNS delegation, with the Issuer.

> Restricting the Issuer to be an eTLD+1 may be too restrictive. Let's get feedback. Having a crisp identifier and an issuer identifier format different than OpenID Connect tokens (no leading https://) simplifies verification and has clean bindings between all the services, DNS record, and token.


## User Experience

Registration Process: The user navigates to the Issuer's website (the apex domain or any subdomain) and is prompted to enable the Issuer to issue verified emails, and the user accepts.

Verified Email Release: The user navigates to any website that requires a verified email address and an input field to enter an email address. The user focusses on the input field and the browser provides one or more verified emails for the user to provide. The user selects a verified email, potentially authenticates with the Issuer, and the app proceeds having obtained a verified email.

# Processing Steps

1.  **Issuer Registration**
2.  **Claim Request**
3.  **Claim Selection**
4.  **Token Request**
5.  **User Authentication**
6.  **Token Generation**
7.  **Token Verification** 




Ahead of time, a website registers itself as an Issuer:
  
- **1.1** - User navigates to the apex or any sub-domain of the Issuer such as `issuer.example` or `www.issuer.example` and logs in.

- **1.2** - The page calls to register claims it is authoritative for as an Issuer with:

```javascript
// This prompts the user to accept "https://issuer.example" as Issuer for john.doe@domain.example.
const response = await IdentityProvider.register({
    email: 'john.doe@domain.example'
});
```
A page can make multiple calls to register claims.

>Q: can the Issuer signal to the browser here to use a Passkey for authentication?

> Q: Does the Issuer do this each time the page is loaded and the user visits? Perhaps an API for the Issuer to get a list of claims the browser has for the Issuer so the Issuer knows which ones to add? The downside is the potential for cross context disclosure. For example, the user may have both a personal and work account at the same Issuer, and the Issuer does not know the two accounts are for the same user, and the browser does not know which account is signed in, so can only provide all claims to the Issuer.

> Q: How are claims removed that are no longer supported by the Issuer? Is this only done by the user? Perhaps prompted for user to delete if Issuer returns an error? For example, they leave a company and no longer can obtain a verified work email address. How is it removed from future selections? Perhaps similar to other autocomplete cleanup where the user can see the Issuers and claims and can delete which ones will be offered?

> TODO: Explore doing this declaratively with HTTP headers and/or HTML metadata.

- **1.3** - The browser then confirms the Issuer will correctly issue VAC-JWTs by performing steps (4) and (7) below.

- **1.4** - If the Issuer has provided a valid VAC-JWT for the email address, the browser prompts the user to accept the Issuer's registration request by displaying the Isssuer domain and the email address verified, and if the user accepts the prompt, the browser records `issuer.example` as an Issuer for the `email` claims `john.doe@domain.example` in its local storage.


## 2. Email Request

User navigates to a site that will act as the RP.

- **2.1** - The RP page has the following HTML in the page:

```html

<input autocomplete="email webidentity">

```

- **2.2** - The page has made this call which has not returned:

> This signature is extending the existing credentials API which has quite different characteristics. A new shape (below) would be more straight forward

```js
try {
  const {token} = await navigator.credentials.get({
    mediation: "conditional",
      providers: [{
        format: "vac-jwt",
        fields: ["email"], 
        nonce: "259c5eae-486d-4b0f-b666-2a5b5ce1c925",
      }]
    }
  });
  // send token to server
} catch ( e ) {
   // no providers or other error
}
```


> It does not seem practical to enable this functionality to be declarative in HTML as a unique `nonce` is required for the RP server to bind the VAC-JWT to the session, but perhaps as a header? 



## 3. Email Selection 

- **4.1** - User focusses on input field with

- **4.2** - The browser displays the list of suggestions of available email addresses could be shared. Emails that would be verified are decorated for user to understand. 

- **4.3** - User selects a verified email from browser selection.

## 4. Token Request

- **4.1** The browser fetches DNS record for Issuer authorization by prepending `email._webidentity.` to the Issuer domain and fetching the `TXT` and confirming it contains the string `iss=issuer.example`

example DNS record:

```
email._webidentity.domain.example   TXT   iss=issuer.example
```

This record confirms that `domain.example` has delegated Verified Email Autocomplete to the Issuer `issuer.example`.

Note this record MUST also exist for `issuer.example`, IE the email domain MUST delegate to itself.


> Access to DNS records and email is often independent of website deployments. This provides assurance that an Issuer is truly authorized as an insider with only access to websites on `issuer.example` could setup an Issuer that would grant them verified emails for any email at `issuer.example`.


- **4.2** - The browser loads `https://issuer.example/.well-known/web-identity` and MUST follow redirects to the same path but with a different subdomain of the Issuer, for example `https://accounts.issuer.example/.well-known/web-identity`. 

> Most apex domains redirect all HTTP calls to a subdomain

- **4.3** - The browser checks that the `.well-known/web-identity` file contains JSON that includes the following properties:

- *vac_issuance_endpoint* - the API endpoint the browser calls to obtain an VAC-JWT
- *vac_jwks_uri* - the URL where the issuer provides its public keys to verify the VAC-JWT

Each of these properties MUST include the issuer domain as the root of their hostname. 

Following is an example `.well-known/web-identity` file

```json
{
  "vac_issuance_endpoint": "https://accounts.issuer.example/vac/issuance",
  "vac_jwks_uri": "https://accounts.issuer.example/vac/jwks.json"
}
```


- **4.3** - browser POSTS `application/json` to the `vac_issuance_endpoint` of the Issuer containing the selected email, the `vac-jst` format, and the nonce along with w/ 1P cookies to get a VAC-JWT:

```json
{ "claims": {
    "email": "john.doe@domain.example"
  },
  "format": "vac-jwt",
  "nonce": "259c5eae-486d-4b0f-b666-2a5b5ce1c925"
}
```

## 5. User Authentication 

- **5.1** - Issuer checks if there is a logged in user with the provided email. If all good the Issuer creates a fresh VAC-JWT per (6) and returns it as the value of `token` in an `application/json` response.


```json
{"token":"eyssss...."}
```

- **5.2** - If the user is not logged, or the email is not associated with the user, the Issuer responds with `application/json` that MUST contain `continue_on` and may contain `webauth`. 

- **continue_on** - the url the browser should load in a popup window if WebAuthN is not available or the user cancels it. The hostname for the `continue_on` url MUST end with the Issuer domain. The URL MAY contain a URL query component.

- **webauthn** - the WebAuthN parameters for the browser to use, that includes:

    - **challenge** - the WebAuthN challenge
    - **rpId** - the RP identifier used by the Issuer when the credential was registered
    - **allowCredentials** - per WebAuthN
    - **userVerification** -  per WebAuthN
    - **timeout** -  per WebAuthN

```json
{ 
  "continue_on": "https://accounts.issuer.example/login?email=john.doe@domain.example",
  "webauthn": { // optional
    "challenge":      [ /* Uint8Array bytes */ ],
    "rpId":         "issuer.example",
    "allowCredentials":[
    {
      "type": "public-key",
      "id":   [ /* the raw byte values of credential.id for john.doe@… */ ],
      "transports": ["internal","usb"]   // optional
    },
    {
      "type": "public-key",
      "id":   [ /* another credential.id if multi-device */ ]
    }
    ],
    "userVerification": "preferred",
    "timeout":        60000
  }
}
```

- **5.2** If the browser successful authenticated the user with WebAuthn, the browser repeats the call to the `vat_issuance_endpoint`, including the WebAuthN **assertion** results.

```json
{
  "claims": {
    "email": "john.doe@domain.example"
  },
  "format": "vac-jwt",
  "nonce": "259c5eae-486d-4b0f-b666-2a5b5ce1c925",
  "assertion": {
    "id": "PLACEHOLDER_FOR_ASSERTION_ID",
    "rawId": [0],
    "clientDataJSON": [0],
    "authenticatorData": [0],
    "signature": [0],
    "userHandle": "PLACEHOLDER_FOR_USER_HANDLE"
  }
}
```

> Q: can the Issuer set session cookies in the response?

> Q: can we include a response that would trigger a Passkey exchange between the browser and the Issuer?

on successful login the Issuer calls `IdentityProvider.resolve(token)` where `token` is the VAC-JWT and the popup window is closed

> Q: what about unsuccessful login? What does the page do? Just close the window after some user notification?




## 6. Token Generation

- **6.1** - The Issuer generates a VAC-JWT with a `typ` set to `vac-jwt` and the `kid` set to the kid value of the key used to sign the VAC-JWT, and a payload that MUST include the following claims:

- `iss`: the Issuer domain
- `iat`: the time the token was issued in UNIX seconds
- `jti`: a unique token identifier
- `email`: the email address
- `nonce`: the nonce value provided in the claim request

```json
// JWT header
{
    "kid": "123456",
    "typ": "vac-jwt"
}
// JWT payload
{
    "iss": "issuer.example",
    "iat": 0000,
    "jti": 9999,
    "email": "john.doe@domain.example",
    "nonce": "259c5eae-486d-4b0f-b666-2a5b5ce1c925"
}

```


## 7. Token Verification

- **6.1** - The `navigator.credentials.get()` call returns and `credential.token` is a VAC-JWT

``` 
// token example and payload
```
- **6.2** - JS code sends `token` to RP server. 

- **6.4** - RP Server retrieves the values from VAC-JWT header and payload and and extracts the email domain from the `email` claim.

- **6.5** - RP Server verifies the `typ` is `vac-jwt`, the `iat` is not earlier than the Verifiers threshold (TODO recommendation), the `jti` has not recently been received, and the `nonce` value matches the browser session `nonce` that was provided.

- **6.6** - RP Server verifies the Issuer is authorized for the email domain by fetching the DNS just as browser did in 4.1

- **6.6** - RP Server fetches `.well-known/web-identity` just as browser did in 4.2. and extracts `vac_jwks_uri` and verifies the host ends with the Issuer domain.

- **6.7** - RP Server verifies VAC-JWT token signature using keys from the `vac_jwks_uri`


