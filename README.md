# verified-autocomplete-claims

Verified Autocomplete Claims - a browser mediated verified claim release. This document specifies how to provide a verified email claim, and extensions can build on the protocol to provide other verified claims such as a phone number.

## Verified Email

Verifying control of an email address is a frequent activity on the web today and is used both to prove the user has provided a valid email address, and as a means of authenticating the user when returning to an application. 

Verification is performed by either:

1) sending the user a link they click on or a verification code. This requires the user to switch from the application they are using to their email address and having to wait for the email arrive, and then perform the verification action. This friction often causes drop off in users completing the task.
2) the user logs in with a social login provider such as Apple or Google that provide a verified email address. This requires the application to have set up a relationship with each social provider, and the user to be using one of those services and wanting to share the additional profile information that is also provided in the OpenID Connect flow.

Verified Autocomplete Claims enables an application to obtain a verified email address from any Issuer the user wishes to use without any prior registration by the application, and to only share the verified email address improving the privacy aspects of the interaction.

The protocol aligns with the issuer->holder->verifier pattern where the holder is the browser and the verifier is the website requesting a verified email address. Having the browser mediate the issuance and presentation blinds the identity of the verifier from the issuer, improving user privacy. The issuer can be any service that a DNS record for the email domain delegates as being authoritative for the email domain.

The protocol also builds on the Issuer having an existing Passkey for the user for authentication by allowing the Issuer to respond with a WebAuthN challenge on issuance that the browser can then start a WebAuthN browser user experience without loading a page from the Issuer, allowing seamless authentication and authorization user experience.

If an Issuer provides a registration URI, browsers can detect if an email address provided by the user to a web page has an Issuer, and offer the user to enable verified autocomplete, similar to how browser prompt to remember email addresses. 


## Key Concepts


- VAC-JWT: An SD-JWT signed by the Issuer containing the claim and nonce provided by the RP and a public key provided by the browser. There are no disclosures in the JWT. The VAC-JWT is requested of the Issuer by the browser at the user's request, and provided to the web page that made the request aka Replying Party (RP) aka verifier. The browser creates a key bound JWT (KB-JWT) that contains the URL of the page as the `aud` claim. The resulting VAC-JWT+KB then binds the Issuer, nonce, claim, browser public key, and web page. The nonce binds the request to the response, and the `aud` prevents replay to another page.

- Issuer: a service that exposes a `vca_issuance_endpoint` that is called by the browser to obtain an VAC-JWT, a `vca_jwks_uri` that contains the public keys used to verify the VAC-JWT, and optionally a `vca_registration_uri` that the browser can open for the user to register an Issuer for an email. The Issuer is identified by its domain, an eTLD+1 (eg `issuer.example`). THe hostname in all URLs from the Issuer's metadata MUST end with the Issuer's domain. This identifier is what binds the VAC-JWT, the DNS delegation, with the Issuer.

> Restricting the Issuer to be an eTLD+1 may be too restrictive. Let's get feedback. Having a crisp identifier and an issuer identifier format different than OpenID Connect tokens (no leading https://) simplifies verification and has clean bindings between all the services, DNS record, and token.


## User Experience

Registration Process: The user navigates to the Issuer's website (the apex domain or any subdomain) and is prompted to enable the Issuer to issue verified emails, and the user accepts.

Verified Email Release: The user navigates to any website that requires a verified email address and an input field to enter an email address. The user focusses on the input field and the browser provides one or more verified emails for the user to provide. The user selects a verified email, potentially authenticates with the Issuer, and the app proceeds having obtained a verified email.

# Issuer Registration

An Issuer is a service that can authenticate the user via a browser and is authorized to make claims about an email domain. An Issuer is identified by an eTLD+1 domain name, such as `issuer.example` or `example.com`. This restricts there to be only one Issuer for an eTLD+1, but ensures that cookies can only be read by a single Issuer. To be recognized as an Issuer, the following must be configured

## .well-known/web-identity


The Issuer must host a document at `.well-known/web-identity` that contains a JSON object that MUST contain:

- **vac_issuance_endpoint** - REQUIRED - the API endpoint the browser calls to obtain a VAC-JWT
- **vac_jwks_uri** - REQUIRED - the URL where the issuer provides its public keys to verify the VAC-JWT


Both of `vac_issuance_endpoint` and `vac_jwks_uri` MUST have an eTLD+1 that matches the eTLD+1 of the Issuer.

