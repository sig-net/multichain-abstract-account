# MultiChain Abstract Account

This is an experimental repository that creates a MultiChain Abstract Account using NEAR Chain Signatures with the following capabilities:

- Users can authorize transactions on their account using any [Auth Method](#auth-methods)
  - Multiple auth methods on one account can be used interchangeably
- No wallet is required to interact with the account
- Users can add and remove [Auth Methods](#auth-methods)
- Multiple independent accounts can be created per [Auth Method](#auth-methods)
- Each [Auth Method](#auth-methods) provides an implicit account on any chain supported by [ChainSignatures MPC](https://github.com/near/mpc) and NEAR
- Account recovery is possible by proving ownership of any [Auth Method](#auth-methods) associated with the account
  - Master Recovery Methods:
    - Email
    - Phone

## Auth Methods

- [OIDC Token](#oidc-auth-contract) (Google, Facebook, Apple)
- [Passkeys](#webauthn-auth-contract)
- [Wallets](#ethereum-auth-contract) (EVM, Solana, Near)
- [Near Account]
- [Phone Number]

## Master Recovery

### Email Recovery
If an account has an OIDC authentication method with an associated email address, the user can authorize transactions by sending a signed email containing the transaction details, even if email authentication was not previously enabled.

#### Why Email?
- Email ownership is required for OIDC authentication
- Email is text-based, making it resistant to blind signing attacks
  - When signing a transaction with OIDC, users cannot verify what they are signing unless the dApp displays the transaction details
- Emails can be cryptographically verified using DKIM signatures
- Like a crypto wallet, email accounts are under sole user control

View: [Email ZK](https://prove.email/)

#### Importance
Email recovery provides a permissionless backup authentication method. This prevents users from being locked out of their accounts if:
- The OIDC provider's service is discontinued
- The client ID website is discontinued

### Phone Recovery
*TODO: Investigate feasibility of implementing similar recovery mechanism using phone numbers*

## Infrastructure

### AbstractAccountContract

This is the EntryPoint contract that manages account storage and key control. It serves as the main interface for account operations, including authentication method management and transaction execution. Below is the proposed interface specification:

```typescript
interface OIDC {
  // The client ID must be included to prevent malicious dApps from taking control of a user's account. Each dApp has its own account that can be merged with other dApp accounts with user authorization
  clientId: string;
  issuer: string;
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
interface AbstractAccountContract {
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
  messageSignature: string, 
  transaction: Transaction, 
  authPath: AuthPath, 
  authTarget: AuthPath
}

class AbstractAccountContract {
  /**
   * @public
   * @call
   */
  async execute(executeArgs: ExecuteArgs): Promise<void> {
    /**
     * 1. Find account based on authPath
     * 2. Validate that account.nonce matches transaction.nonce
     * 3. Validate that authTarget exists on the account
     * 4. Validate that message contains the transaction hash
     *   - Although this logic is auth method specific, it should not be handled by the authContract
     *     since authContract should only validate signatures, not security details of our abstract account
     *   - The message must contain the transaction hash to prevent replay attacks (via nonce) and ensure
     *     that messages can only be used for their intended purpose
     * 5. Validate messageSignature using provided authPath (cross-contract call to corresponding authContract)
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
   * Expensive method as it requires iterating over the account map to find the AuthPath, but should be called rarely since users can provide the accountId for a constant time query
   * 
   * @public
   * @view
   */
  findAccountByAuthPath(authPath: AuthPath): Account | null {}

  /**
   * @public
   * @view
   */
  findAccountByAccountId(accountId: string): Account | null {}

  /**
   * @public
   * @view
   */
  deriveKeyFromPath(authPath: AuthPath): string {}
}
```

#### Auth Paths

Auth Paths act as public keys that are used to authenticate transactions and derive account paths. For example:

```typescript
const authPath: AuthPath = {
  type: "OIDC", 
  value: {
    issuer: "google", 
    clientId: "l2109ufdshf8sdhjf", 
    email: "test@gmail.com"
  }
}

const account = contract.findAccountByAccountId('test@gmail.com')
if (!account.authPaths.includes(authPath)) {
  throw new Error(`authPath: ${JSON.stringify(authPath)} is not authorized on this account`) 
}

const derivedPath = contract.deriveKeyFromPath(authPath)
console.log(derivedPath) // "google,l2109ufdshf8sdhjf,test@gmail.com"
```

### AuthContracts

Stateless contracts used to validate each type of [Auth Method](#auth-methods).

Reasons to deploy them as individual contracts rather than implementing them inside the AbstractAccountContract:

- Reusable across other NEAR contracts
- Independently upgradeable
- Reduced storage on the account management contract

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
    // 1. Fetch the provider's public key from the oracle contract
    // 2. Validate that the token was signed by the provider's public key
    // 3. Return the validation result
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

Contract that fetches and stores OIDC provider public keys on-chain, which are used by OIDCAuthContract to validate token authenticity.

These contracts should support permissionless updates through:

- A decentralized network of oracles
- Zero-knowledge proofs when updating keys

Note: For demo purposes, a centralized trusted server will be used initially.

### Relayer

Relayer REST Server:

- `/sign-near-transaction` endpoint
  - Parameters:
    - receiver_id: Account management contract address (AbstractAccountContract)
    - execute_args: ExecuteArgs
  - Actions:
    - Builds NEAR transaction
    - Signs transaction and sponsors gas fees
    - Submits transaction to RPC node

This relayer enables users to submit blockchain transactions without a wallet.

### OIDC Server (Optional)

Since we cannot guarantee that tokens generated on the client side are 100% authentic, we will introduce this server to use the Client Secret when requesting tokens.

Notes:
- Users can still call the contract without the server by using client-side OIDC token generation
- If we determine that client-generated tokens are authentic and valid, we can remove this server

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

- Since this is an account system within NEAR, interactions with contracts that rely on `predecessor_id` will be less intuitive, as the `predecessor_id` will always be the AbstractAccountContract ID rather than the actual AccountId.
  - This can be addressed by controlling a NEAR account through the ChainSig contract, though this adds some latency due to the extra contract call
    - The cost implications are relative - deploying one AA Smart Contract per account costs 1N/100kB (compared to 2N for a basic stateless contract like the [WebAuthn Contract](https://testnet.nearblocks.io/address/felipe-webauthn.testnet)). A user would need to make approximately 4000 ChainSig calls before the costs equalize.
    - To avoid the latency and extra costs, users can grant dApps a FunctionCall access key to the Near controlled account
- The contract APIs differ from those used with native NEAR accounts, requiring adaptation
  - Build a wallet interface following the wallet-selector standards
 
## Diagrams

[Miro Board](https://miro.com/welcomeonboard/MndKNlAwcFFJWTVDcm5KaVY3VVo1VUdieGNHZzZRNkxDUElMTFJ5d0hrb3JncEY1bGhIMlAzTHZPN1ZlUGF6RHwzMDc0NDU3MzQ1ODc5NjY0NDc0fDI=?share_link_id=667879638731)

![Screenshot 2024-11-07 at 22 35 42](https://github.com/user-attachments/assets/10e8aaae-cd69-49d5-a545-509c5e5e8e23)
![Screenshot 2024-11-07 at 22 35 38](https://github.com/user-attachments/assets/72f20c92-b921-4e1a-9482-f5b76d8a7d47)
