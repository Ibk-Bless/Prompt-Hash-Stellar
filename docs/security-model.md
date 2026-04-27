# Security Model and Threat Analysis

## Overview

PromptHash Stellar implements a multi-layered security architecture that separates content encryption, access control, and payment settlement across client-side, blockchain, and server-side components. This document analyzes the security assumptions, attack vectors, and mitigation strategies for the gated-content flow.

## Security Architecture

### Three-Layer Security Model

1. **Client-Side Encryption Layer**
   - Prompt content is encrypted with AES-256-GCM before leaving the creator's browser
   - AES key is wrapped using libsodium's sealed box (X25519 + XSalsa20-Poly1305)
   - Only encrypted payload and wrapped key are transmitted to the blockchain

2. **Blockchain Access Control Layer**
   - Soroban smart contract maintains authoritative purchase records
   - `has_access` method verifies buyer rights before unlock
   - Payment settlement and access grants are atomic operations

3. **Server-Side Unlock Layer**
   - Challenge-response authentication proves wallet ownership
   - Server unwraps AES key using private key
   - Content integrity verified via SHA-256 hash before delivery

## Trust Assumptions

### What Users Must Trust

1. **Unlock Service Operator**
   - Holds the private key capable of unwrapping encrypted content
   - Must not collude with unauthorized parties to decrypt content
   - Must maintain secure key storage and access controls

2. **Smart Contract Integrity**
   - Contract logic correctly enforces purchase requirements
   - Contract upgrade mechanism is controlled by legitimate admin
   - No bugs allow unauthorized access grants

3. **Stellar Network**
   - Transaction finality is reliable
   - RPC nodes return accurate contract state
   - Network consensus is not compromised

### What Users Do NOT Need to Trust

1. **Content Privacy from Blockchain Observers**
   - Encrypted payload on-chain is cryptographically protected
   - Only holders of the unlock service private key can decrypt

2. **Unlock Service for Payment Settlement**
   - XLM transfers occur entirely on-chain
   - Service cannot steal funds or manipulate prices

3. **Creator Honesty About Content**
   - Content hash allows buyers to verify integrity
   - Mismatch between preview and content is detectable post-purchase

## Attack Vectors and Mitigations

### 1. Service-in-the-Middle Attack

**Attack Scenario:**
A malicious or compromised unlock service operator attempts to:
- Decrypt purchased prompts without authorization
- Sell decrypted content to third parties
- Serve modified content to legitimate buyers

**Mitigations:**

**A. AES Key Wrapping Protection**
- The AES key used for prompt encryption is wrapped using libsodium's `crypto_box_seal`
- Sealed box encryption ensures only the holder of the unlock service private key can unwrap
- Even if the service operator has access to the encrypted payload, they cannot decrypt without the wrapped key being properly unwrapped

**B. Content Integrity Verification**
```typescript
// From api/prompts/unlock.ts
const contentHash = await hashPromptPlaintext(plaintext);
if (contentHash !== prompt.contentHash) {
  res.status(500).json({ error: "Prompt integrity check failed." });
  return;
}
```
- SHA-256 hash of original content is stored on-chain during creation
- Unlock service recomputes hash after decryption
- Any tampering causes hash mismatch and unlock failure

**C. Access Control Enforcement**
```typescript
// From api/prompts/unlock.ts
const access = await hasAccess(config, String(address), id);
if (!access) {
  res.status(403).json({ error: "Prompt access has not been purchased." });
  return;
}
```
- Service queries blockchain state before every unlock
- Cannot bypass on-chain purchase verification
- Audit trail exists in blockchain transaction history

**D. Challenge-Response Authentication**
```typescript
// From src/lib/auth/challenge.ts
const validSignature = verifyChallengeSignature(
  String(address),
  challengeMessage,
  String(signedMessage),
);
```
- Wallet must sign a time-limited challenge message
- Proves cryptographic ownership of the purchasing address
- Prevents impersonation attacks

**Residual Risk:**
- Unlock service operator with private key access can still decrypt content
- Mitigation requires operational security controls:
  - Hardware security module (HSM) for key storage
  - Audit logging of all unlock operations
  - Multi-party computation for key management (future enhancement)

### 2. Double-Spend Attack

**Attack Scenario:**
A malicious buyer attempts to:
- Purchase access to a prompt
- Obtain the decrypted content
- Reverse or cancel the payment transaction
- Retain access to the content without payment

