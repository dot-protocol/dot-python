# USER_MANUAL — dot-python

Python implementation of the DOT wire format: a 151-byte cryptographically signed, hash-linked observation. Ed25519 signing, BLAKE3 chain hashing, AES-256-GCM encryption. Includes a CLI (`dot seal / open / genesis`), a Python library, and an MCP server (TypeScript) for AI agent integration.

---

## What It Is

A DOT is the smallest unit of provable observation:
- **151 bytes minimum** (1-byte payload + fixed overhead)
- **Ed25519 signature** — 32-byte public key IS the identity; no account, no email
- **BLAKE3 chain hash** — each DOT's `prev_hash` = BLAKE3 of the previous DOT; tamper-evident
- **Self-describing** — magic bytes `\x89DOT` at offset 0; a programmer with no docs can parse it from the hex

The format is stable. Version `1.0.0`. Library on PyPI as `dot-protocol`.

---

## Install

```bash
pip install dot-protocol
```

Or from source:

```bash
git clone https://github.com/dot-protocol/dot-python.git
cd dot-python
pip install pynacl blake3
```

Dependencies: `pynacl>=1.5.0`, `blake3>=0.3.4`. Python 3.9+.

---

## CLI

The `dot` command is installed with the package.

```bash
# Seal your first observation (creates observation.dot + keypair on first run)
dot seal "The architecture is complete."
# Output:
#   Generated new keypair → default_key.raw
#   Sealed → observation.dot
#   Size: 196 bytes
#   BLAKE3: 4a3be96f0c5b116965872027eae02732...

# Inspect and verify any .dot file
dot open observation.dot
# Output:
#   Status:       VALID
#   Type:         OBSERVATION (0x02)
#   Timestamp:    1773171415040000 µs (2026-03-10T17:30:15.040000+00:00)
#   Observer:     2a993b4c4ce7146699c2e769994f002c...
#   Prev hash:    000000000000000000000000...  (genesis)
#   Payload (28B): 'The architecture is complete.'
#   BLAKE3 hash:  4a3be96f0c5b116965872027eae02732...
#   Total size:   196 bytes

# Create a genesis DOT (start of a new chain)
dot genesis
# Output:
#   ✓ genesis.dot created
#   Size: 196 bytes
#   Public key: ...
#   Signature valid: True
```

`dot open` exits with code 0 if valid, 1 if invalid/tampered. Wire it into CI:

```bash
dot open some.dot && echo "chain intact"
```

---

## Python API

### dot_core — serialize / deserialize

```python
from dot_core import create_dot, parse_dot, verify_dot, TYPE_OBSERVATION, TYPE_IDENTITY
from dot_crypto import generate_keypair

# Generate a keypair (Ed25519)
signing_key, verify_key = generate_keypair()

# Create an observation DOT
raw = create_dot("I saw something.", signing_key, verify_key)
# raw: bytes, len >= 151

# Verify signature
assert verify_dot(raw)  # True

# Parse all fields
d = parse_dot(raw)
print(d['payload'].decode())       # 'I saw something.'
print(d['timestamp_us'])           # microseconds since epoch
print(d['observer'].hex())         # 32-byte public key as hex
print(d['prev_hash'].hex())        # all zeros = genesis
print(d['total_size'])             # 196 (for 16-byte payload)
print(d['dot_type_name'])          # 'OBSERVATION'
```

`parse_dot()` return dict keys:

| Key | Type | Description |
|---|---|---|
| `magic` | bytes | `\x89DOT` |
| `version` | int | Format version (1) |
| `crypto_suite` | int | 1 = Ed25519+X25519+AES |
| `flags` | int | Bit 0: encrypted, bit 1: has-chain |
| `dot_type` | int | 0x01–0x04 |
| `dot_type_name` | str | `IDENTITY`, `OBSERVATION`, `ENCRYPTED`, or `CHAIN` |
| `tlv_count` | int | Number of TLV extensions |
| `tlvs` | list[dict] | Each: `{tag, length, value}` |
| `timestamp_us` | int | Microseconds since Unix epoch |
| `observer` | bytes | 32-byte Ed25519 public key |
| `prev_hash` | bytes | 32-byte BLAKE3 of previous DOT (zeros = genesis) |
| `payload_len` | int | Payload size in bytes |
| `payload` | bytes | The observation content |
| `signature` | bytes | 64-byte Ed25519 signature |
| `signable` | bytes | Header + TLVs + body (what was signed) |
| `raw` | bytes | Complete DOT bytes |
| `total_size` | int | Total bytes |

