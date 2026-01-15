# Ollama Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Enable local Ollama models in the Openwork desktop app alongside existing cloud providers.

**Architecture:** Add `'ollama'` as a provider type with dynamic model discovery via Ollama API. Store server URL in electron-store (not keychain). Pass `OLLAMA_HOST` env var to OpenCode CLI when using Ollama models.

**Tech Stack:** TypeScript, Electron IPC, React, electron-store, fetch API

---

## Task 1: Update Provider Types

**Files:**
- Modify: `packages/shared/src/types/provider.ts:1-30`

**Step 1: Update ProviderType to include 'ollama'**

```typescript
// Line 5: Change 'local' to 'ollama'
export type ProviderType = 'anthropic' | 'openai' | 'google' | 'ollama' | 'custom';
```

**Step 2: Add OllamaConfig interface after SelectedModel (around line 30)**

```typescript
/**
 * Ollama server configuration
 */
export interface OllamaConfig {
  baseUrl: string;
  enabled: boolean;
  lastValidated?: number;
}
```

**Step 3: Update SelectedModel interface to include optional baseUrl**

```typescript
export interface SelectedModel {
  provider: ProviderType;
  model: string;
  baseUrl?: string;  // For Ollama: the server URL
}
```

**Step 4: Remove the placeholder 'local' provider from DEFAULT_PROVIDERS**

Delete lines 107-120 (the `id: 'local'` provider block).

**Step 5: Verify types compile**

Run: `pnpm -F @accomplish/shared build`
Expected: Build succeeds

**Step 6: Commit**

```bash
git add packages/shared/src/types/provider.ts
git commit -m "feat(types): add ollama provider type and OllamaConfig interface"
```

---

## Task 2: Add Ollama Config Storage

**Files:**
- Modify: `apps/desktop/src/main/store/appSettings.ts:1-88`

**Step 1: Import OllamaConfig type**

Add to the import statement at line 2:

```typescript
import type { SelectedModel, OllamaConfig } from '@accomplish/shared';
```

**Step 2: Update AppSettingsSchema interface (around line 7-14)**

```typescript
interface AppSettingsSchema {
  debugMode: boolean;
  onboardingComplete: boolean;
  selectedModel: SelectedModel | null;
  ollamaConfig: OllamaConfig | null;
}
```

**Step 3: Update store defaults (around line 16-26)**

```typescript
const appSettingsStore = new Store<AppSettingsSchema>({
  name: 'app-settings',
  defaults: {
    debugMode: false,
    onboardingComplete: false,
    selectedModel: {
      provider: 'anthropic',
      model: 'anthropic/claude-opus-4-5',
    },
    ollamaConfig: null,
  },
});
```

**Step 4: Add getter/setter functions at the end of file (before clearAppSettings)**

```typescript
/**
 * Get Ollama configuration
 */
export function getOllamaConfig(): OllamaConfig | null {
  return appSettingsStore.get('ollamaConfig');
}

/**
 * Set Ollama configuration
 */
export function setOllamaConfig(config: OllamaConfig | null): void {
  appSettingsStore.set('ollamaConfig', config);
}
```

**Step 5: Update getAppSettings to include ollamaConfig**

```typescript
export function getAppSettings(): AppSettingsSchema {
  return {
    debugMode: appSettingsStore.get('debugMode'),
    onboardingComplete: appSettingsStore.get('onboardingComplete'),
    selectedModel: appSettingsStore.get('selectedModel'),
    ollamaConfig: appSettingsStore.get('ollamaConfig'),
  };
}
```

**Step 6: Verify build**

Run: `pnpm -F @accomplish/desktop build`
Expected: Build succeeds

**Step 7: Commit**

```bash
git add apps/desktop/src/main/store/appSettings.ts
git commit -m "feat(storage): add ollama config persistence"
```

---

## Task 3: Add Ollama IPC Handlers

