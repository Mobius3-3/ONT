<p align="center">
  <img src="https://raw.githubusercontent.com/neoauth/design-assets/master/logo/colour/neoauth_color.png" width="150px" /> 
</p>

<h1 align="center">NeoAuth Whitepaper</h1>

<p align="center">
  Whitepaper explaining <a href="https://neoauth.org">NeoAuth</a>.
</p>

<p align="center">
  <a href="https://github.com/neoauth/whitepaper/releases">
    <img src="https://img.shields.io/github/release/neoauth/whitepaper.svg?style=flat">
  </a>
</p>

## Contents

- [Aims](#aims)
- [Demo](#demo)
- [Diagram](#diagram)
- [Technical](#technical)
- [Limitations](#limitations)
- [Future](#future)

## Aims

The NeoAuth project aims to:

1. Allow businesses to use the NEO blockchain for **passwordless authentication**.
1. **Replace** email/password with a NEO address.
1. Provide a solution that businesses **outside** of the NEO ecosystem can use **today**.
1. Bring **awareness** to the NEO ecosystem. 
1. Improve the security of web applications by moving away from **email/password** based authentication.

## Demo

Visit [demo.neoauth.org](http://demo.neoauth.org) for a full NeoAuth demo.

## Diagram

![http://res.cloudinary.com/vidsy/image/upload/v1512691723/NeoAuth_Diagram_xgs5lq.svg](http://res.cloudinary.com/vidsy/image/upload/v1512691723/NeoAuth_Diagram_xgs5lq.svg)

## Technical

This section of the whitepaper will going into detail about the NeoAuth flow, which is visualised in the simplified diagram above.

### Setup Server

Each business wishing to use NeoAuth will need to host and maintain their own version of the NeoAuth [server](https://github.com/blockauth/server).

The server generates login attempts which are unique to the business as all interactions are verified with [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) tokens.
Each [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token is signed by a secret, which is unique to the business.

### Setup Client

Each business will need to use the NeoAuth [client](https://github.com/blockauth/client) (Javascript library) in their web application, within a
login form.

The client takes care of communicating with the server, however the business will need to present the data that is returned by the client 
to the user within their web application.

### User Login Attempt

A user will land on the login page of the business' web application. The login form will ask the user for the NEO public address that they wish to login
with. 

The user's NEO public address is passed to a Javascript function within the NeoAuth [client](https://github.com/blockauth/client) library, which contacts the business'
NeoAuth [server](https://github.com/blockauth/server) to create a new login attempt.

The login attempt data returned to the web application contains the following information:

- NeoAuth [smart contract](https://github.com/neoauth/smart-contract) address.
- Smart contract parameters.
- [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token for checking if the login attempt has been successful.
- UNIX timestamp for when the login attempt will expire.

The business' web application should render this information to the user, so that they understand how to invoke the NeoAuth smart contract and how much time
they have left.

### Smart Contract

The user will take the login attempt information, and use it to invoke the NeoAuth [smart contract](https://github.com/neoauth/smart-contract). 

The NeoAuth [smart contract](https://github.com/neoauth/smart-contract) takes two parameters, which are random 
[version 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_.28random.29) UUIDs.

The smart contract verifies that the parameters are both valid UUIDs. It then concatenates the two UUIDs together to form what will be the key
for the `Storage.Put()` operation. The storage key structure is:

```
<random_uuid>.<random_uuid>
```
```
ce8c00c2-4fa5-47de-a07e-1061e62955b0.3b19cfed-9731-4bb4-a5ef-f9fe052bb79e
```

The smart contract will then use the generated storage key to store the transaction hash of the current smart contract invocation.

### Check Login Attempt

The business' web application will then use the NeoAuth [client](https://github.com/blockauth/client) library to check if the login attempt has been
successful.

The [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token returned when the login attempt was created is handed to the business' NeoAuth 
[server](https://github.com/blockauth/server) when checking if the login attempt was successful.

The [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token holds the following information:

- User's public NEO address.
- Smart contract parameters.

The business' NeoAuth [server](https://github.com/blockauth/server) uses the smart contract parameters within the [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token
to generate the same storage key.

The business' NeoAuth [server](https://github.com/blockauth/server) then uses the [NEO Go SDK](https://github.com/CityOfZion/neo-go-sdk) to query the NEO
blockchain directy, using the `getstorage` RPC method.

The `getstorage` RPC method is using the same storage key as the smart contract to check if there is a valid NEO transaction hash stored at that key within
the smart contract.

If a valid NEO transaction hash is returned, and the NEO public address of the transaction matches the user's NEO public address, then the login attempt has been
successful. Thus, the business' NeoAuth [server](https://github.com/blockauth/server) returns a new [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token 
to the business' web application.

### Logged In

After the business' web application receives the final [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token, it can then use that token to identify the user.

Instead of the business' web application storing an email address as the identifier of a user. They instead store the NEO public address of the user.

The NEO public address that the user originally entered into the login form is signed within the JWT token, so the web application can continue to verify
who the user is whilst they have access to the token.

## Limitations

Currently the key used within the `Storage.Put()` operation in the [smart contract](https://github.com/neoauth/smart-contract) is made up of
two randomly generated [version 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_.28random.29) UUIDs.

```
<random_uuid>.<random_uuid>
```
```
ce8c00c2-4fa5-47de-a07e-1061e62955b0.3b19cfed-9731-4bb4-a5ef-f9fe052bb79e
```

This means that all storage keys are shared between all users invoking the smart contract. Therefore a malicious attacker could carry out an 
extremely large [denial-of-service](https://en.wikipedia.org/wiki/Denial-of-service_attack) attack, that could block users from logging in.

This exploit would not allow the malicious attacker to login as a different user, but simply block other users from completing a login attempt.

This however is extremely unlikely due to the number of combinations a UUID could have. The malicious attacker would have a 50% chance of 
blocking another user after generating 2.71 quintillion UUIDs.

This will be fixed in the future by changing how the storage key is generated. It will become:

```
<random_uuid>.<users_neo_public_address>
```
```
ce8c00c2-4fa5-47de-a07e-1061e62955b0.AS3vAZfkFfvyvgSV5e9hwx3oNnJ3ZQ67ct
```

The second part of the storage key will be the NEO public address of the user that invoked the smart contract. Therefore they will only be able
to affect their own login attempts, as the storage key is always unqiue to their NEO public address.

This was not implemented within [v3.0.0](https://github.com/neoauth/smart-contract/releases/tag/3.0.0) of the smart contract because of a 
limitation in [neo-python](https://github.com/CityOfZion/neo-python). NeoAuth will look to implement the necessary features to be able to 
fix this limitation. 

## Future

NeoAuth will continue to build new innovative products in the future.

### Serverless

Currently a business needs to deploy and maintain a hosted instance of the NeoAuth [server](https://github.com/blockauth/server). This is a
barrier to entry, relatively expensive and needs constant monitoring.

In the future the NeoAuth [server](https://github.com/blockauth/server) can be moved to run within [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html).

The deployment to a lambda function can be automated, and so reduces the barrier to entry for a business wishing to use NeoAuth. Serverless is
far more cost effective than running a dedicated host. Lambda functions can not "go down", and so removes the worry of monitoring.

### Transaction Based

The current user experience for invoking a NEO smart contract is quite limited, which introduces a barrier to entry for users interacting with NeoAuth.

Thus a future NeoAuth development could be authentication via a simple GAS transaction, instead of invoking the NeoAuth [smart contract](https://github.com/neoauth/smart-contract).
The GAS amount sent by the user would act as a one-time security code, and would be returned to the user after login had occured.

### Embeddable Login Form

Businesses currently will have to use the NeoAuth [client](https://github.com/blockauth/client) (Javascript library) to implement a NeoAuth solution.

In the future, a business will instead install a single dependency that will act as an embeddable login form for desktop and mobile.

A great example of this product is the [Auth0 Lock](https://auth0.com/lock).

---

<p align="center">
  Check out the NeoAuth <a href="http://demo.neoauth.org/">Demo</a>.
  <br>
  🔐
</p>
