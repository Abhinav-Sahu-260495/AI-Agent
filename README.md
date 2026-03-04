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


Abhi, you want to update on that? Like, we talked about it. So put it like X, Y, Z for his URL and then he should revert to still submit it, make your code ready.  Don't wait for Vishal to give you URL. Once he's ready, he'll give URL replace that you are, and that's all I'm looking at it away. Yeah, sure, Alan.  You want to update on that? Where are we and how long it's gonna take and what is ET and that? Uh, so as of now, as I said, like, uh, I have made some changes and uh, earlier it was not aligned with whatever was expected.  Uh, as per like I I got interviewed by Rahul. So, uh, I'm still making some changes and Right now, the thing is that like the testing part is not yet, uh, Sorry. ready. Hey, you have the call.  No, it is okay. So that is why it is taking a little more time. I'll get it fixed like the Python version and then I'll try to learn earn it locally.  So why can't you do this a bit, right? So you can put a mock service and then call that service itself, thinking that is a Vishal service, right? You't have to wait for each other.  Yeah. So on and like the, for the Asin KP, right, we need like, we are like currently like working on the Samsung enablement. That's like taking a lot of time for us.  So for the asking, like we had to derate the entire process, like you had to split it out and everything. That's fine, but I'm not rushing on that, right? So I will assume that you'll have XOZ API and then we'll just call that, right?  Or no? Yeah, basically what I told Abina was, actually, nothing will change. Like, basically, whatever we do for the sync APN response, right?  Same thing. Just this, we just need to return the same record ID and 200. That's it.  Yeah. If you are accepted, like, so then once you are done, you can call our AP with the same response, whatever you would give it to the CKP, right? Exactly, right?  You were written in acknowledgement, response back, yeah, assuming it's XYG or ABC, whatever, APA, okay? But no, he'll put it like some mock restaurant in college himself. Literally replace that with your endpoint at your endpoint, right?  So, I mean, I think you don't have to wait for him. You need to give an ET and then close this old plan. Yeah, I'll close this off by Tuesday.  Um, on Monday, today's, that's the red wagon. Just do it on Monday, man, it's already been. Okay, I'll try to fix that diamond Anand.