### dot_chain — build and verify chains

```python
from dot_chain import hash_dot, create_chain_dot, verify_chain
from dot_crypto import generate_keypair
from dot_core import create_dot, TYPE_IDENTITY

signing_key, verify_key = generate_keypair()

# Genesis DOT (prev_hash = 32 zero bytes)
dot1 = create_dot("First observation.", signing_key, verify_key, dot_type=TYPE_IDENTITY)

# Chain subsequent DOTs
dot2 = create_chain_dot("Second observation.", signing_key, verify_key, prev_dot=dot1)
dot3 = create_chain_dot("Third observation.", signing_key, verify_key, prev_dot=dot2)

# Verify the chain integrity (checks BLAKE3 links, NOT Ed25519 — call verify_dot per DOT for that)
assert verify_chain([dot1, dot2, dot3])  # True

# Tamper check
tampered = bytearray(dot2); tampered[20] ^= 0xFF
assert not verify_chain([dot1, bytes(tampered), dot3])  # False

# Hash a DOT (used as next DOT's prev_hash)
h = hash_dot(dot1)  # 32-byte BLAKE3 digest
```

### dot_crypto — key management

```python
from dot_crypto import generate_keypair, sign, verify, pubkey_bytes
from dot_crypto import signing_key_from_bytes, verify_key_from_bytes

# Generate
sk, vk = generate_keypair()

# Serialize (persist to disk)
sk_hex = bytes(sk).hex()   # 64 hex chars
vk_hex = pubkey_bytes(vk).hex()  # 64 hex chars

# Deserialize
sk2 = signing_key_from_bytes(bytes.fromhex(sk_hex))
vk2 = verify_key_from_bytes(bytes.fromhex(vk_hex))

# Raw sign/verify (used internally by create_dot / verify_dot)
sig = sign(b"any bytes", sk)   # 64-byte Ed25519 sig
ok = verify(b"any bytes", sig, vk)  # True
```

---

## Wire Format Quick Reference

```
HEADER      10 bytes  Fixed
  magic       4B   \x89DOT
  version     1B   0x01
  crypto_suite 1B  0x01 (Ed25519+X25519+AES-256-GCM)
  flags       2B   big-endian u16 (bit0=encrypted, bit1=has-chain)
  type        1B   0x01=IDENTITY, 0x02=OBSERVATION, 0x03=ENCRYPTED, 0x04=CHAIN
  tlv_count   1B   number of TLV extensions

TLV         variable  (0 bytes for minimal DOT; each: 2B tag + 2B len + NB value)

BODY        variable  (>=76 bytes)
  timestamp   8B   uint64 BE microseconds since epoch
  observer   32B   Ed25519 public key
  prev_hash  32B   BLAKE3 of previous DOT (0x00×32 = genesis)
  payload_len 4B   uint32 BE
  payload     NB   arbitrary bytes (UTF-8 text by convention)

SIGNATURE  64 bytes  Ed25519 over (header + TLVs + body)
```

Minimum valid DOT: **151 bytes** (1-byte payload, no TLVs).

---

## MCP Server (AI Agent Integration)

The `mcp/` directory contains a TypeScript MCP server exposing five tools to Claude Code and other MCP clients.

```json
// Add to ~/.mcp.json
{
  "mcpServers": {
    "open-axxis": {
      "command": "node",
      "args": ["/path/to/dot-python/mcp/dist/index.js"],
      "type": "stdio"
    }
  }
}
```

