# XLS-ConfidentialTransfer: Confidential transfers for XRPL

**Authors:** Murat Cenk, Aanchal Malhotra  
**Status:** Draft  
**Type:** XRPL Standard (XLS)  
**Version:** 0.1

---

## Abstract

This standard proposes a confidential transfer mechanism for the XRPL that hides transaction amounts using elliptic curve encryption and ZKPs. The design introduces encrypted balance handling, homomorphic updates, and optional compliance features. Receiver privacy and full anonymity are supported as optional extensions.

---

## Motivation

While XRPL provides fast and efficient financial transfers, all transaction details are fully transparent. For enterprise, retail, and tokenized asset use cases, confidential transfers are essential to protect business-sensitive information and user privacy.

This proposal aims to:
- Enable confidential transaction amounts.
- Enable validation correctness.
- Offer optional compliance and privacy extensions.

---

## Specification

### Transaction Types

- **ConfidentialMint**: Converts a public XRP balance into an encrypted confidential balance.
- **ConfidentialSend**: Transfers confidential amounts between users using dual encryption and zero-knowledge proofs.

---

### ConfidentialMint Transaction Format

| Field           | Type   | Required | Description                                |
|----------------|--------|----------|--------------------------------------------|
| Amount          | UInt64 | Yes      | The plain XRP amount to convert            |
| EncryptedBalance| Blob   | Yes      | EC-ElGamal ciphertext of the amount        |
| EqualityProof   | Blob   | Yes      | ZK proof that EncryptedBalance equals Amount |
| PublicKey       | Blob   | Yes      | Sender's EC-ElGamal public key             |


---
### EC-ElGamal ciphertext and homomorphic properties

Let G be the generator of the elliptic curve group, and let pk = x * G denote the user’s EC-ElGamal public key for a private scalar x in Z_q. To encrypt a known plaintext amount m in Z_q, the user samples a random blinding scalar r in Z_q and computes the ciphertext:

    C = (c1, c2) = (r * G, m * G + r * pk)

This ciphertext C = (c1, c2), where both components are curve points, encrypts the amount m while preserving homomorphic properties. Given two ciphertexts:

    C1 = (r1 * G, m1 * G + r1 * pk)
    C2 = (r2 * G, m2 * G + r2 * pk)

The component-wise addition yields:

    C1 + C2 = ((r1 + r2) * G, (m1 + m2) * G + (r1 + r2) * pk)

which is a valid EC-ElGamal encryption of (m1 + m2). Similarly, subtraction:

    C1 - C2 = ((r1 - r2) * G, (m1 - m2) * G + (r1 - r2) * pk)

produces a ciphertext encrypting (m1 - m2). These homomorphic properties enable secure balance updates on ledger entries without revealing the plaintext amounts.

---
### Equality Proof Between Plaintext and Ciphertext

In the `ConfidentialMint` transaction, the `EqualityProof` is a zero-knowledge proof showing that a given plaintext amount `m` matches the encrypted value in the EC-ElGamal ciphertext `C = (A, B)`.

The encryption is defined as:

    A = r * G
    B = m * G + r * pk

where:
- `G` is the generator of the elliptic curve group,
- `pk = x * G` is the recipient’s public key,
- `r` is a random blinding scalar in Z_q,
- `m` is the plaintext amount.

The goal is to prove knowledge of `r` such that both:

    A = r * G
    B - m * G = r * pk

This corresponds to a standard **Chaum-Pedersen proof** of equality of discrete logarithms across two bases (`G` and `pk`), without revealing `r`.

#### Steps to Generate the EqualityProof

1. Choose a random scalar `t ∈ Z_q`
2. Compute the commitments:
   `T1 = t * G`
   `T2 = t * pk`
3. Compute the challenge:
   `c = H(A, B, T1, T2, m)`
   where `H` is a cryptographic hash function modeled as a random oracle
4. Compute the response:
   s = t + c * r mod q

#### The Proof

The proof consists of the tuple `(T1, T2, s)`

#### Verification

To verify the proof, the verifier recomputes the challenge `c` and checks that:

    s * G  == T1 + c * A
    s * pk == T2 + c * (B - m * G)

