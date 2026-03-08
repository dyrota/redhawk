**Redhawk**

Redhawk

*Team Reference Overview*

+-----------------------------------+-----------------------------------+
| **Platform**                      | **Deployment**                    |
|                                   |                                   |
| Blazor WASM PWA                   | Web (cross-platform)              |
+-----------------------------------+-----------------------------------+
| **Users**                         | **AI Provider**                   |
|                                   |                                   |
| Insurance Agents (agent-facing)   | Azure AI Foundry (GPT-4o)         |
+-----------------------------------+-----------------------------------+
| **Auth**                          | **Speech**                        |
|                                   |                                   |
| Microsoft Entra ID                | Azure Cognitive Services          |
+-----------------------------------+-----------------------------------+

**1. What We Are Building**

Redhawk is a Progressive Web Application that allows insurance agents to
complete life insurance applications with customers. Agents log in with
their company credentials and work through application forms that adapt
dynamically based on the customer\'s answers.

The system has two modes: when online, a voice-enabled AI assistant
helps the agent fill out the form by listening to responses and
extracting answers automatically. When offline, the form works entirely
without a connection using locally cached rules, and syncs when
connectivity is restored.

**2. Technology Stack**

  -------------------------------------------------------------------------
  **Layer**        **Technology**           **Purpose**
  ---------------- ------------------------ -------------------------------
  Frontend         Blazor WebAssembly (.NET Runs in browser, true offline
                   10)                      via PWA service worker

  State Management Fluxor (Redux for        Single controlled state
                   Blazor)                  mutation path for all consumers

  Rules Engine     Microsoft RulesEngine    Client-side, evaluates
                   (NuGet)                  underwriting/navigation logic
                                            offline

  Local Storage    IndexedDB (via JS        Persists in-progress
                   interop)                 applications encrypted on
                                            device

  AI Orchestration Azure AI Foundry Prompt  Manages LLM calls, context
                   Flow                     assembly, response shaping

  AI Model         GPT-4o (via Foundry)     Extracts structured field
                                            values from voice/text
                                            utterances

  Speech STT       Azure Cognitive Services Converts agent speech to text
                   Speech                   transcript

  Speech TTS       Azure Neural TTS         Reads AI assistant responses
                                            aloud

  Authentication   Microsoft Entra ID +     Agent identity and session
                   MSAL.js                  management

  Backend API      .NET 10 Minimal API      Rules versioning endpoint,
                                            application submission

  Encryption       Web Crypto API (AES-GCM) PII encrypted at rest in
                                            IndexedDB
  -------------------------------------------------------------------------

**3. System Architecture**

**3.1 The Central Concept --- Shared State Document**

Every part of the system reads from and writes to a single JSON
application state document. Three consumers interact with it through
controlled paths:

  -------------------------------------------------------------------------
  **Consumer**   **Role**                                **When Active**
  -------------- --------------------------------------- ------------------
  Rules Engine   Reads full state. Determines the next   Always (online +
                 question and which fields are relevant  offline)
                 context for the AI. May skip questions  
                 or entire sections based on prior       
                 answers.                                

  AI Assistant   Reads only the relevant context slice   Online only
                 the rules engine selects --- never the  
                 full document. Proposes field value     
                 patches from the agent\'s speech.       

  Blazor UI      Reads full state via Fluxor store.      Always
                 Re-renders reactively whenever state    
                 changes, regardless of what caused the  
                 change.                                 
  -------------------------------------------------------------------------

**3.2 Online Mode Flow**

-   Agent presses mic button → Azure STT captures speech → transcript
    sent to Azure AI Foundry

-   Foundry extracts field values from transcript and returns a patch
    proposal

-   Patches applied to state document → Rules engine re-evaluates → next
    question selected

-   UI updates reactively, AI speaks the next question via Azure TTS

**3.3 Offline Mode Flow**

-   Agent enters answers manually via form fields

-   Each answer dispatches through Fluxor → Rules engine evaluates
    client-side

-   Next question determined from cached rules bundle --- no server
    needed

-   Completed applications queue locally and sync when connectivity
    returns

**3.4 What the AI Receives**

To minimize token usage, the AI never receives the full state document.
The rules engine selects only the fields relevant to interpreting the
current question (e.g., the applicant\'s first name for personalization,
and any upstream answers that contextualize the current one). This
context slice is assembled per-question and sent alongside the current
question definition and recent conversation history.

**4. Key Features**

**4.1 Dynamic Form Navigation**

Questions are not a flat list --- they form a graph. The rules engine
evaluates prior answers and may skip questions or entire sections. For
example, a non-smoker skips all tobacco follow-up questions
automatically. An agent answering multiple questions in one sentence can
skip several steps at once.

**4.2 Progressive Reveal UI**

Answered questions remain visible and editable as the form progresses.
Agents can scroll back and correct any answer, which triggers rules
re-evaluation and may change the navigation path forward. The current
unanswered question is highlighted.

