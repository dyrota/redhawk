**AI Agent Build Specification**

Redhawk Platform

  ---------------------- ------------------------------------------------
  **Project**            Redhawk (Redhawk)

  **Document Type**      AI Agent Implementation Specification

  **Primary Stack**      Blazor WASM, .NET 10, Azure AI Foundry,
                         Microsoft RulesEngine

  **Target**             Agent-facing PWA (online + offline)

  **Version**            1.0 --- Proof of Concept Scope
  ---------------------- ------------------------------------------------

**Document Purpose**

This document provides complete technical context for an AI coding agent
to build the Redhawk system. It covers architecture, data contracts,
processing loops, service interfaces, Fluxor state shape, Azure AI
Foundry prompt flow design, and project structure. Follow the build
order in Section 9 to progress incrementally with a demoable system at
each step.

**1. System Overview**

Redhawk is an agent-facing Progressive Web Application built with Blazor
WebAssembly that allows insurance agents to complete life insurance
applications with customers. The system operates in two modes:

  -----------------------------------------------------------------------
  **Mode**               **Capabilities**
  ---------------------- ------------------------------------------------
  **Online Mode**        Full AI voice assistant (Azure Speech STT/TTS +
                         Azure AI Foundry extraction), rules engine runs
                         client-side, completed applications sync to
                         backend.

  **Offline Mode**       Rules engine operates fully client-side from
                         cached rules bundle. Manual form entry only.
                         State persists in IndexedDB and syncs when
                         connectivity is restored.
  -----------------------------------------------------------------------

**1.1 Three-Consumer State Model**

All system behavior derives from a single shared JSON state document.
Three consumers read and write it through controlled mutation paths:

  --------------------------------------------------------------------------------
  **Consumer**       **Role**                           **Availability**
  ------------------ ---------------------------------- --------------------------
  Rules Engine       Reads full state. Outputs          Client-side always
                     NavigationResult (next question +  
                     relevant context + skipped         
                     questions).                        

  AI Assistant       Reads                              Online only
                     NavigationResult.relevantContext   
                     only --- never full state. Writes  
                     FieldPatchProposal.                

  Blazor UI          Reads full state via Fluxor store. Always
                     Renders bound form fields          
                     reactively.                        
  --------------------------------------------------------------------------------

**1.2 Key Architectural Constraints**

-   Rules engine MUST run client-side (Blazor WASM) to support offline
    operation

-   AI assistant is additive --- removing it must not break form
    functionality

-   All state mutations flow through Fluxor actions --- no direct field
    writes

-   Customer PII in IndexedDB must be encrypted at rest using Web Crypto
    API

-   Application submission is final --- no post-submission edits in this
    version

-   Agents may have multiple in-progress applications simultaneously

**2. The Application State Document**

This is the central data contract. Every service, component, and
processing loop references this schema. Store as encrypted JSON in
IndexedDB keyed by applicationId.

**2.1 Full Schema**

> {
>
> \"meta\": {
>
> \"applicationId\": \"uuid-v4\",
>
> \"productType\": \"TermLife \| WholeLife \| Annuity\",
>
> \"rulesVersion\": \"14\",
>
> \"status\": \"InProgress \| Submitted\",
>
> \"createdAt\": \"ISO8601\",
>
> \"lastModified\": \"ISO8601\",
>
> \"agentId\": \"entra-object-id\",
>
> \"currentQuestionId\": \"tobacco_use\",
>
> \"completedSections\": \[\"personal_info\"\],
>
> \"skippedQuestions\": \[\"tobacco_detail_type\"\],
>
> \"completionPercent\": 34
>
> },
>
> \"applicant\": {
>
> \"firstName\": \"John\",
>
> \"lastName\": null,
>
> \"dateOfBirth\": null,
>
> \"tobaccoUse\": false,
>
> \"tobaccoProductType\": null,
>
> \"occupation\": null,
>
> \"annualIncome\": null,
>
> \"stateOfResidence\": null
>
> },
>
> \"coverage\": {
>
> \"requestedAmount\": null,
>
> \"termLength\": null
>
> },
>
> \"beneficiaries\": \[\],
>
> \"healthHistory\": {
>
> \"conditions\": \[\],
>
> \"medications\": \[\]
>
> },
>
> \"conversation\": \[
>
> { \"role\": \"assistant\", \"content\": \"\...\", \"timestamp\":
> \"ISO8601\", \"type\": \"question \| clarification \| summary\" },
>
> { \"role\": \"user\", \"content\": \"\...\", \"timestamp\":
> \"ISO8601\", \"source\": \"voice \| text\" }
>
> \],
>
> \"fieldAudit\": {
>
> \"applicant.firstName\": { \"source\": \"ai_extraction \| manual \|
> agent_voice\", \"timestamp\": \"ISO8601\", \"confidence\": \"high \|
> low\" }
>
> },
>
> \"validation\": {
>
> \"errors\": \[\],
>
> \"warnings\": \[\]
>
> }
>
> }

