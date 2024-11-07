# MultiChain Abstract Account

This is an experimental repository that intents to create a MultiChain Abstract Account using Near Chain Signatures with the following capabilities:

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

```typescript
interface OIDC {
  issuer: string;
  clientId: string;
  email: string;
}

interface WebAuthn {
  keyId: string;
  publicKey: string;
}

// Can be extended to support more auth methods
type AuthPath = 
  | { type: "PubKey", value: string } // Wallet authentication
  | { type: "WebAuthn", value: WebAuthn } // Passkey authentication 
  | { type: "OIDC", value: OIDC } // Social authentication (e.g: Google, Facebook, Apple)
  | { type: "Account", value: string }; // Near Account authentication

interface Account {
  authPaths: AuthPath[];
  nonce: number;
}

// Should implement the Storage Management NEP (https://nomicon.io/Standards/StorageManagement)
interface RootDerivationContract {
  accounts: Map<string, Account>; // AccountId => Account
  authContracts: Map<string, AccountId> // AuthMethodId => AuthMethodContractAddress
}

type Action = 
  | { type: "AddKey", authPath: AuthPath }
  | { type: "RemoveKey", authPath: AuthPath }
  | { 
      type: "CallChainSig", 
      args: {
        payload: Uint8Array;
        keyVersion: Uint8Array;
        path: string;
      }
    };

interface Transaction {
  nonce: number;
  actions: Action[];
}

interface ExecuteArgs {
  message: string,
  signature: string, 
  transaction: Transaction, 
  authPath: AuthPath, 
  authTarget: AuthPath
}

class RootDerivationContract {
  /**
   * @public
   * @call
   */
  async execute(executeArgs: ExecuteArgs): Promise<void> {
    /**
     * 1. Find account based on authPath
     * 2. Validate account.nonce === transaction.nonce
     * 3. Validate if authTarget exist on account
     * 4. Validate if message contains the hash of the transaction
     *   - Even though this logic it's auth method specific it should not be done on the authContract 
     *     because the authContract should only validate the signature and not security details of our abstract account
     *   - The message should contains the transaction hash to: avoid replay attacks (tx includes the nonce), guarantee that even if the message it's intercepted by a malicious actor it can only be used to what's intended for
     * 5. Validate if message => signature using provided authPath (cross-contract call to corresponding authContract)
     * 6. Call executeCallback with authContract result
     */
  }

  async executeCallback(): Promise<void> {
    // 1. If authContract result === true, execute the actions on transaction.actions
  }

  // Methods that can be called by execute
  private async addAuthPath(authPath: AuthPath): Promise<void> {}
  private async removeAuthPath(authPath: AuthPath): Promise<void> {}
  private async callChainSig(authTarget: AuthPath, actions: Action[]): Promise<void> {
    // Call ChainSig contract using authTarget as the path
  }

  /**
   * @public
   * @view
   */
  findAccount(authPath: AuthPath): Account | null {
    return null;
  }

  /**
   * @public
   * @view
   */
  deriveKeyFromPath(authPath: AuthPath): string {
    return "";
  }
}
```

### AuthContracts

Those are stateless contracts used to validate each type of [Auth Method](#auth-methods). 

Reasons to deploy them individually instead of implement inside the RootDerivationContract: 

- Reusable by other Near Contracts
- Upgradable
- Don't take space on account management contract

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
  auth(message: String, signature: String, publicKey: String): boolean {
    // Validate P256 signature
  }
}
```

#### Ethereum Auth Contract

```typescript
class EthereumAuthContract {
  auth(message: String, signature: String, address: String): boolean {
    // Validate secp256k1 signature
  }
}
```

#### Solana Auth Contract

```typescript
class SolanaAuthContract {
  auth(message: String, signature: String, publicKey: String): boolean {
    // Validate ed25519 signature
  }
}
```

### Oracle Contracts

Contract to fetch and store the OIDC provider public key on chain to be used by OIDCAuthContract to validate the validity of the token. 

Those contracts should be able to be updated permission less: 

- Decentralized network of oracles
- ZK when updating key

DEMO: For demo purpose a centralized "trusted" server will be used

### Relayer

Relayer REST server: 

- `/sign-near-transaction`
  - Args: 
    - receiver_id: account management contract address (RootDerivationContract)
    - execute_args: ExecuteArgs
  - Build Near Transaction, sign sponsoring the gas, call RPC

Enables blockchain transactions without wallet

### OIDC Server (Optional)

Currently we are not sure if the token generated on Client Side it's 100% authentic, so we will introduce this server to use the Client Secret to request the token. 

Obs: 
- User still able to call the contract without the server, using OIDC token generation on client flow
- If we come to the conclusion that the Client Token it's authentic and valid we can remove this server

### ChainSignatures Contract

View [ChainSignatures MPC](https://github.com/near/mpc)

- Allow SmartContracts to control keys
- Enable predictable address derivation. 
- Expose the `sign` method

```typescript
type Signature {
  r: string
  s: string
  v: number
}

sign(args: {
  key_version: number,
  payload: Uint8Array, // 32 bytes
  path: string
}): Signature
```

## Issues

- Since this is an account system within NEAR, interactions with contracts that rely on `predecessor_id` will be less intuitive, as the `predecessor_id` will always be the RootDerivationContract ID rather than the actual AccountId.
  - This can be addressed by controlling a NEAR account through the ChainSig contract, though this adds some latency due to the extra contract call
    - The cost implications are relative - deploying one AA Smart Contract per account costs 1N/100kB (compared to 2N for a basic stateless contract like the [WebAuthn Contract](https://testnet.nearblocks.io/address/felipe-webauthn.testnet)). A user would need to make approximately 4000 ChainSig calls before the costs equalize.
- The contract APIs differ from those used with native NEAR accounts, requiring adaptation