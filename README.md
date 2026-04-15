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

Hi, Akilesh. Okay, um, I think the one which you said the other day, right, that is making sense to me now.  Uh, the Autobia thing, right? Let me just share my screen. Is my screen visible to you?  Not yet. Okay. Yeah?  Okay, uh... So basically, I'll tell you what exactly is going on, okay? Basically, the problem is that when we were using the old LLM inside multimodel, right?  That that was breaking. That was breaking for uploading and text. Okay.  Okay. So what the suggestion was that maybe we can change the API. So let's just consider like we have a new API that is like gateway B2, right?  Uh, this is basically, this supports better, uh, like multi- This is a better multimodal API. which supports the PDFN.CSV files. Right? Now, the problem is that when we look when we compare these 2 APAs, right, the 1st one is the old one, which is currently existing.  This follows the Vegas unified format. Right? That means that they have like one particular format and then it alters according to that like what exactly the model which we are using, right?  But then the new model, the new API, this has a native format. Let's just say if you're using Gemini, then it is using the Gemini format. If it is, if we are using clot, then we are using clot.  Right? Correct, yeah. Now, uh, the problem is that um, I've been asked to integrate this new API, right?  In place of the old API. Huh. Okay.  Now, earlier, the one when I tested the old API, I was testing on auto BI flash two, which doesn't exist, as you said, the use case, right? So I was supposed to do it on QVS. Now, when I say that, you know, what so what exactly use case should I use then?  Because since we are using 4 other uh, 4 different use cases, I saw that like it is generally like hard-coded as Q verse pro, right? This is the use case which we are using right now. We use all the 4 use cases.  It's not hardboarded. Uh, okay, for markdowns, I think, yeah, or it is hardcooded for one use case. So, same thing.  Uh, see. Okay, I'll tell you. Uh, your regular alum calls, like, query generation, visualization summary, anything?  Yeah. It is a dynamic thing. So based on whatever model user is selecting, switch the use case accordingly.  Each use case, that was to one mod. Okay. It's basically use case to model one on one mapping.  Okay. Now, uh... All right, sorry, man, give me one second.  Yes. Uh, yeah, uh, but for multimodal, that's not the case, but multimodal, we uh, we wanted only, you know, one static use case. Again, no, user doesn't care what model you're using for or PDFX structure and other stuff, right?  So we hardcoreed it too. I think he was broke, he was flush, whichever was working. We just set it at one news case.  But your other airl are all dynamic. Now, in this case, for your gateway API, also, if Gateway API also needs your use case name and context study. Exactly, yes.  You can continue the same, using the same QS Pro context and whatever come in use case and whatever the context was there. No, my question is, like, let's just say if I, if I'm still using the cubers pro, right? Now we have the different context for different works, right?  So do I have to make like different URL for this? you don't have to do anything. Okay.  You're not doing anything. All you have to do is the pace, you are, whatever is there. Mm hmm.  You're replacing that in doc environment variables that said nothing else. Uh, this is the base URL, Akilesh, right? No, that's not the base, that's the domain name.  Your base URL will be up to B2 chat. Okay, this one. The entire thing.  Mm hmm. The Vito chat is your pace, your... Okay.  Okay, so open the dot in. Yes. Yeah, go to Dotty, okay?  This is, you are, I need to talk. I think you are talking about this thing, right? That is your regular alarm calls.  Yeah. 922 is where it's calling the multimodel API. Okay. Right.  So, this is the base order, or, like, order the API UR allows for multimodal. This URL is what you're going to replace with, uh, Tenu Gator URL. And you're gonna add that use case name and context study.  Again, if you you want usually, you can put the entire thing here. Uh, once again, US Pro slash. Yeah.  One second. So, um, what needs to be changed, you said? This entire thing needs to be replaced with this one.  Okay, within the base, URL. Okay. You replacing the entire U order with that entire organ.  Uh huh. Now, uh, which means you're pointing to the entire thing, even cures prose slash gender mark of the entire thing. Okay.  Uh, no, you made the right API call, right? So you replace the multimodal API with new URL, which means you're hitting the gate to API. Right?  So this is not, this part of it is not, this is completely done. Now you need to look at the request body you're sending you know whatever you information you're sending to the API. That should use the new format.  Okay. So, like, just bring up it's taking curtains, 4 user parts, whatever it is. You follow this format, instead of the old format, whatever we had before.  Okay, very quick question. One quick question. So you said that, like, we have already, basically, we are just, 1st of all, we are linking this thing.  Like, we are we are making our road to how exactly it will get connected to that particular API, which is this, right? This is where it will be connected to this, like, basically, this is, this is basically the link, how it will be accessed, how, where exactly it will go to? Yeah, where exactly it should go to.  Now, this is something which I got, I am not sure. This is something which I got from Claude. I wanted to uh, I wanted to make sure now my question was like, see, here, this generate markdown is one of the context ID, which has, which is linked to one of the prompts, right?  Maybe this is basically for generating the marked on file or I don't know what exactly is that for. Right? It is better for that, yeah.  Okay, yeah. So now my question is that since we have like so many other use, so we'll be having so many contexts, right? There are so many things which is happening.  Like they will be creating, it will be creating the sequel and then it will be regenerating some sequels and then it will be generating the summary and all those things. So my question was that how exactly will this work? I mean, like this, are we not pointing out to one particular use case that only generates model?  Okay. Okay, again, see, there are 2 different APSs. One is for your regular LM calls.  Okay. Then we're just giving you text output or taking text input. Okay.  Multimodal API is, which is something we're just taking only PDFs or like whatever the files that you're uploading as an input. Okay, okay. So, this particular change that you're doing in line 22 is gonna affect only the uploading files functionality.  Okay. It will not affect the sequel and all those things. It won't, unless you do something to line number 19.  Okay. It won't affect any of the other functionality. This is only gonna have an impact on the upload functionality.  So even if you change the payload for this, this is sitting in a different file altogether, the request body for this is sitting in a standalone script. So, you can change the request body directly for this. But if you are planning to change everything to catering, then for query generations or, you know, summaries, socialization, that's when you change line number 19.  And the actual, because, I mean, whatever script we have for Vegas calls. Okay, okay. Got it.  So, uh, this particular thing has to be, uh, we need to add the anthropic also right here. That is working up to you if you want, from Vegas. Then you need to bring a new use screen, a new use case called QOS Anthropic or something.  Uh huh. And create the new context that each generate markdown, and then probably bring it in. Oh, so we have to do it all together.  Like, everything, again, we have to migrate everything, all the things, all the context in the new use case. You'll have to manually copy paste that or probably reach out to Vegas team, ask them to do, copy, paste it for you because we have it on 150 prompts. Okay.  So, you can ask them to, like, you know, properly send an email or raise a cheera ticket, and they will do it for you. They have a plan to do that. Okay.  So, what should be the best bet for this Akhilesh? How should I approach? See, there's nothing to do here.  I mean, your task, I'm assuming it's regarding the upload functionality, which is failing. And they would have asked you to switch to K to API as a solution. right? Okay, yes.  But then, why I'm confused about this particular thing is that because Anjali was asking me to check, to test basically whether this from this API, can we, is the sequel being generated properly or not? This was, yeah. If you're looking for query generation also, then that means you will have to change your line number 19 also, according to the same thing, inference free to chat.  But in that case you won't mention the use case name and context ID. Instead, you will dynamically pass it, like, you know, however, old docuvers is passing a tip, uh, as part breakfast body. Instead you will append it to the URL.  Dynamically every time user is switching between models, they do. But for using anthropic, if, again, if this is using the same format as Vegas, like use case context study sort of format, then you will have to ask Vegas team to create a new use case for you with cloud models. Something like yours, anthropic, okay, was caught, and then you can start using it.  But the existing use cases all are for flash models. So you cannot trust Claud with the existing use case. Correct, correct, because payload will be changed.  Yeah, correct. So, even if you directly go what change theirs, then, I mean, you're sort of breaking your damn environment, now, someone using your dub, my tour, on it, Taurus. So, it's better to create a new use case called QS clot or something.  No, whatever problems you need into it, then change the line number 19 URL. Okay. Uh, and then cross body, and that should be good.  Okay, so, this definitely needs, uh, like Vegas teams intervention, right? I mean, you can do it manually if you know what prompts you need, you can manually go copy paste it. If you need Vegas to do it, they will do it for you.  Okay. Because this is getting critical now, I don't know, like, it has been escalated a lot, so not sure how exactly I need to, okay. Yeah, so 2 is here.  Manually, if you have time, you can manually create a use, you can create a use case. permissions to play it use case. And the APR keys will access all use cases. So, you can create a new use case, uh, start copying all the context from, you know, any one of those 4 years cases is manually copy based into the new use case.  Or, just simply, reach out to Central or anyone to, you know, ask them to reach out to VegasCream to get it done. Okay, okay. And just for my own information.  So it's like whenever we are changing this particular thing, right, which is said, like line number 19, So, uh, it will only be replaced uh, before context ID and use case, right? Correct, yeah. Uh, till that's our chat slash.  You'll paste it till there, and then whatever is there after chart slash, that will be dynamically or just whenever you're calling the APR. But that logic is already existing right here. Uh, we are not adding it to the URL.  We are adding it as a request body paramedal. Here, in this case, you have brought it to the order. So it just placement is different.  Instead of passing it, and it goes more, you are passing it as part of the order itself. Okay, this part I didn't get it. Passing as a URL and passing as a... ideally what you do is when you're calling an API.  You send information as adjacent input into it. Yeah, yeah. Right.  Yeah. Like, because it's not, it's a what, uh, hear what they do when it does. Instead of passing in its Jason input, they wanted us, you are a parameter.  So you order a parameter is nothing but a slash, then your parameter value underlines slash, and there are a parameter value. Okay. If you see line number 22, it was pro one generate markdown in the last two values, right?  Mm hmm. Knows to hear parameters. Okay.  So you can change them. Like, you know, your base, your remains the same. Like, whatever is there before that hockey was pro, that is your base, you are.  Okay. And you simply do base URL plus your parameter slash 2nd parameter. That's it.  Okay, all right. Alright, got it. Claude should be able to do it, like, you know, if you are, if you give a tea or request your order, tell it, what are the parameters that, you know, what are the use cases you have?  We should be able to figure out and update it. Okay. Got it.  Cool, Akilesh. I'll, I'll do this and if I have any route, then I'll, I'll probably reach out to you again. Yeah, yeah, sure.  Thank you so much, Akilesh. You are my survivor. Relax, it's okay.  Yeah. Just, yeah, just play around with it. I don't understand everything.  That rates can get season better. Okay, okay, cool. Yeah.  Thanks a lot. Yeah.
