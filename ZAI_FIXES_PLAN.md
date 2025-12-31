# Zai Integration Fixes - Implementation Plan

## Overview

This plan addresses all in-scope issues identified during the Zai integration audit. Issues are organized by priority and file dependencies.

---

## Critical Priority (Blockers)

### 1. Fix SSE Line Buffering in zai-provider.ts

**File:** `apps/server/src/providers/zai-provider.ts`
**Lines:** 1195-1196

**Problem:** SSE chunks can be split mid-line. Current implementation:

```typescript
const chunk = decoder.decode(value, { stream: true });
const lines = chunk.split('\n');
```

**Solution:** Implement line buffering:

```typescript
let buffer = '';
// In the loop:
const chunk = decoder.decode(value, { stream: true });
buffer += chunk;
const lines = buffer.split('\n');
// Keep the last incomplete line in buffer
if (!chunk.endsWith('\n')) {
  buffer = lines.pop() || '';
} else {
  buffer = '';
}
```

**Estimated:** 15 minutes

---

### 2. Fix Potential Infinite Loop in Tool Call Streaming

**File:** `apps/server/src/providers/zai-provider.ts`
**Lines:** 1294-1306

**Problem:** If stream ends with incomplete tool calls, loop never exits.

**Solution:** Add safety check:

```typescript
if (finishReason === 'tool_calls' && currentToolCalls.size > 0) {
  const allComplete = Array.from(currentToolCalls.values()).every(
    (tc) => tc.id && tc.name && tc.arguments
  );
  if (allComplete) {
    break;
  }
}

// Add before loop iteration
if (done && finishReason === 'tool_calls') {
  const allComplete = Array.from(currentToolCalls.values()).every(
    (tc) => tc.id && tc.name && tc.arguments
  );
  if (!allComplete) {
    logger.warn('[Zai] Stream ended with incomplete tool calls');
    break;
  }
}
```

**Estimated:** 20 minutes

---

### 3. Fix Duplicate Model Definitions in available.ts

**File:** `apps/server/src/routes/models/routes/available.ts`

**Problem:** Models hardcoded instead of using `ProviderFactory.getAllAvailableModels()`

**Solution:** Refactor to use factory:

```typescript
import { ProviderFactory } from '../../providers/provider-factory.js';

export async function GET() {
  const models = ProviderFactory.getAllAvailableModels();
  return Response.json({ success: true, models });
}
```

**Estimated:** 30 minutes

---

### 4. Add verifyZaiAuth to SetupAPI Interface

**File:** `apps/ui/src/lib/electron.ts`
**Lines:** 1058-1130

**Problem:** Interface missing `verifyZaiAuth` method declaration.

**Solution:** Add to interface:

```typescript
interface SetupAPI {
  verifyClaudeAuth: (authMethod?: 'cli' | 'api_key') => Promise<{...}>;
  verifyZaiAuth: (apiKey?: string) => Promise<{
    success: boolean;
    authenticated: boolean;
    error?: string;
  }>;
  // ... rest of interface
}
```

**Estimated:** 10 minutes

---

## High Priority

### 5. Fix maxOutputTokens Inconsistency

**Files:** `apps/server/src/routes/models/routes/available.ts`, `apps/server/src/providers/zai-provider.ts`

**Problem:** `available.ts` has 4096, `zai-provider.ts` has 8192 for most models.

**Solution:** After #3 is complete, values will come from `zai-provider.ts` source of truth.

**Estimated:** Already fixed by #3

---

### 6. Add reasoning Content Block Handling

**File:** `apps/server/src/routes/setup/routes/verify-zai-auth.ts`
**Lines:** 86-100

**Problem:** Only processes `assistant` messages with `text` blocks, missing `reasoning` blocks.

**Solution:** Extend block processing:

```typescript
for (const block of message.content) {
  if (block.type === 'text' && block.text) {
    fullContent += block.text;
  } else if (block.type === 'reasoning' && block.reasoning_content) {
    // Check reasoning content for errors too
    fullContent += block.reasoning_content;
  }
}
```

**Estimated:** 20 minutes

---

### 7. Add Path Validation to describeImages()

**File:** `apps/server/src/providers/zai-provider.ts`
**Lines:** 1480-1504

**Problem:** No validation of image sources before sending to API.