**Files:**
- Modify: `apps/desktop/src/main/ipc/handlers.ts:1-50` (imports)
- Modify: `apps/desktop/src/main/ipc/handlers.ts:850-880` (add new handlers)

**Step 1: Add imports for Ollama config functions (around line 38)**

```typescript
import {
  getDebugMode,
  setDebugMode,
  getAppSettings,
  getOnboardingComplete,
  setOnboardingComplete,
  getSelectedModel,
  setSelectedModel,
  getOllamaConfig,
  setOllamaConfig,
} from '../store/appSettings';
```

**Step 2: Add OllamaConfig to type imports (around line 54)**

```typescript
import type {
  TaskConfig,
  PermissionResponse,
  OpenCodeMessage,
  TaskMessage,
  TaskResult,
  TaskStatus,
  SelectedModel,
  OllamaConfig,
} from '@accomplish/shared';
```

**Step 3: Define Ollama model interface after ALLOWED_API_KEY_PROVIDERS (around line 67)**

```typescript
interface OllamaModel {
  id: string;
  displayName: string;
  size: number;
}
```

**Step 4: Add Ollama handlers after the model:set handler (around line 865)**

```typescript
  // Ollama: Test connection and get models
  handle('ollama:test-connection', async (_event: IpcMainInvokeEvent, url: string) => {
    const sanitizedUrl = sanitizeString(url, 'ollamaUrl', 256);

    try {
      const response = await fetchWithTimeout(
        `${sanitizedUrl}/api/tags`,
        { method: 'GET' },
        API_KEY_VALIDATION_TIMEOUT_MS
      );

      if (!response.ok) {
        throw new Error(`Ollama returned status ${response.status}`);
      }

      const data = await response.json() as { models?: Array<{ name: string; size: number }> };
      const models: OllamaModel[] = (data.models || []).map((m) => ({
        id: m.name,
        displayName: m.name,
        size: m.size,
      }));

      console.log(`[Ollama] Connection successful, found ${models.length} models`);
      return { success: true, models };
    } catch (error) {
      const message = error instanceof Error ? error.message : 'Connection failed';
      console.warn('[Ollama] Connection failed:', message);

      if (error instanceof Error && error.name === 'AbortError') {
        return { success: false, error: 'Connection timed out. Make sure Ollama is running.' };
      }
      return { success: false, error: `Cannot connect to Ollama: ${message}` };
    }
  });

  // Ollama: Get stored config
  handle('ollama:get-config', async (_event: IpcMainInvokeEvent) => {
    return getOllamaConfig();
  });

  // Ollama: Set config
  handle('ollama:set-config', async (_event: IpcMainInvokeEvent, config: OllamaConfig | null) => {
    if (config !== null) {
      if (typeof config.baseUrl !== 'string' || typeof config.enabled !== 'boolean') {
        throw new Error('Invalid Ollama configuration');
      }
    }
    setOllamaConfig(config);
    console.log('[Ollama] Config saved:', config);
  });
```

**Step 5: Verify build**

Run: `pnpm -F @accomplish/desktop build`
Expected: Build succeeds

**Step 6: Commit**

```bash
git add apps/desktop/src/main/ipc/handlers.ts
git commit -m "feat(ipc): add ollama connection test and config handlers"
```

---

## Task 4: Update Preload Script

**Files:**
- Modify: `apps/desktop/src/preload/index.ts:88-99`

**Step 1: Add Ollama methods to accomplishAPI object (after getAllApiKeys, around line 98)**

```typescript
  // Ollama configuration
  testOllamaConnection: (url: string): Promise<{
    success: boolean;
    models?: Array<{ id: string; displayName: string; size: number }>;
    error?: string;
  }> => ipcRenderer.invoke('ollama:test-connection', url),

  getOllamaConfig: (): Promise<{ baseUrl: string; enabled: boolean } | null> =>
    ipcRenderer.invoke('ollama:get-config'),

  setOllamaConfig: (config: { baseUrl: string; enabled: boolean } | null): Promise<void> =>
    ipcRenderer.invoke('ollama:set-config', config),
```