**Mitigations:**

**A. Atomic Purchase and Access Grant**
```rust
// From contracts/prompt-hash/src/contract.rs
fn buy_prompt(env: Env, buyer: Address, prompt_id: u128) -> Result<(), Error> {
    buyer.require_auth();
    // ... validation checks ...
    
    Storage::set_reentrancy_guard(&env)?;
    
    // Transfer XLM to seller and fee wallet
    xlm.transfer_from(&this_contract, &buyer, &prompt.creator, &seller_amount);
    if fee_amount > 0 {
        xlm.transfer_from(&this_contract, &buyer, &fee_wallet, &fee_amount);
    }
    
    // Grant access only after successful payment
    Storage::grant_purchase(&env, prompt_id, &buyer);
    Storage::clear_reentrancy_guard(&env);
    // ...
}
```
- Payment transfer and access grant occur in a single transaction
- Reentrancy guard prevents recursive calls during payment
- Transaction either fully succeeds or fully reverts

**B. Stellar Transaction Finality**
- Stellar provides fast finality (3-5 seconds)
- Once transaction is confirmed, it cannot be reversed
- No blockchain reorganization risk like proof-of-work chains

**C. Unlock Service Verification**
```typescript
// From api/prompts/unlock.ts
const access = await hasAccess(config, String(address), id);
```
- Service queries current blockchain state before unlock
- Uses Stellar RPC to verify purchase record exists
- No caching of access rights that could be stale

**D. Purchase Idempotency**
```rust
// From contracts/prompt-hash/src/contract.rs
ensure(!Storage::has_purchase(&env, prompt_id, &buyer), Error::AlreadyPurchased)?;
```
- Contract prevents duplicate purchases by same buyer
- Protects against accidental double-payment
- Simplifies access verification logic

**Residual Risk:**
- If buyer obtains content before transaction confirms, they could abandon payment
- Mitigation: Unlock service should wait for transaction confirmation before serving content
- Current implementation queries contract state, which reflects confirmed transactions only

### 3. Brute-Force Wallet Signature Attack

**Attack Scenario:**
An attacker attempts to:
- Request many challenge tokens for a prompt
- Try to forge wallet signatures
- Overwhelm the unlock service with requests

**Mitigations:**

**A. Rate Limiting (Implemented)**
```typescript
// From api/auth/challenge.ts and api/prompts/unlock.ts
const rateLimit = checkRateLimit("challenge", clientIp);
if (!rateLimit.success) {
  res.status(429).json({ error: "Too many requests. Please try again later." });
  return;
}
```
- IP-based rate limiting on challenge issuance
- Wallet-based rate limiting on unlock attempts
- Prevents brute-force signature guessing

**B. Time-Limited Challenges**
```typescript
// From src/lib/auth/challenge.ts
const DEFAULT_TTL_MS = 5 * 60 * 1000; // 5 minutes
if (payload.expiresAt < now) {
  throw new Error("Challenge token has expired.");
}
```
- Challenge tokens expire after 5 minutes
- Limits window for signature forgery attempts
- Forces attacker to request new challenges frequently

**C. Cryptographic Signature Verification**
```typescript
// From src/lib/auth/challenge.ts
return Keypair.fromPublicKey(address).verify(
  Buffer.from(message, "utf8"),
  decodeSignature(signedMessage),
);
```
- Uses Stellar's Ed25519 signature verification
- Computationally infeasible to forge valid signatures
- Each challenge includes unique nonce preventing replay

