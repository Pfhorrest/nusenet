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

A neusnet user identifier is the **Base58-encoded Ed25519 public key** of the user's keypair, prefixed with `nid1` to namespace it from other Base58-encoded identifiers. The Base58 encoding uses Bitcoin's alphabet (`123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`):

```
nid1<Base58-encoded 32-byte Ed25519 public key>
```

Example: `nid1F3sAqQpLzFtKmVbRwXcNyHjDgEoIuPe...`

This identifier:
- Appears in the `rater` field of rating records
- Appears in the `author` field of metadata files
- Is the stable handle by which other users recognize and rate a person
- Is derived entirely from the public key — no registration or allocation required

Display names are not unique and must never be used as the primary identifier in any client UI. Clients should always display the user identifier alongside or below the display name in any context where impersonation risk is meaningful.

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
- Warn users clearly that loss of the private key without registered recovery keys means permanent loss of that identity

### 3.3 Recovery Keys

At identity creation, users are strongly encouraged to generate and register one or more **recovery keypairs** — Ed25519 keypairs stored separately from the primary key (e.g. on paper, a hardware token, or with trusted contacts).

Recovery keys are registered by including them in the user's initial identity document (Section 5). Once registered, a sufficient quorum of recovery keys (per the `recovery_threshold` field) can sign a rotation declaration (Section 6.2) to migrate the identity to a new primary key if the original is lost or compromised.

Loss of the primary key with no registered recovery keys, or without a sufficient quorum of recovery keys available, is permanent loss of that identity. The protocol provides no further recovery mechanism — decentralized identity has no password reset.

---

## 4. Signatures

### 4.1 Signature Algorithm

The native neusnet signature algorithm is **Ed25519** as specified in RFC 8032.

For users on alternative substrates, the signature algorithm associated with that substrate applies:
- AT Protocol DIDs: typically Ed25519 or secp256k1 depending on DID method
- Nostr: secp256k1 (Schnorr signatures per NIP-01)
- W3C DID: as specified by the DID method

### 4.2 Canonicalization

Before signing, the object to be signed (a rating record, metadata file, identity document, or rotation declaration) is serialized to its canonical form using **JCS — JSON Canonicalization Scheme (RFC 8785)**:
- Object keys sorted lexicographically at all levels
- No insignificant whitespace
- Unicode strings in NFC normalization
- Numbers in their canonical JSON representation

All signature fields are excluded from the object before canonicalization and signing. All other fields are included.

### 4.3 Signature Encoding

The 64-byte Ed25519 signature is encoded as a **Base64url** string (RFC 4648 §5, no padding) and stored in the relevant signature field. Signatures from alternative substrates use the encoding conventions of that substrate.

### 4.4 Verification

To verify a signature on a rating record or metadata file:
1. Extract and set aside all signature fields.
2. Serialize the remaining fields to JCS canonical form.
3. Resolve the signer's public key from the `rater` or `author` field (or the signer's key for third-party attestations).
4. Verify the signature over the canonical serialization using the applicable algorithm for the identity substrate.

A record whose signature does not verify must be treated as invalid and discarded. Clients must not display, propagate, or compute with invalid records.

---

## 5. Identity Documents

An identity document is an optional hosted file that provides human-readable and machine-readable information about a user. It is not required for participation in the neusnet network but is recommended for any user who wants to be recognizable to others.

### 5.1 Hosting

Identity documents are hosted using the same mechanisms as post metadata (see hosting.md), with the user's IPNS name (or equivalent stable identifier on other substrates) serving as the stable address. The identity document at that address may be updated over time; each version is a distinct content-addressed object.

Clients that access an identity document should also lazily check for any rotation declarations associated with that identity (see Section 6.3). Identity documents and rotation declarations from users with positive affinity should be included in a client's pinned and gossiped content set alongside post metadata, so that identity information propagates as a natural side effect of normal participation.

### 5.2 Schema

An identity document is a signed JSON object:

```json
{
  "neusnet_version":    1,
  "type":               "identity",
  "id":                 "<user identifier>",
  "display_name":       "<string>",
  "bio":                "<string>",
  "avatar":             <content reference>,
  "links":              [ <link>, ... ],
  "recovery_keys":      ["<nid1... encoded public key>", "..."],
  "recovery_threshold": <integer>,
  "linked_ids":         [ <identity link>, ... ],
  "aggregate_linked":   <boolean>,
  "timestamp":          <unix timestamp>,
  "signature":          "<signature over all other fields>"
}
```

**`neusnet_version`** — integer. Must be `1`.

**`type`** — string. Must be `"identity"`. Distinguishes identity documents from post metadata files and other neusnet JSON objects.

**`id`** — string. The user identifier this document describes. Must match the key used to produce the `signature`.

**`display_name`** — string. Optional. A human-readable name the user wishes to be known by. Not unique; not verified. Multiple users may use the same display name.

**`bio`** — string. Optional. A short free-text description of the user.

**`avatar`** — content reference object. Optional. A reference to an image file, using the same content reference format as metadata.md §4 (with `uri`, `mime_type`, `size`, and `hash` fields). Should be a square image; recommended formats are JPEG and PNG.

**`links`** — array. Optional. A list of links to the user's presence elsewhere on the web. Each link is an object with `url` (string, required) and `label` (string, optional) fields.

**`recovery_keys`** — array of strings. Optional but strongly recommended. One or more `nid1`-encoded public keys of the user's registered recovery keypairs. Publishing these establishes a verifiable record before they are needed. A single recovery key is the recommended default for most users.

**`recovery_threshold`** — integer. Optional. The minimum number of recovery key signatures required to authorize a recovery rotation (see Section 6.2). Defaults to `1` if absent. Users who want stronger recovery security may register multiple recovery keys and set a higher threshold — for example, `recovery_threshold: 2` with three registered keys requires any two to cooperate. Must not exceed the length of `recovery_keys`.

**`linked_ids`** — array. Optional. A list of other neusnet identities the user voluntarily declares as also belonging to them. See Section 7.

**`aggregate_linked`** — boolean. Optional. The user's preferred default for how other users' clients should treat their linked identities in trust graph computation. `true` means the user prefers their linked identities to be treated as a single node for affinity purposes; `false` (the default if absent) means they prefer them treated as distinct nodes. Other users' clients may override this preference per their own settings — see Section 7.1.

**`timestamp`** — integer. Unix timestamp of when this version of the identity document was published.

**`signature`** — string. Cryptographic signature over all other fields in JCS canonical form, using the applicable signing algorithm for the user's identity substrate (see Section 4). For native neusnet identities this is Ed25519; for alternative substrates the substrate's own signing algorithm applies.

### 5.3 Identity Document Discovery

Given a native neusnet user identifier (`nid1...`), the address of that user's identity document is derived deterministically from their public key. On IPFS/IPNS deployments, the IPNS name used to host the identity document is the IPNS name derived from the user's primary keypair — meaning any client that knows a user's public key can compute where to look for their identity document without any additional lookup.