The response to the request to the eTLD+1 and MUST be of `Content-Type` `application/json`. The initial request MAY return a redirect to a web server that is on a subdomain of the eTLD+1, but MUST have the same path. 

Following is a non-normative example of a response body:


```json
{
  "vac_issuance_endpoint": "https://accounts.issuer.example/vac/issuance",
  "vac_jwks_uri": "https://accounts.issuer.example/vac/jwks.json"
}
```


## email._web-identity. TXT  DNS Record

The email domain MUST have a `TXT` DNS record for `email._web-identity` where the content is `iss=` followed by the Issuer Domain.

Following is a non-normative example for the email domain `domain.example` and the Issuer Domain of `issuer.domain` :

```
email._web-identity.domain.example TXT iss=issuer.example
```

This record confirms that `domain.example` has delegated Verified Email Autocomplete to the Issuer `issuer.example`.

Note this record MUST also exist for `issuer.example`, IE the email domain MUST delegate to itself.

> Access to DNS records and email is often independent of website deployments. This provides assurance that an Issuer is truly authorized as an insider with only access to websites on `issuer.example` could setup an Issuer that would grant them verified emails for any email at `issuer.example`.


# Processing Steps

1.  **Issuer Registration**
2.  **Claim Request**
3.  **Claim Selection**
4.  **Token Request**
5.  **User Authentication**
6.  **Token Generation**
7.  **Token Verification** 

## 1. Issuer Registration 


Ahead of time, a website registers itself as an Issuer:
  
- **1.1** - User navigates to the apex or any sub-domain of the Issuer such as `issuer.example` or `www.issuer.example` and logs in.

- **1.2** - The page calls to register claims it is authoritative for as an Issuer with:

```javascript
const response = await IdentityProvider.register({
    email: 'john.doe@domain.example'
});
```
A page can make multiple calls to register claims.

> Alternatively the API could take an array of claims

> TODO: Explore doing this declaratively with HTTP headers and/or HTML metadata.

- **1.3** - The browser then confirms the Issuer will correctly issue VAC-JWTs by performing steps (4) and (7) below.

- **1.4** - If the Issuer has provided a valid VAC-JWT for the email address the browser records `issuer.example` as an Issuer for the `email` claims `john.doe@domain.example` in its local storage.



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

> TODO: Explore replacing not having a JS call and doing this declaratively with HTTP headers and/or HTML metadata containing a dynamically generated nonce and a hidden field to accept the token.


## 3. Email Selection 

- **4.1** - User focusses on input field with

- **4.2** - The browser displays the list of suggestions of available email addresses could be shared. Emails that would be verified are decorated for user to understand. 

- **4.3** - User selects a verified email from browser selection.

## 4. Token Request

- **4.1** The browser fetches DNS record for Issuer authorization by prepending `email._webidentity.` to the Issuer domain and fetching the `TXT` and confirming it contains the string `iss=issuer.example`


- **4.2** - The browser loads `https://issuer.example/.well-known/web-identity` checks that file includes `vac_issuance_endpoint` and `vac_jwks_uri` per Issuer Registration above

- **4.3** - The browser generates a public-private key pair and generates a JWT signed by the private key that contains the following  claims:

    - **email** - the email to be verified 
    - **nonce** - the nonce from the RP 
    - **cnf** - the jwk for the public key used to sign the JWT 

> Do we want to register a `typ` for this JWT?
> Do we need a .well-known/web-identity parameter for supported algorithms?
 
- **4.4** - browser POSTS with `Content-Type` `application/x-www-form-urlencoded` the value `vac_request_token` set to the generated JWT along with w/ 1P cookies.

## 5. User Authentication 

- **5.1** - Issuer checks if there is a logged in user with the provided email. If all good the Issuer creates a fresh VAC-JWT per (6) and returns it as the value of `token` in an `application/json` response.


```json
{"token":"eyssss...."}
```

- **5.2** - If the user is not logged, or the email is not associated with the logged n user, the Issuer generates and sets a session cookie to represent the state of the call, and responds with `application/json` that MUST contain `continue_on` and MAY contain `webauth`. 

- **continue_on** - the url the browser should load in a popup window if WebAuthN is not available or the user cancels it. The hostname for the `continue_on` url MUST end with the Issuer domain. The URL MAY contain a URL query component.

