# Claude Console Test Model Fix Design

Date: 2026-03-01
Status: Approved
Owner: Codex + mike

## Background

In the Admin SPA account connectivity test flow, the UI lets users choose a test model. For `claude-console` accounts, this selected model is currently ignored by backend test execution. The backend test path falls back to a hardcoded/default model payload, causing misleading test results.

This is a functional bug because model choice is required to validate account capability and behavior against the intended model.

## Scope

In scope:
- Fix `claude-console` account connectivity test so backend uses the model selected in frontend.
- Make `model` required for the `claude-console` account test endpoint.
- Do not apply `supportedModels` mapping in connectivity test path.
- Add `claude-sonnet-4-6` to Claude model options and place it at position #2 in the list.

Out of scope:
- Refactoring all platform test endpoints to one shared contract.
- Introducing model mapping behavior in account connectivity test.

## Requirements (Confirmed)

1. For `claude-console` account connectivity test, selected model must be used directly.
2. No mapping table transformation should happen in test flow.
3. If `model` is missing/empty, return HTTP 400 (fail fast).
4. Add `claude-sonnet-4-6` (short name, no date suffix) to frontend-selectable Claude models.
5. Put `claude-sonnet-4-6` in the second position of Claude model list.

## Current Problem

Current path:
- Frontend sends `{ model: selectedModel }` for account test.
- `POST /admin/claude-console-accounts/:accountId/test` does not pass model to service.
- `claudeConsoleRelayService.testAccountConnection(...)` does not provide payload model to stream test helper.
- Helper default payload model is used.

Result: UI model selection is ignored.

## Proposed Solution

### 1) API Contract Adjustment (`claude-console` test endpoint)

Endpoint:
- `POST /admin/claude-console-accounts/:accountId/test`

Behavior:
- Read `req.body.model`.
- Validate it is a non-empty string (trimmed).
- On invalid input, return `400` with explicit message (`model is required`).
- On valid input, pass model into service layer.

### 2) Service Layer Model Propagation

Function:
- `claudeConsoleRelayService.testAccountConnection(accountId, responseStream, model)`

Behavior:
- Build payload with explicit selected model:
  - `createClaudeTestPayload(model, { stream: true })`
- Pass payload into `sendStreamTestRequest({ payload, ... })`.
- Keep existing SSE output format unchanged.

### 3) Model List Update for Frontend Selection

File:
- `config/models.js`

Change:
- Insert `{ value: 'claude-sonnet-4-6', label: 'Claude Sonnet 4.6' }` as the second item in `CLAUDE_MODELS`.

Reason:
- Admin SPA test model selector consumes backend model config order.

## Error Handling

For `claude-console` test endpoint:
- `400` when `model` missing/empty.
- Existing service-level failures continue using current SSE error channel and logging.

No silent fallback to default model in this path.

## Compatibility & Risk

Compatibility:
- Changes are scoped to `claude-console` account connectivity test endpoint.
- No behavior change for `claude`/`bedrock`/other platform test endpoints in this iteration.

Risk:
- Low; endpoint-level validation may expose previously hidden bad requests from UI or scripts.
- Mitigation: UI already sends `model`; expected to pass.

## Verification Plan

Manual checks:
1. Choose model A in Admin UI for `claude-console` test; verify outbound payload uses A.
2. Choose model B; verify outbound payload switches to B.
3. Call endpoint without model; verify HTTP 400 + clear message.
4. Ensure SSE success/error rendering still works in UI.
5. Confirm `claude-sonnet-4-6` appears as second option in Claude model selector.

## Acceptance Criteria

- `claude-console` connectivity test uses user-selected model directly.
- No mapping applied in this test path.
- Missing model fails with 400.
- `claude-sonnet-4-6` appears in model list at position #2.

## Implementation Notes

Recommended implementation path:
1. Minimal targeted fix in route + service + model config.
2. No cross-platform endpoint normalization in this bugfix.
3. Follow-up work can standardize test endpoint contracts later.