**2.2 Field Audit Trail**

Every field write must record source, timestamp, and confidence in
fieldAudit. This is required for insurance compliance --- regulators may
ask how each answer was captured. Source values:

-   ai_extraction --- populated by AI from voice utterance

-   agent_voice --- agent spoke the answer, AI extracted it

-   manual --- agent typed directly into form field

**2.3 Null vs Missing**

Use null explicitly for unanswered questions. Never use empty string or
omit the field. The rules engine distinguishes between null (not yet
answered) and false (answered negative) for navigation decisions.

**3. Rules Engine --- Microsoft RulesEngine**

Microsoft.RulesEngine NuGet package. Runs in Blazor WASM (fully
client-side). Rules are loaded from a cached JSON bundle fetched from
the server on first load and version-checked on reconnect.

**3.1 NavigationResult --- The Rules Engine Output Contract**

> public class NavigationResult
>
> {
>
> public Question NextQuestion { get; set; } // null = application
> complete
>
> public Dictionary\<string, object\> RelevantContext { get; set; } //
> AI context slice
>
> public List\<string\> SkippedQuestions { get; set; } // questions
> bypassed this evaluation
>
> public List\<FieldPatch\> EagerPatches { get; set; } // pre-extracted
> future answers to apply
>
> public bool IsComplete { get; set; }
>
> public List\<ValidationError\> ValidationErrors { get; set; }
>
> }

**3.2 Question Definition**

> public class Question
>
> {
>
> public string Id { get; set; }
>
> public string Prompt { get; set; }
>
> public string FieldPath { get; set; } // dot-notation path into state
> doc
>
> public FieldType Type { get; set; } // Boolean, Date, String, Enum,
> Currency
>
> public List\<string\> EnumValues { get; set; } // populated for Type
> == Enum
>
> public string NavigationRule { get; set; } // RulesEngine rule name
> --- must pass to show
>
> public string ValidationRule { get; set; } // RulesEngine rule name
> --- must pass to advance
>
> public List\<ContextRule\> ContextRules { get; set; } // dynamic AI
> context selection
>
> public int DisplayOrder { get; set; }
>
> public string SectionId { get; set; }
>
> public int ConversationWindowSize { get; set; } = 5;
>
> }
>
> public class ContextRule
>
> {
>
> public string Expression { get; set; } // evaluated against state ---
> \'true\' always includes
>
> public string IncludeFieldPath { get; set; }
>
> }

**3.3 Rule Layers**

Rules are organized in three layers. Atomic rules are the single source
of truth and must never be duplicated:

  ------------------------------------------------------------------------
  **Layer**          **Purpose**                **Example**
  ------------------ -------------------------- --------------------------
  Atomic             Single-concern, reusable   IsAdult, IsSmoker,
                                                HighRiskOccupation

  Composite          Assembled from atomics,    TermLifeEligible =
                     per-product                IsAdult + !HighRisk

  Navigation         Drive question sequencing  AskTobaccoFollowUp =
                                                tobaccoUse == true &&
                                                tobaccoProductType == null
  ------------------------------------------------------------------------

**3.4 IRulesEngineService Interface**

> public interface IRulesEngineService
>
> {
>
> Task\<NavigationResult\> EvaluateAsync(ApplicationStateDocument
> state);
>
> Task LoadWorkflowAsync(string productType, string version);
>
> bool IsLoaded { get; }
>
> string LoadedVersion { get; }
>
> }

**3.5 Navigation Loop (Pseudocode)**

