# Timestamp Logic Fix Summary

## Problem
The verification function was failing with:
```
Error: Timestamp looks malformed: '2026-04-14 13:43:30'
```

This occurred because the signing code stored timestamps with a space format (`2026-04-14 13:43:30`) while the verification code expected ISO format with 'T' (`2026-04-14T13:43:30.000Z`).

## Root Cause
1. Signing created `formattedDate` from `utcStamp.replace('T', ' ')` and used it in the PDF title
2. Verification's `parseTitleForSignee()` checked for ISO format using regex `/^\d{4}-\d{2}-\d{2}T/`
3. The mismatch caused verification to fail with "malformed timestamp" error

## Solution

### Signing Phase (index.html)

| Line | Change | Purpose |
|------|--------|---------|
| 1044 | `utcStamp = new Date().toISOString()` | Creates raw ISO timestamp |
| 1046 | `formattedDate` derived from `utcStamp` | For UI display only |
| 1171 | `const timestamp = utcStamp` | Reuse same timestamp for consistency |
| 1323 | `doc.setTitle(\`\${name} — signed \${utcStamp}\`)` | Store RAW ISO in title, not formatted |
| 1320 | `doc.setKeywords([digitalSeal, documentId, timestamp])` | Store RAW ISO in keywords[2] |

### Verification Phase (index.html & verify.html)

| Line | Change | Purpose |
|------|--------|---------|
| `parseTitleForSignee()` | Removed regex check for ISO format | Extract timestamp as-is |
| 1568 / 463 | `verifyParsedTime = keywords[2]` or fallback | Extract RAW ISO from keywords |
| 1577 / 471 | `new Date(parsedTime).toLocaleString()` | Display user-friendly format |
| recalculateSeal() | Uses raw `utcTimestamp` directly | HMAC uses raw ISO string |

## Timestamp Flow

### Signing:
```
new Date().toISOString() → "2026-04-14T13:43:30.000Z" (raw ISO)
    ↓
    • Used for HMAC calculation (correct)
    • Stored in PDF title (correct - raw ISO)
    • Stored in PDF keywords[2] (correct - raw ISO)
    ↓
formattedDate → "2026-04-14 13:43:30" (for display only)
    ↓
    • Displayed in PDF signature box
    • Displayed in UI info panel
```

### Verification:
```
Extract from keywords[2] → "2026-04-14T13:43:30.000Z" (raw ISO)
    ↓
    • Used for HMAC recalculation (correct)
    ↓
Display: new Date(timestamp).toLocaleString() → "4/14/2026, 1:43:30 PM"
```

## Key Principles Applied

1. **Signing Phase**: Use `new Date().toISOString()` for timestamps in cryptographic operations
2. **Verification Phase**: Treat timestamp as raw string for HMAC calculation (no reformatting)
3. **UI Display**: Use `toLocaleString()` only for user-friendly display
4. **HMAC Sync**: Ensure sign and verify use identical timestamp strings in same order

## Files Modified
- `/mnt/d/Code/sandbox/_html/doc-signer/index.html`
- `/mnt/d/Code/sandbox/_html/doc-signer/verify.html`
