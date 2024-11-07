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

impl RootDerivationContract {

  pub auth() {}
  pub execute() {}
  pub add_key(&mut self, auth_path: String) {}
  pub remove_key(&mut self, auth_path: String) {}
  pub find_account(&self, auth_path: String) -> Account {}
  pub derive_key_from_path(&self, auth_path: String) -> String {}
  

}

```

### AuthContracts


### Oracle Contracts


### Relayer


### OIDC Server (Optional)


### ChainSignatures Contract