> foreach (var question in orderedQuestionGraph)
>
> {
>
> if (state already has non-null answer for question.FieldPath)
> continue;
>
> var navigationPasses = await
> rulesEngine.EvaluateRuleAsync(question.NavigationRule, state);
>
> if (!navigationPasses) { result.SkippedQuestions.Add(question.Id);
> continue; }
>
> // Build RelevantContext from ContextRules
>
> result.RelevantContext = question.ContextRules
>
> .Where(cr =\> Evaluate(cr.Expression, state))
>
> .ToDictionary(cr =\> cr.IncludeFieldPath, cr =\> GetFieldValue(state,
> cr.IncludeFieldPath));
>
> result.NextQuestion = question;
>
> break;
>
> }

**4. Fluxor State Store**

Use Fluxor (Redux pattern for Blazor). All state mutations --- AI
patches, rules engine navigation, and manual edits --- dispatch through
Fluxor actions. No component writes state directly.

**4.1 State Shape**

> // Root state --- composed of three slices
>
> public record AgentState
>
> {
>
> public string AgentId { get; init; }
>
> public string DisplayName { get; init; }
>
> public string Email { get; init; }
>
> }
>
> public record ApplicationsState
>
> {
>
> public string ActiveApplicationId { get; init; }
>
> public ImmutableDictionary\<string, ApplicationEntry\> Applications {
> get; init; }
>
> }
>
> public record ApplicationEntry
>
> {
>
> public ApplicationStateDocument Document { get; init; }
>
> public NavigationResult CurrentNavigation { get; init; }
>
> public SyncStatus SyncStatus { get; init; } // Local \| PendingSync \|
> Synced
>
> public DateTimeOffset LastModified { get; init; }
>
> }
>
> public record UIState
>
> {
>
> public bool IsListening { get; init; }
>
> public bool IsProcessing { get; init; }
>
> public string TranscriptBuffer { get; init; }
>
> public ImmutableList\<TranscriptEntry\> ConversationDisplay { get;
> init; }
>
> }

**4.2 Actions**

> // AI returns field patches
>
> public record ApplyFieldPatchesAction(string ApplicationId,
> List\<FieldPatch\> Patches);
>
> // Rules engine returns next navigation state
>
> public record NavigationUpdatedAction(string ApplicationId,
> NavigationResult Result);
>
> // Agent manually edits an already-answered field
>
> public record ManualFieldEditAction(string ApplicationId, string
> FieldPath, object Value);
>
> // Agent saves progress and closes application
>
> public record SaveApplicationAction(string ApplicationId);
>
> // Agent reopens a saved application
>
> public record LoadApplicationAction(string ApplicationId);
>
> // New application started
>
> public record CreateApplicationAction(string ProductType, CustomerInfo
> Customer);
>
> // Application submitted (final, read-only after this)
>
> public record SubmitApplicationAction(string ApplicationId);
>
> // Speech pipeline state
>
> public record SetListeningAction(bool IsListening);
>
> public record SetProcessingAction(bool IsProcessing);
>
> public record AppendTranscriptAction(TranscriptEntry Entry);

**4.3 Effect: ApplyFieldPatches → NavigationUpdated**

After patches are applied, a Fluxor Effect must trigger rules engine
re-evaluation and dispatch NavigationUpdatedAction. This is the core
reactive loop:

> \[EffectMethod\]
>
> public async Task HandleApplyPatches(ApplyFieldPatchesAction action,
> IDispatcher dispatcher)
>
> {
>
> var entry =
> \_store.State.Applications.Applications\[action.ApplicationId\];
>
> var updatedDoc = ApplyPatches(entry.Document, action.Patches);
>
> var navigation = await \_rulesEngine.EvaluateAsync(updatedDoc);
>
> dispatcher.Dispatch(new NavigationUpdatedAction(action.ApplicationId,
> navigation));
>
> await \_storage.SaveAsync(action.ApplicationId, updatedDoc); //
> persist to IndexedDB
>
> }

**5. Azure AI Foundry --- Prompt Flow**

The PWA calls a single Prompt Flow endpoint, never the model directly.
The flow owns all LLM interaction, context assembly, and response
shaping. Model: GPT-4o. Temperature: 0.1 (deterministic extraction).

