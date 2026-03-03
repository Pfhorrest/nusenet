# neusnet Specification Layer 3: Identity

*Working draft — design phase*

This document specifies the neusnet identity model: how users are identified, how their signatures are verified, and how identity persists, rotates, and optionally presents itself to other users. It is a companion to [ratings.md](ratings.md), [metadata.md](metadata.md), and [README.md](README.md).

---

## 1. Design Principles

Identity in neusnet is built on three principles:

**Minimum viable identity is a keypair.** A user who generates an Ed25519 keypair and never publishes anything else has a valid neusnet identity. No registration, no username database, no central authority. The public key is the identifier; the private key is the credential.

**Richer identity is optional and user-controlled.** Users may publish a hosted identity document containing a display name, avatar, profile text, and links to other accounts. This is a convenience layer on top of the keypair, not a requirement.

**Multiple identities are supported.** A user may maintain any number of independent keypairs — one for professional discourse, one for personal use, one for a specific community. Whether and how to link these identities is entirely at the user's discretion. The protocol provides a voluntary linking mechanism but does not require or encourage it.

---

## 2. User Identifiers

A neusnet user identifier is the **Base58-encoded Ed25519 public key** of the user's keypair, prefixed with `nid1` to namespace it from other Base58-encoded identifiers:

```
nid1<Base58-encoded 32-byte Ed25519 public key>
```

Example: `nid1F3sAqQpLzFtKmVbRwXcNyHjDgEoIuPe...`

This identifier:
- Appears in the `rater` field of rating records
- Appears in the `author` field of metadata files
- Is the stable handle by which other users recognize and rate a person
- Is derived entirely from the public key — no registration or allocation required

### 2.1 Alternative Identity Substrates

Users who prefer not to manage raw keypairs may use identifiers from other systems as their neusnet user identifier, at the cost of some degree of centralization:

| Substrate | Identifier form | Notes |
|-----------|----------------|-------|
| AT Protocol | `at://did:plc:...` or `at://did:web:...` | Decentralized; recommended alternative |
| Nostr | `npub1...` (secp256k1) | Large existing ecosystem; different curve from native neusnet |
| W3C DID | `did:key:...`, `did:web:...`, etc. | Most interoperable standard |
| OAuth/SSO | `oauth:<provider>:<user-id>` | Centralized; acceptable for early adoption, not recommended long-term |

When an alternative substrate identifier is used, the signature scheme associated with that substrate applies (see Section 4). Rating records and metadata files from such users are valid neusnet objects; they simply use a different identifier format and signing method.

Clients should display a visual indicator of which identity substrate a user is using, so other users can understand the trust and decentralization properties of that identity.

---

## 3. Keypair Generation and Storage

### 3.1 Key Generation

A neusnet keypair is a standard Ed25519 keypair:
- **Private key**: 32 bytes of cryptographically secure random data
- **Public key**: the corresponding 32-byte Ed25519 public key, derived deterministically from the private key

Key generation must use a cryptographically secure random number generator. Client software must not generate keys from weak entropy sources.

### 3.2 Private Key Storage

The private key must be stored securely. Client software should:
- Encrypt the private key at rest using a user-supplied passphrase (recommended: Argon2id key derivation, AES-256-GCM encryption)
- Never transmit the private key over any network connection
- Warn users clearly that loss of the private key without a registered recovery key means permanent loss of that identity

### 3.3 Recovery Key

At identity creation, users are strongly encouraged to generate and register a **recovery keypair** — a second Ed25519 keypair stored separately from the primary key (e.g. on paper, a hardware token, or a separate device).

The recovery key is registered by including its public key in the user's initial identity document (Section 5) or in a standalone recovery registration record (Section 6.2). Once registered, the recovery key can:
- Sign a key rotation declaration (Section 6.1) to migrate the identity to a new primary key if the original is lost or compromised
- Itself be rotated to a new recovery key by the primary key

Loss of both the primary key and the recovery key is permanent loss of that identity. The protocol provides no further recovery mechanism — decentralized identity has no password reset.

---

## 4. Signatures

### 4.1 Signature Algorithm

The native neusnet signature algorithm is **Ed25519** as specified in RFC 8032.

For users on alternative substrates, the signature algorithm associated with that substrate applies:
- AT Protocol DIDs: typically Ed25519 or secp256k1 depending on DID method
- Nostr: secp256k1 (Schnorr signatures per NIP-01)
- W3C DID: as specified by the DID method

### 4.2 Canonicalization