- **webauthn** - the WebAuthN parameters for the browser to use, that includes:

    - **challenge** - the WebAuthN challenge
    - **rpId** - the RP identifier used by the Issuer when the credential was registered
    - **allowCredentials** - per WebAuthN
    - **userVerification** -  per WebAuthN
    - **timeout** -  per WebAuthN

A non-normative example:

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
      }
    ],
    "userVerification": "preferred",
    "timeout":        60000
  }
}
```

- **5.3** If the Issuer returned a `webauthn` object, and the browser has a credential for the user, then the browser will call the webauthn API to authenticate the user. If the user cancels or the authentication fails, the browser will fall back to loading the `continue_on` URL in a popup window. If the browser successful authenticated the user with WebAuthn, the browser repeats the call to the `vat_issuance_endpoint` passing the results of the WebAuthN call as `Content-Type` `application/json`. 

A non-normative example:

```json
{
  "email": "john.doe@domain.example",
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

- **5.4** If the browser opened a popup window with the `continue_on` URL, after successful login, the Issuer page calls `IdentityProvider.resolve(token)` where `token` is the VAC-JWT and the popup window is closed

> Q: what about unsuccessful login? What does the page do? Just close the window after some user notification?


## 6. VAC-JWT+KB Generation

- **6.1** - The Issuer generates a VAC-JWT that does not have any selective discloser claims that MUST have the following claims: 

- `iss`: the Issuer domain
- `iat`: the time the token was issued in UNIX seconds
- `jti`: a unique token identifier
- `email`: the email address
- `nonce`: the nonce value provided in the claim request
- `cnf`: the public key to the key pair used by browser in making the request.

and returns it to the browser.

- **6.2** - The browser generates a Key Binding JWT per [SD-JWT draft 18](https://www.ietf.org/archive/id/draft-ietf-oauth-selective-disclosure-jwt-18.html#name-key-binding-jwt) and includes the `aud` claim set to the URL of the page, and then concatenates the SD-JWT, a '~', and the KB and presents that SD-KWT+KB to the page.



## 7. Token Verification

- **7.1** - The `navigator.credentials.get()` call returns and `credential.token` is a SD-JWT+KB.

- **7.2** - JS code sends `token` to RP server. 

- **7.4** - RP Server separates the VAC-JWT from the KB-JWT, and retrieves the values from the VAC-JWT header and payload and and extracts the email domain from the `email` claim that MUST exist.

- **7.5** - RP Server verifies the `typ` is `vac+jwt`, the `iat` is not earlier than the Verifiers threshold (TODO recommendation), the `jti` has not recently been received, and the `nonce` value matches the browser session `nonce` that was provided. 

- **7.6** - RP Server verifies the Issuer is authorized for the email domain by fetching the DNS just as browser did in 4.1

- **7.6** - RP Server fetches `.well-known/web-identity` just as browser did in 4.2. and extracts `vac_jwks_uri` and verifies the host ends with the Issuer domain.

- **7.7** - RP Server verifies VAC-JWT token signature using keys from the `vac_jwks_uri`

- **7.8** - RP Server verifies KB-JWT key matches `cnf` in VAC-JWT.

- **7.9** - RP Server verifies `nonce` in KB-JWT matches `nonce` in VAC-JWT.

- **7.10** - RP Server verifies the KB-JWT.

- **7.11** - RP Server verifies the `aud` value in the KB-JWT matches the URL that requested the email claim.

## 8. VAC-JWT+KB


```text
<VAC-JWT>~<KB-JWT>
```

Following is an ABNF [RFC5234] for the VAC-JWT+KB, and various constituent parts:

ALPHA = %x41-5A / %x61-7A ; A-Z / a-z
DIGIT = %x30-39 ; 0-9
BASE64URL = 1*(ALPHA / DIGIT / "-" / "_")
JWT = BASE64URL "." BASE64URL "." BASE64URL
VAC-JWT = JWT
KB-JWT = JWT
VAC-JWT-KB = VAC-JWT "~" KB-JWT

## 8.1 SD-JWT

- `typ`: "vac+jwt"

> register a vac type?

- `iss`: the Issuer domain
- `iat`: the time the token was issued in UNIX seconds
- `jti`: a unique token identifier
- `email`: the email address
- `nonce`: the nonce value provided in the claim request
- `cnf`: the public key to the key pair used by browser in making the request.


## 8.2 KB-JWT

- `typ`: "kb+jwt"
- `alg`:

- `iat`:
- `aud`: the url of the page
- `vac_hash`: hash of the SD-JWT




