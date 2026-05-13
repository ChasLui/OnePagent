# Model Reasoning Content Return Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-model OpenAI-compatible `reasoning_content` preservation and multi-turn return settings, with DeepSeek and MiMo effective On by default.

**Architecture:** Keep the feature inside `onepagent.html`, matching the existing provider-profile and Model Context Lengths patterns. Add one model-bound settings map (`reasoning_content_models`), runtime helpers for the effective policy, Settings UI rows, and replace all reasoning-path DeepSeek checks with the new helper.

**Tech Stack:** Plain browser JavaScript, single-file HTML/CSS UI, localStorage provider profiles, manual browser validation.

---

## File structure

- Modify: `onepagent.html`
  - Settings modal HTML near `setModelContextsList`: add the “Reasoning Content Return” section.
  - i18n maps: add English and Chinese labels for the new Settings section.
  - Provider/settings initialization near `MODEL_CONTEXTS`: add `MODEL_REASONING_CONTENT` and helper functions.
  - Provider switching and settings save/load: include `reasoning_content_models` in provider profiles and runtime `_keys`.
  - Settings UI helpers near Model Context Length helpers: add row rendering/import/add/read functions for reasoning-content modes.
  - Request assembly and stream parsing: use `shouldKeepReasoningForModel(model)` instead of `isDeepSeekActiveProvider()`.
- No automated test files exist for this single-file app. Validation is static grep plus browser/manual request-path checks.

---

### Task 1: Add Settings UI labels and markup

**Files:**
- Modify: `onepagent.html` around the Settings modal, near lines 1627-1639.
- Modify: `onepagent.html` i18n English map near lines 2196-2200.
- Modify: `onepagent.html` i18n Chinese map near lines 2602-2607.

- [ ] **Step 1: Add the Settings modal section**

Insert this block immediately after the Model Context Lengths add row and before the Tavily API Key label:

```html
      <div style="border-top:1px solid var(--border);margin:12px 0 10px"></div>
      <label style="display:flex;align-items:center;justify-content:space-between">
        <span><span data-i18n="settings.reasoningContent">Reasoning Content Return</span> <span style="text-transform:none;color:var(--text-dim)" data-i18n="settings.reasoningContentHint">(OpenAI-compatible reasoning_content)</span></span>
        <button type="button" class="modal-btn secondary" onclick="syncReasoningContentFromSelected()" style="padding:2px 10px;font-size:10px">&#x21A1; <span data-i18n="settings.rcImport">Import Selected</span></button>
      </label>
      <div id="setReasoningContentList" class="rc-list"></div>
      <div id="setReasoningContentEmpty" class="rc-empty" data-i18n="settings.rcEmpty">No per-model reasoning settings. Auto keeps DeepSeek and MiMo on, others off.</div>
      <div class="rc-add-row">
        <input type="text" id="setRcNewModel" placeholder="model id" data-i18n-placeholder="settings.mcModelPh" list="setRcModelList" onkeydown="if(event.key==='Enter'){event.preventDefault();addReasoningContentModelFromInputs();}">
        <datalist id="setRcModelList"></datalist>
        <select id="setRcNewMode" style="width:92px;padding:6px 8px;background:var(--bg-root);border:1px solid var(--border);border-radius:5px;color:var(--text-primary);font-family:'JetBrains Mono',monospace;font-size:11px;margin-bottom:0">
          <option value="auto">Auto</option>
          <option value="on">On</option>
          <option value="off">Off</option>
        </select>
        <button type="button" class="modal-btn secondary" onclick="addReasoningContentModelFromInputs()" style="padding:4px 10px;font-size:10px"><svg class="ui-icon" aria-hidden="true"><use href="#i-plus"></use></svg> <span data-i18n="settings.addModel">Add</span></button>
      </div>
```

Expected: the new section appears between Model Context Lengths and Tavily API Key.

- [ ] **Step 2: Add compact CSS for reasoning rows**

Insert this CSS block immediately after the existing Model-context list CSS block:

