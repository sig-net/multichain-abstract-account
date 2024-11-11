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
type AuthIdentity = 
  | { type: "PubKey", value: string } // Wallet authentication
  | { type: "WebAuthn", value: WebAuthn } // Passkey authentication 
  | { type: "OIDC", value: OIDC } // Social authentication (e.g: Google, Facebook, Apple)
  | { type: "Account", value: string }; // Near Account authentication

interface Account {
  authIdentities: AuthIdentity[];
  nonce: number;
}

// Should implement the Storage Management NEP (https://nomicon.io/Standards/StorageManagement)
interface AbstractAccountContract {
  accounts: Map<string, Account>; // AccountId => Account
  authContracts: Map<string, AccountId> // AuthMethodId => AuthMethodContractAddress
}

type Action = 
  | { type: "AddKey", authIdentity: AuthIdentity }
  | { type: "RemoveKey", authIdentity: AuthIdentity }
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
  authIdentity: AuthIdentity, 
  authTarget: AuthIdentity
}

class AbstractAccountContract {
  /**
   * @public
   * @call
   */
  async execute(executeArgs: ExecuteArgs): Promise<void> {
    /**
     * 1. Find account based on authIdentity
     * 2. Validate that account.nonce matches transaction.nonce
     * 3. Validate that authTarget exists on the account
     * 4. Validate that message contains the transaction hash
     *   - Although this logic is auth method specific, it should not be handled by the authContract
     *     since authContract should only validate signatures, not security details of our abstract account
     *   - The message must contain the transaction hash to prevent replay attacks (via nonce) and ensure
     *     that messages can only be used for their intended purpose
     * 5. Validate messageSignature using provided authIdentity (cross-contract call to corresponding authContract)
     * 6. Call executeCallback with authContract result
     */
  }

  async executeCallback(): Promise<void> {
    // 1. If authContract result === true, execute the actions on transaction.actions
  }

  /**
   * @public
   * @call
   */
  async addAccount(accountId: string, authIdentity: AuthIdentity): Promise<void>{}

  // Methods that can be called by executeCallback
  private async addAuthIdentity(authIdentity: AuthIdentity): Promise<void> {}
  private async removeAuthIdentity(authIdentity: AuthIdentity): Promise<void> {}
  private async callChainSig(authTarget: AuthIdentity, actions: Action[]): Promise<void> {
    // Call ChainSig contract using authTarget as the path
  }
  private async deleteAccount(accountId: string): Promise<void> {}

  /**
   * Expensive method as it requires iterating over the account map to find the AuthIdentity, but should be called rarely since users can provide the accountId for a constant time query
   * 
   * @public
   * @view
   */
  findAccountByAuthIdentity(authIdentity: AuthIdentity): Account | null {}

  /**
   * @public
   * @view
   */
  findAccountByAccountId(accountId: string): Account | null {}

  /**
   * @public
   * @view
   */
  deriveKeyFromPath(authIdentity: AuthIdentity): string {}
}
```

#### Auth Identity

Auth Identity act as identities that are authorized to execute transactions and derive account paths. For example:

```typescript
const authIdentity: AuthIdentity = {
  type: "OIDC", 
  value: {
    issuer: "google", 
    clientId: "l2109ufdshf8sdhjf", 
    email: "test@gmail.com"
  }
}

const account = contract.findAccountByAccountId('test@gmail.com')
if (!account.authIdentities.includes(authIdentity)) {
  throw new Error(`authIdentity: ${JSON.stringify(authIdentity)} is not authorized on this account`) 
}

const derivedPath = contract.deriveKeyFromPath(authIdentity)
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

### OIDC Zero-Knowledge Proof Server

To enhance user privacy, we plan to implement a zero-knowledge proof server that verifies OIDC token authenticity without exposing the full token on-chain.

Proposed Zero-Knowledge Proof Interface:

Public Inputs:
- OIDC Provider's Public Key
- User's Email Address

Private Inputs:
- Complete OIDC Token

Public Outputs:
- Boolean value indicating token authenticity

Using this interface, we can verify email ownership without exposing the complete OIDC token on-chain.

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

## Notes on OIDC Token

- Client-side flow is possible and secure using PKCE (Proof Key for Code Exchange) as defined in [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
- According to OIDC standards, the OIDC token can optionally include a nonce provided by the client. By using the transaction hash as this nonce, we ensure the OIDC token can only be used for its intended transaction, thus preventing replay attacks

## Issues

- Since this is an account system within NEAR, interactions with contracts that rely on `predecessor_id` will be less intuitive, as the `predecessor_id` will always be the AbstractAccountContract ID rather than the actual AccountId.
  - This can be addressed by controlling a NEAR account through the ChainSig contract, though this adds some latency due to the extra contract call
    - The cost implications are relative - deploying one AA Smart Contract per account costs 1N/100kB (compared to 2N for a basic stateless contract like the [WebAuthn Contract](https://testnet.nearblocks.io/address/felipe-webauthn.testnet)). A user would need to make approximately 4000 ChainSig calls before the costs equalize.
    - To avoid the latency and extra costs, users can grant dApps a FunctionCall access key to the Near controlled account
- The contract APIs differ from those used with native NEAR accounts, requiring adaptation
  - Build a wallet interface following the wallet-selector standards
 
## Diagrams

[Miro Board](https://miro.com/welcomeonboard/MndKNlAwcFFJWTVDcm5KaVY3VVo1VUdieGNHZzZRNkxDUElMTFJ5d0hrb3JncEY1bGhIMlAzTHZPN1ZlUGF6RHwzMDc0NDU3MzQ1ODc5NjY0NDc0fDI=?share_link_id=667879638731)

### OIDC Auth Flow
![Screenshot 2024-11-07 at 22 49 00](https://github.com/user-attachments/assets/40b70aef-c463-45ff-a609-f7ca1fc73185)

### Wallet/Passkey Auth Flow
![Screenshot 2024-11-07 at 22 49 10](https://github.com/user-attachments/assets/f486fd72-8102-40da-b2ea-9531f2a61fea)

### Control Near Account Flow
![Screenshot 2024-11-07 at 22 49 50](https://github.com/user-attachments/assets/a9d673d4-9db3-4728-8d65-ada1f7276b10)
