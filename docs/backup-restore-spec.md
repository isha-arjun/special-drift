# My Finance — encrypted backup and restore specification

## User experience

Settings → Data & Backup:

- Export Backup
- Upload Backup
- Last backup status (local UI only; do not imply server synchronization)

### Export

1. Read the complete app dataset from the local persistence layer.
2. Serialize a versioned envelope payload.
3. Generate a cryptographically random salt and AES-GCM IV using the browser Web Crypto API.
4. Derive an encryption key in-browser using PBKDF2-SHA-256 (minimum 250,000 iterations).
5. Encrypt with AES-GCM.
6. Trigger a download named `my-finance-backup-YYYY-MM-DD.backup`.
7. Do not store the passphrase after export; do not send the payload off-device.

### Upload / restore

1. Render a visible **Upload Backup** button linked to a hidden file input with accept=".backup,application/json".
2. Read the selected file using the browser File API; enforce a conservative size limit (e.g. 25 MB).
3. Parse JSON and validate the envelope shape and supported format version before any decryption.
4. Ask the user for the passphrase in a dedicated dialog.
5. Derive the key using the parameters embedded in the validated envelope, with strict bounds to prevent unreasonable iteration counts.
6. Decrypt with AES-GCM. A failed authentication tag means wrong passphrase or damaged file; do not modify app state.
7. Validate the decrypted payload schema and every record before showing a preview.
8. Show export timestamp, backup version and record counts. Do not show sensitive values in a notification.
9. Require explicit confirmation before restoring.
10. Prefer staging into a temporary object/store, then write the full dataset as one transaction. Keep a pre-restore snapshot in memory until persistence succeeds. On failure, preserve the prior dataset.
11. Reload derived UI state and show a confirmation after successful restore.

## Envelope sketch

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

The decrypted payload should include its own schema version and app data. Do not trust values from a backup until all validation completes.

## Required test cases

- Valid export can be uploaded and restored.
- Wrong passphrase reports a generic decryption error and preserves current data.
- Altered ciphertext fails authentication and preserves current data.
- Invalid JSON or missing envelope fields are rejected.
- Unknown format version is rejected with a useful explanation.
- Unsupported or out-of-range KDF parameters are rejected.
- Malformed inner data is rejected.
- Cancelled restore leaves data unchanged.
- Successful restore reproduces all supported records and settings.
- iPadOS Safari Files picker can select the downloaded backup.

## Non-goals

- No cloud database.
- No upload to GitHub, Vercel, or other hosted storage.
- No automatic background backups.
- No bank integrations.
- No server-side decryption.