```css
/* Reasoning-content return list (Settings modal) */
.rc-list { display: flex; flex-direction: column; gap: 4px; max-height: 180px; overflow-y: auto; margin-bottom: 6px; padding: 4px; border: 1px solid var(--border); border-radius: 5px; background: var(--bg-root); }
.rc-list:empty { display: none; }
.rc-empty { font-size: 10px; color: var(--text-dim); padding: 10px; text-align: center; border: 1px dashed var(--border); border-radius: 5px; margin-bottom: 6px; background: var(--bg-root); }
.rc-row { display: flex; align-items: center; gap: 6px; padding: 4px 6px; border-radius: 4px; background: var(--bg-panel); border: 1px solid var(--border); transition: border-color 0.15s; }
.rc-row:hover { border-color: var(--border-active); }
.rc-row .rc-model { flex: 1; min-width: 0; font-family: 'JetBrains Mono', monospace; font-size: 11px; color: var(--text-primary); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rc-row .rc-mode { width: 92px; flex-shrink: 0; padding: 4px 6px !important; margin: 0 !important; font-family: 'JetBrains Mono', monospace; font-size: 11px; background: var(--bg-root); border: 1px solid var(--border); border-radius: 4px; color: var(--text-primary); }
.rc-row .rc-effective { width: 78px; flex-shrink: 0; font-size: 10px; color: var(--text-dim); text-align: right; }
.rc-remove { padding: 0 6px; font-size: 14px; line-height: 1; background: transparent; color: var(--text-dim); border: none; cursor: pointer; border-radius: 3px; flex-shrink: 0; }
.rc-remove:hover { color: var(--accent-red); background: rgba(255,68,102,0.1); }
.rc-add-row { display: flex; gap: 6px; align-items: center; margin-bottom: 12px; }
.rc-add-row > input { margin: 0 !important; padding: 6px 8px !important; font-size: 11px; font-family: 'JetBrains Mono', monospace; }
.rc-add-row > input[type=text] { flex: 1; min-width: 0; }
```

Expected: new rows use the same compact visual style as the existing model context rows.

- [ ] **Step 3: Add English i18n keys**

In the English settings i18n map, after `settings.mcModelPh`, add:

```js
    'settings.reasoningContent': 'Reasoning Content Return',
    'settings.reasoningContentHint': '(OpenAI-compatible reasoning_content)',
    'settings.rcImport': 'Import Selected',
    'settings.rcEmpty': 'No per-model reasoning settings. Auto keeps DeepSeek and MiMo on, others off.',
```

Expected: the English Settings modal has localized labels for the new section.

- [ ] **Step 4: Add Chinese i18n keys**

In the Chinese settings i18n map, after `settings.mcModelPh`, add:

```js
    'settings.reasoningContent': '思考内容回传',
    'settings.reasoningContentHint': '（OpenAI-compatible reasoning_content）',
    'settings.rcImport': '导入已选模型',
    'settings.rcEmpty': '尚未设置模型思考内容回传。Auto 默认 DeepSeek 和 MiMo 开启，其他关闭。',
```

Expected: the Chinese Settings modal has localized labels for the new section.

- [ ] **Step 5: Smoke-check UI strings**

Run:

```bash
grep -n "settings.reasoningContent\|setReasoningContentList\|rc-list" onepagent.html
```

Expected: matches cover CSS, Settings markup, English i18n, and Chinese i18n.

- [ ] **Step 6: Commit Task 1**

```bash
git add onepagent.html
git commit -m "feat(settings): add reasoning content return controls"
```

---

### Task 2: Add runtime map and reasoning-content helpers

**Files:**
- Modify: `onepagent.html` around legacy provider migration and `MODEL_CONTEXTS`, near lines 3048-3108.
- Modify: `onepagent.html` around `isDeepSeekActiveProvider`, near lines 3301-3305.

- [ ] **Step 1: Preserve legacy/config reasoning-content maps during provider migration**

In `migrateLegacyProviderSettingsIntoProfiles()`, after `legacyModelContexts`, add:

```js
  const legacyReasoningContentModels = normalizeReasoningContentModelsMap(_userSettings.reasoning_content_models || CFG.reasoning_content_models || {});
```

Then replace the migrated provider object with:

```js
      [id]: { id, name, type: legacyType, endpoint: legacyEndpoint, apiKey: legacyKey, models: legacyModels, defaultModel: legacyDefaultModel, model_contexts: legacyModelContexts, reasoning_content_models: legacyReasoningContentModels }
```

Expected: a first-run migrated provider carries any existing `reasoning_content_models` config into provider-profile storage.

- [ ] **Step 2: Add model reasoning-content state beside `MODEL_CONTEXTS`**

Replace:

```js
const MODEL_CONTEXTS = { ...(CFG.model_contexts || {}) };
let currentMaxContextTokens = DEFAULT_MAX_CONTEXT_TOKENS;
```

with:

```js
const MODEL_CONTEXTS = { ...(((ACTIVE_PROVIDER && ACTIVE_PROVIDER.model_contexts) || CFG.model_contexts || {})) };
const MODEL_REASONING_CONTENT = { ...normalizeReasoningContentModelsMap((ACTIVE_PROVIDER && ACTIVE_PROVIDER.reasoning_content_models) || CFG.reasoning_content_models || {}) };
let currentMaxContextTokens = DEFAULT_MAX_CONTEXT_TOKENS;
```

Expected: runtime state starts from the active provider profile when available.

- [ ] **Step 3: Add normalizers and helper functions before provider migration is called**

Insert these functions above `migrateLegacyProviderSettingsIntoProfiles()` so migration can call the map normalizer:

```js
const REASONING_CONTENT_MODES = new Set(['auto', 'on', 'off']);
function normalizeReasoningContentMode(mode) {
  const m = String(mode || 'auto').trim().toLowerCase();
  return REASONING_CONTENT_MODES.has(m) ? m : 'auto';
}
function normalizeReasoningContentModelsMap(map) {
  const out = {};
  if (!map || typeof map !== 'object' || Array.isArray(map)) return out;
  Object.entries(map).forEach(([model, mode]) => {
    const key = String(model || '').trim();
    if (key) out[key] = normalizeReasoningContentMode(mode);
  });
  return out;
}
function isMimoReasoningModel(model = API_MODEL) {
  const normalized = String(model || '').toLowerCase().replace(/[^a-z0-9]+/g, '');
  return normalized.includes('xiaomimimo') || normalized.includes('mimo');
}
function isDefaultReasoningContentModel(model = API_MODEL) {
  const p = ACTIVE_PROVIDER || {};
  const haystack = [p.name, p.endpoint, p.defaultModel, model, _keys?.api_endpoint, CFG.api_endpoint].filter(Boolean).join(' ').toLowerCase();
  return haystack.includes('deepseek') || isMimoReasoningModel(model);
}
function getReasoningContentModeForModel(model = API_MODEL) {
  const key = String(model || '').trim();
  return normalizeReasoningContentMode(key ? MODEL_REASONING_CONTENT[key] : 'auto');
}
function shouldKeepReasoningForModel(model = API_MODEL) {
  const mode = getReasoningContentModeForModel(model);
  if (mode === 'on') return true;
  if (mode === 'off') return false;
  return isDefaultReasoningContentModel(model);
}
```

Expected: the feature has one effective policy helper that all runtime paths can call.

- [ ] **Step 4: Remove the old DeepSeek-only helper**

Delete this function:

```js
function isDeepSeekActiveProvider() {
  const p = ACTIVE_PROVIDER || {};
  const haystack = [p.name, p.endpoint, p.defaultModel, API_MODEL, _keys?.api_endpoint, CFG.api_endpoint].filter(Boolean).join(' ').toLowerCase();
  return haystack.includes('deepseek');
}
```

Expected: no standalone provider-level DeepSeek helper remains; DeepSeek defaulting now lives inside `isDefaultReasoningContentModel()`.

- [ ] **Step 5: Smoke-check helper names**

Run:

```bash
grep -n "MODEL_REASONING_CONTENT\|normalizeReasoningContentMode\|isMimoReasoningModel\|shouldKeepReasoningForModel\|isDeepSeekActiveProvider" onepagent.html
```

Expected: matches for new helper names and no `isDeepSeekActiveProvider` match.

- [ ] **Step 6: Commit Task 2**

```bash
git add onepagent.html
git commit -m "feat(models): add reasoning content policy helpers"
```

---

### Task 3: Persist reasoning-content settings with provider profiles

**Files:**
- Modify: `onepagent.html` provider switching near lines 3245-3267.
- Modify: `onepagent.html` Settings profile loading/new profile near lines 7841-7890.
- Modify: `onepagent.html` `saveSettings()` near lines 8197-8305.

- [ ] **Step 1: Load reasoning-content map into the Settings modal**

In `_applyProviderProfileToSettingsModal(p)`, after:

```js
  const ctxs = p.model_contexts || {};
  renderModelContextsList(ctxs);
```

add:

```js
  renderReasoningContentModelsList(p.reasoning_content_models || {});
```

