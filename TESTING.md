# AFv2 Pattern #3: Routing - Test Cases

## Overview

**Pattern:** Intent-based routing to domain-specific agents with confidence threshold validation
**Flow:** Start → Router → [Billing | Technical | General | FAIL] → Synthesize → Direct Reply
**Repository:** https://github.com/snedea/afv2-pattern-03-routing

---

## Prerequisites

### 1. Import Pattern into Flowise

1. Open Flowise UI (http://localhost:3000)
2. Navigate to **Agentflows** section
3. Click **"Add New"** → **"Import Agentflow"**
4. Upload `03-routing.json`
5. Pattern should load with 10 nodes:
   - Start
   - Intent Router (ConditionAgent)
   - Agent.Billing
   - Agent.Technical
   - Agent.General
   - Agent.FAIL
   - Agent.Synthesize
   - Direct Reply
   - 3 Sticky Notes (documentation)

### 2. Configure API Keys

**All 6 agents require Anthropic API key configuration:**

1. Click on **Intent Router** node
2. In the **Model** dropdown, select model configuration:
   - **Credential:** Select your "Anthropic API Key" credential
   - **Model Name:** `claude-sonnet-4-5-20250929`
   - **Temperature:** `0.1` (for Router - deterministic)
   - **Streaming:** `true` (default)

3. Repeat for domain agents (temperature: 0.3):
   - **Agent.Billing**
   - **Agent.Technical**
   - **Agent.General**
   - **Agent.FAIL**

4. Repeat for synthesis agents (temperature: 0.2):
   - **Agent.Synthesize**

5. Save the workflow

### 3. Verify Node Configuration

**Expected Configuration:**

| Node | Type | Tools | State Updates | Key Feature |
|------|------|-------|---------------|-------------|
| Intent Router | ConditionAgent | N/A | Produces routing decision JSON | Classifies intent with confidence score |
| Agent.Billing | Agent | `currentDateTime`, `calculator` | `domain.response`, `domain.name="Billing"` | Handles billing queries |
| Agent.Technical | Agent | `currentDateTime` | `domain.response`, `domain.name="Technical"` | Handles technical issues |
| Agent.General | Agent | `currentDateTime` | `domain.response`, `domain.name="General"` | Handles general queries |
| Agent.FAIL | Agent | N/A | `domain.response`, `domain.name="FAIL"` | Handles invalid input |
| Agent.Synthesize | Agent | `currentDateTime` | `output` | Formats final response |
| Direct Reply | DirectReply | N/A | N/A | Terminal node (hideOutput: true) |

**Router Configuration:**
- **Confidence Threshold:** 0.6
- **Output Format:** JSON with `{route, confidence, rationale, alternates}`
- **Routes:** Billing (0), Technical (1), General (2), FAIL (3)

---

## Test Cases

### TC-3.1: High-Confidence Routing to Domain Agent

**Objective:** Verify Router correctly routes to domain agent when confidence ≥ 0.6

#### Test 3.1a: Billing Domain Routing

**Input:**
```
I was charged twice for my subscription this month. Can you explain the duplicate charge and issue a refund?
```

**Expected Execution Flow:**

1. **Intent Router Classifies:**
   - Analyzes input for intent
   - Identifies billing keywords: "charged", "subscription", "refund"
   - Produces routing decision:
     ```json
     {
       "route": "Billing",
       "confidence": 0.95,
       "rationale": "Clear billing inquiry with payment/refund request",
       "alternates": [
         {"route": "General", "confidence": 0.04}
       ]
     }
     ```
   - **Decision:** Confidence 0.95 ≥ 0.6 → Route to Billing path (output 0)

2. **Agent.Billing Executes:**
   - Receives routed question
   - Uses `calculator` tool for refund calculations
   - Uses `currentDateTime` to check billing period
   - Produces billing-specific response
   - Updates state:
     ```json
     {
       "domain.response": "[billing response with refund details]",
       "domain.name": "Billing"
     }
     ```

3. **Agent.Synthesize Executes:**
   - Reads `domain.response` and `domain.name` from state
   - Formats final response with domain context
   - Updates state: `output` with formatted response

4. **Direct Reply Returns:**
   - Displays final response to user

**Validation Checklist:**

- [ ] **Router Classification:**
  - Router output confidence ≥ 0.6 (should be ~0.95)
  - Router output route = "Billing"
  - Router output includes rationale
  - Router output includes alternates list

- [ ] **Routing Decision:**
  - Router selected output anchor 0 (Billing path)
  - Agent.Billing executed (NOT Technical/General/FAIL)
  - **CRITICAL:** Only Billing agent executed (verify others did NOT run)

- [ ] **Domain Agent Execution:**
  - Agent.Billing used `calculator` tool (if calculations needed)
  - Agent.Billing used `currentDateTime` tool
  - State contains `domain.response` with billing details
  - State contains `domain.name="Billing"`

- [ ] **Synthesize Execution:**
  - Agent.Synthesize accessed `domain.response` from state
  - Agent.Synthesize included domain context (e.g., "Billing Support:")
  - State contains `output` with formatted response

- [ ] **Response Quality:**
  - Response addresses billing concern
  - Response includes refund process details
  - Response uses billing-specific terminology

**Success Criteria:**
- Router confidence ≥ 0.6 for Billing route
- ONLY Agent.Billing executed (other agents skipped)
- Response includes billing-specific details
- Workflow completed via Billing path only

---

#### Test 3.1b: Technical Domain Routing

**Input:**
```
I'm getting a 500 Internal Server Error when calling the /api/users endpoint. The error started this morning. Here's the stack trace: [stack trace].
```

**Expected Execution Flow:**

1. **Intent Router Classifies:**
   - Identifies technical keywords: "500 error", "API", "endpoint", "stack trace"
   - Produces routing decision:
     ```json
     {
       "route": "Technical",
       "confidence": 0.92,
       "rationale": "API error with technical details (500 error, endpoint, stack trace)",
       "alternates": [
         {"route": "General", "confidence": 0.06}
       ]
     }
     ```
   - **Decision:** Confidence 0.92 ≥ 0.6 → Route to Technical path (output 1)

2. **Agent.Technical Executes:**
   - Analyzes error details
   - Uses `currentDateTime` to correlate with deployment timeline
   - Produces technical troubleshooting response
   - Updates state: `domain.response`, `domain.name="Technical"`

3. **Agent.Synthesize → Direct Reply**

**Validation Checklist:**

- [ ] Router confidence ≥ 0.6 (~0.92)
- [ ] Router route = "Technical"
- [ ] ONLY Agent.Technical executed
- [ ] State contains `domain.name="Technical"`
- [ ] Response includes technical troubleshooting steps
- [ ] Response addresses API error specifically

**Success Criteria:**
- Correct routing to Technical domain
- Response includes API debugging guidance
- No execution of Billing/General/FAIL agents

---

### TC-3.2: Low-Confidence Fallback to General

**Objective:** Verify Router routes to General when confidence < 0.6

**Input:**
```
What are your business hours and office locations?
```

**Expected Execution Flow:**

1. **Intent Router Classifies:**
   - Analyzes input
   - Not clearly Billing (no payment/invoice keywords)
   - Not clearly Technical (no error/bug keywords)
   - Could be General, but confidence below threshold
   - Produces routing decision:
     ```json
     {
       "route": "General",
       "confidence": 0.45,
       "rationale": "General information query, but low specificity",
       "alternates": [
         {"route": "Billing", "confidence": 0.28},
         {"route": "Technical", "confidence": 0.15}
       ]
     }
     ```
   - **Decision:** Confidence 0.45 < 0.6 → Route to General path (output 2) as fallback

2. **Agent.General Executes:**
   - Handles general information query
   - Produces general-purpose response
   - Updates state:
     ```json
     {
       "domain.response": "[general info response]",
       "domain.name": "General"
     }
     ```

3. **Agent.Synthesize → Direct Reply**

**Validation Checklist:**

- [ ] **Low Confidence Detection:**
  - Router output confidence < 0.6 (e.g., 0.45)
  - Router still produces route decision (General)
  - Router includes alternate routes with scores

- [ ] **Fallback Routing:**
  - Router selected output anchor 2 (General path)
  - Agent.General executed
  - **CRITICAL:** General agent used as fallback (not FAIL)

- [ ] **General Agent Execution:**
  - Agent.General provided helpful response
  - State contains `domain.name="General"`
  - Response doesn't claim specialized domain expertise

- [ ] **Transparency:**
  - Router metadata tracks low confidence (visible in logs/state)
  - Alternate routes documented for debugging

**Success Criteria:**
- Router confidence < 0.6 triggers General fallback
- ONLY Agent.General executed
- Response is helpful despite low confidence
- No false routing to Billing/Technical

**Confidence Threshold Validation:**

```
Routing Logic:

High Confidence (≥ 0.6):
  Billing query (0.95) → Billing Agent  ✅
  Technical query (0.92) → Technical Agent  ✅

Low Confidence (< 0.6):
  General query (0.45) → General Agent (fallback)  ✅
  Ambiguous query (0.55) → General Agent (fallback)  ✅

Invalid Input:
  Gibberish (N/A) → FAIL Agent  ✅
```

---

### TC-3.3: FAIL Path for Invalid Input

**Objective:** Verify Router correctly routes invalid/unroutable input to FAIL path

**Input (Test 3.3a - Gibberish):**
```
asdfghjkl qwertyuiop zxcvbnm 12345
```

**Expected Execution Flow:**

1. **Intent Router Attempts Classification:**
   - Analyzes input
   - Detects no recognizable keywords or structure
   - Cannot classify into any domain
   - Produces routing decision:
     ```json
     {
       "route": "FAIL",
       "confidence": 0.0,
       "rationale": "Unrecognizable input (gibberish, no valid keywords)",
       "alternates": []
     }
     ```
   - **Decision:** Cannot route → FAIL path (output 3)

2. **Agent.FAIL Executes:**
   - Handles invalid input gracefully
   - Produces helpful error message
   - Updates state:
     ```json
     {
       "domain.response": "I'm unable to understand your request. Please provide a clear question about billing, technical issues, or general information.",
       "domain.name": "FAIL"
     }
     ```

3. **Agent.Synthesize → Direct Reply:**
   - Formats error response
   - Provides guidance for valid input

**Validation Checklist:**

- [ ] **FAIL Detection:**
  - Router identified input as unroutable
  - Router route = "FAIL"
  - Router confidence = 0.0 or very low
  - Router rationale explains why input is invalid

- [ ] **FAIL Path Execution:**
  - Router selected output anchor 3 (FAIL path)
  - Agent.FAIL executed
  - **CRITICAL:** No domain agent (Billing/Technical/General) executed

- [ ] **Error Handling:**
  - Agent.FAIL provided helpful error message
  - Error message suggests valid input types
  - State contains `domain.name="FAIL"`
  - No stack traces or technical errors shown to user

- [ ] **User Experience:**
  - Error message is polite and constructive
  - Provides examples of valid queries
  - Doesn't expose system internals

**Success Criteria:**
- Gibberish input correctly routed to FAIL
- ONLY Agent.FAIL executed
- User receives helpful error message
- No false routing to domain agents

---

**Input (Test 3.3b - Off-Topic):**
```
What's the weather like in Paris today?
```

**Expected Execution Flow:**

1. **Intent Router Classifies:**
   - Detects off-topic query (weather, not billing/technical/general company info)
   - Produces routing decision:
     ```json
     {
       "route": "FAIL",
       "confidence": 0.0,
       "rationale": "Off-topic query (weather), not related to billing/technical/company information",
       "alternates": []
     }
     ```
   - **Decision:** Off-topic → FAIL path

2. **Agent.FAIL Executes:**
   - Politely declines to answer off-topic query
   - Redirects to appropriate domain
   - Updates state: `domain.response`, `domain.name="FAIL"`

**Validation Checklist:**

- [ ] Router identified query as off-topic
- [ ] Router route = "FAIL"
- [ ] Agent.FAIL provided redirection guidance
- [ ] Error message is polite (not abrupt)
- [ ] No domain agents executed

**Success Criteria:**
- Off-topic query routed to FAIL
- User receives polite redirection
- System boundaries clearly communicated

---

**Input (Test 3.3c - Empty Input):**
```
[empty string or whitespace only]
```

**Expected Execution Flow:**

1. **Intent Router Detects Empty Input:**
   - Input is empty or contains only whitespace
   - Cannot classify
   - Produces: `route="FAIL"`, `confidence=0.0`

2. **Agent.FAIL → Synthesize → Direct Reply**

**Validation Checklist:**

- [ ] Empty input routed to FAIL
- [ ] Error message requests valid input
- [ ] No crash or error thrown

---

## Common Issues & Debugging

### Issue 1: Router Always Routes to General (Ignoring Confidence)

**Symptoms:**
- All queries routed to General agent
- Billing/Technical agents never execute
- Confidence scores appear correct but routing is wrong

**Debugging Steps:**
1. Verify Router output anchor connections:
   - Output 0 should connect to Agent.Billing
   - Output 1 should connect to Agent.Technical
   - Output 2 should connect to Agent.General
   - Output 3 should connect to Agent.FAIL

2. Check `edges` array in JSON:
   ```json
   [
     { "source": "router_condition", "sourceHandle": "router_condition-output-0", "target": "agent_billing" },
     { "source": "router_condition", "sourceHandle": "router_condition-output-1", "target": "agent_technical" },
     { "source": "router_condition", "sourceHandle": "router_condition-output-2", "target": "agent_general" },
     { "source": "router_condition", "sourceHandle": "router_condition-output-3", "target": "agent_fail" }
   ]
   ```

3. Verify Router scenarios match output order:
   - Scenario 0: Billing
   - Scenario 1: Technical
   - Scenario 2: General
   - Scenario 3: FAIL

**Solution:**
- Ensure output anchors connect to correct agents
- Verify scenario order matches output anchor order
- Re-import pattern if edges are corrupted

---

### Issue 2: Router Confidence Always Below Threshold

**Symptoms:**
- Clear Billing/Technical queries routed to General
- Confidence scores consistently < 0.6 even for obvious queries
- Router rationale seems confused

**Debugging Steps:**
1. Check Router instructions:
   - Should include domain keywords
   - Should specify confidence threshold (0.6)
   - Should provide clear classification criteria

2. Verify Router temperature:
   - Should be `0.1` (deterministic)
   - Higher temperatures may cause inconsistent classification

3. Test with explicit queries:
   ```
   INPUT: "I need a refund for invoice #12345"
   EXPECTED: route="Billing", confidence ≥ 0.8
   ```

**Solution:**
- Update Router instructions with clearer keywords
- Set temperature to 0.1 for consistent classification
- Add examples to Router instructions

---

### Issue 3: Multiple Domain Agents Execute for Single Query

**Symptoms:**
- Billing AND Technical agents both execute
- Multiple `domain.response` values in state
- Conflicting responses in final output

**Debugging Steps:**
1. Verify ConditionAgent is being used (not parallel agents):
   - Router node type should be `ConditionAgent`
   - Should have 4 distinct output anchors (0, 1, 2, 3)
   - Only ONE output should activate per query

2. Check for duplicate edges:
   ```bash
   jq '.edges[] | select(.source == "router_condition")' 03-routing.json
   # Should show exactly 4 edges (one per output)
   ```

3. Verify no parallel execution:
   - Start should NOT have 4 outgoing edges to domain agents
   - Start should have 1 edge to Router only

**Solution:**
- Use ConditionAgent (not regular Agent) for Router
- Ensure only 4 edges from Router (one per output)
- Remove any duplicate or parallel edges

---

### Issue 4: FAIL Path Never Executes

**Symptoms:**
- Gibberish input routed to General instead of FAIL
- Off-topic queries handled by General
- Agent.FAIL never executes

**Debugging Steps:**
1. Check Router FAIL detection logic:
   - Instructions should specify FAIL conditions
   - Should detect gibberish, empty, off-topic input
   - Should output `route="FAIL"` for invalid input

2. Verify FAIL path connection:
   - Output 3 should connect to Agent.FAIL
   - Check edge exists: `router_condition → agent_fail`

3. Test with obvious FAIL input:
   ```
   INPUT: "asdfghjkl qwertyuiop"
   EXPECTED: route="FAIL"
   ```

**Solution:**
- Add explicit FAIL detection to Router instructions
- Verify edge connects output 3 to Agent.FAIL
- Test with gibberish to validate FAIL path

---

### Issue 5: Synthesize Agent Missing Domain Context

**Symptoms:**
- Final response doesn't indicate which domain handled query
- Generic response regardless of domain agent
- `domain.name` not included in output

**Debugging Steps:**
1. Verify domain agents update `domain.name`:
   - **Agent.Billing:** `domain.name="Billing"`
   - **Agent.Technical:** `domain.name="Technical"`
   - **Agent.General:** `domain.name="General"`
   - **Agent.FAIL:** `domain.name="FAIL"`

2. Check Agent.Synthesize reads from state:
   - Should access `{{ domain.name }}` variable
   - Should access `{{ domain.response }}` variable
   - Should format response with domain context

3. Verify memory enabled:
   ```json
   {
     "agentEnableMemory": true  // ✅ Required
   }
   ```

**Solution:**
- Add `domain.name` state update to all domain agents
- Configure Agent.Synthesize to read state variables
- Enable memory for all agents

---

## Test Execution Report

**Test Date:** `[YYYY-MM-DD]`
**Flowise Version:** `[version]`
**Pattern Version:** `1.0`

### Test Results Summary

| Test Case | Status | Duration | Notes |
|-----------|--------|----------|-------|
| TC-3.1a: Billing Routing | ⬜ Pass / ⬜ Fail | `[time]` | |
| TC-3.1b: Technical Routing | ⬜ Pass / ⬜ Fail | `[time]` | |
| TC-3.2: General Fallback | ⬜ Pass / ⬜ Fail | `[time]` | |
| TC-3.3a: FAIL (Gibberish) | ⬜ Pass / ⬜ Fail | `[time]` | |
| TC-3.3b: FAIL (Off-Topic) | ⬜ Pass / ⬜ Fail | `[time]` | |
| TC-3.3c: FAIL (Empty) | ⬜ Pass / ⬜ Fail | `[time]` | |

### Configuration Details

- **Anthropic Model:** `claude-sonnet-4-5-20250929`
- **Router Temperature:** `0.1` (deterministic)
- **Domain Agent Temperature:** `0.3`
- **Confidence Threshold:** `0.6`
- **Streaming:** `Enabled`
- **Memory Enabled:** `Yes`

### Router Performance Metrics

| Input Type | Expected Route | Actual Route | Confidence | Pass/Fail |
|------------|----------------|--------------|------------|-----------|
| Billing query | Billing | | | |
| Technical query | Technical | | | |
| General query | General | | | |
| Gibberish | FAIL | | | |
| Off-topic | FAIL | | | |
| Empty | FAIL | | | |

### Issues Encountered

`[List any issues encountered during testing]`

### Additional Notes

`[Any additional observations or recommendations]`

---

## Pattern Validation

This pattern has been validated against `validate_workflow.py` with the following checks:

✅ **Structural Validation:**
- Proper JSON structure with nodes and edges arrays
- All required node fields present
- All required edge fields present

✅ **Node Type Validation:**
- Start node present
- 1 ConditionAgent (Router) with 4 output anchors
- 4 domain Agent nodes (Billing, Technical, General, FAIL)
- 1 synthesis Agent node
- 1 DirectReply terminal node with hideOutput: true
- 3 StickyNote nodes for documentation

✅ **Routing Logic:**
- Router has 4 distinct output anchors (0, 1, 2, 3)
- Each output connects to correct domain agent
- Confidence threshold (0.6) enforced in Router instructions
- FAIL path exists for invalid input

✅ **State Management:**
- All agents have agentEnableMemory: true
- Domain agents update `domain.response` and `domain.name`
- Synthesize agent reads from state
- State tracking enables domain context in responses

✅ **Tool Configuration:**
- Billing: `currentDateTime`, `calculator` (for billing calculations)
- Technical: `currentDateTime`
- General: `currentDateTime`
- FAIL: No tools (error handling only)
- Synthesize: `currentDateTime`

✅ **Edge Validation:**
- Start → Router (1 edge)
- Router → Domain agents (4 edges, one per output)
- Domain agents → Synthesize (4 edges converge)
- Synthesize → Direct Reply (1 edge)

---

## Additional Resources

- **Pattern Library:** [Context Foundry AFv2 Patterns](https://github.com/snedea/afv2-patterns-index)
- **Pattern #3 Repository:** https://github.com/snedea/afv2-pattern-03-routing
- **Flowise Documentation:** https://docs.flowiseai.com/
- **AgentFlow v2 Specification:** [AFv2 Docs](https://docs.flowiseai.com/agentflows)
- **ConditionAgent Documentation:** [Conditional Routing](https://docs.flowiseai.com/agentflows/condition-agent)

---

🤖 Built with Context Foundry
