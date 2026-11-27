# Mobile App Technical Changes: P1-058 Rate Limiting and Abuse Protection

## 1. Overview and Business Context

In accordance with Phase 1 Case **P1-058** ("Add rate limits and anti-automation protections"), the backend API now enforces strict rate limiting and abuse controls across authentication, parent linking, and interactive student answer submissions.

When rate limits are exceeded, the API answers with:
- **HTTP Status**: `429 Too Many Requests`
- **Response Error Code**: `RATE_LIMIT_EXCEEDED`
- **Standard Header**: `Retry-After: <seconds>` (indicating the exact number of seconds the client must wait before retrying)

This document outlines the required changes in the React Native / Expo mobile application repository (`D:\lm`) to handle throttling cleanly, provide intuitive user feedback, and prevent answer loss during practice sessions or exams.

---

## 2. Backend Rate Limit Policies and Client Impacts

| Category | Backend Endpoints | Threshold (per 60s) | Mobile App Impact & UX Requirement |
|---|---|---|---|
| **Email OTP** | `POST /auth/otp/send`<br>`POST /auth/otp/verify`<br>`POST /auth/otp/resend` | 5 req/min per IP<br>5 req/min per email | 1. Enforce a 60-second in-app countdown timer after requesting an OTP before re-enabling the "Resend Code" button.<br>2. On 429, display an inline alert: "Too many code requests. Please wait X seconds before trying again." |
| **Authentication** | `POST /auth/login`<br>`POST /auth/register`<br>`POST /auth/google` | 5 req/min per IP<br>5 req/min per email | 1. Disable submit buttons immediately upon tap to prevent rapid double-submits.<br>2. On 429, show: "Too many sign-in attempts. Please try again in X seconds." |
| **Parent Invites** | `POST /parent/invites/redeem`<br>`POST /student/parent-invites/redeem` | 20 req/min per IP | 1. Limit code redemption retries.<br>2. On 429, show: "Too many redemption attempts. Please wait X seconds before trying again." |
| **Answer Submissions** | `PUT /student/practice/sessions/:id/items/:id/answer`<br>`POST /student/practice/sessions/:id/submit`<br>`PUT /student/mock-exams/attempts/:id/items/:id/answer`<br>`POST /student/mock-exams/attempts/:id/submit` | 60 req/min per student | 1. If a student taps answers rapidly or network queue flushes, buffer requests locally.<br>2. If a 429 occurs, pause submission queue for `Retry-After` seconds and automatically retry without losing student input or showing scary error modals. |

---

## 3. Step-by-Step Implementation Guide for Mobile App

### Step 3.1: Global 429 Interceptor in API Client

Configure the centralized HTTP client (e.g. Axios or Fetch wrapper in `src/services/api/` or `src/lib/api.ts`) to parse the `Retry-After` header on any HTTP 429 response:

```typescript
// Example: src/services/api/interceptors.ts

export interface RateLimitError {
  isRateLimited: true;
  retryAfterSeconds: number;
  message: string;
}

export function parseRateLimitResponse(error: any): RateLimitError | null {
  if (error?.response?.status === 429) {
    const rawRetryAfter = error.response.headers['retry-after'];
    const parsedSeconds = parseInt(rawRetryAfter, 10);
    const retryAfterSeconds = Number.isFinite(parsedSeconds) && parsedSeconds > 0 ? parsedSeconds : 60;

    return {
      isRateLimited: true,
      retryAfterSeconds,
      message: `Too many requests. Please wait ${retryAfterSeconds} seconds before retrying.`,
    };
  }
  return null;
}
```

---

### Step 3.2: OTP Resend Button Cooldown and 429 Handling

In the OTP verification screen (e.g. `src/screens/auth/OtpVerificationScreen.tsx`):
1. **Local Timer**: Initialize an in-memory countdown of 60 seconds whenever an OTP is requested.
2. **Disabled State**: Disable the "Resend Code" button while the timer is greater than zero.
3. **429 Feedback**: If a resend request triggers a 429 error, reset the timer to the `retryAfterSeconds` returned in the error response and display an amber banner.

