# Chapter 10: Native Encryption

> **Source:** `include/sys/dsl_crypt.h`, `include/sys/zio_crypt.h`, `include/sys/spa.h`, `include/sys/dmu.h`, `include/sys/dmu_objset.h`, `include/sys/dsl_dir.h`, `include/sys/fs/zfs.h`, `module/zfs/dsl_crypt.c`, `module/os/*/zfs/zio_crypt.c`, `module/zcommon/zfeature_common.c`

ZFS native encryption is a per-dataset feature (`feature@encryption`) that encrypts data blocks while preserving on-disk integrity and copy-on-write semantics. Encryption is implemented at the block pointer layer, with dataset keys stored in the MOS.

## 10.1 Feature Flag and Scope

- Feature flag: `com.datto:encryption` (`feature@encryption`)
- Dependencies: `feature@extensible_dataset`, `feature@bookmark_v2`
- Type: per-dataset (`ZFEATURE_FLAG_PER_DATASET`)

Encrypted datasets form **encryption roots**. Datasets that inherit encryption reference the same on-disk key object as their root. Clones share the same key object (refcounted).

## 10.2 DSL Crypto Key Object (On Disk)

Each encryption root has a DSL crypto key object stored in the MOS as a **ZAP** (`DMU_OTN_ZAP_METADATA`). The DSL directory holds its object ID in the extensible dsl_dir ZAP field:

- `com.datto:crypto_key_obj` (`DD_FIELD_CRYPTO_KEY_OBJ`)

### ZAP Entries (On-Disk Strings)

| ZAP Entry (string) | Macro Name | Size | Description |
|--------------------|------------|------|-------------|
| `DSL_CRYPTO_SUITE` | `DSL_CRYPTO_KEY_CRYPTO_SUITE` | `uint64` | Encryption algorithm (`zio_encrypt`) |
| `DSL_CRYPTO_GUID` | `DSL_CRYPTO_KEY_GUID` | `uint64` | Unique key GUID |
| `DSL_CRYPTO_IV` | `DSL_CRYPTO_KEY_IV` | 12 bytes | IV for wrapping key material (`WRAPPING_IV_LEN`) |
| `DSL_CRYPTO_MAC` | `DSL_CRYPTO_KEY_MAC` | 16 bytes | MAC over wrapped key data (`WRAPPING_MAC_LEN`) |
| `DSL_CRYPTO_MASTER_KEY_1` | `DSL_CRYPTO_KEY_MASTER_KEY` | 32 bytes | Wrapped master key (`MASTER_KEY_MAX_LEN`) |
| `DSL_CRYPTO_HMAC_KEY_1` | `DSL_CRYPTO_KEY_HMAC_KEY` | 64 bytes | Wrapped HMAC key (`SHA512_HMAC_KEYLEN`) |
| `DSL_CRYPTO_ROOT_DDOBJ` | `DSL_CRYPTO_KEY_ROOT_DDOBJ` | `uint64` | DSL dir object ID of the encryption root |
| `DSL_CRYPTO_REFCOUNT` | `DSL_CRYPTO_KEY_REFCOUNT` | `uint64` | Number of datasets sharing this key object |
| `DSL_CRYPTO_VERSION` | `DSL_CRYPTO_KEY_VERSION` | `uint64` | Key format version (`ZIO_CRYPT_KEY_CURRENT_VERSION`) |
| `keyformat` | `ZFS_PROP_KEYFORMAT` | `uint64` | `ZFS_PROP_KEYFORMAT` value |
| `pbkdf2salt` | `ZFS_PROP_PBKDF2_SALT` | `uint64` | `ZFS_PROP_PBKDF2_SALT` value |
| `pbkdf2iters` | `ZFS_PROP_PBKDF2_ITERS` | `uint64` | `ZFS_PROP_PBKDF2_ITERS` value |

The master and HMAC keys are stored **wrapped** (encrypted) with the dataset's wrapping key. The IV and MAC provide authenticated wrapping.

## 10.3 Keyformat and Wrapping Keys

The ZAP fields `keyformat`, `pbkdf2salt`, and `pbkdf2iters` preserve the parameters needed to reconstruct the wrapping key for passphrase-based datasets.

`zfs_keyformat_t` values stored on disk:

| Value | Meaning |
|-------|---------|
| `ZFS_KEYFORMAT_NONE` | No key format |
| `ZFS_KEYFORMAT_RAW` | Raw binary key |
| `ZFS_KEYFORMAT_HEX` | Hex-encoded key |
| `ZFS_KEYFORMAT_PASSPHRASE` | PBKDF2-derived key |

## 10.4 Key Change Commands (`dcp_cmd_t`)

Key changes and rewrap operations are driven by `dcp_cmd_t` and affect how the on-disk key object is updated:

| Command | Meaning | On-Disk Effect |
|---------|---------|----------------|
| `DCP_CMD_NONE` | No action | None |
| `DCP_CMD_NEW_KEY` | Rewrap key as a new encryption root | Re-encrypts the DSL crypto key with a new wrapping key and updates `DSL_CRYPTO_ROOT_DDOBJ` |
| `DCP_CMD_INHERIT` | Rewrap key with parent wrapping key | Updates wrapped key material to match parent |
| `DCP_CMD_FORCE_NEW_KEY` | Change to encryption root without rewrap | Updates root relationship without rewriting key material |
| `DCP_CMD_FORCE_INHERIT` | Inherit without rewrap | Updates root relationship without rewriting key material |
| `DCP_CMD_RAW_RECV` | Raw receive | Populates key object from stream data |

These commands determine whether the DSL crypto key object is rewritten (rewrapped) or only its relationships are updated.

## 10.5 Block Pointer Encryption Fields

Encrypted (or authenticated) blocks use the `X` bit in the block pointer and store encryption parameters directly in the blkptr. The encrypted blkptr layout repurposes fields:

- **Salt**: 8 bytes (`ZIO_DATA_SALT_LEN`), stored in the 3rd DVA payload
- **IV**: 12 bytes (`ZIO_DATA_IV_LEN`)
  - IV1 (first 64 bits) stored alongside the salt
  - IV2 (last 32 bits) stored in the upper bits of the fill count field
- **MAC**: 16 bytes (`ZIO_DATA_MAC_LEN`), stored in checksum words 2 and 3
- **Checksum**: truncated to 128 bits (checksum words 0 and 1)

Because the 3rd DVA is used for encryption parameters, encrypted blocks are limited to **two physical copies**.

The `X` bit indicates one of three cases:

- **Encrypted**: level 0 block and encrypted object type
- **Authenticated**: level 0 block and unencrypted object type
- **Indirect MAC**: level > 0 block (MAC-of-MACs)

The encryption decision uses the object type's `DMU_OT_ENCRYPTED` flag and the dataset's encryption root.

## 10.6 What Gets Encrypted vs Authenticated

- **Encrypted blocks**: Level-0 blocks for encrypted object types (`DMU_OT_IS_ENCRYPTED`), including file data and other per-object payloads.
- **Authenticated-only blocks**: Level-0 blocks for unencrypted object types (e.g., metadata that must remain readable). These carry an HMAC instead of ciphertext.
- **Indirect blocks**: For levels above 0, ZFS stores a SHA512 checksum of the MACs from the level below (MAC-of-MACs) in the blkptr checksum fields to protect the block pointer tree structure.

ZIL blocks are handled specially: their MAC is stored in the embedded checksum within the `zil_chain_t` header, and only the sensitive portions of the ZIL block are encrypted.

## 10.7 Objset MACs

`objset_phys_t` includes two 32-byte MAC fields:

- `os_portable_mac`
- `os_local_mac`

These provide authenticated protection for the objset metadata when encryption is active. The portable MAC covers a mask of flags intended for raw send/receive portability; currently the portable mask is empty (`OBJSET_CRYPT_PORTABLE_FLAGS_MASK = 0`).

## 10.8 Encryption Algorithm IDs

The on-disk crypto suite is stored as `enum zio_encrypt`:

| Enum | Description |
|------|-------------|
| `ZIO_CRYPT_INHERIT` | Inherit encryption setting |
| `ZIO_CRYPT_ON` | Use default cipher |
| `ZIO_CRYPT_OFF` | No encryption |
| `ZIO_CRYPT_AES_128_CCM` | AES-128-CCM |
| `ZIO_CRYPT_AES_192_CCM` | AES-192-CCM |
| `ZIO_CRYPT_AES_256_CCM` | AES-256-CCM |
| `ZIO_CRYPT_AES_128_GCM` | AES-128-GCM |
| `ZIO_CRYPT_AES_192_GCM` | AES-192-GCM |
| `ZIO_CRYPT_AES_256_GCM` | AES-256-GCM |

Cipher values start at 3 (`ZIO_CRYPT_AES_128_CCM`). `ZIO_CRYPT_ON_VALUE` defaults to `ZIO_CRYPT_AES_256_GCM`.

## 10.9 Encrypted Deduplication

Encrypted dedup requires deterministic salts and IVs so identical plaintext produces identical ciphertext. ZFS derives these from an HMAC of the plaintext:

- First 64 bits of the HMAC are used as the salt
- Next 96 bits are used as the IV

Because this uses the dataset's master and HMAC keys, encrypted dedup only works within the same encryption root (clone family).

## 10.10 Raw Send/Receive Notes

Raw encrypted send streams include the portable objset MAC and IV set GUIDs:

- `portable_mac` is stored in the stream and written to `os_portable_mac` on receive.
- `from_ivset_guid` and `to_ivset_guid` are used to validate raw receives. The snapshot's IV set GUID is stored in `com.datto:ivset_guid` (`DS_FIELD_IVSET_GUID`) in the dataset's extensible fields.

The `os_local_mac` is cleared on raw receive because user accounting objects are not transferred in raw streams.