Before signing, the object to be signed (a rating record or metadata file) is serialized to its canonical form using **JCS — JSON Canonicalization Scheme (RFC 8785)**:
- Object keys sorted lexicographically at all levels
- No insignificant whitespace
- Unicode strings in NFC normalization
- Numbers in their canonical JSON representation

The `signature` field is excluded from the object before canonicalization and signing. All other fields are included.

### 4.3 Signature Encoding

The 64-byte Ed25519 signature is encoded as a **Base64url** string (RFC 4648 §5, no padding) and stored in the `signature` field.

### 4.4 Verification

To verify a signature on a rating record or metadata file:
1. Extract and set aside the `signature` field.
2. Serialize the remaining fields to JCS canonical form.
3. Resolve the signer's public key from the `rater` or `author` field (or the `signature`'s associated key for third-party attestations).
4. Verify the Ed25519 signature over the canonical serialization.

A record whose signature does not verify must be treated as invalid and discarded. Clients must not display, propagate, or compute with invalid records.

---

## 5. Identity Documents

An identity document is an optional hosted file that provides human-readable and machine-readable information about a user. It is not required for participation in the neusnet network but is recommended for any user who wants to be recognizable to others.

### 5.1 Hosting

Identity documents are hosted using the same mechanisms as post metadata (see hosting.md), with the user's IPNS name (or equivalent stable identifier on other substrates) serving as the stable address. The identity document at that address may be updated over time; each version is a distinct content-addressed object.

### 5.2 Schema

An identity document is a signed JSON object:

```json
{
  "neusnet_version": 1,
  "type":            "identity",
  "id":              "<user identifier>",
  "display_name":    "<string>",
  "bio":             "<string>",
  "avatar":          "<content reference>",
  "links":           [ <link>, ... ],
  "recovery_keys":   ["<nid1... encoded public key>", "..."],
  "recovery_threshold": <integer>,
  "linked_ids":      [ <identity link>, ... ],
  "timestamp":       <unix timestamp>,
  "signature":       "<signature over all other fields>"
}
```

**`neusnet_version`** — integer. Must be `1`.

**`type`** — string. Must be `"identity"`. Distinguishes identity documents from post metadata files.

**`id`** — string. The user identifier this document describes. Must match the key used to produce the `signature`.

**`display_name`** — string. Optional. A human-readable name the user wishes to be known by. Not unique; not verified. Multiple users may use the same display name.

**`bio`** — string. Optional. A short free-text description of the user.

**`avatar`** — content reference object. Optional. A reference to an image file, using the same content reference format as metadata.md §4 (with `uri`, `mime_type`, `size`, and `hash` fields). Should be a square image; recommended formats are JPEG and PNG.

**`links`** — array. Optional. A list of links to the user's presence elsewhere on the web. Each link is an object with `url` (string, required) and `label` (string, optional) fields.

**`recovery_key`** — string. Optional but strongly recommended. The `nid1`-encoded public key of the user's registered recovery keypair. Publishing this in the identity document establishes a verifiable record of the recovery key before it is needed.

**`linked_ids`** — array. Optional. A list of other neusnet identities the user voluntarily declares as also belonging to them. See Section 7.

**`timestamp`** — integer. Unix timestamp of when this version of the identity document was published.

**`signature`** — string. Cryptographic signature over all other fields in JCS canonical form, using the applicable signing algorithm for the user's identity substrate (see Section 4). For native neusnet identities this is Ed25519; for alternative substrates the substrate's own signing algorithm applies.

### 5.3 Identity Document Discovery

Given a native neusnet user identifier (`nid1...`), the address of that user's identity document is derived deterministically from their public key using the same keypair that generates the identifier. On IPFS/IPNS deployments, the IPNS name used to host the identity document is the IPNS name derived from the user's primary keypair — meaning any client that knows a user's public key can compute where to look for their identity document without any additional lookup.

For alternative substrate identifiers, the discovery mechanism follows the conventions of that substrate:
- AT Protocol DIDs: the identity document is discoverable via the AT Protocol DID resolution mechanism
- Nostr: the identity document may be published as a kind-0 metadata event on Nostr relays, or hosted at the IPNS name derived from the corresponding keypair
- W3C DIDs: resolved via the DID resolution mechanism specified for that DID method

Clients encountering a user identifier for the first time should attempt identity document discovery lazily — it is not required before displaying or computing with rating records or metadata files that reference that identifier.

---

## 6. Key Rotation and Recovery

### 6.1 Key Rotation Declaration

