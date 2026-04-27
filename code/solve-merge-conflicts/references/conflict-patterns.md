# Common Conflict Patterns — Worked Examples

---

## Pattern: Duplicate Imports

**Symptom:** Both branches added an import at the top of a file.

```
<<<<<<< HEAD
import { Button, Modal } from '@/components/ui';
import { useAuth } from '@/hooks/useAuth';
=======
import { Button, Tooltip } from '@/components/ui';
import { useFeatureFlag } from '@/hooks/useFeatureFlag';
>>>>>>> feature/tooltips
```

**Resolution:** Merge all symbols, deduplicate, keep sorted.
```typescript
import { Button, Modal, Tooltip } from '@/components/ui';
import { useAuth } from '@/hooks/useAuth';
import { useFeatureFlag } from '@/hooks/useFeatureFlag';
```

Auto-resolvable: ✅ (both additions are additive and independent)

---

## Pattern: Same Function Modified Differently

**Symptom:** Both branches changed the same function body.

```
<<<<<<< HEAD
async function processPayment(amount: number) {
  const result = await stripe.charge(amount);
  return result.id;
}
=======
async function processPayment(amount: number, currency = 'USD') {
  const result = await stripe.charge(amount);
  await analytics.track('payment', { amount });
  return result.id;
}
>>>>>>> feature/multi-currency
```

**Analysis:**
- HEAD: no semantic change from base (same logic)
- Incoming: added `currency` parameter + analytics tracking

**Resolution:** Both changes are additive — synthesize.
```typescript
async function processPayment(amount: number, currency = 'USD') {
  const result = await stripe.charge(amount, currency);
  await analytics.track('payment', { amount, currency });
  return result.id;
}
```

Auto-resolvable: ⚠️ Proceed with caution — verify `stripe.charge` actually accepts currency.

---

## Pattern: Config Object — Both Sides Added Keys

```javascript
<<<<<<< HEAD
const config = {
  apiUrl: process.env.API_URL,
  timeout: 5000,
  retries: 3,
};
=======
const config = {
  apiUrl: process.env.API_URL,
  timeout: 5000,
  featureFlags: {
    newCheckout: true,
  },
};
>>>>>>> feature/flags
```

**Resolution:** Merge all keys (both are additive).
```javascript
const config = {
  apiUrl: process.env.API_URL,
  timeout: 5000,
  retries: 3,
  featureFlags: {
    newCheckout: true,
  },
};
```

Auto-resolvable: ✅

---

## Pattern: Renamed Function + Caller

**Symptom:** One branch renamed a function; the other branch calls it under its old name.

```
# file: auth.ts (their branch renamed the function)
<<<<<<< HEAD
export function validateUser(token: string) {
=======
export function verifyUserToken(token: string) {
>>>>>>> refactor/naming

# file: api.ts (our branch still calls the old name)
const isValid = validateUser(req.headers.authorization);
```

This is a **semantic conflict** — the merge will succeed textually but fail at runtime.

**Resolution:**
1. Decide which name to keep (ask user if unclear — this is an API surface change)
2. Update ALL callers across the codebase: `grep -rn "validateUser" src/`
3. Update exports and any TypeScript interfaces

Ask user: ⚠️ "Which function name should we standardize on: `validateUser` or `verifyUserToken`? I'll update all call sites."

---

## Pattern: Test File Conflicts

**Symptom:** Both branches added tests to the same `describe` block.

```
<<<<<<< HEAD
  it('returns 404 when user not found', async () => {
    const res = await request(app).get('/users/999');
    expect(res.status).toBe(404);
  });
=======
  it('returns 401 when token is expired', async () => {
    const res = await request(app).get('/users/1').set('Authorization', 'expired');
    expect(res.status).toBe(401);
  });
>>>>>>> feature/auth-errors
```

**Resolution:** Keep both tests — they test different behaviors.
Auto-resolvable: ✅

---

## Pattern: CSS / Tailwind Class Conflicts

```
<<<<<<< HEAD
<button className="px-4 py-2 bg-blue-500 text-white rounded">
=======
<button className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700">
>>>>>>> feature/hover-states
```

**Analysis:** One side changed the shade (500→600) AND added hover state. Both are likely
intentional design decisions.

**Ask user:** "Our branch uses `bg-blue-500`, their branch uses `bg-blue-600` with hover
state added. Should I use `bg-blue-600 hover:bg-blue-700` (their full version) or keep
`bg-blue-500` with the hover state added?"

---

## Pattern: Version Bumps (package.json `version` field)

```json
<<<<<<< HEAD
  "version": "2.3.1",
=======
  "version": "2.4.0",
>>>>>>> release/2.4.0
```

**Resolution:** Use the higher version.
Auto-resolvable: ✅ (take `2.4.0` — it's the intentional release version)

---

## Pattern: Environment Config

```
<<<<<<< HEAD
DATABASE_URL=postgres://localhost:5432/myapp_dev
REDIS_URL=redis://localhost:6379
=======
DATABASE_URL=postgres://localhost:5432/myapp_dev
NEW_FEATURE_ENABLED=true
>>>>>>> feature/new-feature
```

**Resolution:** Merge both — each side added a different key.
Auto-resolvable: ✅ (but flag `NEW_FEATURE_ENABLED` to user — env vars affect runtime behavior)

---

## Heuristics for Auto vs Ask

| Situation | Action |
|-----------|--------|
| Both sides added independent new code | Auto-resolve, report |
| One side is a clear bugfix on top of the other | Auto-resolve, report |
| Both sides modified the same logic differently | Ask |
| A function was renamed or moved | Ask |
| A public API signature changed | Ask |
| Config values changed (not just added) | Ask |
| Database migration numbering conflicts | Always ask |
| Lock file | Auto-regenerate, report |
| Version bump | Take the higher version, report |