If both equalities hold, the verifier is convinced that the ciphertext `(A, B)` correctly encrypts the known plaintext `m` without learning `r`.

### ConfidentialSend Transaction Format

| Field         | Type | Required | Description                                       |
|---------------|------|----------|---------------------------------------------------|
| C_send        | Blob | Yes      | Ciphertext encrypted with sender's key           |
| C_receive     | Blob | Yes      | Ciphertext encrypted with recipient’s key        |
| EqualityProof | Blob | Yes      | ZK proof: both ciphertexts encrypt the same value |
| RangeProof    | Blob | Yes      | ZK proof: sender balance - amount ≥ 0            |
| AuditorField  | Blob | Optional | Ciphertext encrypted with auditor's public key   |
| PublicKeys    | Blob | Yes      | Public keys used in the transaction              |

---
### Equality Proof Between Two EC-ElGamal Ciphertexts

This section describes a ZKP that two EC-ElGamal ciphertexts encrypt the same plaintext value `m`, possibly using different public keys and randomness. The proof reveals nothing about `m`, `r1`, or `r2`.

#### Setting

Let:

    C1 = (R1, S1) = (r1 * G, m * G + r1 * P1)
    C2 = (R2, S2) = (r2 * G, m * G + r2 * P2)

where:
- `G` is the curve generator,
- `P1`, `P2` are EC-ElGamal public keys,
- `r1`, `r2 ∈ Z_q` are blinding scalars (nonces),
- `m ∈ Z_q` is the plaintext scalar value to encrypt.

The prover wants to convince the verifier that:

    S1 - r1 * P1 = S2 - r2 * P2 = m * G

which implies that both ciphertexts encrypt the same `m`.

This reduces to proving that two group elements `A1` and `A2` have the same discrete logarithm base `G`:

    A1 = S1 - r1 * P1
    A2 = S2 - r2 * P2
    log_G(A1) = log_G(A2) = m

#### Protocol (Non-Interactive via Fiat–Shamir)

1. Prover computes:
   A1 = S1 - r1 * P1 and A2 = S2 - r2 * P2

2. Chooses a random scalar `w ∈ Z_q`

3. Computes:
   t1 = w * G and t2 = w * G

4. Computes challenge:
   c = H(G, A1, A2, t1, t2)

5. Computes response:
   z = w + c * m mod q

#### The Proof

The proof consists of A1, A2, t1, t2, and z.

#### Verification

The verifier recomputes `c` and checks that:

    z * G == t1 + c * A1
    z * G == t2 + c * A2

Since `A1 = A2`, verifying either is sufficient, but verifying both adds consistency and protection against malformed inputs.

If the equations hold, the verifier is convinced that both ciphertexts encrypt the same plaintext `m` without learning anything about `m`, `r1`, or `r2`.


---
### Bitwise Range Proof

In the `ConfidentialSend` transaction, the `RangeProof` field is a ZKP
that ensures the  `balanace - m ≥  0`. This prevents encoding invalid or negative amounts
while maintaining confidentiality.

#### Bit Decomposition

The prover represents the amount `m ∈ Z_q` as a binary sum:

    m = b_0 * 2^0 + b_1 * 2^1 + ... + b_{n-1} * 2^{n-1}, where each b_i ∈ {0,1}

#### Commitments to Bits

Each bit `b_i` is committed using a Pedersen commitment:

    C_i = b_i * G + r_i * H

where
- `G` and `H` are independent elliptic curve generators
- `r_i ∈ Z_q` is a blinding factor for bit `i`

#### Aggregation of Commitments

The prover computes the overall commitment to the amount:

    C = ∑_{i=0}^{n-1} 2^i * C_i = m * G + r * H

where

    r = ∑_{i=0}^{n-1} 2^i * r_i

This `C` serves as the link between the bitwise decomposition and the encrypted value.

#### Bit Validity Proofs (b_i ∈ {0,1})

For each bit commitment `C_i`, the prover constructs a zero-knowledge OR-proof
demonstrating that:

    C_i = 0 * G + r_0 * H    or    C_i = 1 * G + r_1 * H

