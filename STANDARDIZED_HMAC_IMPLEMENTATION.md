# Standardized HMAC Implementation

## Problem Solved
Fixed string-matching errors between signing and verification by standardizing the hash string format.

## Key Changes

### 1. Normalizer Function (Both Files)
```javascript
const normalize = (val) => (val ? String(val).trim() : '');
```

### 2. Standardized Hash String Format
Both signing and verification now build the hash string identically:

```javascript
const dataToHash = [
    normalize(data.name),
    normalize(data.email).toLowerCase(),  // Lowercase email for consistency
    normalize(data.organization),
    normalize(data.jobTitle),
    normalize(data.timestamp),
    normalize(data.documentId)
].join('|');
```

### 3. Character Encoding
Uses `new TextEncoder().encode(dataToHash)` before passing to crypto.subtle.sign

### 4. Console Logging for Debugging
Both functions log the exact hash string:
- Signing: `console.log('DEBUG - Final Hash String (Signing):', dataToHash);`
- Verification: `console.log('DEBUG - Final Hash String (Verification):', dataToHash);`

### 5. Data Source: JSON-Only
Verification now reads EXCLUSIVELY from the Subject field JSON:
- No fallback to Keywords
- No parsing of comma-separated strings
- Direct `JSON.parse(subjectRaw)` from Subject field

### 6. Signing Cleanup
Keywords field now only stores fallback data for backward compatibility:
```javascript
doc.setKeywords([digitalSeal, documentId, timestamp]);  // Old format fallback
doc.setSubject(factsJson);  // Primary source (JSON)
```

## Hash String Examples

### Example 1: Complete Data
```
John Doe|jane@example.com|Acme Corp|Software Engineer|2026-04-14T13:43:30.000Z|12345678-1234-1234-1234-123456789abc
```

### Example 2: Missing Organization/JobTitle
```
John Doe|jane@example.com|||2026-04-14T13:43:30.000Z|12345678-1234-1234-1234-123456789abc
```

## Files Modified

### index.html
- Added `normalize()` function
- Updated `generateDigitalSeal(pin, data)` signature
- Standardized hash string format in both signing and verification
- Subject field is primary source for verification
- Added comprehensive console logging

### verify.html
- Added `normalize()` function
- Updated `recalculateSeal(pin, data)` signature
- Standardized hash string format
- Subject field is ONLY source (Keywords ignored)
- Added console logging for debugging

## Testing

1. Sign a document
2. Check browser console for:
   - `DEBUG - Final Hash String (Signing): ...`
3. Verify the document
4. Check browser console for:
   - `DEBUG - Final Hash String (Verification): ...`
5. Compare the two strings - they should be IDENTICAL

## PIN Handling

The PIN is used as the HMAC key (not part of the data string):
- Key derivation: `importPinKey(pin)` → SHA-256 hash of PIN
- HMAC key: derived from PIN hash
- Data string: metadata fields joined by '|'

This ensures:
- PIN never appears in the hash string
- PIN is used to generate the signing key
- Verification uses the same PIN-derived key