**4.3 Multi-Application Management**

Agents may have multiple in-progress applications simultaneously --- one
per customer. The dashboard lists all in-progress and submitted
applications with completion percentage and last-modified date.
Applications are saved automatically and can be resumed at any time.

**4.4 Voice Assistant**

Available online only. The agent speaks to fill out the form --- the AI
extracts one or more field values from a single utterance. If the AI is
uncertain about an extracted value (low confidence), it asks a
clarifying question before writing to the form. The agent can also type
if preferred.

**4.5 Offline First**

Rules are cached on the client after first load. A version check on
reconnect fetches updated rules if the server has a newer version.
In-progress applications survive browser refresh and offline periods via
encrypted IndexedDB storage.

**5. Data & Privacy**

  -----------------------------------------------------------------------
  **Data Type**          **Handling**
  ---------------------- ------------------------------------------------
  Customer PII in        Encrypted at rest with AES-GCM. Key derived from
  IndexedDB              agent\'s session token. Unreadable without
                         active authenticated session.

  Data sent to AI        Only the relevant context slice (1-3 fields) +
                         current question + recent conversation. Full
                         state never leaves the client.

  Conversation history   Stored in the state document for audit. Older
                         turns summarized before sending to AI to
                         minimize context size.

  Submitted applications Transmitted to backend API over HTTPS after
                         agent confirms submission. Local copy retained
                         as read-only.

  Field audit trail      Every field records how it was populated: manual
                         entry, agent voice, or AI extraction. Required
                         for compliance.
  -----------------------------------------------------------------------

**6. Application Lifecycle**

  -----------------------------------------------------------------------
  **Status**         **Meaning**
  ------------------ ----------------------------------------------------
  InProgress         Active, editable. Saved locally. Agent can leave and
                     return.

  PendingSync        All questions answered, submitted by agent, waiting
                     for connectivity to transmit.

  Submitted          Transmitted to backend. Read-only. Visible in
                     submitted applications list.
  -----------------------------------------------------------------------

Note: Post-submission amendments are out of scope for the proof of
concept. Submitted applications cannot be edited.

**7. Proof of Concept Build Phases**

Each phase delivers a working, demoable system. The AI assistant is
additive and not required for earlier phases to function.

  ----------------------------------------------------------------------------
  **Phase**   **Name**        **Deliverable**
  ----------- --------------- ------------------------------------------------
  Phase 1     Auth +          Entra ID login. Agent dashboard.
              Dashboard       Create/open/delete applications. IndexedDB
                              storage wired up.

  Phase 2     Manual Form +   Fluxor store. Progressive reveal form. Manual
              State           field entry working end-to-end.

  Phase 3     Rules Engine    Microsoft RulesEngine integrated. Question
              Navigation      skipping and section navigation working offline.

  Phase 4     AI Text         Foundry Prompt Flow. Text input (type) sends to
              Extraction      AI, patches returned and applied. Multi-field
                              extraction.

  Phase 5     Voice Pipeline  Azure STT/TTS integrated. Full voice loop. Mic
                              button UI. Offline graceful degradation.

  Phase 6     Encryption +    PII encrypted in IndexedDB. Rules version gate.
              Sync            Application sync on reconnect.
  ----------------------------------------------------------------------------

**8. Prototype Considerations**

The following small items (\~4-5 hours total effort) are included in the
proof of concept scope. Each prevents a meaningful problem during the
build or demo without adding significant complexity.

  -----------------------------------------------------------------------------
  **Item**           **Effort**   **Purpose**
  ------------------ ------------ ---------------------------------------------
  Patch Validation   \~1 hour     Validates AI-returned field values match
                                  expected types before writing to state.
                                  Prevents silent state corruption.

  Schema Version     5 min        Add schemaVersion: 1 to the state document.
  Field                           No logic needed now --- makes future
                                  migrations possible.

  AI Timeout +       \~30 min     3-second timeout on Foundry calls. On
  Fallback                        timeout, surface manual input. Agents must
                                  always be able to continue.

  IndexedDB Write    \~30 min     Coalesce rapid state mutations into a single
  Debounce                        500ms debounced write. Prevents excessive
                                  storage churn.

  Structured Logging \~20 min     Console log AI extraction results, rule
                                  evaluations, and navigation decisions in
                                  consistent format.

  Undo Last Answer   \~1-2 hours  Maintain a shallow undo stack (last 5 states)
                                  in Fluxor. Allows agents to correct
                                  mis-spoken answers instantly.

  Developer Debug    \~1 hour     Toggle with ?debug=true. Shows current
  Panel                           question, last AI result, and
                                  NavigationResult. Hidden in production.
  -----------------------------------------------------------------------------

The following are explicitly deferred to production: event log,
multi-device sync conflict resolution, streaming STT, prompt injection
guardrails, and AI section summaries.

*Redhawk Team Reference v1.0 --- Proof of Concept Scope*