This is instantiated using a standard disjunctive Σ-protocol based on by Cramer, Damgård, and Schoenmakers (CRYPTO 1994).  The interactive protocol is converted to a non-interactive form via the Fiat–Shamir heuristic.

#### Linking to EC-ElGamal Ciphertext

The bit commitment `C` is linked to the EC-ElGamal ciphertext:

    (A, B) = (r * G, m * G + r * pk)

This is done using the `EqualityProof` described earlier, via a Chaum–Pedersen protocol
showing that:

    C - m * G = r * H
    B - m * G = r * pk

Without revealing `m` or `r`.

#### Summary

The complete bitwise range proof includes:
- `n` Pedersen commitments to bits: C_0, ..., C_{n-1}
- `n` OR-proofs that each b_i ∈ {0,1}
- An aggregation check that ∑ 2^i * C_i = C
- An EqualityProof that C and the ciphertext represent the same `m`

This ensures that `m` is a valid non-negative amount encrypted under EC-ElGamal
while maintaining zero knowledge.

---
### Base-3 Range Proof

As an alternative to bitwise decomposition, the `RangeProof` may use a base-3 (ternary) decomposition,
where the plaintext amount `m ∈ [0, 3^n)` is expressed using digits in `{0, 1, 2}`.

This method reduces the number of proof elements compared to bitwise proofs, at the cost of more complex
per-digit validity checks. It is particularly useful when optimizing for proof size.

#### Base-3 Decomposition

The prover expresses the value `m` as:

    m = d_0 * 3^0 + d_1 * 3^1 + ... + d_{n-1} * 3^{n-1}, where each d_i ∈ {0, 1, 2}

#### Commitments to Digits

Each ternary digit `d_i` is committed using a Pedersen commitment:

    C_i = d_i * G + r_i * H

Where:
- `G` and `H` are independent elliptic curve generators
- `r_i ∈ Z_q` is a blinding scalar

#### Aggregation of Commitments

The prover computes the aggregate commitment to the value:

    C = ∑_{i=0}^{n-1} 3^i * C_i = m * G + r * H

Where:

    r = ∑_{i=0}^{n-1} 3^i * r_i

This `C` links the ternary digit commitments to the encrypted value `m`.

#### Digit Validity Proofs (d_i ∈ {0,1,2})

To ensure each committed digit `d_i` is in `{0,1,2}`, the prover uses a **zero-knowledge OR-proof** over three cases:

    C_i = 0 * G + r_0 * H
    C_i = 1 * G + r_1 * H
    C_i = 2 * G + r_2 * H

This is a 3-ary disjunctive Σ-protocol, following the generalized Cramer–Damgård–Schoenmakers (CDS) framework.

As with bitwise proofs, this interactive protocol is made non-interactive via the Fiat–Shamir transform.

#### Linking to EC-ElGamal Ciphertext

The aggregate commitment `C` is linked to the EC-ElGamal ciphertext `(A, B)` using the same
`EqualityProof` described earlier:

    A = r * G
    B = m * G + r * pk

The `EqualityProof` confirms that both representations encode the same plaintext `m`.

#### Summary

A base-3 range proof includes:
- `n` Pedersen digit commitments: C_0, ..., C_{n-1}
- `n` 3-ary OR-proofs ensuring d_i ∈ {0,1,2}
- An aggregation check: ∑ 3^i * C_i = C
- An EqualityProof linking `C` and the ciphertext

Using base-3 decomposition reduces the number of digits required (compared to base-2),
trading off slightly more complex per-digit proofs for fewer total commitments.

---
### Bulletproof Range Proof

As an alternative to bitwise or base-3 decompositions, the `RangeProof` may use **Bulletproofs** by Bünz, Bootle, Boneh, Poelstra, Wuille, and Maxwell that is
a logarithmic-size zero-knowledge proof system efficiently proving that a committed value lies
within a certain range without revealing it.

This approach significantly reduces proof size and verification time for typical 64-bit confidential
amounts, and is suitable when compactness and scalability are important.

