# Admin Panel Technical Changes: P1-058 Rate Limiting and Abuse Protection

## 1. Overview and Business Context

In accordance with Phase 1 Case **P1-058** ("Add rate limits and anti-automation protections"), the backend API has added rate limiting to admin authentication and question bulk import endpoints:
- `POST /api/v1/admin/auth/login`: Limited to 5 requests per minute per IP and per account.
- `POST /api/v1/admin/questions/bulk`: Limited to 20 chunk uploads per minute per admin IP (previously marked with `@SkipThrottle()`).

When rate limits are exceeded, the API answers with HTTP 429 `RATE_LIMIT_EXCEEDED` and a standard `Retry-After: <seconds>` response header.

This document outlines the required adjustments in the React/Vite admin panel repository (`learning-app-admin`) to ensure smooth bulk question imports and friendly error feedback.

---

## 2. Admin Panel Impacts and Requirements

### 2.1 Bulk Question Import Pacing (Critical)
The question bulk import workflow (`src/features/questions/`) reads a CSV or Excel file and sends rows in chunks of 50.
- For files with fewer than 1,000 questions (up to 20 chunks), imports finish in under a minute without hitting limits.
- For files with more than 1,000 questions (e.g. 2,000 questions = 40 chunks), sending chunks back-to-back without pacing will exceed the 20 chunks/minute limit and trigger a 429 error.
- **Requirement**: The admin upload loop must:
  1. Introduce a short 200 ms inter-chunk pacing delay between sequential chunk uploads.
  2. In the event of an HTTP 429 response, automatically pause for the number of seconds specified in the `Retry-After` header and then resume uploading chunks, rather than aborting the entire file import.

---

### 2.2 Admin Authentication Throttle Feedback
On the login screen (`src/features/auth/`):
- If an admin or an attacker fails login 5 times within a minute, the backend returns a 429 error.
- **Requirement**: Display a clear countdown warning: "Too many login attempts. Please wait X seconds before trying again."

---

## 3. Step-by-Step Implementation Guide for Admin Panel

### Step 3.1: Bulk Import Chunk Upload Loop with Automatic 429 Retry

In `src/features/questions/` (e.g. `src/features/questions/components/bulk-import-dialog.tsx` or `src/features/questions/api/import.ts`):
Update the chunk uploading loop to include inter-chunk delay and automatic retry on 429:

```typescript
// Example: src/features/questions/api/bulk-import-runner.ts

interface ChunkUploadOptions {
  batchId?: string;
  dryRun?: boolean;
  onProgress?: (processed: number, total: number, statusText?: string) => void;
}

export async function uploadQuestionChunks(
  rows: any[],
  chunkSize = 50,
  options?: ChunkUploadOptions,
) {
  const totalRows = rows.length;
  let processed = 0;
  const results: any[] = [];

  for (let i = 0; i < totalRows; i += chunkSize) {
    const chunk = rows.slice(i, i + chunkSize);
    let success = false;
    let attempts = 0;

    while (!success && attempts < 5) {
      attempts += 1;
      try {
        const response = await apiRequest({
          url: '/api/v1/admin/questions/bulk',
          method: 'POST',
          body: {
            rows: chunk,
            dryRun: options?.dryRun ?? false,
          },
        });

        results.push(...response.data.results);
        processed += chunk.length;
        options?.onProgress?.(processed, totalRows);
        success = true;

        // Pacing delay of 200ms between chunks under normal conditions
        await new Promise((res) => setTimeout(res, 200));
      } catch (error: any) {
        if (error?.status === 429 || error?.statusCode === 429) {
          // Read Retry-After header or default to 5 seconds
          const retryAfter = parseInt(error.headers?.['retry-after'] ?? '5', 10);
          const waitSeconds = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter : 5;

          options?.onProgress?.(
            processed,
            totalRows,
            `Rate limit reached. Pausing for ${waitSeconds}s before resuming...`,
          );

          // Wait out the throttle window and retry the same chunk
          await new Promise((res) => setTimeout(res, waitSeconds * 1000));
        } else {
          // Non-throttle errors should be thrown to fail or report
          throw error;
        }
      }
    }
  }

  return results;
}
```

---

### Step 3.2: Login Screen Throttle Feedback

In `src/features/auth/components/login-form.tsx`:
Catch HTTP 429 errors and extract `Retry-After`:

```tsx
// Example snippet in login submit handler:
try {
  await login(email, password);
  navigate('/dashboard');
} catch (error: any) {
  if (error?.status === 429 || error?.statusCode === 429) {
    const waitSeconds = error.headers?.['retry-after'] ?? 60;
    setErrorMessage(`Too many sign-in attempts. Please wait ${waitSeconds} seconds before trying again.`);
  } else {
    setErrorMessage(error.message || 'Invalid email or password.');
  }
}
```

---

## 4. Verification and Acceptance Checklist for Admin Panel

- [ ] Uploading a CSV file with 100 questions (2 chunks) completes cleanly with progress updates.
- [ ] Uploading a large CSV file with 1,500 questions paces chunk requests by 200 ms.
- [ ] If throttled during a large import, the import dialog pauses with an informative status message and resumes automatically without aborting or losing previously committed rows.
- [ ] After 5 consecutive failed login attempts, the login form clearly communicates the cooldown period.