| Tool | Description | Key inputs | Output |
|---|---|---|---|
| `axxis.seal` | Create a signed DOT offline | `payload`, `keypair` | raw DOT bytes + BLAKE3 hash |
| `axxis.verify` | Verify a DOT's signature | DOT bytes or hash | `valid: bool` + parsed fields |
| `axxis.observe` | Seal + submit to public chain | `payload`, `keypair` | DOT hash + Nostr event ID |
| `axxis.room` | Read the live chain | `limit?` | array of DOT dicts |
| `axxis.search` | Semantic search across chain | `query`, `limit?` | ranked DOT array with scores |

Semantic search uses `Xenova/all-MiniLM-L6-v2` (384-dim, 25 MB, runs locally). Cache at `~/.axxis/embed-cache.json`.

---

## Observe State

The library is stateless. There is no daemon, no server. State is in `.dot` files on disk.

```bash
# Is the genesis DOT present and valid?
dot open genesis.dot
# → Status: VALID (exit 0) or INVALID (exit 1)

# Inspect a chain of DOTs
for f in *.dot; do dot open "$f" | grep "Status:"; done

# Check library import
python -c "from dot_core import create_dot, parse_dot; print('OK')"

# Verify keypair files exist (created by 'dot seal' or 'dot genesis')
ls -la default_key.raw genesis_key.raw genesis_key.pub
```

---

## Experiments

```bash
pip install pynacl blake3
python experiments/exp1_genesis.py    # Create and verify genesis
python experiments/exp2_integrity.py  # Tamper detection
python experiments/exp3_minimum.py    # Minimum 151-byte DOT
python experiments/exp4_cold_open.py  # Parse the included genesis.dot cold
```

Results summary: `experiments/RESULTS.md`.

---

## Troubleshooting

**`ImportError: No module named 'nacl'`**
→ `pip install pynacl`

**`ImportError: No module named 'blake3'`**
→ `pip install blake3`

**`ValueError: Invalid magic`**
→ The file is not a DOT. Magic bytes at offset 0 must be `\x89DOT` (hex `89 44 4f 54`). Check with: `xxd yourfile.dot | head -1`

**`ValueError: DOT too short: N bytes`**
→ Minimum valid DOT is 151 bytes. Truncated file or wrong file.

**`verify_dot()` returns False**
→ The signature is invalid. Either the file was tampered with after sealing, or it was parsed with the wrong observer key. The observer key embedded in the DOT must match the signing key used.

**`verify_chain()` returns False**
→ Chain link broken: `parse_dot(dots[i]).prev_hash != BLAKE3(dots[i-1])`. A DOT was modified or the list is out of order. Note: `verify_chain()` only checks hash links; call `verify_dot()` per DOT for Ed25519 validity.

**MCP server not found**
→ Build it first: `cd mcp && npm install && npm run build`. Then point `args` at `mcp/dist/index.js`.

---

## Files

| File | What |
|---|---|
| `dot_core.py` | Wire format serialization/deserialization (`create_dot`, `parse_dot`, `verify_dot`) |
| `dot_crypto.py` | Ed25519 key gen, sign, verify, key import/export |
| `dot_chain.py` | BLAKE3 hashing, chain construction, chain verification |
| `dot_cli.py` | CLI entry point (`seal`, `open`, `genesis` commands) |
| `pyproject.toml` | Package metadata, entry point (`dot` command), dependencies |
| `SPEC.md` | Full wire format specification |
| `ARCHITECTURE.md` | Three-layer design (Core / Protocol APIs / Applications) |
| `QUICKSTART.md` | 30-second getting started guide |
| `genesis.dot` | The first DOT in the public chain (binary) |
| `genesis.dot.hex` | Annotated hex dump of genesis.dot |
| `genesis_key.pub` | Public key of the genesis observer |
| `cold_open.hex` | Hex of genesis.dot for cold-parse experiments |
| `cold_open_annotated.txt` | Field-by-field annotation of genesis.dot |
| `experiments/` | Four standalone validation experiments |
| `mcp/` | TypeScript MCP server (`axxis.*` tools) |

## License

Public Domain — CC0 1.0. `pip install dot-protocol`.