**Step 2: Verify build**

Run: `pnpm -F @accomplish/desktop build`
Expected: Build succeeds

**Step 3: Commit**

```bash
git add apps/desktop/src/preload/index.ts
git commit -m "feat(preload): expose ollama IPC methods to renderer"
```

---

## Task 5: Update OpenCode Adapter for Ollama

**Files:**
- Modify: `apps/desktop/src/main/opencode/adapter.ts:330-400` (buildEnvironment)
- Modify: `apps/desktop/src/main/opencode/adapter.ts:399-424` (buildCliArgs)

**Step 1: Import getOllamaConfig (around line 12)**

```typescript
import { getSelectedModel, getOllamaConfig } from '../store/appSettings';
```

**Step 2: Update buildEnvironment to set OLLAMA_HOST (around line 380, after groq key)**

```typescript
    if (apiKeys.groq) {
      env.GROQ_API_KEY = apiKeys.groq;
      console.log('[OpenCode CLI] Using Groq API key from settings');
    }

    // Set Ollama host if configured
    const selectedModel = getSelectedModel();
    if (selectedModel?.provider === 'ollama' && selectedModel.baseUrl) {
      env.OLLAMA_HOST = selectedModel.baseUrl;
      console.log('[OpenCode CLI] Using Ollama host:', selectedModel.baseUrl);
    }
```

**Step 3: Verify build**

Run: `pnpm -F @accomplish/desktop build`
Expected: Build succeeds

**Step 4: Commit**

```bash
git add apps/desktop/src/main/opencode/adapter.ts
git commit -m "feat(adapter): set OLLAMA_HOST env var for local models"
```

---

## Task 6: Update OpenCode Config Generator

**Files:**
- Modify: `apps/desktop/src/main/opencode/config-generator.ts:366-395`

**Step 1: Import getOllamaConfig (around line 5)**

```typescript
import { PERMISSION_API_PORT } from '../permission-api';
import { getOllamaConfig } from '../store/appSettings';
```

**Step 2: Update enabled_providers to conditionally include ollama (around line 370)**

Replace:
```typescript
    enabled_providers: ['anthropic', 'openai', 'google', 'groq'],
```

With:
```typescript
    // Enable providers - add ollama if configured
    const ollamaConfig = getOllamaConfig();
    const baseProviders = ['anthropic', 'openai', 'google', 'groq'];
    const enabledProviders = ollamaConfig?.enabled
      ? [...baseProviders, 'ollama']
      : baseProviders;
```

And update the config object:
```typescript
    enabled_providers: enabledProviders,
```

**Step 3: Verify build**

Run: `pnpm -F @accomplish/desktop build`
Expected: Build succeeds

**Step 4: Commit**

```bash
git add apps/desktop/src/main/opencode/config-generator.ts
git commit -m "feat(config): enable ollama provider when configured"
```

---

## Task 7: Update Settings Dialog UI - Add Tabs

**Files:**
- Modify: `apps/desktop/src/renderer/components/layout/SettingsDialog.tsx`

**Step 1: Add tab state and Ollama state variables (around line 49)**

```typescript
  const [activeTab, setActiveTab] = useState<'cloud' | 'local'>('cloud');
  const [ollamaUrl, setOllamaUrl] = useState('http://localhost:11434');
  const [ollamaModels, setOllamaModels] = useState<Array<{ id: string; displayName: string; size: number }>>([]);
  const [ollamaConnected, setOllamaConnected] = useState(false);
  const [ollamaError, setOllamaError] = useState<string | null>(null);
  const [testingOllama, setTestingOllama] = useState(false);
  const [selectedOllamaModel, setSelectedOllamaModel] = useState<string>('');
  const [savingOllama, setSavingOllama] = useState(false);
```

**Step 2: Add useEffect to load Ollama config (inside the existing useEffect, around line 102)**