#### Commitment

The prover commits to the amount `m` using a standard Pedersen commitment:

    C = m * G + r * H

Where:
- `G`, `H` are independent elliptic curve generators
- `r ∈ Z_q` is a blinding scalar
- `m ∈ Z_q` is the confidential plaintext amount

#### Range Statement

The prover proves in zero knowledge that:

    0 ≤ m < 2^n

For example, with `n = 64`, the proof shows that `m` is a valid 64-bit non-negative integer.

Unlike bitwise proofs, Bulletproofs do **not** require committing to each bit individually. Instead,
the full range proof is built from the single commitment `C`.

#### Proof Protocol

The Bulletproof protocol constructs an inner product argument that proves `m` is composed of valid bits
without sending each bit commitment separately. The protocol includes:

1. Vector polynomials encoding the bits of `m`
2. An interactive proof of the inner product
3. Fiat–Shamir transformation to make it non-interactive

The resulting proof size is:

    Size = 2 * log₂(n) + 9 group elements

For `n = 64`, the proof is ~700 bytes — significantly smaller than bitwise or base-3 alternatives.

#### Linking to EC-ElGamal Ciphertext

The Pedersen commitment `C` used in the Bulletproof must be shown to match the plaintext value
used in the EC-ElGamal ciphertext:

    A = r * G
    B = m * G + r * pk

This linkage is proven using the standard `EqualityProof` described earlier:

    C - m * G = r * H
    B - m * G = r * pk

#### Summary

A Bulletproof-based range proof includes:
- A single Pedersen commitment `C = m * G + r * H`
- A compact logarithmic-size ZK proof that `m ∈ [0, 2^n)`
- An `EqualityProof` linking `C` and the EC-ElGamal ciphertext
  
Bulletproofs are well-suited for high-performance applications where minimizing proof size and verification cost is critical, while preserving strong zero-knowledge guarantees.

---
### Validation Rules

- Validate equality and range proofs
- Perform EC-ElGamal homomorphic updates:
  - `Sender_Balance' = Sender_Balance - C_send`
  - `Receiver_Balance' = Receiver_Balance + C_receive`
- Verify optional auditor encryption if provided
- Store updated encrypted balances in ledger

---

### Ledger Changes

- New encrypted balance field in AccountRoot object
- New transaction types: `ConfidentialMint` and `ConfidentialSend`
- Optional support for auditor ciphertexts and note objects
---
### Compliance: Selective Disclosure

Two supported methods:
- **Auditor Encryption**: Encrypted output for auditor + equality proof
- **On-Demand Disclosure**: Users may share decryption key/amount when required





## Appendix A: Receiver Privacy Extensions

### A.1 Use Case 1: Receiver Anonymity Only (No Amount Confidentiality)

**Transaction Type:** `ReceiverPrivacySend`

| Field          | Type   | Required | Description                                |
|----------------|--------|----------|--------------------------------------------|
| Amount         | UInt64 | Yes      | Public amount of XRP being sent            |
| EphemeralKey   | Blob   | Yes      | One-time public key `R = r·G`              |
| StealthAddress | Blob   | Yes      | `P = H(r·B_view) + B_spend`                |
| Metadata       | Blob   | Optional | Encrypted memo or payload                  |

### Stealth Protocol

This section describes how sender and receiver generate unlinkable, one-time addresses using elliptic curve Diffie-Hellman and hashing. The goal is to enable recipient privacy by ensuring that only the receiver can detect and spend funds sent to a stealth address.

#### Participants

- **Alice (Sender)**:
  - Holds keypair: private key `a`, public key `A = a * G`
- **Bob (Receiver)**:
  - Holds keypair: private key `b`, public key `B = b * G`

Here, `G` is the generator point of the elliptic curve group.

#### Step 1: Receiver Key Generation (Bob)

- Bob generates a static keypair:
  - Private key: `b ∈ Z_q`
  - Public key: `B = b * G`

This keypair is used for deriving stealth addresses and later recovering received notes.

#### Step 2: Stealth Address Generation (Alice)