**5.1 AI Context Envelope (PWA → Foundry)**

The PWA sends a lean envelope --- never the full state document:

> {
>
> \"currentQuestion\": {
>
> \"id\": \"tobacco_use\",
>
> \"prompt\": \"Have you used tobacco or nicotine in the last 12
> months?\",
>
> \"fieldPath\": \"applicant.tobaccoUse\",
>
> \"fieldType\": \"boolean\",
>
> \"enumValues\": null,
>
> \"constraints\": null
>
> },
>
> \"relevantContext\": {
>
> \"applicant.firstName\": \"John\",
>
> \"applicant.age\": 34
>
> },
>
> \"conversationHistory\": \[
>
> { \"role\": \"assistant\", \"content\": \"\...\", \"timestamp\":
> \"\...\" },
>
> { \"role\": \"user\", \"content\": \"No I dont smoke\", \"timestamp\":
> \"\...\" }
>
> \],
>
> \"userUtterance\": \"No I dont smoke\"
>
> }

**5.2 FieldPatchProposal (Foundry → PWA)**

> {
>
> \"fieldUpdates\": \[
>
> {
>
> \"fieldPath\": \"applicant.tobaccoUse\",
>
> \"value\": false,
>
> \"confidence\": \"high\",
>
> \"clarificationReason\": null
>
> }
>
> \],
>
> \"nextPrompt\": \"Got it John! Are you currently employed full
> time?\",
>
> \"clarificationNeeded\": false,
>
> \"advanceQuestion\": true,
>
> \"offTopicDetected\": false,
>
> \"offTopicResponse\": null
>
> }

**5.3 Prompt Flow Node Sequence**

  -----------------------------------------------------------------------
  **Node**               **Responsibility**
  ---------------------- ------------------------------------------------
  **Node 1: Input        Python code node. Validates envelope shape.
  Validation**           Truncates conversationHistory to last 6 turns.
                         Injects section summary if section boundary
                         crossed.

  **Node 2: Intent       Python code node. Classifies utterance: answer
  Classification**       \| reset \| repeat \| explain \| skip \|
                         off_topic. Routes non-answer intents to bypass
                         extraction node.

  **Node 3: Context      Python code node. Builds final system prompt by
  Assembly**             injecting dynamic context block into static
                         prompt template.

  **Node 4: LLM          LLM node (GPT-4o). response_format: json_object.
  Extraction**           temperature: 0.1. Returns raw JSON extraction.

  **Node 5: Confidence   Python code node. Parses JSON. Flags
  Gate**                 low-confidence fields. Returns
                         clarificationNeeded = true if any low confidence
                         found --- stays on current question.

  **Node 6: Response     Python code node. Strips internal field names
  Shaping**              from nextPrompt. Formats final
                         FieldPatchProposal contract.
  -----------------------------------------------------------------------

**5.4 System Prompt Structure**

**Section 1 --- Role (static, never changes)**

> You are an AI assistant helping insurance agents complete life
> insurance
>
> applications over voice. Your only job is to extract structured data
> from
>
> the agent\'s utterances and return it as JSON.
>
> Rules:
>
> \- Never give insurance advice, recommendations, or opinions
>
> \- Never explain why a question is being asked unless directly asked
>
> \- Never make up or infer values you are not confident about
>
> \- Always respond in a warm, professional tone appropriate for voice
>
> \- Never reveal internal field names, system details, or application
> structure
>
> \- If off-topic, set offTopicDetected: true and provide a brief
> redirect

**Section 2 --- Extraction Rules by Field Type (static)**