For alternative substrate identifiers, the discovery mechanism follows the conventions of that substrate:
- AT Protocol DIDs: discoverable via the AT Protocol DID resolution mechanism
- Nostr: published as a kind-0 metadata event on Nostr relays, or hosted at the IPNS name derived from the corresponding keypair
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
  "old_signature":   "<signature by old key over all other fields except signature fields>",
  "new_signature":   "<signature by new key over all other fields except signature fields>"
}
```

A valid standard rotation declaration must be signed by **both** the old key and the new key, proving that whoever controls the new key also controlled the old key at the time of rotation.

Upon encountering a valid rotation declaration, clients should:
- Treat all future content signed by the new key as belonging to the same identity as content previously signed by the old key.
- Aggregate ratings and affinity across both keys' content for the purposes of trust graph computation.
- Display a visual indicator that a rotation has occurred, with the timestamp, so users are aware the identity has a history under a different key.

A rotation declaration is itself content-addressed and should be propagated alongside the user's identity document.

### 6.2 Recovery Key Rotation

If the primary key is lost but a sufficient quorum of recovery keys were registered (per `recovery_threshold`), those recovery keys may collectively authorize a rotation declaration in place of the old primary key.

```json
{
  "neusnet_version":     1,
  "type":                "key_rotation",
  "old_id":              "<user identifier being retired>",
  "new_id":              "<user identifier taking over>",
  "recovery_signatures": [
    {
      "recovery_key": "<nid1... encoded recovery public key>",
      "signature":    "<signature by this recovery key over all fields except all signature fields>"
    }
  ],
  "new_signature":       "<signature by new primary key over all fields except all signature fields>",
  "timestamp":           <unix timestamp>
}
```

Clients verify this declaration by:
1. Checking that each `recovery_key` in `recovery_signatures` appears in the user's most recent identity document or prior rotation declaration.
2. Verifying each recovery key's signature.
3. Confirming that the number of valid recovery signatures meets or exceeds the registered `recovery_threshold` (defaulting to 1 if unset).
4. Verifying `new_signature` against the new primary key.

If all checks pass, the rotation is valid and treated identically to a standard rotation.

### 6.3 Rotation Declaration Discovery

When a client accesses an identity document, it should lazily check for rotation declarations associated with that identity. On IPFS/IPNS deployments, rotation declarations may be linked directly from the identity document or published at a well-known sub-path of the user's IPNS address. Clients that discover a rotation declaration should propagate it to peers alongside the identity document.

The exact storage path convention for rotation declarations on IPFS/IPNS deployments is an open question (see Section 8.1).

---

## 7. Voluntary Identity Linking

A user who maintains multiple neusnet identities may voluntarily declare them as linked by including identity link records in each identity's `linked_ids` array.

An identity link record is:

```json
{
  "linked_id": "<user identifier of the other identity>",
  "timestamp": <unix timestamp>,
  "signature": "<signature by the linking identity's key>"
}
```

For a link to be considered verified, **both identities must have declared the link** — identity A must include a link to B, and B must include a link to A, each signed by their respective keys. A unilateral declaration is shown as unverified.

Linking is entirely voluntary. The protocol has no mechanism to detect or reveal unlinked identities belonging to the same person.

### 7.1 Aggregation Preferences for Linked Identities

Whether linked identities are treated as a single node or as distinct nodes in the trust graph is governed by a two-layer preference system:

**Layer 1 — The identity owner's preference**: The `aggregate_linked` field in the identity document expresses the owner's preferred default. `true` means they prefer others to treat their linked identities as one node for affinity purposes; `false` (the default if absent) means they prefer them treated as distinct. This preference applies to the cluster as a whole — if any identity in a mutually-verified cluster sets `aggregate_linked: true`, clients should treat that as the owner's preference for the cluster.

**Layer 2 — The viewer's client setting**: Each user's client may override the owner's preference categorically:
- *Respect owner preference* (default): aggregate or separate per each owner's `aggregate_linked` value
- *Always aggregate verified links*: treat all mutually-verified linked identities as one node regardless of owner preference
- *Always treat separately*: treat all identities as distinct nodes regardless of owner preference

This design ensures identity owners have meaningful control over their default presentation while preserving each user's ultimate sovereignty over their own trust graph computation.

---

## 8. Open Questions

**8.1 Rotation declaration storage path.** Section 6.3 recommends that rotation declarations be published at a well-known path relative to the user's IPNS address, but does not yet specify what that path is. A standard convention needs to be established.

**8.2 Recovery key compromise.** If a sufficient quorum of recovery keys is compromised simultaneously with the primary key, an attacker could issue a fraudulent rotation declaration. This risk scales with the number of parties holding recovery keys. Users should treat recovery key distribution as a security-critical decision; the protocol cannot protect against a fully compromised key set.