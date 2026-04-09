🚀 COMPLETE REQUEST-TO-RESPONSE JOURNEY

DETAILED CODE FLOW WITH LINE NUMBERS

STEP 0: SERVER STARTUP (main.py)

FILE: main.py (Lines 1-279)

Line 87-93: app = FastAPI(...)

FastAPI instance created

Line 110-117: app.add_middleware(CORSMiddleware, ...)

CORS middleware added (cross-origin requests)

Line 119-150: app.add_middleware(BackendAuthMiddleware, ...)

Auth middleware added (token validation)

Line 218: app.add_middleware(RequestIDMiddleware)

Request ID middleware added

Line 248-259: app.include_router(api_router, prefix="/ntwgenie/api/workflows")

Routes registered

Line 277: uvicorn.run(app, host="0.0.0.0", port=8000)

Server starts listening on port 8000

STEP 1: CLIENT SENDS REQUEST

Client sends:
POST https://api.example.com/ntwgenie/api/workflows/wireless/oos_analyzer/v2

Headers:
Content-Type: application/json
X-Apigee-Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

Body:
{
"metadata": {
"iop_record_id": "OUT-20260208-001",
"user_id": "IOP"
},
"json_input": [
{
"topic": "amos",
"commands": [
{
"cmd": "show alarms",
"log": "CRITICAL: Power failure detected..."
}
]
}
]
}

STEP 2: REQUEST ARRIVES AT FASTAPI SERVER

FastAPI Internal Processing

Receives HTTP request on port 8000

Parses HTTP headers and body

Starts middleware chain

STEP 3: MIDDLEWARE PROCESSING

MIDDLEWARE 1: CORSMiddleware

Checks if cross-origin request allowed

Adds CORS headers to response

Passes to next middleware

MIDDLEWARE 2: BackendAuthMiddleware
FILE: app/core/middleware/backend_auth.py (Lines 30-51)

Line 32: backend_token = request.headers.get(settings.APIGEE_HEADER_NAME)

Extracts: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

Line 35-36: if self._is_exempt_path(request.url.path):

Path: "/ntwgenie/api/workflows/wireless/oos_analyzer/v2"

NOT in exempt list → Continue auth check

Line 42-44: if not backend_token:

Token exists ✅ → Continue

Line 46-48: if await self.is_key_valid_in_atlas(backend_token) == False:

🔹 Token Validation Process:

Line 74-77: Check cache first
if api_key in self._api_key_cache:
return cached_entry["is_valid"]

Line 81-91: Call Atlas API for validation
response = await client.post(
url=settings.ATLAS_VALIDATION_URL,
auth=(username, password),
json={"token": api_key}
)

Line 92-96: Cache result for 10 hours
self._api_key_cache[api_key] = {
"is_valid": True,
"expiry": datetime.now() + timedelta(hours=10)
}

Token valid ✅ → Continue

Line 51: return await call_next(request)

Passes to next middleware

MIDDLEWARE 3: RequestIDMiddleware
FILE: app/core/middleware/request_id.py (Lines 20-40)

Line 22: request_id_value = request.headers.get("X-Request-ID")
or str(uuid.uuid4())

Client didn’t send ID → Generate new UUID

Generated: "req-abc-123-xyz-456"

Line 25: token = request_id.set(request_id_value)

Sets context variable (available throughout request lifecycle)

Line 28: ensure_request_id()

Ensures logging system has request ID

Line 31: response = await call_next(request)

Passes to route handler

STEP 4: ROUTE MATCHING

FastAPI Internal Router
URL: /ntwgenie/api/workflows/wireless/oos_analyzer/v2
Method: POST

Matches route definition:

FILE: app/wireless/api/router.py

Line 10: api_router.include_router(oos_analyzer.router, ...)

FILE: app/wireless/api/oos_analyzer.py (Lines 58-66)

Line 58-66:
@router.post(
"/oos_analyzer/v2",
response_model=OOSServiceResponse,
summary="Analyze OOS Diagnostic Logs - Direct Vegas Endpoint",
tags=["oos_analyzer"],
)
async def analyze_oos_endpoint_v2(request: OOSInput):

MATCHED! ✅ → Calls analyze_oos_endpoint_v2()

STEP 5: REQUEST BODY PARSING (PYDANTIC VALIDATION)

🔹 Pydantic Automatic Validation

JSON Body → Pydantic Models

FILE: app/wireless/workflows/oos/models.py

Line 9-12:
class RequestMetadata(BaseModel):
iop_record_id: str
user_id: str

Validates: {"iop_record_id": "OUT-20260208-001", "user_id": "IOP"}
✅ Valid