Expected: switching provider profiles refreshes the new reasoning section.

- [ ] **Step 2: Initialize new provider profiles with an empty reasoning map**

In `newProviderProfile()`, add `reasoning_content_models: {}` after `model_contexts: {}`:

```js
    model_contexts: {},
    reasoning_content_models: {}
```

Expected: newly created provider profiles have an explicit empty map.

- [ ] **Step 3: Reset reasoning UI when no provider is active**

In `deleteCurrentProviderProfile()`, replace:

```js
  else renderModelContextsList({});
```

with:

```js
  else {
    renderModelContextsList({});
    renderReasoningContentModelsList({});
  }
```

In `openSettingsModal()`, after:

```js
  renderModelContextsList({}); // reset; overridden below if a profile is active
```

add:

```js
  renderReasoningContentModelsList({});
```

Expected: stale rows are cleared when the modal opens or provider profiles are removed.

- [ ] **Step 4: Update runtime map on top-bar provider switch**

In the `providerSelect` change handler, replace the `_keys` assignment with:

```js
  _keys = Object.assign(_keys || {}, {
    api_key: (ACTIVE_PROVIDER.apiKey || ''),
    api_endpoint: (ACTIVE_PROVIDER.endpoint || ''),
    api_models: (ACTIVE_PROVIDER.models || []),
    api_model: API_MODEL,
    model_contexts: (ACTIVE_PROVIDER.model_contexts || {}),
    reasoning_content_models: normalizeReasoningContentModelsMap(ACTIVE_PROVIDER.reasoning_content_models || {})
  });
```

After the `MODEL_CONTEXTS` update block, add:

```js
  Object.keys(MODEL_REASONING_CONTENT).forEach(k => delete MODEL_REASONING_CONTENT[k]);
  Object.assign(MODEL_REASONING_CONTENT, _keys.reasoning_content_models || {});
```

Expected: switching providers updates both context limits and reasoning-content policy state.

- [ ] **Step 5: Read reasoning-content settings in `saveSettings()`**

After:

```js
  const model_contexts = readModelContextsFromUI();
```

add:

```js
  const reasoning_content_models = readReasoningContentModelsFromUI();
```

Expected: Settings save has both per-model maps available.

- [ ] **Step 6: Save reasoning-content settings into provider profiles**

Replace the provider profile assignment in `saveSettings()` with:

```js
  providers[profId] = { id: profId, name, type, endpoint, apiKey, models, defaultModel: defaultModel || models[0] || '', imageModels, videoModels, defaultImageModel, defaultVideoModel, imageGenerationEndpoint, imageEditEndpoint, videoGenerationEndpoint, videoStatusEndpoint, model_contexts, reasoning_content_models };
```

Expected: provider profiles persist `reasoning_content_models` alongside `model_contexts`.

- [ ] **Step 7: Save reasoning-content settings into runtime `_keys`**

In the runtime `_keys` assignment inside `saveSettings()`, replace:

```js
    api_model: ACTIVE_PROVIDER.defaultModel || '',
    model_contexts
```

with:

```js
    api_model: ACTIVE_PROVIDER.defaultModel || '',
    model_contexts,
    reasoning_content_models
```

Then after the `MODEL_CONTEXTS` update block, add:

```js
  Object.keys(MODEL_REASONING_CONTENT).forEach(k => delete MODEL_REASONING_CONTENT[k]);
  Object.assign(MODEL_REASONING_CONTENT, reasoning_content_models);
```

Expected: saving settings updates in-memory policy immediately without reload.

- [ ] **Step 8: Smoke-check persistence references**

Run:

```bash
grep -n "reasoning_content_models" onepagent.html
```

Expected: matches cover migration, runtime initialization, provider switch, new provider creation, settings load, and settings save.

- [ ] **Step 9: Commit Task 3**

```bash
git add onepagent.html
git commit -m "feat(settings): persist reasoning content model modes"
```

---

### Task 4: Add Settings UI behavior for reasoning-content rows

**Files:**
- Modify: `onepagent.html` near Model Context UI helpers, around lines 8078-8195.

- [ ] **Step 1: Add row rendering and empty-state helpers**

Insert this block after `readModelContextsFromUI()` and before `addModelContextFromInputs()`:

```js
function _rcEffectiveLabel(model, mode) {
  const effective = mode === 'on' || (mode === 'auto' && isDefaultReasoningContentModel(model));
  return mode === 'auto' ? (effective ? 'Auto: On' : 'Auto: Off') : (effective ? 'On' : 'Off');
}
function _rcBuildRow(model, mode = 'auto') {
  const normalizedMode = normalizeReasoningContentMode(mode);
  const row = document.createElement('div');
  row.className = 'rc-row';
  row.dataset.model = model;
  row.innerHTML =
    '<span class="rc-model" title="' + _mcEsc(model) + '">' + _mcEsc(model) + '</span>' +
    '<select class="rc-mode"><option value="auto">Auto</option><option value="on">On</option><option value="off">Off</option></select>' +
    '<span class="rc-effective"></span>' +
    '<button type="button" class="rc-remove" title="Remove" aria-label="Remove">&times;</button>';
  const select = row.querySelector('.rc-mode');
  const effective = row.querySelector('.rc-effective');
  const refresh = () => { effective.textContent = _rcEffectiveLabel(model, normalizeReasoningContentMode(select.value)); };
  select.value = normalizedMode;
  select.addEventListener('change', refresh);
  row.querySelector('.rc-remove').addEventListener('click', () => {
    row.remove();
    _rcUpdateEmptyState();
    _rcUpdateModelDatalist();
  });
  refresh();
  return row;
}
function _rcUpdateEmptyState() {
  const list = document.getElementById('setReasoningContentList');
  const empty = document.getElementById('setReasoningContentEmpty');
  if (!list || !empty) return;
  const hasRows = list.children.length > 0;
  empty.style.display = hasRows ? 'none' : 'block';
  list.style.display = hasRows ? '' : 'none';
}
function _rcUpdateModelDatalist() {
  const dl = document.getElementById('setRcModelList');
  if (!dl) return;
  const selected = getCurrentSelectedModels();
  const have = new Set(Array.from(document.querySelectorAll('#setReasoningContentList .rc-row')).map(r => r.dataset.model));
  dl.innerHTML = selected.filter(m => !have.has(m)).map(m => '<option value="' + _mcEsc(m) + '"></option>').join('');
}
function renderReasoningContentModelsList(configs) {
  const list = document.getElementById('setReasoningContentList');
  if (!list) return;
  list.innerHTML = '';
  Object.entries(normalizeReasoningContentModelsMap(configs || {})).forEach(([model, mode]) => {
    list.appendChild(_rcBuildRow(model, mode));
  });
  _rcUpdateEmptyState();
  _rcUpdateModelDatalist();
}
function readReasoningContentModelsFromUI() {
  const out = {};
  document.querySelectorAll('#setReasoningContentList .rc-row').forEach(row => {
    const model = String(row.dataset.model || '').trim();
    const mode = normalizeReasoningContentMode(row.querySelector('.rc-mode')?.value || 'auto');
    if (model) out[model] = mode;
  });
  return out;
}
```

Expected: the Settings modal can render and read persisted reasoning-content rows.

- [ ] **Step 2: Add manual add and import functions**

Insert this block after `syncModelContextsFromSelected()`:

```js
function addReasoningContentModelFromInputs() {
  const mi = document.getElementById('setRcNewModel');
  const si = document.getElementById('setRcNewMode');
  const status = document.getElementById('setStatus');
  const model = (mi.value || '').trim();
  const mode = normalizeReasoningContentMode(si.value || 'auto');
  if (!model) { status.textContent = 'Enter a model id'; mi.focus(); return; }
  const list = document.getElementById('setReasoningContentList');
  const existing = Array.from(list.querySelectorAll('.rc-row')).find(r => r.dataset.model === model);
  if (existing) {
    existing.querySelector('.rc-mode').value = mode;
    existing.querySelector('.rc-mode').dispatchEvent(new Event('change'));
    existing.scrollIntoView({ block: 'nearest' });
  } else {
    list.appendChild(_rcBuildRow(model, mode));
  }
  mi.value = '';
  si.value = 'auto';
  status.textContent = existing ? ('Updated reasoning mode for "' + model + '"') : ('Added reasoning mode for "' + model + '"');
  _rcUpdateEmptyState();
  _rcUpdateModelDatalist();
  mi.focus();
}

function syncReasoningContentFromSelected() {
  const selected = getCurrentSelectedModels();
  const status = document.getElementById('setStatus');
  if (!selected.length) { status.textContent = 'No selected models to import'; return; }
  const list = document.getElementById('setReasoningContentList');
  const have = new Set(Array.from(list.querySelectorAll('.rc-row')).map(r => r.dataset.model));
  let added = 0;
  selected.forEach(m => { if (!have.has(m)) { list.appendChild(_rcBuildRow(m, 'auto')); added++; } });
  status.textContent = added ? ('Added ' + added + ' reasoning mode row(s)') : 'All selected models already listed';
  _rcUpdateEmptyState();
  _rcUpdateModelDatalist();
}
```