```tsx
// Example Snippet: OtpScreen.tsx
const handleResend = async () => {
  if (resendCooldown > 0) return;
  setIsLoading(true);

  try {
    await authApi.resendOtp(email);
    setResendCooldown(60);
  } catch (error) {
    const rateLimit = parseRateLimitResponse(error);
    if (rateLimit) {
      setResendCooldown(rateLimit.retryAfterSeconds);
      showToast(rateLimit.message, 'warning');
    } else {
      showToast('Failed to resend code. Please check your network.', 'error');
    }
  } finally {
    setIsLoading(false);
  }
};
```

---

### Step 3.3: Student Answer Queue Pacing and Resilience

During practice sessions or timed mock exams, students may skip through questions quickly or recover from intermittent network drops. If multiple queued answers are submitted in rapid succession, they could exceed the 60 req/min threshold.

**Implementation Strategy**:
1. **Immediate Local State Update**: Immediately update the UI state and mark the selected option as chosen so the student feels zero lag.
2. **Sequential Background Queue**: Submit answers sequentially with a minimal delay (e.g. 100 ms between requests).
3. **Automatic 429 Backoff**:
   - If a background answer submission receives a 429:
     - Read the `Retry-After` header.
     - Pause the background queue for `retryAfterSeconds`.
     - Automatically retry sending the answer without prompting the student or discarding their choice.

```typescript
// Example: src/services/practice/answerQueue.ts

class AnswerSubmissionQueue {
  private queue: Array<() => Promise<void>> = [];
  private isProcessing = false;
  private pausedUntil: number = 0;

  async enqueue(submitFn: () => Promise<void>) {
    this.queue.push(submitFn);
    this.processQueue();
  }

  private async processQueue() {
    if (this.isProcessing) return;
    this.isProcessing = true;

    while (this.queue.length > 0) {
      const now = Date.now();
      if (now < this.pausedUntil) {
        await new Promise((res) => setTimeout(res, this.pausedUntil - now));
      }

      const task = this.queue.shift();
      if (!task) break;

      try {
        await task();
        // Safe 100ms spacing between answers to avoid burst throttling
        await new Promise((res) => setTimeout(res, 100));
      } catch (error: any) {
        const rateLimit = parseRateLimitResponse(error);
        if (rateLimit) {
          // Pause queue for specified Retry-After window and re-queue the task
          this.pausedUntil = Date.now() + rateLimit.retryAfterSeconds * 1000;
          this.queue.unshift(task);
        } else {
          console.error('Answer submission failed:', error);
        }
      }
    }

    this.isProcessing = false;
  }
}

export const answerQueue = new AnswerSubmissionQueue();
```

---

### Step 3.4: Parent Invite Code Redemption

In `src/screens/parent/RedeemInviteScreen.tsx`:
1. Limit code submission to 1 tap per request.
2. On 429 `RATE_LIMIT_EXCEEDED`: Display an alert explaining that too many attempts have been made, indicating the remaining cooldown.
3. On 429 `INVITE_LOCKED`: Display "This account is temporarily locked out of redeeming codes due to repeated wrong codes. Please try again in 5 minutes."

---

## 4. Verification and Acceptance Checklist for Mobile

- [ ] Requesting an OTP starts a 60-second visual countdown on the "Resend" button.
- [ ] Repeated rapid taps on the sign-in button do not send duplicate requests.
- [ ] If a 429 is received on login, a toast displays the countdown from `Retry-After`.
- [ ] Rapidly clicking answers across 10 practice questions updates the UI instantly and syncs sequentially without triggering 429 errors.
- [ ] Entering 20 invalid invite codes in under a minute displays a friendly throttling message rather than an unhandled network error crash.
- [ ] Submitting the exam or session finalizes cleanly even under slow connection throttling.