Line 14-17:
class CommandResult(BaseModel):
cmd: str
log: str

Validates: {"cmd": "show alarms", "log": "CRITICAL..."}
✅ Valid

Line 19-22:
class TopicResult(BaseModel):
topic: str
commands: List[CommandResult]

Validates: {"topic": "amos", "commands": [...]}
✅ Valid

Line 24-27:
class OOSInput(BaseModel):
metadata: RequestMetadata
json_input: List[TopicResult]

Validates entire request body
✅ Valid → OOSInput object created

request = OOSInput(
metadata=RequestMetadata(iop_record_id="OUT-20260208-001", user_id="IOP"),
json_input=[TopicResult(topic="amos", commands=[...])]
)

STEP 6: ENDPOINT FUNCTION EXECUTION

FILE: app/wireless/api/oos_analyzer.py (Lines 67-115)

Line 68:
logger.info(f"Received request at /oos_analyzer/v2 for iop_record_id: {request.metadata.iop_record_id}")

Logs: "Received request at /oos_analyzer/v2 for iop_record_id: OUT-20260208-001"

Line 70: try:

Line 72:
analysis_result, session_id, agent_transaction_id, parent_transaction_id = await oos_workflow_v2.analyze_oos_v2(request)

🚀 Calls workflow → Go to STEP 7

STEP 7: WORKFLOW EXECUTION (OOSAnalyzerWorkflowV2)

FILE: app/wireless/workflows/oos/oos_workflow_v2.py (Lines 203-248)

Line 204:
async def analyze_oos_v2(self, request: OOSInput):
Entry point

Line 210:
logger.info("[OOS-V2] [WORKFLOW] Starting workflow...")

Line 213-215:
initial_state = {
"oos_input": request,
}

Prepares initial state for LangGraph

Line 218-219:
final_state = await self.agent.ainvoke(initial_state)

🔥 LangGraph DAG Execution starts!
Graph compiled at Line 52-68:

graph.add_node("start", self.node_start)
graph.add_node("prepare_logs", self.node_prepare_logs)
graph.add_node("build_glossary", self.node_build_glossary)
graph.add_node("llm_analyst", self.node_llm_analyst)
graph.add_node("format_output", self.node_format_output)

graph.add_edge("start", "prepare_logs")
graph.add_edge("prepare_logs", "build_glossary")
graph.add_edge("build_glossary", "llm_analyst")
graph.add_edge("llm_analyst", "format_output")

STEP 8: NODE 1 - START

FILE: app/wireless/workflows/oos/oos_workflow_v2.py (Lines 70-74)

Line 72: run_id = str(uuid.uuid4())
Generates: "run-789-def-012"

Line 73: iop_record_id = state["oos_input"].metadata.iop_record_id
Extracts: "OUT-20260208-001"

Line 74: return {"run_id": run_id}
Updates state with run_id

State now: {"oos_input": {...}, "run_id": "run-789-def-012"}

STEP 9: NODE 2 - PREPARE_LOGS

FILE: app/wireless/workflows/oos/oos_workflow_v2.py (Lines 76-85)

Line 81: iop_record_id, combined_logs = extract_core_fields(state["oos_input"])

Goes to tools.py:
FILE: app/wireless/workflows/oos/tools.py (Lines 44-49)

Line 48: iop_record_id = oos.metadata.iop_record_id
Line 49: combined_text = flatten_topic_results(oos.json_input)

Flattens logs:
Line 32-41: def flatten_topic_results(topics):
blocks = []
for topic in topics:
blocks.append(f"--- TOPIC: {topic.topic.upper()} ---")
for command in topic.commands:
blocks.append(f"$ {command.cmd}\n{command.log}")
return "\n\n".join(blocks)

Result:

TOPIC: AMOS

$ show alarms
CRITICAL: Power failure detected...

Line 85: return {"iop_record_id": iop_record_id, "combined_logs": combined_logs}

State updated: {..., "iop_record_id": "OUT-...", "combined_logs": "..."}

STEP 10: NODE 3 - BUILD_GLOSSARY

FILE: app/wireless/workflows/oos/oos_workflow_v2.py (Lines 87-103)

Line 95: glossary_text = build_relevant_glossary(state["combined_logs"])

Goes to glossary.py:

Searches for technical terms in logs

Adds definitions for found terms

Result:

GLOSSARY:

Power failure: Loss of electrical supply to equipment

Critical alarm: Highest severity level alert

Line 103: return {"glossary_section_text": glossary_text}

State updated: {..., "glossary_section_text": "..."}