Expected: “Import Selected” adds selected models as `auto`, and manual add validates a non-empty model id.

- [ ] **Step 3: Keep both datalists refreshed when selected models change**

In `setSelectedModels(list)`, replace:

```js
  _mcUpdateModelDatalist();
```

with:

```js
  _mcUpdateModelDatalist();
  _rcUpdateModelDatalist();
```

Expected: both the context-length and reasoning-content add rows suggest selected models that are not already listed.

- [ ] **Step 4: Smoke-check UI helper functions**

Run:

```bash
grep -n "function _rcBuildRow\|function renderReasoningContentModelsList\|function readReasoningContentModelsFromUI\|function syncReasoningContentFromSelected\|function addReasoningContentModelFromInputs" onepagent.html
```

Expected: all five functions are present.

- [ ] **Step 5: Commit Task 4**

```bash
git add onepagent.html
git commit -m "feat(settings): manage reasoning content model rows"
```

---

### Task 5: Route request assembly and parsing through the per-model helper

**Files:**
- Modify: `onepagent.html` `buildSubAgentRequestBody()` near lines 11721-11734.
- Modify: `onepagent.html` sub-agent stream parsing near lines 11804-11812.
- Modify: `onepagent.html` `assembleRequestBodyFromParts()` near lines 14016-14068.
- Modify: `onepagent.html` main stream parsing and usage accounting near lines 14797-14823.

- [ ] **Step 1: Make request assembly accept the effective model**

Change the function signature and top of `assembleRequestBodyFromParts()` from:

```js
function assembleRequestBodyFromParts(parts) {
  if (PROVIDER === 'anthropic_compat') {
    const body = {
      model: API_MODEL,
```

to:

```js
function assembleRequestBodyFromParts(parts, options = {}) {
  const requestModel = String(options.model || API_MODEL || '').trim() || API_MODEL;
  if (PROVIDER === 'anthropic_compat') {
    const body = {
      model: requestModel,
```

Expected: all request-body construction can use the lead model or a sub-agent override model.

- [ ] **Step 2: Replace DeepSeek-only request assembly policy**

Inside `assembleRequestBodyFromParts()`, replace:

```js
  const keepReasoning = isDeepSeekActiveProvider();
```

with:

```js
  const keepReasoning = shouldKeepReasoningForModel(requestModel);
```

At the end of the function, replace:

```js
  return applyThinkingToRequestBody({ model: API_MODEL, messages: oai, tools: parts.openaiTools }, PROVIDER);
```

with:

```js
  return applyThinkingToRequestBody({ model: requestModel, messages: oai, tools: parts.openaiTools }, PROVIDER);
```

Expected: historical assistant reasoning blocks are returned as `reasoning_content` when the effective model policy is On, including tool-call assistant messages with stored reasoning.

- [ ] **Step 3: Pass sub-agent model override into assembly before history is serialized**

In `buildSubAgentRequestBody()`, replace:

```js
  const body = assembleRequestBodyFromParts(parts);
  if (options.modelOverride) body.model = options.modelOverride;
  return body;
```

with:

```js
  return assembleRequestBodyFromParts(parts, { model: options.modelOverride || API_MODEL });
```

Expected: sub-agent history assembly and body model use the same model id.

- [ ] **Step 4: Use the helper for sub-agent stream parsing**

In `runSubAgentLoop()`, before `const result = PROVIDER === 'anthropic_compat'`, add:

```js
    const keepReasoning = shouldKeepReasoningForModel(body.model || API_MODEL);
```

Then replace the OpenAI parse call with:

```js
      : await parseOpenAIStream(reader, () => {}, () => {}, () => {}, () => {}, { keepReasoning, includeReasoningUsage: keepReasoning });
```

Expected: sub-agent stream parsing preserves reasoning blocks only when the worker model’s effective policy is On.

