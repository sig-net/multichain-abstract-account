# MultiChain Abstract Account

This is an experimental repository that intents to create a MultiChain Abstract Account with the following capabilities:

- User can authorize transaction on the account using any [Auth Method](#auth-methods)
  - If two auth methods are added to one account, they can be used interchangeably
- User don't need a Wallet to interact with the Account
- User can add/remove [Auth Methods](#auth-methods)
- User can have multiple accounts per [Auth Method](#auth-methods) and each account is independent
- User have an implicit account in any chain supported by the [ChainSignatures MPC](https://github.com/near/mpc) and Near for each [Auth Method](#auth-methods)
- User can recover an AccountId by providing ownership of one [Auth Method](#auth-methods) available on the account
  - Master Recovery: Email, Phone

## Auth Methods

- [OIDC Token](#oidc-token) (Google, Facebook, Apple)
- [Phone Number](#phone-number)
- [Passkeys](#passkeys)
- [Wallets](#wallets) (EVM, Solana, Near)
- [Near Account](#near-account)

## Master Recovery

- Email: if the account has an OIDC with Email the user should be able to authorize transaction on the account by sending an email with a transaction to it, even if the email method is not previously added.
  - Why email?
    - The user can only have the OIDC if he has the email
    - The email is text-based, it means no website can trick the user
    - The email can be easily validated by the DKIM
    - THe email is controlled only by the user and can only be sent by the user like a wallet
  - View (Email ZK)[https://prove.email/]
  - Why we need this? 
    - If the account can only be controlled by OIDC token the user can be locked out/censored of the account if the website with ClientID shutdown or acts malicious, so the user need a permission-less method to authenticate on his own account
- Phone: [TODO: investigate if it's possible to do the same for phone as email]

## Infrastructure

This section describes the parts necessary to implement this project

### RootDerivationContract

This is the EntryPoint contract that stores the Accounts Mapping and control the accounts keys. Bellow you can find a description of the proposed interface

```rust
// Validate if all OIDC providers have this properties
struct OIDC {
    issuer: String,
    client_id: String,
    email: String
}

struct WebAuthn {
    key_id: String,
    public_key: String
}

// Can be extended to support more auth methods
enum AuthPath {
    PubKey(String), // Wallet authentication
    WebAuthn(WebAuthn), // Passkey authentication
    OIDC(OIDC) // Social authentication (e.g: Google, Facebook, Apple)
    Account(AccountId) // Near Account authentication
}

struct Account {
    auth_paths: Vec<AuthPath>,
    nonce: u64,
}

// Should implement the (Storage Management NEP)[https://nomicon.io/Standards/StorageManagement]
struct RootDerivationContract {
    accounts: HashMap<String, Account> // AccountId => Account
}

struct Action {
  AddKey({
    auth_path: AuthPath
  })
  RemoveKey({
    auth_path: AuthPath
  })
  CallChainSig({
    args: { 
      payload: [u8, 32],
      key_version: [u8, 32],
      path: String
    }
  })
}

// TODO: Decide if message signed will be the following object or the hash of the
struct Transaction {
  nonce: u64,
  actions: Vec<Action> 
}

impl RootDerivationContract {
  #[public call]
  pub execute(&mut self, message: String, signature: String, transaction: Transaction, auth_path: AuthPath, auth_target: AuthPath) {
    /**
    1. Find account based on auth_path
    2. Validate account nonce
    3. Validate if auth_target exist on account
    4. Validate if message contains the hash of the transaction
      - Even though this logic it's method specific it should not be done on the auth_contract because the auth_contract should only validate the signature and not security details of our abstract account
    5. Validate if message => signature using provided auth_path (cross-contract call to corresponding auth_contract)
    6. Call auth_callback with auth_contract result
    **/
  }

  pub execute_callback(&self) {
    // 1. If auth_contract result === true, execute the actions on transaction.actions
  } // Execute transaction

  // Methods that can be called by execute
  #[private]
  pub add_auth_path(&mut self, auth_path: AuthPath) {}
  #[private]
  pub remove_auth_path(&mut self, auth_path: AuthPath) {}
  #[private]
  pub call_chain_sig(&mut self, auth_target: AuthPath, actions: Vec<Actions>) {
    // Call ChainSig contract using auth_target as the path
  }

  #[public view]
  pub find_account(&self, auth_path: AuthPath) -> Account {}
  #[public view]
  pub derive_key_from_path(&self, auth_path: AuthPath) -> String {}
}

```

### AuthContracts

Those are individual contract for each type of authentication we want to support for the accounts. This includes

#### OIDC Auth Contract

```typescript
interface OIDCToken {
  issuer: string;
  clientId: string;
  email: string;
}

type Provider = 'google' | 'facebook' | 'apple'

class OIDCAuthContract {
  auth(token: OIDCToken, tokenSignature: string, provider: Provider): boolean {
    // 1. Fetches the public key of the provider from the oracle contract
    // 2. Validate if the token was signed by the providerPublicKey
    // 3. Return the result
  }
}
```

#### WebAuthn Auth Contract

```typescript
class WebAuthnAuthContract {
  auth(message: String, signature: String, publicKey: String): boolean {}
}

```

#### Ethereum Auth Contract
#### Solana Auth Contract


### Oracle Contracts

Those contracts are necessary to fetch and store the current OIDC provider public key as they are rotated frequently

### Relayer

REST server that receives a payload, build the NearTx and call the RootDerivationContract sponsoring the gas

It enables users to use any blockchain without a wallet, simply by signing a payload with his OIDC token

### OIDC Server (Optional)

We are not 100% sure if OIDC token generation on client side is secure so we will initially have a server that has the clientSecret and request the token. 

Obs: The user still able to submit the client token to auth on the account without the server

### ChainSignatures Contract

View [ChainSignatures MPC](https://github.com/near/mpc)

This contract allow contracts to control private keys, it has a single method that matter for us: 

```typescript
type Signature {
  r: String
  s: String
  v: String
}

sign(args: {
  key_type: Number, 
  payload: Uint8Array(32),
  path: String
}): Signature
```

## Issues

- It's an account system inside Near, so it interacting with other contracts that uses predecessor_id won't be natural as the predecessor_id will always be the RootDerivationContract Id and not the AccountId. 
  - The solution it's control a Near account using ChainSig contract (slower than native account because requires a call to ChainSig)
    - More expensive it's relative because if we deploy one AA Smart Contract for each account it cost 1N/100kB (stateless basic contract cost 2N, see (WebAuthn Contract)[https://testnet.nearblocks.io/address/felipe-webauthn.testnet], it means the user has to performs around 4000 ChaiSig to make the cost even)
- The APIs to talk with this contract are different from using a native account on Near