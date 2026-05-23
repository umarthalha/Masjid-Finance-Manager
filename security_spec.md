# Security Specifications - Masjid Finance Tracker

This specification describes the security rules and data invariants for our Firestore deployment, detailing how we block malicious data manipulation.

## 1. Data Invariants

1. **Masjid Integrity**:
   - A `FinanceEntry` or `TempCategory` must always be bound to a valid `masjidId`.
   - A user can only read, create, update, or delete entries or categories for a Masjid they have access to.

2. **Temporal Correctness**:
   - Timestamps like `createdAt` must exactly match the Firestore server time (`request.time`).
   - Entry date picker only allows past or present dates (format `YYYY-MM-DD`).

3. **Value Sanitization**:
   - Financial entry amounts must be numbers and strictly positive (> 0).
   - Category names and IDs must not be oversized (e.g., limit id sizes to prevent exhaustion).

4. **Self-Locked RBAC**:
   - Users cannot change their emails or spoof their `uid` to other accounts.

---

## 2. The "Dirty Dozen" Payloads

Here are 12 specific JSON payloads designed to break the laws of Identity, Integrity, and State that our security rules will prevent:

### Pillar: Identity Spoofing & Owner Protection
1. **Malicious Transaction Owner Spotting**: Saving a finance entry with a custom fake `userId`.
2. **Masjid Spoofing**: Saving an entry under a `masjidId` that belongs to someone else's mosque, which the current user is not authorized to edit.
3. **User Profile Theft**: Updating another user's document under `/users/{someOtherUid}`.

### Pillar: Integrity & Type Pollution
4. **Negative Value Poisoning**: Saving a FinanceEntry with `amount: -500`.
5. **Blank Type Pollution**: Saving a FinanceEntry with an empty `type: ""` or `type: "fakenews"`.
6. **Immutable Hijacking**: Trying to change `createdAt` or `userId` in a transaction during an update.
7. **Temporal Fraud**: Setting `createdAt` of a new entry to a future year 2030 or any custom client-side timestamp instead of `request.time`.
8. **Invalid Date Injection**: Saving an entry with `date: "2026-12-31"` (future date) or invalid string format.

### Pillar: Denial of Wallet (Resource Poisoning)
9. **Oversized String Injection**: Trying to write a transaction description that is 500KB to bloat DB size and cause indexing issues.
10. **Junk Path ID Poisoning**: Specifying an entry ID with special script characters or extra long strings (e.g. `entry_123456789_very_long_junk_payload_...`).

### Pillar: State Shortcutting & Unauthorized Schema Keys
11. **Shadow Field Insertion**: Sending an entry with a ghost field `isSuperAdmin: true`.
12. **Blank Category Creation**: Adding `TempCategory` with empty text name.

---

## 3. Security Rules Draft

We will enforce these validations directly inside `/firestore.rules`.