- Alice generates a fresh ephemeral scalar `r ∈ Z_q`
- Computes the temporary public key `R = r * G`

This `R` is published with the transaction.

- Computes the shared secret:

      s = H(r * B)

  where `H` is a cryptographic hash function mapping to `Z_q`.

- Derives the one-time public key for the recipient:

      P = s * G + B

This `P` is the stealth address used in the note or output.

#### Step 3: Recipient Detection and Key Recovery (Bob)

- Bob monitors the ledger for transactions containing `R`.
- For each detected `R`, Bob computes:

      s = H(b * R)

- Reconstructs the one-time public key:

      P = s * G + B

- Computes the corresponding one-time private key:

      a' = s + b  ∈ Z_q

This ensures:

      a' * G = (s + b) * G = s * G + B = P

Bob can now use `a'` to spend the funds sent to stealth address `P`, and only Bob can compute this key.

#### Validation
- Sender transfers public XRP to stealth address.
- Receiver scans ledger with `b_view` key.

---

### A.2 Use Case 2: Receiver Anonymity with Confidential Amounts

**Transaction Type:** `ReceiverPrivacyConfidentialSend`

| Field          | Type   | Required | Description                                 |
|----------------|--------|----------|---------------------------------------------|
| C_send         | Blob   | Yes      | Ciphertext encrypted with sender’s key      |
| C_receive      | Blob   | Yes      | Ciphertext encrypted to stealth recipient   |
| EqualityProof  | Blob   | Yes      | ZK proof that both ciphertexts match        |
| RangeProof     | Blob   | Yes      | ZK proof of sender balance sufficiency      |
| EphemeralKey   | Blob   | Yes      | Sender’s one-time key                       |
| StealthAddress | Blob   | Yes      | Receiver’s computed address                 |
| AuditorField   | Blob   | Optional | Encrypted payload for auditor               |

- Validation same as `ConfidentialSend`
- No additional ledger change

---

## Appendix B: Full Confidential and Private Transfers

Combines:
- Confidential balances and amounts
- Receiver anonymity (stealth addresses)
- Sender anonymity (ring signatures)

---

### Transaction Types

| Transaction Type          | Description                                           |
|---------------------------|-------------------------------------------------------|
| NoteMintFromPublic        | Creates a confidential note from public XRP           |
| NoteMint                  | Creates a confidential note from confidential balance |
| ConfidentialTransfer      | Spends a note with ring signature and stealth address |
| ConfidentialBurn          | Converts a note back into public XRP                  |

---

### NoteMintFromPublic Transaction Format

| Field         | Type   | Required | Description                                  |
|---------------|--------|----------|----------------------------------------------|
| Amount        | UInt64 | Yes      | Plain XRP amount                             |
| Commitment    | Blob   | Yes      | Pedersen commitment `C = r·G + m·H`          |
| OwnerKey      | Blob   | Yes      | `P = s·G + B_spend`                          |
| EphemeralKey  | Blob   | Yes      | One-time key `R = k·G`                        |
| EqualityProof | Blob   | Yes      | ZK proof `C` encodes `Amount`                |

**Validator Actions:**
- Verify `EqualityProof`
- Deduct `Amount` from public balance
- Store Note: `(Commitment, OwnerKey, EphemeralKey, Spent = false)`

---

### NoteMint Transaction Format

| Field         | Type | Required | Description                              |
|---------------|------|----------|------------------------------------------|
| Commitment    | Blob | Yes      | Pedersen commitment `C = r·G + m·H`      |
| OwnerKey      | Blob | Yes      | One-time key for note ownership          |
| EphemeralKey  | Blob | Yes      | For stealth detection                    |
| RangeProof    | Blob | Yes      | ZK proof: balance covers note amount     |

**Validator Actions:**
- Verify `RangeProof`
- Deduct note amount homomorphically
- Store Note with `Spent = false`

---

### ConfidentialTransfer Transaction Format

