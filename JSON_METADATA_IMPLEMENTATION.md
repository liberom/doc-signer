## Data Flow

### Signing:
1. Collect all facts into a JavaScript object
2. Generate HMAC-SHA256 seal using PIN + PDF bytes + metadata
3. Add seal to the facts object
4. JSON.stringify() the complete object
5. Store in PDF **Subject** field (primary) and Keywords (fallback)
6. Set Author field to just the person's name

### Verification:
1. Read Subject field (primary source)
2. If Subject is valid JSON, parse it
3. If not, try parsing from Keywords (handles comma-separated strings)
4. If still no data, fallback to old format (seal, documentId, timestamp)
5. Extract all fields from the facts object
6. Re-calculate seal using PIN + PDF bytes
7. Compare recalculated seal with stored seal

## Example JSON Structure

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "organization": "Acme Corp",
  "jobTitle": "Software Engineer",
  "timestamp": "2026-04-14T13:43:30.000Z",
  "documentId": "12345678-1234-1234-1234-123456789abc",
  "seal": "a1b2c3d4e5f6..."
}
```

## Files Modified

- `index.html` (signing logic + verification logic in SPA mode)
- `verify.html` (standalone verification page)

## Verification Order in index.html

1. Standard metadata:
   - `setAuthor(name)` - Just the person's name for PDF readers
   
2. JSON-in-Metadata (Subject field):
   - `setSubject(factsJson)` - Complete JSON string as source of truth
   
3. Fallback (Keywords field):
   - `setKeywords([digitalSeal, documentId, timestamp])` - For backward compatibility

## Console Logging for Debugging

The verification logic includes comprehensive console logging:
- Raw Subject and Keywords content before parsing
- Success/failure messages for each parsing attempt
- Extracted metadata object for field inspection
- Organization, Job Title, Document ID, and Seal values

### 2. Verification Logic (index.html and verify.html)

#### Reading JSON from Keywords
```javascript
const factsJson = keywords[0];
try {
  factsData = JSON.parse(factsJson);
} catch (e) {
  throw new Error('Invalid JSON in Keywords metadata — file may have been signed with an older version.');
}

// Extract data from JSON object (source of truth)
storedSeal = factsData.seal;
parsedName = factsData.name;
parsedEmail = factsData.email || '(not stored)';
parsedTime = factsData.timestamp;
```

#### Title Parsing with "Document Signed by" Prefix
```javascript
function parseTitleForSignee(titleStr) {
  const marker = ' — signed ';
  const idx = titleStr.lastIndexOf(marker);
  if (idx === -1) throw new Error('Title metadata missing " — signed " marker.');
  let name = titleStr.slice(0, idx).trim();
  // Remove "Document Signed by " prefix if present
  if (name.startsWith('Document Signed by ')) {
    name = name.slice('Document Signed by '.length).trim();
  }
  const utcStr = titleStr.slice(idx + marker.length).trim();
  return { name, utcTimestamp: utcStr };
}
```

## Key Benefits

1. **All Data Preserved**: Organization, JobTitle, and other metadata are no longer lost
2. **Single Source of Truth**: The JSON in Keywords contains all facts needed for verification
3. **Backward Compatible**: Old signature verification still works (falls back to parsing title)
4. **Standard PDF Reader Friendly**: Author field shows just the name, making it look nice in PDF viewers
5. **Error Handling**: Clear error messages when JSON parsing fails

## Data Flow

### Signing:
1. Collect all facts into a JavaScript object
2. Generate HMAC-SHA256 seal using PIN + PDF bytes + metadata
3. Add seal to the facts object
4. JSON.stringify() the complete object
5. Store in PDF Keywords[0] field
6. Set Author field to just the person's name

### Verification:
1. Read Keywords[0] from PDF metadata
2. JSON.parse() to get the facts object
3. Extract seal, name, email, timestamp from JSON
4. Re-calculate seal using the same PIN + PDF bytes
5. Compare recalculated seal with stored seal

## Example JSON Structure

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "organization": "Acme Corp",
  "jobTitle": "Software Engineer",
  "timestamp": "2026-04-14T13:43:30.000Z",
  "documentId": "12345678-1234-1234-1234-123456789abc",
  "seal": "a1b2c3d4e5f6..."
}
```

## Files Modified

- `index.html` (signing logic + verification logic in SPA mode)
- `verify.html` (standalone verification page)

## Verification Order in index.html

1. Standard metadata:
   - `setAuthor(name)` - Just the person's name for PDF readers
   
2. JSON-in-Metadata:
   - `setKeywords([factsJson])` - Complete JSON string as source of truth
   - `setSubject(...)` - Descriptive subject line
   - `setTitle(...)` - "Document Signed by {name} — signed {timestamp}"