**Residual Risk:**
- Distributed attacks from many IPs could bypass rate limits
- Mitigation: Implement Redis-backed distributed rate limiting (Issue #62)
- Add CAPTCHA for suspicious request patterns

### 4. Creator Front-Running Attack

**Attack Scenario:**
A malicious creator attempts to:
- List a prompt with attractive preview
- Wait for purchase transaction to be submitted
- Front-run with a transaction to deactivate the prompt
- Keep payment without providing access

**Mitigations:**

**A. Purchase Validation**
```rust
// From contracts/prompt-hash/src/contract.rs
ensure(prompt.active, Error::PromptInactive)?;
```
- Contract checks prompt is active at purchase time
- Transaction reverts if prompt is inactive
- No payment occurs if validation fails

**B. Transaction Atomicity**
- Stellar transactions are atomic
- Either all operations succeed or all fail
- No partial state where payment succeeds but access is denied

**C. Access Grant Independence**
```rust
// From contracts/prompt-hash/src/contract.rs
fn has_access(env: Env, user: Address, prompt_id: u128) -> Result<bool, Error> {
    let prompt = Storage::require_prompt(&env, prompt_id)?;
    Ok(prompt.creator == user || Storage::has_purchase(&env, prompt_id, &user))
}
```
- Once purchase is recorded, access persists regardless of prompt status
- Creator cannot revoke access after sale
- Deactivating prompt only prevents new purchases

**Residual Risk:**
- Creator could update prompt content after sale (not currently prevented)
- Mitigation: Implement versioning system (Issue #65)

### 5. Unlock Service Denial of Service

**Attack Scenario:**
An attacker attempts to:
- Overwhelm unlock service with requests
- Exhaust server resources
- Prevent legitimate users from unlocking content

**Mitigations:**

**A. Rate Limiting**
- IP-based and wallet-based request limits
- Prevents single attacker from monopolizing resources

**B. Challenge Token Validation**
```typescript
// From src/lib/auth/challenge.ts
const expectedSignature = signPayload(secret, encodedPayload);
if (!timingSafeEqual(received, expected)) {
  throw new Error("Invalid challenge token signature.");
}
```
- Server-signed tokens prevent unauthorized unlock attempts
- Attacker must first obtain valid challenge tokens
- Challenge endpoint also rate-limited

**C. Observability and Monitoring**
```typescript
// From api/prompts/unlock.ts
metrics.trackUnlockFailure(String(address), String(promptId), "invalid_signature");
req.logger.warn({ address, promptId }, "Invalid wallet signature");
```
- Structured logging tracks attack patterns
- Metrics enable real-time alerting
- Incident response procedures documented in `docs/operations/`

**Residual Risk:**
- Sophisticated DDoS attacks could still overwhelm service
- Mitigation: Deploy behind CDN with DDoS protection
- Implement Redis-backed distributed rate limiting (Issue #62)

## Security Best Practices

### For Creators

1. **Content Verification**
   - Verify preview accurately represents full content
   - Test unlock flow before listing
   - Monitor sales and buyer feedback

2. **Pricing Strategy**
   - Set appropriate prices for content value
   - Consider that content may be shared after purchase
   - Use preview to demonstrate value without revealing full content

### For Buyers

1. **Preview Evaluation**
   - Carefully review preview before purchase
   - Check creator reputation and sales history
   - Understand that purchases are final

2. **Wallet Security**
   - Use secure wallet with signature verification
   - Verify transaction details before signing
   - Keep wallet private keys secure

### For Operators

1. **Key Management**
   - Store unlock service private key in secure environment
   - Rotate challenge token secret regularly
   - Use HSM for production key storage

2. **Monitoring and Response**
   - Monitor unlock success/failure rates
   - Alert on unusual request patterns
   - Follow incident response procedures

3. **Infrastructure Security**
   - Keep dependencies updated
   - Use HTTPS for all endpoints
   - Implement distributed rate limiting

## Future Security Enhancements

1. **Multi-Party Computation (MPC)**
   - Distribute unlock key across multiple parties
   - Require threshold of parties to decrypt
   - Eliminates single point of trust

2. **Zero-Knowledge Proofs**
   - Prove purchase without revealing buyer identity
   - Enable privacy-preserving access verification
   - Reduce on-chain data exposure

3. **Decentralized Unlock Network**
   - Multiple independent unlock service operators
   - Buyers choose which service to trust
   - Market competition improves security incentives

4. **Content Versioning**
   - Track prompt updates over time
   - Buyers access version they purchased
   - Prevents bait-and-switch attacks (Issue #65)

5. **Reputation System**
   - On-chain creator reputation scores
   - Buyer reviews and ratings
   - Dispute resolution mechanism

## Conclusion

PromptHash Stellar's security model balances practical usability with strong cryptographic protection. The three-layer architecture ensures that:

- Content remains encrypted on-chain and in transit
- Access control is enforced by immutable smart contract logic
- Payment settlement is atomic and irreversible
- Wallet ownership is cryptographically verified

The primary trust assumption is the unlock service operator's integrity, which is mitigated through operational security controls, audit logging, and content integrity verification. Future enhancements can further decentralize trust through MPC and distributed unlock networks.

For production deployment, operators should implement the recommended security best practices, particularly around key management, rate limiting, and monitoring infrastructure.