STEP 11: NODE 4 - LLM_ANALYST (🔥 VEGAS LLM)

FILE: app/wireless/workflows/oos/oos_workflow_v2.py (Lines 105-177)

Line 113-117: prompt = build_llm_prompt(
iop_record_id=state["iop_record_id"],
combined_logs=state["combined_logs"],
glossary_text=state["glossary_section_text"]
)

Builds prompt for LLM with logs + glossary

Line 122: vegas_transaction_id = await call_llm(prompt, run_id)

🚀 VEGAS LLM CALL!
FILE: app/wireless/workflows/oos/llm.py (Lines 28-30)

Line 29: return await asyncio.to_thread(_submit_llm_sync, prompt, run_id)

Submits to Vegas API:
Line 60-111: def _submit_llm_sync(prompt, run_id):
# Prepare payload
payload = {
"useCase": "LLM_STUDIO",
"contextId": "gemini-2.5-flash",
"preSeed_injection_map": {"{INPUT}": prompt},
"parameters": {
"temperature": 0.05,
"maxOutputTokens": 65535,
...
}
}

Submit to Vegas

submit_resp = requests.post(
"https://oa.verizon.com/vegas/apps/prompt/poll/text/LLMInsight
",
headers=headers,
json=payload
)

transaction_id = submit_json.get("vegasTransactionId")
return transaction_id

Returns: "vegas-tx-456-abc"

Line 125: ai_message = await get_result_from_transaction(vegas_transaction_id)

Polls Vegas for result:
FILE: app/wireless/workflows/oos/llm.py (Lines 145-169)

Line 155-162: @retry(stop_after_attempt(10), wait=wait_exponential(...))
def _poll_for_result(poll_url, headers, transaction_id):
resp = requests.get(poll_url, headers=headers)

if "prediction" not in result:  
    raise RequestException("Not ready, retrying")

return result.get("prediction", "")


Vegas response:

{
"status": "success",
"rca_summary": "Power supply failure in eNodeB-789",
"rca_description": "Critical power failure affecting 10 cells...",
"rca_devices": "eNodeB-789, Cells 1-10",
"evidence_logs": "$ show alarms: CRITICAL...",
"suggested_resolution": "1. Dispatch engineer...",
"confidence_score": 0.92
}

Line 133: parsed_output = parse_llm_json(raw_output)
Parses JSON response (utils.py)

Line 134: normalized_output = normalize_llm_output(parsed_output)
Normalizes/validates output (utils.py)

Line 167-172: return {
"llm_prompt": prompt,
"llm_ai_message": ai_message,
"llm_raw_output": raw_output,
"llm_parsed_output": normalized_output
}

State updated with LLM results

STEP 12: NODE 5 - FORMAT_OUTPUT

FILE: app/wireless/workflows/oos/oos_workflow_v2.py (Lines 179-199)

Line 186-195: analysis = OOSAnalysis(
status=llm_output["status"],
rca_summary=llm_output["rca_summary"],
rca_description=llm_output["rca_description"],
rca_devices=llm_output["rca_devices"],
secondary_issues=llm_output["secondary_issues"],
evidence_logs=llm_output["evidence_logs"],
suggested_resolution=llm_output["suggested_resolution"],
confidence_score=llm_output["confidence_score"]
)

Creates OOSAnalysis object

Line 199: return {"analysis": analysis}

Final state: {..., "analysis": OOSAnalysis(...)}

STEP 13: WORKFLOW RETURNS TO ENDPOINT

Back to: app/wireless/workflows/oos/oos_workflow_v2.py (Line 219)

Line 219: final_state = await self.agent.ainvoke(initial_state)
DAG execution complete ✅

Line 221: analysis_result = final_state["analysis"]
Line 222: session_id = final_state["run_id"]

Line 232-239: agent_transaction_id, parent_transaction_id = await log_transaction(...)
Logs to network genie database for tracking

Line 246: return analysis_result, session_id, agent_transaction_id, parent_transaction_id

STEP 14: BUILD RESPONSE OBJECT

Back to: app/wireless/api/oos_analyzer.py (Line 72)

Line 72: analysis_result, session_id, agent_transaction_id, parent_transaction_id =
await oos_workflow_v2.analyze_oos_v2(request)

Received ✅

Line 75-79: response_metadata = ResponseMetadata(
iop_record_id=request.metadata.iop_record_id,
session_id=session_id,
agent_transaction_id=agent_transaction_id,
parent_transaction_id=parent_transaction_id
)

Line 82-85: response = OOSServiceResponse(
response=analysis_result,
metadata=response_metadata
)