```typescript
    const fetchOllamaConfig = async () => {
      try {
        const config = await accomplish.getOllamaConfig();
        if (config) {
          setOllamaUrl(config.baseUrl);
          // Auto-test connection if previously configured
          if (config.enabled) {
            const result = await accomplish.testOllamaConnection(config.baseUrl);
            if (result.success && result.models) {
              setOllamaConnected(true);
              setOllamaModels(result.models);
            }
          }
        }
      } catch (err) {
        console.error('Failed to fetch Ollama config:', err);
      }
    };

    fetchOllamaConfig();
```

**Step 3: Add handler functions after handleDeleteApiKey (around line 195)**

```typescript
  const handleTestOllama = async () => {
    const accomplish = getAccomplish();
    setTestingOllama(true);
    setOllamaError(null);
    setOllamaConnected(false);
    setOllamaModels([]);

    try {
      const result = await accomplish.testOllamaConnection(ollamaUrl);
      if (result.success && result.models) {
        setOllamaConnected(true);
        setOllamaModels(result.models);
        if (result.models.length > 0) {
          setSelectedOllamaModel(result.models[0].id);
        }
      } else {
        setOllamaError(result.error || 'Connection failed');
      }
    } catch (err) {
      setOllamaError(err instanceof Error ? err.message : 'Connection failed');
    } finally {
      setTestingOllama(false);
    }
  };

  const handleSaveOllama = async () => {
    const accomplish = getAccomplish();
    setSavingOllama(true);

    try {
      // Save the Ollama config
      await accomplish.setOllamaConfig({
        baseUrl: ollamaUrl,
        enabled: true,
      });

      // Set as selected model
      await accomplish.setSelectedModel({
        provider: 'ollama',
        model: `ollama/${selectedOllamaModel}`,
        baseUrl: ollamaUrl,
      });

      setSelectedModel({
        provider: 'ollama',
        model: `ollama/${selectedOllamaModel}`,
        baseUrl: ollamaUrl,
      });

      setModelStatusMessage(`Model updated to ${selectedOllamaModel}`);
    } catch (err) {
      setOllamaError(err instanceof Error ? err.message : 'Failed to save');
    } finally {
      setSavingOllama(false);
    }
  };

  const formatBytes = (bytes: number): string => {
    const gb = bytes / (1024 * 1024 * 1024);
    return `${gb.toFixed(1)} GB`;
  };
```

**Step 4: Update the Model Selection section JSX to include tabs (replace lines 204-247)**

