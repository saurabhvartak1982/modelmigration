# GenAI Model Migration Approach (Microsoft Foundry)

<b>Disclaimer:</b> Views are personal. </br>

## A) Discovery: Understand the application and current model usage

1. **Workload type & modality needs**
   - What does the app do (chat/Q&A, RAG, extraction, summarization, coding assistant, agentic orchestration)?
   - What modalities are used/required (text, vision, audio, etc.)?

2. **Interaction pattern**
   - Stateless single-turn vs multi-turn conversation (history included?)
   - RAG vs non-RAG
   - Tool/function calling or not (and how many tools)

3. **Architecture & rollout feasibility**
   - Can you run parallel deployments (blue/green, canary, A/B)?
   - Where can traffic be split (app layer, APIM, gateway, service mesh)?

4. **Token profile and context window fit**
   - Current p50/p95/max input tokens (prompt + history + RAG docs + tool schemas)
   - Current output length and max_output_tokens / max_tokens usage
   - Validate: effective output = min(max_output, context_window − input_tokens)

5. **Output contract requirements**
   - Do you rely on "prompted JSON" (format in instructions) or strict structured outputs?
   - Any schema validation in code? (required keys, enums, ordering assumptions, etc.)

6. **Model + deployment constraints**
   - Current model family and candidate target model(s)
   - Deployment type: Standard/PTU/Batch + Regional/ Global / Data Zone
   - Region/location boundaries and compliance requirements
   - Quota posture (TPM/RPM) and headroom

7. **API surface / URL pattern used by the app**
   - Legacy: .../openai/deployments/{deployment}/... ?api-version=YYYY-MM-DD
   - v1 GA: .../openai/v1/... where api-version is not required
   - Is api-version hardcoded in URL, config, or SDK parameter?

8. **How deployment name is referenced**
   - Is deployment name provided as a parameter/config (SDK builds URL), or embedded directly in URL/base_url? - This determines how big the code/config change is.

9. **Baseline quality, performance, and known issues**
   - How was the current model validated? Any golden dataset? edge cases?
   - How was the earlier model and application performance tested?
   - Any production pain points (latency, cost spikes, JSON validity, hallucinations, tool-call mistakes)?
   - Any reliability incidents tied to input size/tool schemas?

---

## B) Model delta analysis (Source vs Target) — fill this for any migration

Create a short "delta sheet" for your source and target models:

1. **Capabilities**
   - Instruction following / reasoning style differences
   - Coding/tool-use differences (if relevant)
   - Safety behaviour differences (if relevant)

2. **Limits**
   - Context window, input capacity and max output tokens
   - Known limitations or special constraints

3. **Cost and rate-limit implications**
   - Pricing changes (input/output token pricing; long-context tiering if applicable)
   - Quota, latency and throughput differences (TPM/RPM and how you allocate them)

4. **Knowledge cutoff differences (if relevant to your use case)**

5. **Expected impact summary**
   - What could change functionally (edge cases, formatting strictness, tool selection, verbosity)
   - What must be retested because of the above

---

## C) Deployment Strategy

1. Create a new deployment for the target model (do not replace in-place).
2. Ensure quota allocation supports parallel testing:
   - If using PTU, plan capacity split; if standard, ensure quota headroom.
   - If the same model deployment is to be used as the "judge" model for AI-assisted evaluations, then plan the appropriate time for running the evaluations (e.g.: during off-peak hours).
3. Keep rollback easy: don’t delete the old deployment until cutover is stable.

---

## D) Evaluation & testing (make it repeatable)

Split into two layers:

### D1) Functional testing (humans + automated assertions)

- Functional correctness (workflow-specific, including edge cases)
- Schema correctness (JSON validity, required fields, enum values, parsing robustness)
- Tool/function calling correctness (tool selection + parameter correctness)
- Token/request drift (input and output token changes)
- Safety behaviour where relevant

### D2) Foundry Evaluations as regression gates (automated + scalable)

Use evaluations as a repeatable “quality gate” alongside functional testing:

- For Q&A/RAG: QAEvaluator (Relevance, Groundedness, Fluency, Coherence, Similarity, F1)
- For agents/tools: ToolCallAccuracyEvaluator (relevance, parameter correctness, etc.)
- Run evaluations locally and track in Foundry via the Azure AI Evaluation SDK workflow.
- Treat eval runs as comparable baselines (source vs target), and define thresholds.

**Golden dataset guidance**  
Label expected behaviours explicitly (examples):

- “must call tool X”
- “must not hallucinate beyond retrieved context”
- “must return schema-valid JSON”
- “must refuse / safe-complete”

---

## Model-specific notes

If the target model has known issues (e.g., tool schema size limits), add targeted tests. 

---

## Observability

Observability (including Evaluations) is part of GenAIOps; use it during test and after release.

---

## E) Performance and cost validation

- Load test with realistic concurrency and prompt sizes
- Track p50/p95 latency, error rate, retries, and timeouts
- Track cost drivers: token/request drift and any long-context usage patterns

---

## F) Production rollout (operationalizing)

1. **Traffic shifting mechanism**
   - In-app routing (percentage or stable hashing)
   - APIM weighted routing
   - Mesh/canary at service layer

2. **Rollout plan**
   - 1% → 5% → 25% → 50% → 100% with “hold points” tied to metrics/evals

3. **Post-cutover continuous evaluation**
   - Periodically run Foundry evals to detect drift/regression

4. **Rollback conditions**
   - invalid JSON spike
   - tool-call accuracy drop
   - p95 latency breach
   - token/request or cost spike
   - elevated error rates (429/5xx)

## Acknowledgements
Thank you Prafulla (https://github.com/prwani) for inputs 

