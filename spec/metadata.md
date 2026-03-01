# neusnet Specification Layer 2: Content Metadata

*Working draft — design phase*

This document specifies the neusnet content metadata format — the atomic unit of a neusnet post. It is a companion to [ratings.md](ratings.md) (Layer 1: Ratings) and [README.md](README.md) (overview and motivation).

The examples in this document use IPFS/IPNS terminology (CIDs, IPNS names) as the reference implementation. All concepts generalize to other hosting substrates; see Section 2 for the abstract model and Section 8 for notes on non-IPFS deployments.

---

## 1. Purpose and Role

Every neusnet post is represented by a small **metadata file**: a signed JSON document that identifies the post, its author, its place in the conversation graph, and where its content can be retrieved. The metadata file is designed to be lightweight enough that clients can retrieve and index many of them cheaply, deferring retrieval of actual content payloads until the user chooses to read a post.

The metadata file is the object that gets pinned, gossiped, and indexed. Content payloads — text, images, audio, video, and other media — live separately, referenced by the metadata file but not embedded in it (except for short inline text, as described in Section 4).

---

## 2. Identifiers: Stable ID and Version Identifier

Every neusnet post has two distinct identifiers serving different purposes.

**The stable identifier** (`id`) is an author-controlled address that persists across all edits. It is the anchor of a post's identity over its lifetime. The stable identifier:
- Is declared once at first publication and never changes across versions
- Is what clients navigate to in order to find the current version of a post
- Is the anchor from which canonical version history is constructed (see Section 5)

**The version identifier** is the content-addressed identifier of a specific metadata file — for example, an IPFS CID on IPFS-based deployments, or an equivalent immutable identifier on other substrates. The version identifier:
- Changes with every edit, uniquely identifying each revision
- Is what rating records reference as the `item` field, permanently recording which exact version of a post was rated
- Is what `parent` fields in reply posts reference, recording which exact version was being replied to
- Is what `previous` fields reference to form the version chain

Clients aggregate ratings across all versions of a post by building the canonical version history from the stable `id` (see Section 5.3) and summing ratings across all versions in that history.

### 2.1 Stable Identifier Forms

The stable identifier may take different forms depending on the hosting substrate:

| Substrate | Stable identifier form |
|-----------|----------------------|
| IPFS/IPNS | IPNS name (e.g. `ipns://k51qzi5...`) |
| AT Protocol | AT URI (e.g. `at://did:plc:abc.../app.bsky.feed.post/xyz`) |
| Nostr | Replaceable event identifier |
| HTTP | A canonical URL the author demonstrably controls |
| Other | Any stable, author-controlled address |

The stable identifier must be genuinely author-controlled: the author must be able to update what it points to, and no other party should be able to do so. This property is what makes the canonical version history trustworthy (see Section 5.2).

---

## 3. Metadata File Schema

A metadata file is a JSON object. Fields marked **required** must be present in every valid metadata file. Fields marked **optional** may be omitted.

```json
{
  "neusnet_version": 1,
  "id":             "<stable identifier>",
  "author":         "<user identifier>",
  "subject":        "<string>",
  "summary":        "<string>",
  "tags":           ["<normalized tag>", "..."],
  "content":        [ <content-reference>, "..." ],
  "parent":         "<version identifier of parent post>",
  "previous":       "<version identifier of previous version of this post>",
  "timestamp":      <unix timestamp, integer seconds>,
  "signature":      "<cryptographic signature>"
}
```

### Required Fields

**`neusnet_version`** — integer. The version of the neusnet metadata format this file conforms to. Must be `1` for files conforming to this specification. Clients encountering an unrecognized version number should treat the file as opaque and not attempt to interpret it.

**`id`** — string. The stable identifier for this post (see Section 2). Must be present in every version of the post and must not change between versions.

**`author`** — string. The user identifier of the post's author (see identity.md). Must match the key used to produce the `signature`, if a signature is present. May be omitted only for unsigned bridged posts (see Section 6).

**`tags`** — array of strings. Zero or more tags, each normalized per ratings.md §6. An empty array is valid. Order is not significant.