- [ ] **Step 5: Use the helper for main chat stream parsing and usage accounting**

In `_runAgentLoop()`, after `const body = await buildRequestBody(...)`, add:

```js
      const keepReasoning = shouldKeepReasoningForModel(body.model || API_MODEL);
```

Replace the OpenAI parse call with:

```js
          : await parseOpenAIStream(reader, onText, onReasoning, onToolStart, onToolDelta, { keepReasoning, includeReasoningUsage: keepReasoning });
```

Replace usage details extraction with:

```js
      const usageDetails = extractUsageDetails(result.usage, { includeReasoningDetails: keepReasoning });
```

Expected: main chat parsing, stored reasoning blocks, and reasoning usage accounting are all controlled by the same model-bound decision.

- [ ] **Step 6: Smoke-check old and new call sites**

Run:

```bash
grep -n "isDeepSeekActiveProvider\|shouldKeepReasoningForModel\|parseOpenAIStream(reader" onepagent.html
```

Expected: no `isDeepSeekActiveProvider` matches; `shouldKeepReasoningForModel` appears in request assembly, main chat parsing, sub-agent parsing, and helper definitions.

- [ ] **Step 7: Commit Task 5**

```bash
git add onepagent.html
git commit -m "feat(models): apply reasoning content policy to requests"
```

---

### Task 6: Validate policy behavior with lightweight static/runtime checks

**Files:**
- Modify only if validation reveals bugs: `onepagent.html`.

- [ ] **Step 1: Check syntax-critical tokens**

Run:

```bash
grep -n "MODEL_REASONING_CONTENT\|reasoning_content_models\|shouldKeepReasoningForModel\|reasoning_content" onepagent.html
```

Expected: matches cover Settings UI, persistence, helper policy, request assembly, and existing `reasoning_content` serialization.

- [ ] **Step 2: Confirm no old hard-coded DeepSeek call remains**

Run:

```bash
grep -n "isDeepSeekActiveProvider" onepagent.html || true
```

Expected: no output.

- [ ] **Step 3: Start a local static server**

Run:

```bash
python -m http.server 8765
```

Expected: server starts and serves the repository root. If port `8765` is busy, retry with another unprivileged port such as `8787`.

- [ ] **Step 4: Confirm OnePagent loads**

Open:

```text
http://127.0.0.1:8765/onepagent.html
```

Expected: OnePagent loads without a console syntax error.

- [ ] **Step 5: Validate Settings add/save/delete behavior manually**

In the browser:

1. Open Settings.
2. Add selected/chat models to “Reasoning Content Return” with “Import Selected”.
3. Verify each imported row defaults to `Auto`.
4. Change one normal OpenAI-compatible model to `On`.
5. Change one DeepSeek or MiMo model to `Off`.
6. Save Settings.
7. Reopen Settings and verify the modes persisted.
8. Delete one row, save, reopen, and verify the row is gone.

Expected: `reasoning_content_models` rows save/load with the active provider profile, and deleting a row reverts that model to default detection.

- [ ] **Step 6: Validate effective defaults manually in the browser console**

Run these expressions in the loaded app console after configuring a provider profile:

```js
shouldKeepReasoningForModel('deepseek-chat')
shouldKeepReasoningForModel('xiaomimimo-72b')
shouldKeepReasoningForModel('gpt-4o')
```

Expected:

```text
true
true
false
```

Then configure `gpt-4o` to `On` and `deepseek-chat` to `Off`, save settings, and run:

```js
shouldKeepReasoningForModel('gpt-4o')
shouldKeepReasoningForModel('deepseek-chat')
```

Expected:

```text
true
false
```

- [ ] **Step 7: Validate multi-turn request assembly manually**

In the browser console, create a minimal request-parts object:

```js
const reasoningParts = {
  systemPrompt: 'system',
  stableSystemPrompt: 'system',
  volatileSystemPrompt: '',
  projectedMessages: [
    { role: 'user', content: 'first' },
    { role: 'assistant', content: [
      { type: 'reasoning', reasoning_content: 'stored reasoning' },
      { type: 'text', text: 'I will call a tool.' },
      { type: 'tool_use', id: 'call_1', name: 'Bash', input: { command: 'pwd' } }
    ] },
    { role: 'user', content: [
      { type: 'tool_result', tool_use_id: 'call_1', content: 'ok' },
      { type: 'text', text: 'next' }
    ] }
  ],
  openaiTools: [],
  anthropicTools: []
};
```