Line 88: logger.info("Successfully completed request...")

Line 89: return response
Returns OOSServiceResponse object

STEP 15: RESPONSE PROCESSING

FastAPI Internal Processing

Pydantic object → JSON conversion:

response.json() produces:

{
"response": {
"status": "success",
"rca_summary": "Power supply failure in eNodeB-789",
"rca_description": "Critical power failure affecting 10 cells...",
"rca_devices": "eNodeB-789, Cells 1-10",
"secondary_issues": null,
"evidence_logs": "$ show alarms: CRITICAL...",
"suggested_resolution": "1. Dispatch engineer...",
"confidence_score": 0.92
},
"metadata": {
"iop_record_id": "OUT-20260208-001",
"session_id": "run-789-def-012",
"agent_transaction_id": "tx-network-genie-123",
"parent_transaction_id": "parent-tx-456"
}
}

STEP 16: MIDDLEWARE ADDS HEADERS

RequestIDMiddleware (app/core/middleware/request_id.py)

Line 36: response.headers["X-Request-ID"] = request_id_value
Adds: X-Request-ID: req-abc-123-xyz-456

STEP 17: HTTP RESPONSE SENT TO CLIENT

HTTP/1.1 200 OK
Content-Type: application/json
X-Request-ID: req-abc-123-xyz-456

{
"response": {
"status": "success",
"rca_summary": "Power supply failure in eNodeB-789",
...
},
"metadata": {
"iop_record_id": "OUT-20260208-001",
...
}
}

STEP 18: CLIENT RECEIVES RESPONSE

✅ Request successfully processed!
⏱ Total time: ~10-15 seconds (Vegas LLM takes most time)

🔥 VEGAS/LLM INTEGRATION DIAGRAM

OOS WORKFLOW

Node: START
→ Generate run_id

Node: PREP_LOGS
→ Flatten command logs

Node: GLOSSARY
→ Build technical glossary

Node: LLM_ANALYST

Build prompt (logs + glossary)

Submit to Vegas API

Get transaction ID
(vegas-tx-456-abc)

Poll for result (retry loop)

Parse JSON response

Normalize/validate output

Node: FORMAT
→ Create OOSAnalysis object

Return Result

VEGAS LLM SERVICE
(External Verizon AI Service)

POST /vegas/apps/prompt/poll/text/LLMInsight

Request:
{
"useCase": "LLM_STUDIO",
"contextId": "gemini-2.5-flash",
"preSeed_injection_map": {
"{INPUT}": "Analyze these logs..."
},
"parameters": {
"temperature": 0.05,
"maxOutputTokens": 65535
}
}

Response:
{
"vegasTransactionId": "vegas-tx-456"
}

GET /vegas/apps/prompt/poll/LLMInsight/vegas-tx-456
(Poll every 2s, 4s, 8s... max 10 attempts)

Final Response:
{
"prediction": {
"status": "success",
"rca_summary": "Power failure",
...
}
}

📊 TIMING BREAKDOWN

Total Request Time: ~10-15 seconds

Operation	Time
Middleware processing	~10ms
Route matching + Pydantic validation	~20ms
Node: START	~5ms
Node: PREPARE_LOGS	~50ms
Node: BUILD_GLOSSARY	~30ms
Node: LLM_ANALYST	
├─ Build prompt	~10ms
├─ Submit to Vegas	~500ms
├─ Poll for result (retry loop)	5-10s 🔥
├─ Parse JSON	~20ms
Node: FORMAT_OUTPUT	~10ms
Log transaction to DB	~200ms
Build response + JSON serialization	~30ms

🔥 Vegas LLM polling is the slowest part (5-10 seconds)


form of a spreadsheet or a document or anything, which gives me a comparative analysis. Okay. With the existing API to the new API and test it with at least 10 questions. 
 Okay. And onboard of base level PDF for CSV or anything with respect to it, and test it out. And depending on each size, like the way you set the 1st one, the optimal sizes, you have the PDS or the CSVs.  How is it working? Okay. I want like a clear documentation comparison between both of them.  Okay. Okay. Because based on that, we can evaluate if it's giving us the right output on the questions based on the data available.  And then we can compare it with converse. We'll see the response time from here, response time from Converse Vegas api is way the same way along the way. And then determine, which is better for answering the question, because this API has claude as well.  Claude for SQL generation is much better than Gemini. Okay. Yeah, here it is using, I think gemini 2.0 flash.  Yeah, so I would like to see how its skills are working. You do have access to advert dashboard, right? Yeah, now I do have, yes.  Perfect. Okay. In the admin outboard.  Let me find where Manisha is situated. So Manisha is the domain catalog is the one that we have to be testing on because this was their complaint. Okay.  