**`content`** — array of one or more content reference objects (see Section 4). Must contain at least one entry.

**`timestamp`** — integer. Unix timestamp (seconds since epoch) of when this version was published.

### Optional Fields

**`subject`** — string. A short human-readable title. Analogous to an HTML document's `<title>`. Recommended for top-level posts and link aggregation posts; may be omitted for short replies where the content speaks for itself.

**`summary`** — string. A short human-readable excerpt or description for feed previews. Analogous to an HTML `<meta name="description">`. Recommended when content is a long document or an external URL. May be omitted when inline content is present and short enough to display directly.

At least one of `subject`, `summary`, or short inline content should be present so clients have something to display in feed views without fetching the full payload. This is a client recommendation, not a protocol requirement.

**`parent`** — string. The version identifier of the metadata file of the post this is a reply to. Omitted for top-level posts. References the specific version being replied to, not the stable identifier — permanently recording the exact content the reply was responding to.

**`previous`** — string. The version identifier of the immediately preceding version of this post's own metadata file. Omitted in the first published version. Forms a backwards-linked chain that clients can traverse to reconstruct version history. On hosting substrates with native versioning (e.g. IPFS/IPNS), this field is redundant but included for clients that do not query native version history. On substrates without native versioning, this field is the primary mechanism for version chain traversal.

**`signature`** — string. Cryptographic signature over all other fields in JCS canonical form (RFC 8785: keys sorted, no insignificant whitespace), using the author's private key. Unsigned posts are permitted in limited circumstances (see Section 6); unsigned posts must omit this field entirely rather than including an empty or null value.

---

## 4. Content References

The `content` field is an array of content reference objects. Multiple references are encouraged for redundancy — a client retrieves content from whichever reference it can reach first. All references in the array must point to semantically identical content; they are mirrors, not alternatives.

Each content reference object:

```json
{
  "uri":            "<content URI>",
  "mime_type":      "<MIME type string>",
  "size":           <size in bytes>,
  "inline_content": "<string>"
}
```

**`uri`** — string. Required. The location or identifier of the content. See Section 4.1 for URI types and their mutability properties.

**`mime_type`** — string. Optional but strongly recommended. A standard MIME type (e.g. `text/plain`, `text/html`, `image/jpeg`, `video/mp4`, `audio/ogg`). Allows clients to make display and prefetch decisions before fetching content.

**`size`** — integer. Optional. Size of the content payload in bytes. Allows clients to make informed prefetch decisions, particularly for large media files. Should reflect the size of the raw content, not any transport encoding.

**`inline_content`** — string. Present only when `uri` is `inline:` (see Section 4.2).

### 4.1 URI Types and Mutability

Content URIs fall into two categories with importantly different trust properties:

**Immutable URIs** — the content at this URI is permanently fixed and cannot be changed after publication. The URI is typically a content-addressed hash of the content itself.

| URI form | Example |
|----------|---------|
| IPFS CID | `ipfs://bafybeig...` |
| Inline marker | `inline:` |
| BitTorrent magnet | `magnet:?xt=urn:btih:...` |

**Mutable URIs** — the content at this URI may change after publication. The author (or a server operator) may alter what is served at this address without any record of the change in the neusnet metadata file.

| URI form | Example | Notes |
|----------|---------|-------|
| IPNS name | `ipns://k51qzi5...` | Mutable IPFS pointer |
| HTTPS URL | `https://example.com/post` | Server-controlled |
| Other HTTP | `http://...` | Server-controlled |

This list is not exhaustive; other URI schemes may be used as neusnet deployments evolve.

