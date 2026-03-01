# Claude Console Test Model Fix Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Ensure `claude-console` account connectivity tests use the model selected in Admin UI, require `model` input, and expose `claude-sonnet-4-6` as the second Claude model option.

**Architecture:** Keep scope minimal and local: enforce request validation in admin route, propagate model explicitly into relay test payload, and update model catalog ordering. No mapping logic changes; no scheduler changes.

**Tech Stack:** Node.js (Express), Jest, Vue Admin SPA configuration via backend `config/models.js`.

---

### Task 1: Add Failing Route Validation Test (`model` required)

**Files:**
- Create: `tests/claudeConsoleAccounts.test.js`
- Modify: `src/routes/admin/claudeConsoleAccounts.js`

**Step 1: Write the failing test**

```js
jest.mock('../src/middleware/auth', () => ({
  authenticateAdmin: (_req, _res, next) => next()
}))

jest.mock('../src/services/relay/claudeConsoleRelayService', () => ({
  testAccountConnection: jest.fn().mockResolvedValue(undefined)
}))

const express = require('express')
const request = require('supertest')
const router = require('../src/routes/admin/claudeConsoleAccounts')

describe('POST /claude-console-accounts/:accountId/test', () => {
  it('returns 400 when model is missing', async () => {
    const app = express()
    app.use(express.json())
    app.use('/admin', router)

    const res = await request(app).post('/admin/claude-console-accounts/a1/test').send({})

    expect(res.status).toBe(400)
    expect(res.body.error).toContain('model')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `npm test -- tests/claudeConsoleAccounts.test.js -i`
Expected: FAIL (current route does not validate `model`).

**Step 3: Write minimal implementation**

In route handler:
- Read `const model = typeof req.body?.model === 'string' ? req.body.model.trim() : ''`
- If empty, `return res.status(400).json({ error: 'model is required' })`
- Else call `await claudeConsoleRelayService.testAccountConnection(accountId, res, model)`

**Step 4: Run test to verify it passes**

Run: `npm test -- tests/claudeConsoleAccounts.test.js -i`
Expected: PASS.

**Step 5: Commit**

```bash
git add tests/claudeConsoleAccounts.test.js src/routes/admin/claudeConsoleAccounts.js
git commit -m "fix: require model for claude-console account test endpoint"
```

### Task 2: Add Failing Service Test (selected model used in payload)

**Files:**
- Create: `tests/claudeConsoleRelayService.test.js`
- Modify: `src/services/relay/claudeConsoleRelayService.js`

**Step 1: Write the failing test**

```js
jest.mock('../src/services/account/claudeConsoleAccountService', () => ({
  getAccount: jest.fn().mockResolvedValue({
    id: 'a1',
    name: 'acc',
    apiUrl: 'https://example.com',
    apiKey: 'k',
    proxy: null,
    userAgent: ''
  }),
  _createProxyAgent: jest.fn().mockReturnValue(null)
}))

const helper = require('../src/utils/testPayloadHelper')
jest.spyOn(helper, 'sendStreamTestRequest').mockResolvedValue(undefined)

const svc = require('../src/services/relay/claudeConsoleRelayService')

describe('claudeConsoleRelayService.testAccountConnection', () => {
  it('passes selected model in payload', async () => {
    const res = { headersSent: true, write: jest.fn(), end: jest.fn() }
    await svc.testAccountConnection('a1', res, 'claude-sonnet-4-6')

    const args = helper.sendStreamTestRequest.mock.calls[0][0]
    expect(args.payload.model).toBe('claude-sonnet-4-6')
    expect(args.payload.stream).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `npm test -- tests/claudeConsoleRelayService.test.js -i`
Expected: FAIL (service currently does not pass payload model).

**Step 3: Write minimal implementation**

In `testAccountConnection(accountId, responseStream, model)`:
- Add `model` argument.
- Build payload via `createClaudeTestPayload(model, { stream: true })`.
- Pass payload to `sendStreamTestRequest({ payload, ... })`.

**Step 4: Run test to verify it passes**

Run: `npm test -- tests/claudeConsoleRelayService.test.js -i`
Expected: PASS.

**Step 5: Commit**

```bash
git add tests/claudeConsoleRelayService.test.js src/services/relay/claudeConsoleRelayService.js
git commit -m "fix: use selected model in claude-console connectivity test payload"
```

### Task 3: Add Failing Config Test (`claude-sonnet-4-6` at position #2)

**Files:**
- Create: `tests/modelsConfig.test.js`
- Modify: `config/models.js`

**Step 1: Write the failing test**

```js
const { CLAUDE_MODELS } = require('../config/models')

describe('config/models Claude list', () => {
  it('contains claude-sonnet-4-6 as second item', () => {
    expect(CLAUDE_MODELS[1]).toEqual({
      value: 'claude-sonnet-4-6',
      label: 'Claude Sonnet 4.6'
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `npm test -- tests/modelsConfig.test.js -i`
Expected: FAIL.

**Step 3: Write minimal implementation**

In `config/models.js`:
- Insert `{ value: 'claude-sonnet-4-6', label: 'Claude Sonnet 4.6' }` into `CLAUDE_MODELS` at index 1.

**Step 4: Run test to verify it passes**

Run: `npm test -- tests/modelsConfig.test.js -i`
Expected: PASS.

**Step 5: Commit**

```bash
git add tests/modelsConfig.test.js config/models.js
git commit -m "feat: add claude sonnet 4.6 model option for tests"
```

### Task 4: Regression and Quality Gate

**Files:**
- Modify: none
- Test: `tests/claudeConsoleAccounts.test.js`, `tests/claudeConsoleRelayService.test.js`, `tests/modelsConfig.test.js`

**Step 1: Run focused tests**

Run:
- `npm test -- tests/claudeConsoleAccounts.test.js -i`
- `npm test -- tests/claudeConsoleRelayService.test.js -i`
- `npm test -- tests/modelsConfig.test.js -i`

Expected: all PASS.

**Step 2: Run full test suite**

Run: `npm test -- -i`
Expected: PASS (or document unrelated pre-existing failures).

**Step 3: Manual verification checklist**

- In Admin UI, select `claude-sonnet-4-6` and run `claude-console` connectivity test.
- Confirm backend outbound payload `model` equals selected value.
- Call endpoint without `model`; confirm `400` + explicit error.
- Confirm SSE success/error rendering still works.

**Step 4: Final commit (if needed)**

```bash
git add -A
git commit -m "test: add regression coverage for claude-console connectivity test model handling"
```

(Only if there are non-committed verification-related changes.)