Abhinav- I have no clue who Manisha. Yeah, I'll find that domain catalogue for you. Okay. 
 Anjali- And I'll tell you. Give me one. As using thought, reduce the time that the API takes to respond?  Yes, and clon is a coding LLM. Okay. Rather than Gemini.  Manisha as in Minerva. Okay. So Mener was root or something like that, right?  And take that metadata and test it out. In admin Ashboard, go to Minerva. 
Abhinav- I'm not sure why I'm not able to log in until yesterday I was able to.  Anjali- Maybe it's a cashing issue. Okay, yeah, 
Abhinav- I'm able to log in now. Yeah.  Oh, okay. 
Anjali- Go on into usage logs and look for Minerva route. Okay.  Okay. Okay. Minerva has, which catalog do I have to go?  This is okay. So right now, you see all these questions that you see here in the usage logs. These are all minerwA questions and they were expecting all of these to work with their own CSV as well.  Okay. Okay. So let's take these questions.  The 1st 10 questions as an example, whichever catalog they're associated with, and if you click on each of these questions, you'll also see the SQL that was generated. Yeah. Yeah, perfect. You'll see the SQ and the summary and everything. See, use the claude part of it.  And with the multimodal API, see, if you're still generating the same SQL, and the same summary, I'll ask her which document she used. Okay. So we test on this usage, uh, this domain.  So look for these questions, test it on these questions, if she gives me a document, I'll forward that to you. Let me know. I want like a clear report, comparative report between both of them.  Abhinav- Okay, so one thing which you want me to check is, like, the use cases, whatever you will be, uh, uh, basically the the things which I'll be comparing, right? First is basically the things which I'll be testing out, like the PDA, large PDF and all those stuff. This is one part of a report.  What are the other things which you want me to include? 
Anjali- We need speed. We need accuracy.  And we need load, 3 kinds of testings that have to go through, and 3 of them are the capacity of AB testing. So speed testing for Converse, for right now, Converse, or right now, multimodal API to the one that they gave us, and load testing with the size of the document, and accuracy testing with the SQL comparison. Okay.  Yeah, we need a report of this testing completely. It can be multipage document also, but we need a testing comparison of the entire thing. Okay.  So we understand where we stand and if we need to integrate those. Okay. 
Abhinav- One part which I was not clear was Anjali, when we talk about the converse API, I was just wondering that basically the upload part is basically that that one particular API which is being hit, right?  Why are we replacing this entire thing with the converse? Converse itself is like a broader thing, which? 
Anjali- No, no, we launch a replacing converse with this.  So in Converse, the 1st step of SQL generation. If this works better than that, that part, the 1st part, then they replace that Vegas called me, this Vegas called. Okay, okay.  Abhinav - And they were also mentioning about like different LLM calls, which is being there, right? So are we trying to reduce those LLM call and just putting it as one or something? 
Anjali- No, no, no.  That is not something that you are doing right now. We're just trying to see if the SQL that is generated with the API is better than the one that is being generated with the existing API. Okay.  Okay. Nothing. Cool, cool, cool.  Abhinav- I thought that this this API just works with the uploads, but okay, fine. It also works with the sql as well. So okay.  Anjali - It has to give us an output of a sql. So, see, right now in Converse API. Under that, the Vegas call has a certain context ID.  Use the same context ID or use the same prompt. Give this information on it as well, and let's see how it works. Okay.  If you change the prompt, also, I have no problem with that. Okay, see what works, just if there is a better way that you can construct that prompt and do it, then that is on you. Try it out.  We need SQL. 
Abhinav- Okay, just a quick question. So how do I play with prompts here?  Do I have to go to Vegas and do it or? 

Anjali- No, no. So right now in the API of your calling, you're already giving an input of a prop, correct?  Anjaly- As an input, you're giving a prompt, you're giving mitadata, you're giving the PDF file everything. Yeah. Right now when you're testing on the multimodal API.  Abhinav- When we are doing this, I don't think so. We are giving prompt anywhere. 
Anjali- in the test part of request payload  You can also give the entire prompt there. Okay. Correct.  So the entirety of the, if you're writing a prompt, a new prompt, then write a new prompt, where you take in the document, you take in all of this information, and you give it a set of instructions as to, I want an SQL generated such as, with this document information such as, and write a prompt there, and see what is the level of accuracy? And keep speaking of the prompt according to your requirement. 
Abhinav- Okay.  Yeah? Okay, I'll test that out, yeah. Perfect, okay.  