| Field          | Type | Required | Description                                   |
|----------------|------|----------|-----------------------------------------------|
| Ring           | Blob | Yes      | Ring of public keys                           |
| KeyImage       | Blob | Yes      | Unique tag for double-spend prevention        |
| RingSignature  | Blob | Yes      | Proof of signer ownership                     |
| NewNotes       | List | Yes      | New notes with (Commitment, OwnerKey, EphemeralKey) |
| RangeProofs    | Blob | Yes      | ZK range proofs on outputs                    |
| BalanceProof   | Blob | Yes      | ZK proof: input = sum(output)                 |

### One-Time Ring Signatures and Key Images

This section describes a cryptographic scheme that allows a sender to prove, in zero knowledge, that they control one of a set of public keys without revealing which one. It preserves sender anonymity while preventing double-spending via key images.

#### Key Definitions

Let `G` be the elliptic curve generator (e.g., Ed25519 or secp256k1). Let:

- `x ∈ Z_l` be the sender’s private key
- `P = x * G` be the corresponding public key
- `Hp(P)` be a hash-to-curve function that maps `P` deterministically to a curve point
- The **key image** is defined as:

      I = x * Hp(P)

This `I` is unlinkable to `x` but uniquely identifies a spent key.

---

### Signature Construction

Let `m` be the message to sign, and `S = {P_0, P_1, ..., P_n}` be a set of public keys (the ring). The signer is at index `s` with private key `x_s`.

1. **Generate randomness:**
  - Choose random scalars `q_i ∈ Z_l` for all `i ∈ [0, n]`
  - Choose random responses `w_i ∈ Z_l` for all `i ≠ s`

2. **Compute commitments:**

   For all `i ∈ [0, n]`:

  - If `i == s`:

        L_i = q_i * G  
        R_i = q_i * Hp(P_i)

  - If `i ≠ s`:

        L_i = q_i * G + w_i * P_i  
        R_i = q_i * Hp(P_i) + w_i * I

3. **Compute challenge:**

   c = H(m || L_0 || ... || L_n || R_0 || ... || R_n)

4. **Compute responses for signer’s index `s`:**

   c_s = c - Σ_{i ≠ s} c_i mod l  
   r_s = q_s - c_s * x_s mod l

5. **The final ring signature is:**

   σ = (I, {c_i}, {r_i}) for i = 0 to n

---

### Signature Verification

Given message `m`, signature `σ = (I, {c_i}, {r_i})`, and public key ring `S = {P_0, ..., P_n}`:

1. **Recompute commitments:**

   For each `i`:

       L_i' = r_i * G + c_i * P_i  
       R_i' = r_i * Hp(P_i) + c_i * I

2. **Recompute challenge:**

       c' = H(m || L_0' || ... || L_n' || R_0' || ... || R_n')

3. **Accept the signature if:**

       c' = Σ_{i=0 to n} c_i mod l

---

### Key Image Linkability

To prevent double-spending:

- Validators maintain a global set of used key images.
- If the key image `I` has already been recorded, the transaction is rejected.
- Otherwise, the key image `I` is added to the ledger as spent.

Since the key image is deterministically derived from the signer’s private key (`I = x * Hp(P)`), it uniquely identifies a spent note without revealing which ring member signed the transaction.



**Validator Actions:**
- Verify `RingSignature`, `KeyImage`, `RangeProofs`, `BalanceProof`
- Reject reused `KeyImage`
- Mark input note as spent
- Store new output notes

---

### ConfidentialBurn Transaction Format

| Field         | Type | Required | Description                              |
|---------------|------|----------|------------------------------------------|
| Commitment    | Blob | Yes      | Note commitment to burn                  |
| EqualityProof | Blob | Yes      | ZK proof: commitment encodes amount      |

**Validator Actions:**
- Verify `EqualityProof`
- Mark note as spent
- Credit XRP to public balance

---

## Security Considerations

- ZKPs must be sound
- EC-ElGamal ciphertexts must be randomized
- Key images must be collision-resistant
- Auditing metadata must not leak confidential values

---

## References

[1] Bulletproofs: Short Proofs for Confidential Transactions  
[2] EC-ElGamal Encryption  
[3] Schnorr-based Ring Signatures  
[4] XRPLF XLS-9d: Blinded Tags