A key rotation declaration is a signed record that supersedes one key with another. It enables a user to migrate their identity to a new primary keypair — for example, if their current key is compromised, or if they wish to upgrade their key for any reason.

```json
{
  "neusnet_version": 1,
  "type":            "key_rotation",
  "old_id":          "<user identifier being retired>",
  "new_id":          "<user identifier taking over>",
  "timestamp":       <unix timestamp>,
  "old_signature":   "<signature by old key over all other fields>",
  "new_signature":   "<signature by new key over all other fields>"
}
```

A valid rotation declaration must be signed by **both** the old key and the new key, proving that whoever controls the new key also controlled the old key at the time of rotation (or is in possession of the registered recovery key — see Section 6.2).

**`old_signature`** — signature by the old private key over all fields except both signature fields.
**`new_signature`** — signature by the new private key over all fields except both signature fields.

Upon encountering a valid rotation declaration, clients should:
- Treat all future content signed by the new key as belonging to the same identity as content previously signed by the old key.
- Aggregate ratings and affinity across both keys' content for the purposes of trust graph computation.
- Display a visual indicator that a rotation has occurred, with the timestamp, so users are aware the identity has a history under a different key.

A rotation declaration is itself content-addressed and should be propagated alongside the user's identity document.

### 6.2 Recovery Key Rotation

If the primary key is lost but a recovery key was registered (in the identity document or a prior rotation declaration), the recovery key may sign a rotation declaration in place of the old key:

- `old_signature` is signed by the **recovery key** rather than the old primary key
- The rotation declaration must reference the recovery key's public key in an additional `recovery_key` field, so clients can verify the signature against the registered recovery key rather than the old primary
- `new_signature` is signed by the new primary key as normal

```json
{
  "neusnet_version": 1,
  "type":            "key_rotation",
  "old_id":          "<user identifier being retired>",
  "new_id":          "<user identifier taking over>",
  "recovery_key":    "<nid1... encoded recovery public key>",
  "timestamp":       <unix timestamp>,
  "old_signature":   "<signature by recovery key over all other fields except signatures>",
  "new_signature":   "<signature by new primary key over all other fields except signatures>"
}
```

Clients verify the `old_signature` against the `recovery_key` field, and verify that the `recovery_key` matches the one registered in the user's most recent identity document or prior rotation declaration. If both checks pass, the rotation is valid.

---

## 7. Voluntary Identity Linking

A user who maintains multiple neusnet identities may voluntarily declare them as linked by including an `identity_link` record in each identity's `linked_ids` array, or by publishing a standalone linking record.

An identity link is:

```json
{
  "linked_id":   "<user identifier of the other identity>",
  "timestamp":   <unix timestamp>,
  "signature":   "<signature by the linking identity's key>"
}
```

For a link to be considered verified, **both identities must have declared the link** — identity A must include a link to B, and B must include a link to A, each signed by their respective keys. A unilateral declaration (A claims to be linked to B but B has no corresponding declaration) is shown as unverified.

Linking is entirely voluntary. Users may maintain completely separate identities with no links between them, and the protocol has no mechanism to detect or reveal unlinked identities belonging to the same person.

### 7.1 Implications for the Trust Graph

Linked identities are treated as distinct nodes in the trust graph by default. Clients may optionally offer users the ability to merge linked identities' affinity scores, but this should be an explicit user choice rather than automatic behavior — the user whose identities are linked may not want their professional and personal reputations to influence each other's trust graph position.

---

## 8. Open Questions

**8.1 Identifier encoding details.** The `nid1` prefix and Base58 encoding are specified here as the recommended native format. The exact Base58 alphabet (Bitcoin's, Flickr's, or another) should be explicitly specified to ensure interoperability. Bitcoin's alphabet (`123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`) is recommended as the most widely implemented.

**8.2 Rotation declaration propagation.** A rotation declaration needs to reach peers who hold affinity data for the old key so they can migrate it to the new key. There is no specified mechanism yet for ensuring rotation declarations propagate reliably through the network.

**8.3 Display name uniqueness and squatting.** Display names are explicitly non-unique. This is correct for a decentralized system but creates opportunities for impersonation. Client software should display the user identifier alongside or below the display name in contexts where impersonation risk is meaningful, and should never display display names alone as the primary identifier.

**8.4 Recovery key compromise.** If both primary and recovery keys are compromised simultaneously, an attacker could issue a fraudulent rotation declaration. This is an inherent limitation of two-key schemes. Hardware security keys and threshold schemes (requiring M of N keyholders) are potential mitigations but are out of scope for this version of the spec.