```tsx
          {/* Model Selection Section */}
          <section>
            <h2 className="mb-4 text-base font-medium text-foreground">Model</h2>
            <div className="rounded-lg border border-border bg-card p-5">
              {/* Tabs */}
              <div className="flex gap-2 mb-5">
                <button
                  onClick={() => setActiveTab('cloud')}
                  className={`px-4 py-2 rounded-lg text-sm font-medium transition-colors ${
                    activeTab === 'cloud'
                      ? 'bg-primary text-primary-foreground'
                      : 'bg-muted text-muted-foreground hover:text-foreground'
                  }`}
                >
                  Cloud Providers
                </button>
                <button
                  onClick={() => setActiveTab('local')}
                  className={`px-4 py-2 rounded-lg text-sm font-medium transition-colors ${
                    activeTab === 'local'
                      ? 'bg-primary text-primary-foreground'
                      : 'bg-muted text-muted-foreground hover:text-foreground'
                  }`}
                >
                  Local Models
                </button>
              </div>

              {activeTab === 'cloud' ? (
                <>
                  <p className="mb-4 text-sm text-muted-foreground leading-relaxed">
                    Select a cloud AI model. Requires an API key for the provider.
                  </p>
                  {loadingModel ? (
                    <div className="h-10 animate-pulse rounded-md bg-muted" />
                  ) : (
                    <select
                      value={selectedModel?.provider !== 'ollama' ? selectedModel?.model || '' : ''}
                      onChange={(e) => handleModelChange(e.target.value)}
                      className="w-full rounded-md border border-input bg-background px-3 py-2 text-sm"
                    >
                      <option value="" disabled>Select a model...</option>
                      {DEFAULT_PROVIDERS.filter((p) => p.requiresApiKey).map((provider) => {
                        const hasApiKey = savedKeys.some((k) => k.provider === provider.id);
                        return (
                          <optgroup key={provider.id} label={provider.name}>
                            {provider.models.map((model) => (
                              <option
                                key={model.fullId}
                                value={model.fullId}
                                disabled={!hasApiKey}
                              >
                                {model.displayName}{!hasApiKey ? ' (No API key)' : ''}
                              </option>
                            ))}
                          </optgroup>
                        );
                      })}
                    </select>
                  )}
                  {modelStatusMessage && (
                    <p className="mt-3 text-sm text-success">{modelStatusMessage}</p>
                  )}
                  {selectedModel && selectedModel.provider !== 'ollama' && !savedKeys.some((k) => k.provider === selectedModel.provider) && (
                    <p className="mt-3 text-sm text-warning">
                      No API key configured for {DEFAULT_PROVIDERS.find((p) => p.id === selectedModel.provider)?.name}. Add one below.
                    </p>
                  )}
                </>
              ) : (
                <>
                  <p className="mb-4 text-sm text-muted-foreground leading-relaxed">
                    Connect to a local Ollama server to use models running on your machine.
                  </p>

                  {/* Ollama URL Input */}
                  <div className="mb-4">
                    <label className="mb-2 block text-sm font-medium text-foreground">
                      Ollama Server URL
                    </label>
                    <div className="flex gap-2">
                      <input
                        type="text"
                        value={ollamaUrl}
                        onChange={(e) => {
                          setOllamaUrl(e.target.value);
                          setOllamaConnected(false);
                          setOllamaModels([]);
                        }}
                        placeholder="http://localhost:11434"
                        className="flex-1 rounded-md border border-input bg-background px-3 py-2 text-sm"
                      />
                      <button
                        onClick={handleTestOllama}
                        disabled={testingOllama}
                        className="rounded-md bg-muted px-4 py-2 text-sm font-medium hover:bg-muted/80 disabled:opacity-50"
                      >
                        {testingOllama ? 'Testing...' : 'Test'}
                      </button>
                    </div>
                  </div>

                  {/* Connection Status */}
                  {ollamaConnected && (
                    <div className="mb-4 flex items-center gap-2 text-sm text-success">
                      <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M5 13l4 4L19 7" />
                      </svg>
                      Connected - {ollamaModels.length} model{ollamaModels.length !== 1 ? 's' : ''} available
                    </div>
                  )}

                  {ollamaError && (
                    <div className="mb-4 flex items-center gap-2 text-sm text-destructive">
                      <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
                      </svg>
                      {ollamaError}
                    </div>
                  )}

                  {/* Model Selection (only show when connected) */}
                  {ollamaConnected && ollamaModels.length > 0 && (
                    <div className="mb-4">
                      <label className="mb-2 block text-sm font-medium text-foreground">
                        Select Model
                      </label>
                      <select
                        value={selectedOllamaModel}
                        onChange={(e) => setSelectedOllamaModel(e.target.value)}
                        className="w-full rounded-md border border-input bg-background px-3 py-2 text-sm"
                      >
                        {ollamaModels.map((model) => (
                          <option key={model.id} value={model.id}>
                            {model.displayName} ({formatBytes(model.size)})
                          </option>
                        ))}
                      </select>
                    </div>
                  )}

                  {/* Save Button */}
                  {ollamaConnected && selectedOllamaModel && (
                    <button
                      onClick={handleSaveOllama}
                      disabled={savingOllama}
                      className="w-full rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground hover:bg-primary/90 disabled:opacity-50"
                    >
                      {savingOllama ? 'Saving...' : 'Use This Model'}
                    </button>
                  )}

                  {/* Help text when not connected */}
                  {!ollamaConnected && !ollamaError && (
                    <p className="text-sm text-muted-foreground">
                      Make sure{' '}
                      <a
                        href="https://ollama.ai"
                        target="_blank"
                        rel="noopener noreferrer"
                        className="text-primary hover:underline"
                      >
                        Ollama
                      </a>{' '}
                      is installed and running, then click Test to connect.
                    </p>
                  )}

                  {/* Current Ollama selection indicator */}
                  {selectedModel?.provider === 'ollama' && (
                    <div className="mt-4 rounded-lg bg-muted p-3">
                      <p className="text-sm text-foreground">
                        <span className="font-medium">Currently using:</span>{' '}
                        {selectedModel.model.replace('ollama/', '')}
                      </p>
                    </div>
                  )}
                </>
              )}
            </div>
          </section>
```