No problem. So I've been out, there's a brand new feature that we want to come up with, okay? And we have no fucking clue of how to do it.  Okay. But I trust you to do it. Okay?  How are we going to do it? Have you used ChatGPT? Yes.  Have you used Gemini browser? Yes. Right?  When you talk to ChatGPT, let's say you've been using your profile ChatGPT for a month or 2 months or 3 months now. Now when you go to ChatGPT and ask it that he described me, describe me. based on everything that you've asked before, based on the entire profile building that has happened on ChatGPT about you. It will be able to describe, right?  This is a similar thing that we want to bring on cubers. Okay. We, we calling it tidbits, but let's say uh, Rakshit, Rakshit is using cubas, and I have a tendency that I will usually go to reports as my Tomi, and BCDC you put as my catalog.  You notice this trend that, okay, every time Lakshit comes to on US, he goes to this thing. He goes to this particular tombing catalogue. Okay, let me store it somewhere.  Then on this particular domain in catalog, also, these are the set of questions that Rakshiti usually asks. Okay, let me store it somewhere. Okay?  Then, This is Rakshit uses Axiom, or Rakshit uses custom prompt, or Rakshit changes a lot of catalog descriptions using the configuration screen. This is all what Takshit is doing. On a regular basis.  Let me stop somewhere. Basically, like recording the pattern, recording the behavior of that user. Yeah, like, Hooser's pattern is what we want to record.  And then store it somewhere. Right now we just want to store it and create some sense out of it. Why is user doing that?  What is user doing that? It's basically a kind of profile building for that particular user on our back end and we store it somewhere. In future, let's say, one fine day, I log into cubers, I go to reports, VCGCU report, and without any context, if I just ask, give me top 5, not even give me top five.  I would just say that, uh, you know, give me the thoughts. You mean last week's data? Give me last week's report.  That is where I will use that entire profile building and make sense out of it, that user is asking me, give me last week's data, but what data? What are the probable data I can give to user? Now I have the entire a month or 2 months of pattern that this user has been doing.  Based on it, let me make out some sense. Let me figure out what can be the probable ending of this question. Give me last week's data, but of what?  So that what will be answered from that profile building of that user? Okay. Okay, so that is an entire tidbit that this we are calling a tidbit.  So this is what we want to build. What do we call that? Take bit.  Yeah, TIDBID. Okay, tidbit. Yeah.  I mean, forget about the normal culture. just a profile building that we like. Yeah. This is what we want to bring on cubas.  This is the big idea. This is, we don't have any in-depth idea of it. We don't have any fucking clue how to do it, okay?  Okay. We just have this thing that we want to bring it on cues because few of the domain experts, we have one of the people who, every time she comes on cures, she will ask questions related to Middle East data or there's one person who will come and he will only ask questions related to Texas as a state. okay? So in future, we want to store all all of that, that okay, user explicitly tries to answer, use questions regarding this particular thing.  When in future they just come and ask a vague question, we can use that context and enrich their question. Without them knowing, but yet providing the correct answer. Okay.  Because this is what ChatGPT does. This is what Gemini does. is what Claude also does. Right now also when you're using copilot.  Sometimes your explanation is not good enough, but copilot still understands what you want. Yeah. That is what we want, humors to be.  So as far as I know, like, uh, I did check, uh, so there'll be like some custom uh, prompt which we can set on ChatGPT, right? And whenever you see those, so have you ever come under a situation where, you know, it will be showing that your memory is full, right? That is fine.  Yeah, yeah, that is... I think like it, you already have it. You already have custom now.  So basically, like what it does, like, I'm just talking with respect to ChatGPT, so it basically like records few of the things. I'm not sure what are the activities which it records. Maybe need to ideate more on what are the things which we actually need to store and what are the things which we can probably ignore.  But, yeah, I get a sense of it, yeah. You get a sense of what I'm trying. Yeah, yeah, yeah.  It's like a profile building that we want to do for users. We'll definitely need a different table in the database for it. We'll definitely need certain number of columns.  I don't know how what wave you want to do it. You can vectorize it. You can use PM 25 or you can just save it semantically. anything is fine with me.  But do you understand this concept, what I'm trying to say? Yeah, yeah, I got it. Yes.  Yeah, that's why I say we won't be needing to record this because it's a very broader concert and you just understand when I see it. What I want you to do is go through the current code base. spend some time with a copilot, come up with a plan. Okay.  That is all I want to do it. And if possible, can you send the plan to me by 2 PM or 3 PM? Uh, I, I'll try my, I don't want, I don't want you to spend the entire day this day just for planning.  Because with copilot and all these AI tools, planning should be 30 minutes of work. And that is what we expect here also. So just come up with a plan.  Maybe use Gemini, Gemini 3.5 Pro is also really good. Use that, come up with a plan. But the fact copilot has access to your code base, you will know what you have.  You know how you can track users movement over the queues. Okay. Yeah, you probably have to check with like Gemini and ChatGPT.  How execrate, they do it. And then maybe like I can, I can see. Yeah, yeah, yeah. do whatever you have to do. take 1st half of today and maybe around 2 PM, 3 PM, we can meet again and you can walk me through the plan.  Because there will be a lot of hydrations in the plan itself. Once you come up with a plan, I'll give you my input. Okay, Lesh will give is important, then modify the plan, and then we'll start with coding.  Okay. So it's a long shot. Okay, got it, yeah.  Is it what we are to do? get started on this and yeah, I think we have enough on our plate, right? Okay.  So I'll propose this idea. I'm not sure how exactly the architecture will be working, whether we'll be using the existing. you know, we'll probably need a new database for this, a table. No, not new database. existing database, just a new table.  A table, okay. store user's identity and that this user, this particular column. Okay. So basically, like, we'll be still using the, uh, the, the, the one which way is, which one is that?  The PG vector? Table, yeah. Yeah, you just check the models dot PY.  We will create a new table in it. Okay, okay, cool, yeah. Yeah, yeah.  It should database should be a simple table generation. Anjali can explain it to you. She's been working on that.  Are you actually working with any database changes? Great. I'm just pulling data from them.  Okay, okay. But do you know where we are executing, right? You know where art models.  TY is and how the tables are in. Battle, yes, yeah. Yeah, yeah, yeah.  So, yeah, I mean, you can discuss it with that, really, also, if you guys are in the same office or whatever, maybe you can just meet and discuss this. Yeah. Okay, cool.  Yeah, but okay, let's connect some with me. Just create a new call somewhere around 2 PM. Let's connect and we'll discuss whatever plan you have.  Okay, can we connect at 3 PM? Sure. Because I'll be commuting to office, so again, some time will be taken there.  Oh, okay. Yeah, no problem. Okay.  You go to office at 3 PM? No, no, not at, like, right now I'll be going, right? So.  Yeah, that's fine. That fine. Yeah, okay, yeah.  Yeah. Okay, cool. I mean, but at least the 1st cut of plan I'm expecting.  Sure, sure, sure, yeah. All right, all right. Thank you guys.  Thanks, Rakshad, bye. Thanks, Anjali. Bye.  You can go back.
