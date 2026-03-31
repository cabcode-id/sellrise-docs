# PRD 3: Secure Photo Management

## Goal
Implement a secure, encrypted photo upload and retrieval pipeline for patient medical photos.

## Key Features
- **Direct Upload**: Upload photo directly from browser to Phlastic API (bypassing Sellrise).
- **Storage Rules**: 
    - Files stored in non-web-accessible directory.
    - UUID-based filenames (no PII in paths).
- **Access Control**: 
    - Authenticated API-only access.
    - No public URLs.

## Success Metrics
- Photo successfully uploaded and stored in encrypted path.
- Photo binary can only be downloaded by authenticated staff.
- Photo metadata (type, consent) is correctly stored in `photos` table.
