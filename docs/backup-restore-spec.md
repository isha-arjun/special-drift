# My Finance — encrypted backup and restore specification

My Finance remains device-local. No cloud database, server-side decryption, or automatic backup upload.

## Settings → Data & Backup
- **Export Backup** downloads an encrypted backup file.
- **Upload Backup** opens the iPadOS Files picker for a previously exported .backup file.

## Upload and restore flow
1. A visible Upload Backup button opens a file input accepting .backup and application/json.
2. Read the selected file locally and enforce a reasonable size limit (e.g. 25 MB).
3. Parse and validate the envelope shape and supported format version before requesting a passphrase.
4. Derive the decryption key locally using PBKDF2-SHA-256 with bounded parameters and at least 250,000 iterations.
5. Decrypt with AES-GCM. Wrong passphrase or corrupted ciphertext must fail authentication without changing existing records.
6. Validate the decrypted payload and all records before displaying a preview.
7. Preview backup version, export date, and record counts without exposing sensitive values.
8. Require explicit confirmation before restore.
9. Stage the new dataset and commit it atomically. Any failure must preserve the current dataset.
10. Refresh the UI and report success only after persistence succeeds.

## Suggested encrypted envelope
```json
{
  "format": "my-finance-backup",
  "formatVersion": 1,
  "kdf": {
    "name": "PBKDF2",
    "hash": "SHA-256",
    "iterations": 250000,
    "salt": "<base64>"
  },
  "cipher": {
    "name": "AES-GCM",
    "iv": "<base64>",
    "ciphertext": "<base64>"
  },
  "createdAt": "ISO-8601 timestamp"
}
```

## Acceptance tests
- Valid backup uploads and restores successfully.
- Wrong passphrase, corrupted ciphertext, invalid JSON, malformed payload, unsupported version, or unsafe KDF values are rejected without data loss.
- Cancelling restore leaves current records unchanged.
- iPadOS Safari Files picker works.
- No financial records, backup files, or passphrases enter GitHub or hosted deployment source.

## Preserve existing product rules
- INR and dark theme.
- IndexedDB with localStorage fallback.
- 50% protected savings allocation and original allocation percentages.
- Category-level rollover.
- Existing sections: Overview, Allocations, Transactions, Assets, MacCompat, Future Ventures, Education, Goals, Reports, Settings.