Clients should warn users when a content reference uses a mutable URI, as the content may have changed since the post was published and since any ratings were given. Immutable URIs are strongly preferred for content payloads. (Note that mutable URIs are appropriate and expected for the post's `id` field — that is their purpose — but content payload URIs should be immutable wherever possible.)

### 4.2 Inline Content

For short text posts, content may be embedded directly in the metadata file by setting `uri` to `inline:` and including an `inline_content` field:

```json
{
  "uri":            "inline:",
  "mime_type":      "text/plain",
  "inline_content": "The actual text of the post goes here."
}
```

Inline content eliminates a separate content fetch for short posts. There is no hard size limit in the protocol; clients may apply their own limits and may decline to display or propagate metadata files they consider excessively large.

---

## 5. Version History and Integrity

### 5.1 Publishing an Edit

When an author publishes a revised version of a post:

1. They produce a new metadata file with the same `id`, updated fields, a new `timestamp`, and a `previous` field pointing to the prior version's identifier.
2. They sign the new metadata file with their author key.
3. They publish it to their hosting substrate, obtaining a new version identifier.
4. They update their stable `id` pointer to the new version.
5. Their client automatically issues a self-rating of +1 for the new version (per ratings.md §2.4).

Prior versions remain accessible at their original version identifiers as long as any peer retains them. Clients should not discard cached prior versions merely because a newer version exists.

### 5.2 Canonical Version History

The canonical version history of a post is constructed as follows:

1. Resolve the stable `id` to obtain the current version's metadata file. Because the author controls the stable `id`, this is the author's authoritative current version.
2. Follow `previous` links backwards through the chain to collect all prior versions.
3. The resulting ordered list is the canonical version history.

Any metadata file that claims a given `id` but does not appear in its canonical version history is **anomalous**. Anomalous posts fall into two categories distinguishable by signature:

- **Memory-holed versions**: signed by the genuine author but excluded from the canonical chain. This occurs when an author intentionally skips a version in their `previous` chain to suppress it. The omitted version is still accessible via its version identifier to any peer who retained it.
- **False claimants**: not signed by the genuine author. These are posts by other parties falsely claiming authorship or association with the stable `id`.

Clients must handle anomalous posts as follows:
- Build the canonical version history as described above; use this as the primary version list.
- Collect anomalous posts separately (e.g. from peers' cached collections).
- **Memory-holed versions**: include their ratings in the aggregate score for the stable `id`. An author cannot erase community response to their own words.
- **False claimants**: exclude their ratings from the aggregate. Display them only if directly linked, with a clear "not part of canonical history" flag.
- When directly linked to a post, verify lazily that it appears in the canonical history of its claimed `id`. Flag posts that do not.

### 5.3 Aggregating Scores Across Versions

Ratings reference version identifiers, not stable identifiers. To compute the aggregate score for a post:

1. Build the canonical version history (Section 5.2).
2. Collect all rating records whose `item` field matches any version identifier in that history, plus any memory-holed versions (whose ratings count per Section 5.2).
3. Deduplicate by rater: for each rater, use only their most recent rating across all versions (by timestamp). This prevents score inflation from a rater re-rating multiple versions of the same post, and prevents authors from self-boosting by publishing many near-identical versions each attracting a self-rating.
4. Apply the trust graph algorithm (ratings.md §4–5) to the deduplicated rating set.

The default view shows the current version with the aggregate score. Clients should additionally:
- Provide access to per-version score breakdowns in the version history view.
- Visually flag posts where ratings are substantially concentrated on a version other than the current one, as this may indicate content was changed after attracting significant attention.

### 5.4 Disavowal

An author who wishes to disavow a post may publish a new version with a `subject` of `"[retracted]"`, an empty or minimal `content` array, and a self-rating of −1 for the new version. This does not delete prior versions — content-addressed storage is immutable — but records the author's explicit disavowal in the canonical version chain and signals to clients that the author no longer endorses the content.

---

## 6. Unsigned and Bridged Posts

### 6.1 Unsigned Posts

Posts may be published without a `signature` field. Unsigned posts are permitted to support bridging use cases where the neusnet metadata file is created by someone other than the original author (see Section 6.2), and to allow participation in the neusnet graph by authors who have not established a neusnet identity.

Clients must display unsigned posts with a clear visual indicator distinguishing them from author-signed posts. Three trust levels apply to any metadata file:

| Status | Condition | Display |
|--------|-----------|---------|
| **Author-verified** | Signed by the key matching the `author` field | Full trust indicators |
| **Third-party attested** | Signed by a key that does not match `author` | "Introduced by [introducer]" indicator |
| **Unverified** | No signature | Clearly flagged as unverified |

### 6.2 Bridged Posts

Posts originating on other platforms can participate in the neusnet graph by having a metadata file created for them, either by the original author or by any third party.

For bridged posts:
- The `id` field should be the platform's native stable identifier (e.g. an AT Protocol URI, a canonical URL).
- The `author` field should be the author's neusnet user identifier if known. If the author has no neusnet identity, the field may be set to a platform-scoped identifier (e.g. `at://did:plc:abc123`) as an attribution hint, with the understanding that this cannot be cryptographically verified against a neusnet signature.
- If the metadata file is created by a third party (not the author), it should be signed by the introducer's key, making it "third-party attested." The `author` field still reflects the original author; the `signature` reflects the introducer.
- The `content` array should include a URI pointing to the original content.

When a bridged post's original author later establishes a neusnet identity, they may publish their own author-signed metadata file for the same stable `id`. Clients should prefer author-signed metadata files over third-party-attested ones for the same `id`.

---

## 7. Open Questions

**7.1 Tag, subject, and summary length guidance.** No hard limits are specified for tag count, subject length, summary length, or inline content length. These are left as client implementation decisions — a client may truncate fields for display, or decline to propagate metadata files it considers excessively large. Future versions of this spec may add recommended (non-mandatory) guidance if the community converges on norms.

**7.2 Non-IPFS stable identifier binding.** For authors using AT Protocol DIDs, Nostr keypairs, or other identity systems as their stable identifier substrate, the precise binding between the neusnet `author` field and the stable `id` address space needs elaboration. The identity.md spec should address this.

**7.3 Third-party attestation scope.** The "third-party attested" status in Section 6.1 covers the case where an introducer signs a bridged post. It does not yet specify what claims the introducer's signature covers (e.g. "I retrieved this content at this URL at this time and it matched this hash") versus what it does not (authorship). A more precise attestation schema may be useful.

---

## Appendix A: Minimal Valid Metadata File

A top-level post with short inline text, no subject, no tags:

```json
{
  "neusnet_version": 1,
  "id":        "ipns://k51qzi5uqu5dh6lfh0....",
  "author":    "npub1abc123...",
  "tags":      [],
  "content":   [
    {
      "uri":            "inline:",
      "mime_type":      "text/plain",
      "inline_content": "Hello, neusnet."
    }
  ],
  "timestamp": 1740000000,
  "signature": "sig1abc..."
}
```

## Appendix B: Full-Featured Metadata File

A reply post with redundant content references, edit history, summary, and tags:

```json
{
  "neusnet_version": 1,
  "id":        "ipns://k51qzi5uqu5dh6lfh0....",
  "author":    "npub1abc123...",
  "subject":   "Re: The case for decentralized moderation",
  "summary":   "A counterargument focusing on the cold-start problem and Sybil resistance.",
  "tags":      ["decentralization", "moderation", "trust-graphs"],
  "content":   [
    {
      "uri":       "ipfs://bafybeig...",
      "mime_type": "text/html",
      "size":      142080
    },
    {
      "uri":       "https://example.com/posts/counterargument.html",
      "mime_type": "text/html"
    }
  ],
  "parent":    "<version identifier of parent post>",
  "previous":  "<version identifier of prior version of this post>",
  "timestamp": 1740003600,
  "signature": "sig1abc..."
}
```

## Appendix C: Unsigned Bridged Post

A Bluesky post introduced to neusnet by a third party:

```json
{
  "neusnet_version": 1,
  "id":        "at://did:plc:abc123.../app.bsky.feed.post/xyz",
  "author":    "at://did:plc:abc123...",
  "subject":   "Interesting thread on trust graphs",
  "tags":      ["trust-graphs", "bluesky"],
  "content":   [
    {
      "uri":       "https://bsky.app/profile/abc123.bsky.social/post/xyz",
      "mime_type": "text/html"
    }
  ],
  "timestamp": 1740007200,
  "signature": "sig_of_introducer_not_author..."
}
```