**Solution:** Add validation:

```typescript
private async describeImages(
  images: Array<{ type: string; source?: object }>,
  originalPrompt: string
): Promise<string> {
  for (const img of images) {
    if (img.source && typeof img.source === 'object') {
      const source = img.source as { type?: string; data?: string };
      if (source.data && source.data.startsWith('file://')) {
        throw new Error('file:// URLs not supported in image sources');
      }
    }
  }
  // ... rest of method
}
```

**Estimated:** 15 minutes

---

### 8. Add Delete API Key Functionality for Zai

**Files:**

- `apps/ui/src/components/views/settings-view/api-keys/api-keys-section.tsx`
- `apps/ui/src/components/views/setup-view/steps/zai-setup-step.tsx`

**Problem:** Zai has no delete button (only Anthropic does).

**Solution:**

1. Add `deleteZaiKey` function in `api-keys-section.tsx`
2. Add delete button with `Trash2` icon in both places
3. Wire up to `api.setup.deleteApiKey('zai')`

**Estimated:** 45 minutes

---

### 9. Add Icon to Unverified Status Badge

**File:** `apps/ui/src/components/views/setup-view/steps/zai-setup-step.tsx`
**Line:** 131

**Problem:** Badge has no icon while other status badges do.

**Solution:** Add `AlertCircle` icon:

```typescript
import { AlertCircle } from 'lucide-react';

// In badge:
<div className="inline-flex items-center gap-1.5...">
  <AlertCircle className="w-3.5 h-3.5" />
  Unverified
</div>
```

**Estimated:** 10 minutes

---

### 10. Enable Skipped Integration Tests

**File:** `apps/server/tests/unit/lib/provider-query.test.ts`
**Lines:** 245-256

**Problem:** 3 integration tests skipped with TODO comment.

**Solution:** Fix mocking and enable tests for:

- Provider routing for glm models
- Provider routing for claude models
- Structured output prompt injection

**Estimated:** 2 hours

---

### 11. Add Image/Vision Handling Tests

**File:** `apps/server/tests/unit/providers/zai-provider.test.ts`

**Problem:** No tests for `describeImages()`, image fallback, `modelSupportsVision()`.

**Solution:** Add test suite:

```typescript
describe('image handling', () => {
  it('should describe images using GLM-4.6v for non-vision models');
  it('should format image content for GLM-4.6v');
  it('should handle image description failures gracefully');
  it('should detect vision support correctly');
});
```

**Estimated:** 2 hours

---

## Medium Priority

### 12. Fix Incomplete Tool Call Data Handling

**File:** `apps/server/src/providers/zai-provider.ts`
**Lines:** 1334-1342

**Problem:** If JSON.parse fails, error is yielded but execution continues with invalid tool call.

**Solution:** Skip tool call on parse error:

```typescript
try {
  toolArgs = JSON.parse(toolCall.arguments);
} catch {
  yield {
    type: 'error',
    error: `Invalid tool arguments: ${toolCall.arguments}`,
  };
  continue; // Skip this tool call entirely
}
```

**Estimated:** 15 minutes

---

### 13. Refactor Hardcoded Provider Lists

**Files:**

- `apps/server/src/routes/setup/routes/store-api-key.ts`
- `apps/server/src/routes/setup/routes/delete-api-key.ts`

**Problem:** Hardcoded switch statements for provider names.

**Solution:** Create provider registry:

```typescript
// In common.ts or new file
const SUPPORTED_PROVIDERS = new Set(['anthropic', 'anthropic_oauth_token', 'zai']);
const PROVIDER_ENV_KEYS = {
  anthropic: 'ANTHROPIC_API_KEY',
  zai: 'ZAI_API_KEY',
  // ...
};
```

**Estimated:** 45 minutes

---

### 14. Clarify extendedThinking vs thinking Feature Names

**Files:**

- `apps/server/src/providers/types.ts`
- `apps/server/src/providers/base-provider.ts`
- `apps/server/src/providers/zai-provider.ts`

**Problem:** Ambiguous whether `extendedThinking` means Claude's extended thinking or Zai's thinking mode.

**Solution:** Add clear JSDoc or consider renaming:

```typescript
export type ProviderFeature =
  | 'tools'
  | 'text'
  | 'vision'
  | 'mcp'
  | 'browser'
  | 'extendedThinking' // Claude: extended thinking, Zai: thinking mode
  | 'structuredOutput';
```

Or use discriminated approach:

```typescript
// Add to ModelDefinition
thinkingSupport?: {
  claudeExtended?: boolean;
  zaiMode?: boolean;
};
```

**Estimated:** 30 minutes

---

### 15. Fix API Key Detection Duplication

**File:** `apps/server/src/lib/provider-query.ts`
**Lines:** 179-191

**Problem:** Duplicates factory's model→provider detection logic.

**Solution:** Use factory result:

```typescript
const provider = ProviderFactory.getProviderForModel(resolvedModel);
const providerName = provider.getName();

const apiKeyMap: Record<string, string | undefined> = {
  claude: apiKeys.anthropic,
  zai: apiKeys.zai,
  google: apiKeys.google,
  openai: apiKeys.openai,
};

const effectiveApiKey = apiKey || apiKeyMap[providerName];
```

**Estimated:** 20 minutes

---

### 16. Fix Asymmetric Model Equivalence

**File:** `libs/types/src/model.ts`

**Problem:** GLM-4.6v maps to Sonnet, but Sonnet maps to GLM-4.6 (not GLM-4.6v). Vision capability lost.

**Solution:** Document or create separate mappings:

```typescript
// Option 1: Document the limitation
// GLM-4.6v (vision) maps to Sonnet, but reverse loses vision
// Option 2: Create vision-aware equivalence
const MODEL_EQUIVALENCE_WITH_VISION = {
  // ... separate mapping for vision use cases
};
```

**Estimated:** 20 minutes

---

### 17. Add Credential Loading to AgentService

**File:** `apps/server/src/services/agent-service.ts`

**Problem:** AutoModeService loads credentials explicitly, AgentService doesn't.

**Solution:** Add similar pattern:

```typescript
// Load credentials from SettingsService
let apiKey: string | undefined;
if (this.settingsService) {
  const credentials = await this.settingsService.getCredentials();
  const lowerModel = model.toLowerCase();

  if (lowerModel.startsWith('glm-') || lowerModel === 'glm') {
    apiKey = credentials.apiKeys?.zai;
  } else if (/* ... */) {
    apiKey = credentials.apiKeys?.anthropic;
  }
}

const provider = ProviderFactory.getProviderForModel(
  model,
  apiKey ? { apiKey } : undefined
);
```

**Estimated:** 30 minutes

---

## Implementation Order (Dependencies)

```
Phase 1 - Critical (Day 1)
├── 1. SSE line buffering (15 min)
├── 2. Infinite loop fix (20 min)
├── 4. SetupAPI interface (10 min)
└── 3. Duplicate model defs (30 min) - fixes #5 automatically

Phase 2 - High Priority (Day 1-2)
├── 6. Reasoning content block (20 min)
├── 7. Path validation in describeImages (15 min)
├── 9. Unverified badge icon (10 min)
├── 8. Delete API key for Zai (45 min)
├── 15. API key detection refactor (20 min)
└── 13. Hardcoded provider lists (45 min)

Phase 3 - High Priority Tests (Day 2-3)
├── 10. Skipped integration tests (2 hours)
└── 11. Image/vision tests (2 hours)

Phase 4 - Medium Priority (Day 3-4)
├── 12. Tool call data handling (15 min)
├── 14. Feature naming clarity (30 min)
├── 16. Model equivalence (20 min)
└── 17. AgentService credentials (30 min)
```

---

## Total Time Estimate

| Priority  | Tasks  | Time         |
| --------- | ------ | ------------ |
| Critical  | 4      | ~75 min      |
| High      | 7      | ~4.5 hours   |
| Medium    | 6      | ~2.5 hours   |
| **Total** | **17** | **~8 hours** |

---

## Testing Checklist

After each fix:

- [ ] Run existing tests: `npm test`
- [ ] Check TypeScript compilation: `npx tsc --noEmit`
- [ ] Manual smoke test of affected functionality
- [ ] Review related files for unintended side effects

---

## Notes

1. **Critical issues should block merge** - They can cause runtime failures
2. **High priority issues affect UX or feature parity** - Should be fixed before release
3. **Medium priority are code quality/maintainability** - Can be addressed in follow-up