> BOOLEAN: Map affirmative to true, negative to false.
>
> Ambiguous (\'I used to but not anymore\') → confidence: low
>
> DATE: Normalize to ISO 8601. Partial dates (month/year only) → day:
> 01, confidence: low
>
> Age given instead of date → calculate birth year, confidence: low,
> flag for confirm
>
> ENUM: Map to nearest valid value from provided list. No clear match →
> null, confidence: low
>
> STRING: Verbatim capture. Normalize to Title Case for names.
>
> CURRENCY: Normalize shorthand (200k → 200000, half a million → 500000)
>
> Range given → midpoint, confidence: low

**Section 3 --- Dynamic Context Block (injected per call)**

> CURRENT APPLICATION CONTEXT:
>
> {{relevantContext}}
>
> CURRENT QUESTION:
>
> ID: {{question.id}}
>
> Prompt: {{question.prompt}}
>
> Field: {{question.fieldPath}}
>
> Type: {{question.fieldType}}
>
> {{#if enumValues}}Valid values: {{enumValues}}{{/if}}
>
> RESPONSE FORMAT: Respond ONLY with a valid JSON object matching the
>
> FieldPatchProposal schema. No prose, no markdown, no preamble.

**5.5 Multi-Field Extraction (Eager Mode)**

If the agent\'s utterance answers multiple questions in one turn, the AI
should extract all of them in a single fieldUpdates array. The rules
engine will receive all patches simultaneously, evaluate the full
updated state, and may skip several questions in one jump. This is the
desired behavior --- do not limit extraction to the current question
only.

**5.6 IFoundryService Interface**

> public interface IFoundryService
>
> {
>
> Task\<FieldPatchProposal\> ExtractAsync(
>
> NavigationResult navigation,
>
> string userUtterance,
>
> List\<ConversationTurn\> recentHistory);
>
> bool IsAvailable { get; } // false when offline --- UI hides mic
> button
>
> }

**6. Speech Pipeline --- Azure Speech Services**

Azure Cognitive Services Speech SDK for both STT and TTS in online mode.
Graceful degradation to text-only input offline. The speech pipeline is
entirely decoupled from the AI extraction loop --- it produces a string
transcript that is passed to IFoundryService.

**6.1 ISpeechService Interface**

> public interface ISpeechService
>
> {
>
> Task\<string\> ListenAsync(CancellationToken ct); // Returns
> transcript string
>
> Task SpeakAsync(string text, CancellationToken ct);
>
> bool IsAvailable { get; }
>
> event EventHandler\<string\> OnInterimTranscript; // for live display
> while speaking
>
> }

**6.2 Online Voice Loop**

> 1\. Agent presses mic button → UIState.IsListening = true
>
> 2\. ISpeechService.ListenAsync() → Azure STT → transcript string
>
> 3\. UIState.IsListening = false, UIState.IsProcessing = true
>
> 4\. Dispatch AppendTranscriptAction({ role: \'user\', content:
> transcript })
>
> 5\. IFoundryService.ExtractAsync(currentNavigation, transcript,
> history)
>
> 6\. Dispatch ApplyFieldPatchesAction(patches)
>
> 7\. Effect triggers NavigationUpdatedAction
>
> 8\. ISpeechService.SpeakAsync(fieldPatchProposal.nextPrompt)
>
> 9\. UIState.IsProcessing = false

**7. IndexedDB Persistence & Sync**

**7.1 IndexedDB Stores**

  ------------------------------------------------------------------------
  **Store**          **Key**                    **Value Shape**
  ------------------ -------------------------- --------------------------
  applications       applicationId              { id, agentId,
                                                productType, customerName,
                                                status, lastModified,
                                                encryptedStateDocument,
                                                rulesVersion }

  rules_cache        productType + version      { productType, version,
                                                fetchedAt, workflowJson,
                                                questionGraph }

  agent_meta         \'profile\'                { agentId, lastSync,
                                                preferences }
  ------------------------------------------------------------------------

**7.2 Encryption at Rest**

State documents contain customer PII and must be encrypted before
writing to IndexedDB. Use the Web Crypto API with AES-GCM. Derive the
encryption key from the agent\'s MSAL access token on login. The key
lives in memory only --- if the session expires, data requires
re-authentication to read.

> // Key derivation (on login)
>
> const keyMaterial = await crypto.subtle.importKey(\'raw\', tokenBytes,
> \'PBKDF2\', false, \[\'deriveKey\'\]);
>
> const encryptionKey = await crypto.subtle.deriveKey(
>
> { name: \'PBKDF2\', salt, iterations: 100000, hash: \'SHA-256\' },
>
> keyMaterial, { name: \'AES-GCM\', length: 256 }, false, \[\'encrypt\',
> \'decrypt\'\]
>
> );

**7.3 Rules Version Gate**

> On application load:
>
> 1\. GET /api/rules/version?productType={type} → { version: \'15\' }
>
> 2\. Compare to rules_cache entry for productType
>
> 3\. If offline → use cached rules, log version mismatch warning
>
> 4\. If version mismatch → fetch new rules bundle, update cache, reload
> engine
>
> 5\. If version matches → proceed with cached rules

**7.4 IApplicationStorageService Interface**

> public interface IApplicationStorageService
>
> {
>
> Task SaveAsync(string applicationId, ApplicationStateDocument doc);
>
> Task\<ApplicationStateDocument\> LoadAsync(string applicationId);
>
> Task\<List\<ApplicationSummary\>\> ListAsync(string agentId);
>
> Task DeleteAsync(string applicationId);
>
> Task MarkPendingSyncAsync(string applicationId);
>
> }

**8. Project Structure**

> Redhawk.sln
>
> ├── src/
>
> │ ├── Redhawk.Client/ (Blazor WASM PWA)
>
> │ │ ├── wwwroot/
>
> │ │ │ ├── manifest.json
>
> │ │ │ └── service-worker.js
>
> │ │ ├── Store/ (Fluxor)
>
> │ │ │ ├── ApplicationsState/
>
> │ │ │ │ ├── ApplicationsState.cs
>
> │ │ │ │ ├── ApplicationsActions.cs
>
> │ │ │ │ ├── ApplicationsReducers.cs
>
> │ │ │ │ └── ApplicationsEffects.cs
>
> │ │ │ ├── AgentState/
>
> │ │ │ └── UIState/
>
> │ │ ├── Services/
>
> │ │ │ ├── IRulesEngineService.cs
>
> │ │ │ ├── RulesEngineService.cs
>
> │ │ │ ├── IFoundryService.cs
>
> │ │ │ ├── FoundryService.cs
>
> │ │ │ ├── ISpeechService.cs
>
> │ │ │ ├── AzureSpeechService.cs
>
> │ │ │ ├── IApplicationStorageService.cs
>
> │ │ │ ├── IndexedDbStorageService.cs
>
> │ │ │ └── ISyncService.cs
>
> │ │ ├── Pages/
>
> │ │ │ ├── Dashboard.razor
>
> │ │ │ ├── Application.razor
>
> │ │ │ └── SubmittedApplications.razor
>
> │ │ ├── Components/
>
> │ │ │ ├── ApplicationShell/
>
> │ │ │ │ ├── ProgressBar.razor
>
> │ │ │ │ ├── SectionTabs.razor
>
> │ │ │ │ └── QuestionRenderer.razor
>
> │ │ │ ├── AIPanel/
>
> │ │ │ │ ├── AssistantPanel.razor
>
> │ │ │ │ ├── TranscriptDisplay.razor
>
> │ │ │ │ └── MicButton.razor
>
> │ │ │ └── Shared/
>
> │ │ └── Models/
>
> │ │ ├── ApplicationStateDocument.cs
>
> │ │ ├── Question.cs
>
> │ │ ├── NavigationResult.cs
>
> │ │ └── FieldPatch.cs
>
> │ │
>
> │ ├── Redhawk.Api/ (.NET 10 minimal API)
>
> │ │ ├── Endpoints/
>
> │ │ │ ├── RulesEndpoints.cs (GET /api/rules/version, GET
> /api/rules/{type})
>
> │ │ │ └── ApplicationsEndpoints.cs (POST /api/applications/submit)
>
> │ │ └── Services/
>
> │ │
>
> │ └── Redhawk.Shared/ (shared models, interfaces)
>
> │ ├── Models/
>
> │ └── Contracts/
>
> │
>
> ├── foundry/ (Azure AI Foundry Prompt Flow)
>
> │ ├── flow.dag.yaml
>
> │ ├── nodes/
>
> │ │ ├── input_validation.py
>
> │ │ ├── intent_classification.py
>
> │ │ ├── context_assembly.py
>
> │ │ ├── llm_extraction.jinja2
>
> │ │ ├── confidence_gate.py
>
> │ │ └── response_shaping.py
>
> │ └── prompts/
>
> │ └── system_prompt.md
>
> │
>
> └── rules/ (RulesEngine workflow JSON)
>
> ├── TermLife_v1.json
>
> └── questions/
>
> └── TermLife_questions.json

**9. Prototype Considerations**

The following improvements are scoped for the 2-week proof of concept.
Each is small in effort but prevents significant problems during the
build or demo. Items deferred to production are listed separately.

**9.1 Implement During Prototype (\~4-5 hours total)**

  -----------------------------------------------------------------------
  **Item**               **Detail**
  ---------------------- ------------------------------------------------
  **Patch Validation     The AI will occasionally return the wrong type
  Layer**                for a field (e.g. string \'no\' for a boolean
                         field). A FieldPatchValidator must check type
                         compatibility before any patch is applied to
                         state. Without this, state corruption is silent
                         and hard to debug. Effort: \~1 hour.

  **schemaVersion        Add schemaVersion: 1 to the state document now.
  Field**                It never needs to be used during the prototype
                         but retrofitting migration support later without
                         it is painful. Effort: 5 minutes.

  **AI Timeout + Manual  If the Foundry call stalls, the UI must not
  Fallback**             freeze. Wrap IFoundryService.ExtractAsync in a
                         3-second timeout. On timeout, surface the manual
                         input field and log a warning. Agents must
                         always be able to continue. Effort: \~30
                         minutes.

  **IndexedDB Write      Every Fluxor state mutation currently triggers
  Debounce**             SaveAsync. At peak this is dozens of writes per
                         minute. Add a 500ms debounce to the storage
                         effect so rapid patch sequences coalesce into a
                         single write. Effort: \~30 minutes.

  **Basic Structured     Log AI extraction results, rule evaluations, and
  Logging**              navigation decisions to the browser console in a
                         consistent format. During prototype debugging
                         this saves hours of guesswork. Effort: \~20
                         minutes.

  **Undo Last Action**   Agents will mis-speak answers. Support
                         UndoLastPatchAction by maintaining a shallow
                         previousStateStack in the Fluxor store (last 5
                         states). A single Ctrl+Z or UI button restores
                         the prior state and re-evaluates navigation.
                         Effort: \~1-2 hours.

  **Developer Debug      A toggleable panel (activated via ?debug=true
  Panel**                query param) showing current question ID, field
                         path, last AI extraction JSON, and last
                         NavigationResult. Invaluable during demo setup
                         and debugging. Hide completely in production
                         builds. Effort: \~1 hour.
  -----------------------------------------------------------------------

**9.2 Patch Validator Implementation**

> public class FieldPatchValidator
>
> {
>
> public bool Validate(FieldPatch patch, Question question)
>
> {
>
> if (patch.FieldPath != question.FieldPath) return false;
>
> return question.Type switch
>
> {
>
> FieldType.Boolean =\> patch.Value is bool,
>
> FieldType.Currency =\> patch.Value is decimal or int,
>
> FieldType.Date =\> patch.Value is DateTime,
>
> FieldType.Enum =\>
> question.EnumValues?.Contains(patch.Value?.ToString()) ?? false,
>
> \_ =\> true
>
> };
>
> }
>
> }

**9.3 Undo State Stack**

> // In ApplicationsState
>
> public record ApplicationEntry
>
> {
>
> public ApplicationStateDocument Document { get; init; }
>
> public ImmutableStack\<ApplicationStateDocument\> UndoStack { get;
> init; } // max depth 5
>
> // \...
>
> }
>
> // UndoAction reducer
>
> public static ApplicationsState Reduce(ApplicationsState state,
> UndoLastPatchAction action)
>
> {
>
> var entry = state.Applications\[action.ApplicationId\];
>
> if (entry.UndoStack.IsEmpty) return state;
>
> var previous = entry.UndoStack.Peek();
>
> return state with { Applications = state.Applications.SetItem(
>
> action.ApplicationId,
>
> entry with { Document = previous, UndoStack = entry.UndoStack.Pop() }
>
> )};
>
> }

**9.4 Deferred to Production**

The following are good ideas but out of scope for a 2-week proof of
concept. Do not implement these during the prototype sprint:

  -----------------------------------------------------------------------
  **Feature**            **Reason Deferred**
  ---------------------- ------------------------------------------------
  **Event Log /          Per-session audit timeline. Useful for
  Timeline**             analytics, not needed for POC.

  **Multi-Device Sync    Unlikely scenario in a single-agent prototype
  Conflict Resolution**  demo.

  **Rule Dependency      Partial rule evaluation. Question graph will be
  Graph Optimization**   small enough that full evaluation is fast.

  **Streaming STT**      Adds integration complexity. Current
                         request-response STT is sufficient for POC.

  **Prompt Injection /   Important in production. Risk is negligible for
  Guardrail Layer**      an internal prototype.

  **AI Section           Nice UX improvement, not demo-critical.
  Summaries**            
  -----------------------------------------------------------------------

**10. Build Order**

Each step produces a demoable system. Do not skip steps or combine them.
Prototype items from Section 9 are noted inline --- implement them as
part of the step they belong to, not as a separate pass.

  -----------------------------------------------------------------------
  **Step**               **Deliverable**
  ---------------------- ------------------------------------------------
  **Step 1**             Auth + Dashboard Shell. Entra ID login with
                         MSAL.js. Blazor WASM project with Fluxor wired
                         up. IndexedDB storage service with 500ms write
                         debounce (Section 9.1). Agent can create, open,
                         delete applications. Add schemaVersion: 1 to
                         state document.

  **Step 2**             State Document + Manual Form. Fluxor store fully
                         wired. Question renderer bound to state. Manual
                         field input dispatches ManualFieldEditAction.
                         Undo stack implemented (Section 9.3). Rules
                         engine loads from hardcoded JSON. Progressive
                         reveal UI working.

  **Step 3**             Rules Engine Navigation. IRulesEngineService
                         implemented with Microsoft.RulesEngine.
                         Navigation loop evaluates full state.
                         NextQuestion drives form progression. Skip logic
                         working. Add structured console logging for rule
                         evaluation (Section 9.1). Developer debug panel
                         wired up (Section 9.1).

  **Step 4**             Foundry Text Extraction. IFoundryService
                         implemented with 3-second timeout and manual
                         fallback (Section 9.1). FieldPatchValidator
                         added to patch application pipeline (Section
                         9.2). Text input sends to Foundry. Multi-field
                         eager extraction working.

  **Step 5**             Azure Speech STT/TTS. ISpeechService wraps Azure
                         Speech SDK. Mic button in UI. Voice loop
                         end-to-end. Interim transcript display. Graceful
                         offline degradation.

  **Step 6**             Encryption + Rules Version Gate. Web Crypto API
                         encryption of IndexedDB. Rules version check on
                         load. Sync service for submitted applications.
  -----------------------------------------------------------------------

**11. NuGet & NPM Dependencies**

  -------------------------------------------------------------------------------------------------
  **Package**                                 **Source**                 **Purpose**
  ------------------------------------------- -------------------------- --------------------------
  Microsoft.RulesEngine                       NuGet                      Client-side rules
                                                                         evaluation in Blazor WASM

  Fluxor.Blazor.Web                           NuGet                      Redux state management for
                                                                         Blazor

  Microsoft.Authentication.WebAssembly.Msal   NuGet                      Entra ID auth in WASM

  Blazored.LocalStorage                       NuGet                      IndexedDB / storage
                                                                         abstraction

  Microsoft.CognitiveServices.Speech          NPM (JS SDK)               Azure STT/TTS --- loaded
                                                                         via JS interop

  \@microsoft/fetch-event-source              NPM                        SSE streaming for speech
                                                                         if needed
  -------------------------------------------------------------------------------------------------

**12. Configuration & Environment**

> // appsettings.json (Redhawk.Client)
>
> {
>
> \"AzureAd\": {
>
> \"Authority\": \"https://login.microsoftonline.com/{tenantId}\",
>
> \"ClientId\": \"{client-id}\",
>
> \"ValidateAuthority\": true
>
> },
>
> \"Foundry\": {
>
> \"Endpoint\": \"https://{foundry-endpoint}.azureml.ms/\...\",
>
> \"ApiKey\": \"\" // use managed identity in production
>
> },
>
> \"Speech\": {
>
> \"SubscriptionKey\": \"\",
>
> \"Region\": \"eastus\"
>
> },
>
> \"Api\": {
>
> \"BaseUrl\": \"https://redhawk-api.azurewebsites.net\"
>
> }
>
> }

*Redhawk --- AI Agent Build Specification v1.0 --- Proof of Concept
Scope*