Set `PROVIDER = 'openai_compat'`, then verify On behavior:

```js
MODEL_REASONING_CONTENT['gpt-4o'] = 'on';
assembleRequestBodyFromParts(reasoningParts, { model: 'gpt-4o' }).messages.find(m => m.role === 'assistant').reasoning_content;
```

Expected:

```text
stored reasoning
```

Then verify Off behavior:

```js
MODEL_REASONING_CONTENT['gpt-4o'] = 'off';
assembleRequestBodyFromParts(reasoningParts, { model: 'gpt-4o' }).messages.find(m => m.role === 'assistant').reasoning_content;
```

Expected: `undefined`.

- [ ] **Step 8: Validate missing historical reasoning fallback**

In the browser console, create a parts object with a tool-call assistant message but no reasoning block:

```js
const missingReasoningParts = {
  systemPrompt: 'system',
  stableSystemPrompt: 'system',
  volatileSystemPrompt: '',
  projectedMessages: [
    { role: 'user', content: 'first' },
    { role: 'assistant', content: [
      { type: 'text', text: 'I will call a tool.' },
      { type: 'tool_use', id: 'call_2', name: 'Bash', input: { command: 'pwd' } }
    ] },
    { role: 'user', content: [
      { type: 'tool_result', tool_use_id: 'call_2', content: 'ok' },
      { type: 'text', text: 'next' }
    ] }
  ],
  openaiTools: [],
  anthropicTools: []
};
MODEL_REASONING_CONTENT['gpt-4o'] = 'on';
assembleRequestBodyFromParts(missingReasoningParts, { model: 'gpt-4o' }).messages;
```

Expected: no fabricated `reasoning_content`; the assistant tool-call message and matching tool result are skipped according to the existing fallback behavior, while any text fallback remains as a user message.

- [ ] **Step 9: Stop the local server**

Stop the `python -m http.server 8765` process with Ctrl+C.

- [ ] **Step 10: Commit any validation fixes**

If validation required code fixes, commit them:

```bash
git add onepagent.html
git commit -m "fix(models): validate reasoning content return settings"
```

If no fixes were needed, do not create an empty commit.

---

### Task 7: Final review

**Files:**
- Inspect: `onepagent.html`
- Inspect: `docs/superpowers/specs/2026-05-13-model-reasoning-content-return-design.md`

- [ ] **Step 1: Confirm implementation matches spec**

Run:

```bash
grep -n "reasoning_content_models\|MODEL_REASONING_CONTENT\|shouldKeepReasoningForModel\|reasoning_content\|parseOpenAIStream(reader" onepagent.html
```

Expected: matches cover model-bound settings, runtime helpers, Settings UI persistence, request assembly, normal chat parsing, and sub-agent parsing.

- [ ] **Step 2: Confirm no unrelated reasoning behavior changed**

Run:

```bash
grep -n "THINK_LEVEL\|applyThinkingToRequestBody\|thinking" onepagent.html | head -40
```

Expected: Think level behavior still only controls request parameters such as `reasoning_effort`; it is not mixed with `reasoning_content_models`.

- [ ] **Step 3: Confirm git state**

Run:

```bash
git status --short
```

Expected: no unexpected uncommitted files. If the spec or plan files are uncommitted, commit them separately with an accurate docs message.

- [ ] **Step 4: Report results**

Final report should include:

```text
Implemented per-model reasoning_content return settings.
DeepSeek and MiMo default to effective On in Auto mode; other models default Off.
Validated Settings persistence, explicit On/Off overrides, request assembly with stored reasoning_content, missing-reasoning fallback, and normal/sub-agent stream parsing call sites.
```

---

## Self-review notes

- Spec coverage: The plan covers per-model `auto`/`on`/`off`, DeepSeek/MiMo defaults, Settings UI, provider-profile persistence, provider switching, normal chat parsing, sub-agent parsing, usage accounting, request assembly, tool-call history return, and missing historical reasoning fallback.
- Placeholder scan: The plan uses concrete file paths, code snippets, commands, and expected outcomes; it contains no TBD/TODO placeholders.
- Type consistency: The stored key is consistently `reasoning_content_models`; runtime map is `MODEL_REASONING_CONTENT`; effective policy is `shouldKeepReasoningForModel(model)`; modes are `auto`, `on`, and `off`.