**Step 5: Verify build**

Run: `pnpm -F @accomplish/desktop build`
Expected: Build succeeds

**Step 6: Test manually**

Run: `pnpm dev`
Expected: Settings dialog shows Cloud/Local tabs, Ollama config works

**Step 7: Commit**

```bash
git add apps/desktop/src/renderer/components/layout/SettingsDialog.tsx
git commit -m "feat(ui): add ollama configuration with tab-based settings"
```

---

## Task 8: Update Accomplish Library Types

**Files:**
- Modify: `apps/desktop/src/renderer/lib/accomplish.ts`

**Step 1: Read the file to understand current structure**

**Step 2: Add Ollama method types to the AccomplishAPI interface**

The methods should already work via the preload script, but we need to ensure TypeScript types are correct.

**Step 3: Verify build**

Run: `pnpm -F @accomplish/desktop build`
Expected: Build succeeds

**Step 4: Commit**

```bash
git add apps/desktop/src/renderer/lib/accomplish.ts
git commit -m "feat(lib): add ollama method types to accomplish API"
```

---

## Task 9: Export OllamaConfig from Shared Package

**Files:**
- Modify: `packages/shared/src/index.ts`

**Step 1: Add OllamaConfig to exports**

```typescript
export type { OllamaConfig } from './types/provider';
```

**Step 2: Verify build**

Run: `pnpm build`
Expected: All packages build successfully

**Step 3: Commit**

```bash
git add packages/shared/src/index.ts
git commit -m "feat(shared): export OllamaConfig type"
```

---

## Task 10: Full Integration Test

**Step 1: Build all packages**

Run: `pnpm build`
Expected: All packages build successfully

**Step 2: Run type checking**

Run: `pnpm typecheck`
Expected: No type errors

**Step 3: Run linting**

Run: `pnpm lint`
Expected: No lint errors

**Step 4: Manual testing**

Run: `pnpm dev`

Test flow:
1. Open Settings
2. Click "Local Models" tab
3. Enter Ollama URL (default localhost:11434)
4. Click "Test" - should show models if Ollama running
5. Select a model
6. Click "Use This Model"
7. Verify model shows as selected
8. Start a task - should use Ollama model

**Step 5: Final commit**

```bash
git add -A
git commit -m "feat: complete ollama integration with local model support

- Add ollama provider type and OllamaConfig interface
- Add IPC handlers for connection testing and config storage
- Update OpenCode adapter to set OLLAMA_HOST env var
- Add tab-based Settings UI for cloud/local model selection
- Dynamic model discovery via Ollama API

Closes #ollama-support"
```

---

## Summary

**Total tasks:** 10
**Files modified:** 8
- `packages/shared/src/types/provider.ts`
- `packages/shared/src/index.ts`
- `apps/desktop/src/main/store/appSettings.ts`
- `apps/desktop/src/main/ipc/handlers.ts`
- `apps/desktop/src/preload/index.ts`
- `apps/desktop/src/main/opencode/adapter.ts`
- `apps/desktop/src/main/opencode/config-generator.ts`
- `apps/desktop/src/renderer/components/layout/SettingsDialog.tsx`

**Files created:** 0

**Commits:** 10 small commits following conventional commit format
