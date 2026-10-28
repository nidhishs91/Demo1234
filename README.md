JavaScript evaluation error on:
(function execute(inputs, outputs) {
inputs = new AdobeSignUtils().trimStringInputs(inputs);
var mandatoryInputs = {
'name': inputs.name,
'group_record': inputs.group_record,
'participant_info': inputs.participant_info,
'documents': inputs.documents
}
new AdobeSignUtils().checkMandatoryInputs(mandatoryInputs);

outputs.name = inputs.name;
var inputDocs = inputs.documents;
var documents = [];
for (var i = 0; i < inputDocs.length; i++) {
var docObj = {};
var tempObj = inputDocs[i];
var document_type = "libraryDocumentId";
var id = tempObj.document_id;
switch (tempObj.document_source) {
case "library_document":
document_type = "libraryDocumentId";
break;
case "transient_document":
document_type = "transientDocumentId";
break;
}
docObj[document_type] = id;
documents.push(docObj);
}

var json_body = {};
var memberinfo = {};
var emailadd1 = {};
var memberinfo_array = [];
var memberinfo = {};
json_body["fileInfos"] = documents;
json_body["name"] = inputs.name;
if (inputs.participant_info.length > 0) {
var participants = [];
inputs.participant_info.forEach(function(property) {
var participant = {};
if (property.order) {
var order_syntex = /^[-+]?[0-9]+$/;
try {
var order = property.order.toString().trim();
if (order.match(order_syntex)) {
participant.order = order;
} else
throw new Error("Provided Signing Order is invalid.");
} catch (err) {
throw new Error("Provided Signing Order is invalid.");
}
}

if (property.role)
participant.role = property.role;
if (!gs.nil(property.private_message))
participant.privateMessage = property.private_message.trim();
if (property.email) {
var email_syntex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
try {
var emailAdd = property.email.toString().trim();
if (emailAdd.match(email_syntex)) {
var memberInfos = [];
var memberInfo = {};
memberInfo.email = emailAdd;
if(!gs.nil(property.name)) memberInfo.name = property.name.trim();
memberInfos.push(memberInfo);
participant.memberInfos = memberInfos;
} else {
throw new Error("Provided Participant Email is invalid.");
}
} catch (err) {
throw new Error("Provided Participant Email is invalid.");
}
}
participants.push(participant);
});
if (participants.length > 0) {
json_body["participantSetsInfo"] = participants;
var participants_obj = {};
participants_obj['participantSetsInfo'] = participants;
outputs.participants_info = JSON.stringify(participants_obj);
}
}
if (!inputs.agreement_record) {} else if (inputs.agreement_record && (inputs.agreement_record.status == "") || (inputs.agreement_record.status == "DRAFT")) {

} else {
throw new Error("Provided agreement status should be DRAFT or null.");
}
json_body["message"] = inputs.message;
json_body["signatureType"] = "ESIGN";
json_body["state"] = inputs.state;
json_body["externalId"] = "API_SERVICENOW_SPOKE";

if (inputs.suppress_notification == true) {
json_body["emailOption"] = {};
json_body["emailOption"]["sendOptions"] = {};
json_body["emailOption"]["sendOptions"]["completionEmails"] = "NONE";
json_body["emailOption"]["sendOptions"]["inFlightEmails"] = "NONE";
json_body["emailOption"]["sendOptions"]["initEmails"] = "NONE";
}

outputs.group_record = new AdobeSignUtils().validateGroupRecord(inputs.group_record);
outputs.json_body = JSON.stringify(json_body);
var connection_sys_id = inputs.connection_sys_id;
outputs.adobe_version = new AdobeSignUtils().validateAdobeVersion(connection_sys_id);
}(inputs, outputs));
Stack trace:
at sys_script_include.e3b33778db4227003e146583ca961970.script:117
at Process Automation.707db6de43702110f588ead67bb8f219:9 (execute)
at Process Automation.707db6de43702110f588ead67bb8f219:108
at sys_script_include.35cb33ee476a21105da8ff58936d4390.script:94
at sys_script_include.64e874a243f52110ff9fead67bb8f2b4.script:191 (sendForSignature)
at sys_script_include.53b5af02849a2110fa9bfd0f8bd07b03.script:51
at sys_script_include.cd6e5e7d84922110fa9bfd0f8bd07bdb.script:105 (implementation)
at sp_widget.1ebf13f7e0a63010fa9b460b5b232860.script:66
at sp_widget.1ebf13f7e0a63010fa9b460b5b232860.script:3
at sp_widget.1ebf13f7e0a63010fa9b460b5b232860.script:1
: org.mozilla.javascript.JavaScriptException: Error: Mandatory inputs cannot be empty (sys_script_include.e3b33778db4227003e146583ca961970.script; line 117): org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1200)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sys_script_include_e3b33778db4227003e146583ca961970_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.Process_Automation_707db6de43702110f588ead67bb8f219.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$exec$1(InterpretedFunction.java:148)
com.glide.caller.gen.Process_Automation_707db6de43702110f588ead67bb8f219.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.exec(InterpretedFunction.java:146)
com.glide.script.ScriptCompiler.executeAndPublishMetric(ScriptCompiler.java:83)
com.glide.script.ScriptEvaluator.execute(ScriptEvaluator.java:552)
com.glide.script.ScriptEvaluator.evaluate(ScriptEvaluator.java:254)
com.glide.script.fencing.GlideScopedEvaluator.evaluateScript(GlideScopedEvaluator.java:429)
com.glide.script.fencing.GlideScopedEvaluator.evaluateScript(GlideScopedEvaluator.java:311)
com.glide.script.fencing.GlideScopedEvaluator.evaluateScript(GlideScopedEvaluator.java:288)
com.snc.process_flow.operation.GlidePFScriptEvaluator.evaluateScript(GlidePFScriptEvaluator.java:74)
com.snc.process_flow.operation.GlidePFScriptEvaluator.runScript(GlidePFScriptEvaluator.java:34)
com.snc.process_flow.operation.script.ScriptOperationBase.runScript(ScriptOperationBase.java:56)
com.snc.process_flow.operation.script.ScriptOperation.run(ScriptOperation.java:75)
com.snc.process_flow.engine.Operation.run2(Operation.java:90)
com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:74)
com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:68)
com.snc.process_flow.v2.exec.OpInstructionExecutor.runOperation(OpInstructionExecutor.java:62)
com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:29)
com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:16)
com.snc.process_flow.v2.exec.EngineV2.executeInstructions(EngineV2.java:183)
com.snc.process_flow.v2.exec.EngineV2.run(EngineV2.java:107)
com.snc.process_flow.v2.exec.EngineRunner.lambda$run$0(EngineRunner.java:28)
com.snc.process_flow.v2.exec.enviroment.execution.ExecutionScopeRunner.run(ExecutionScopeRunner.java:37)
com.snc.process_flow.v2.exec.EngineRunner.run(EngineRunner.java:28)
com.snc.process_flow.engine.EngineV2Adapter.runWithExecutionResult(EngineV2Adapter.java:160)
com.snc.process_flow.engine.EngineV2Adapter.resumeWithExecutionResult(EngineV2Adapter.java:443)
com.snc.process_flow.engine.ProcessAutomation.runWithExecutionResult(ProcessAutomation.java:107)
com.snc.process_flow.engine.ProcessAutomation.run(ProcessAutomation.java:70)
com.snc.process_flow.engine.GlideProcessAutomation.runSync(GlideProcessAutomation.java:254)
com.snc.process_flow.engine.GlideProcessAutomation.runWithDomain(GlideProcessAutomation.java:375)
com.snc.process_flow.engine.GlideProcessAutomation.lambda$runAsUserSync$2(GlideProcessAutomation.java:349)
com.snc.process_flow.engine.PFSessionClone.run(PFSessionClone.java:50)
com.snc.process_flow.engine.GlidePFSession.run(GlidePFSession.java:72)
com.snc.process_flow.engine.GlideProcessAutomation.runAsUserSync(GlideProcessAutomation.java:348)
com.snc.process_flow.engine.GlideProcessAutomation.messageFlow(GlideProcessAutomation.java:445)
com.snc.process_flow.engine.GlideProcessAutomation._start(GlideProcessAutomation.java:753)
com.snc.process_flow.engine.GlideProcessAutomation$StartBuilder.start(GlideProcessAutomation.java:1872)
com.snc.process_flow.engine.subflow.StartFlowOperation.run(StartFlowOperation.java:435)
com.snc.process_flow.engine.Operation.run2(Operation.java:90)
com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:74)
com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:68)
com.snc.process_flow.v2.exec.OpInstructionExecutor.runOperation(OpInstructionExecutor.java:62)
com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:29)
com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:16)
com.snc.process_flow.v2.exec.EngineV2.executeInstructions(EngineV2.java:183)
com.snc.process_flow.v2.exec.EngineV2.run(EngineV2.java:107)
com.snc.process_flow.v2.exec.EngineRunner.lambda$run$0(EngineRunner.java:28)
com.snc.process_flow.v2.exec.enviroment.execution.ExecutionScopeRunner.run(ExecutionScopeRunner.java:37)
com.snc.process_flow.v2.exec.EngineRunner.run(EngineRunner.java:28)
com.snc.process_flow.engine.EngineV2Adapter.runWithExecutionResult(EngineV2Adapter.java:160)
com.snc.process_flow.engine.EngineV2Adapter.resumeWithExecutionResult(EngineV2Adapter.java:443)
com.snc.process_flow.engine.ProcessAutomation.runWithExecutionResult(ProcessAutomation.java:107)
com.snc.process_flow.engine.ProcessAutomation.run(ProcessAutomation.java:70)
com.snc.process_flow.engine.GlideProcessAutomation.runSync(GlideProcessAutomation.java:254)
com.snc.process_flow.engine.GlideProcessAutomation.runWithDomain(GlideProcessAutomation.java:375)
com.snc.process_flow.engine.GlideProcessAutomation.lambda$runAsUserSync$2(GlideProcessAutomation.java:349)
com.snc.process_flow.engine.PFSessionClone.run(PFSessionClone.java:50)
com.snc.process_flow.engine.GlidePFSession.run(GlidePFSession.java:72)
com.snc.process_flow.engine.GlideProcessAutomation.runAsUserSync(GlideProcessAutomation.java:348)
com.snc.process_flow.engine.GlideProcessAutomation.messageFlow(GlideProcessAutomation.java:445)
com.snc.process_flow.engine.GlideProcessAutomation._start(GlideProcessAutomation.java:753)
com.snc.process_flow.engine.GlideProcessAutomation$StartBuilder.start(GlideProcessAutomation.java:1872)
com.glide.plan.runners.FlowObjectExecutor.startFlowObjectSynchronously(FlowObjectExecutor.java:340)
com.glide.plan.runners.FlowObjectExecutor.executeToCompletion(FlowObjectExecutor.java:198)
com.glide.plan.runners.FlowObjectExecutor.executeInternalWithResponse(FlowObjectExecutor.java:188)
com.glide.plan.runners.FlowObjectExecutor.lambda$executeWithResponse$0(FlowObjectExecutor.java:135)
com.glide.sys.Transaction.executeWithTimeout(Transaction.java:4018)
com.glide.plan.runners.FlowObjectExecutor.executeWithResponse(FlowObjectExecutor.java:135)
com.glide.plan.runners.FlowObjectAPI.executeWithResponse(FlowObjectAPI.java:109)
com.glide.plan.runners.FlowRunner.run(FlowRunner.java:355)
com.glide.plan.runners.scriptable.ScriptableFlowRunner.jsFunction_run(ScriptableFlowRunner.java:206)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:569)
org.mozilla.javascript.MemberBox.invoke(MemberBox.java:229)
org.mozilla.javascript.FunctionObject.doInvoke(FunctionObject.java:693)
org.mozilla.javascript.FunctionObject.call(FunctionObject.java:622)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3194)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sys_script_include_35cb33ee476a21105da8ff58936d4390_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sys_script_include_64e874a243f52110ff9fead67bb8f2b4_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sys_script_include_53b5af02849a2110fa9bfd0f8bd07b03_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sys_script_include_cd6e5e7d84922110fa9bfd0f8bd07bdb_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
org.mozilla.javascript.ContextFactory.doTopCall(ContextFactory.java:722)
org.mozilla.javascript.ScriptRuntime.doTopCall(ScriptRuntime.java:4812)
org.mozilla.javascript.InterpretedFunction.exec(InterpretedFunction.java:141)
com.glide.script.ScriptCompiler.executeAndPublishMetric(ScriptCompiler.java:83)
com.glide.script.ScriptEvaluator.execute(ScriptEvaluator.java:552)
com.glide.script.ScriptEvaluator.evaluate(ScriptEvaluator.java:254)
com.glide.script.ScriptEvaluator.evaluateString(ScriptEvaluator.java:442)
com.glide.script.ScriptEvaluator.evaluateString(ScriptEvaluator.java:166)
com.glide.service_portal.SPScriptEvaluator.eval(SPScriptEvaluator.java:72)
com.glide.service_portal.SPScriptEvaluator.eval(SPScriptEvaluator.java:87)
com.glide.service_portal.widget.SPScriptRunner.eval(SPScriptRunner.java:260)
com.glide.service_portal.widget.SPScriptRunner.runScript(SPScriptRunner.java:132)
com.glide.service_portal.widget.SPWidget.get(SPWidget.java:122)
com.glide.service_portal.widget.SPWidget.handleWidget(SPWidget.java:569)
com.glide.service_portal.SPRestService.widget(SPRestService.java:162)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:569)
com.glide.rest.handler.impl.ServiceHandlerImpl.invokeService(ServiceHandlerImpl.java:45)
com.glide.rest.processors.RESTAPIProcessor.invokeService(RESTAPIProcessor.java:508)
com.glide.rest.processors.RESTAPIProcessor.processInternal(RESTAPIProcessor.java:490)
com.glide.rest.processors.RESTAPIProcessor.process(RESTAPIProcessor.java:475)
com.glide.processors.AProcessor.runProcessor(AProcessor.java:919)
com.glide.processors.AProcessor.processTransaction(AProcessor.java:345)
com.glide.processors.ProcessorRegistry.process0(ProcessorRegistry.java:200)
com.glide.processors.ProcessorRegistry.process(ProcessorRegistry.java:188)
com.glide.ui.GlideServletTransaction.process(GlideServletTransaction.java:62)
com.glide.sys.Transaction.run(Transaction.java:3248)
com.glide.ui.HTTPTransaction.run(HTTPTransaction.java:44)
com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runTransaction(SemaphoreQueueThreadPool.java:338)
com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runThreadImpl(SemaphoreQueueThreadPool.java:303)
com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runThread(SemaphoreQueueThreadPool.java:150)
java.base/java.lang.Thread.run(Thread.java:841)
2026-10-28 15:20:32

Contract Core: Electronic Signature Adob...	Error	Encountered error executing instruction: OpInstruction{id=19, opClass=com.snc.process_flow.operation.script.ScriptOperation, io=ReadOnlyDefaultOutputsIo{input={mid_selection=StringValue{fValue='auto_select'}, capabilities=StringValue{fValue=''}, mid_server=StringValue{fValue=''}, documents=FlowReference{value=s4o.documents}, mid_cluster=StringValue{fValue=''}, participant_info=FlowReference{value=s2o.participants}, mid_selection_type=StringValue{fValue='any'}, message=StringValue{fValue='Please sign this document'}, connection_sys_id=FlowReference{value=s5i.connection_sys_id}, script=StringValue{fValue='(function execute(inputs, outputs) {
inputs = new AdobeSignUtils().trimStringInputs(inputs);
var mandatoryInputs = {
'name': inputs.name,
'group_record': inputs.group_record,
'participant_info': inputs.participant_info,
'documents': inputs.documents
}
new AdobeSignUtils().checkMandatoryInputs(mandatoryInputs);

outputs.name = inputs.name;
var inputDocs = inputs.documents;
var documents = [];
for (var i = 0; i < inputDocs.length; i++) {
var docObj = {};
var tempObj = inputDocs[i];
var document_type = "libraryDocumentId";
var id = tempObj.document_id;
switch (tempObj.document_source) {
case "library_document":
document_type = "libraryDocumentId";
break;
case "transient_document":
document_type = "transientDocumentId";
break;
}
docObj[document_type] = id;
documents.push(docObj);
}

var json_body = {};
var memberinfo = {};
var emailadd1 = {};
var memberinfo_array = [];
var memberinfo = {};
json_body["fileInfos"] = documents;
json_body["name"] = inputs.name;
if (inputs.participant_info.length > 0) {
var participants = [];
inputs.participant_info.forEach(function(property) {
var participant = {};
if (property.order) {
var order_syntex = /^[-+]?[0-9]+$/;
try {
var order = property.order.toString().trim();
if (order.match(order_syntex)) {
participant.order = order;
} else
throw new Error("Provided Signing Order is invalid.");
} catch (err) {
throw new Error("Provided Signing Order is invalid.");
}
}

if (property.role)
participant.role = property.role;
if (!gs.nil(property.private_message))
participant.privateMessage = property.private_message.trim();
if (property.email) {
var email_syntex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
try {
var emailAdd = property.email.toString().trim();
if (emailAdd.match(email_syntex)) {
var memberInfos = [];
var memberInfo = {};
memberInfo.email = emailAdd;
if(!gs.nil(property.name)) memberInfo.name = property.name.trim();
memberInfos.push(memberInfo);
participant.memberInfos = memberInfos;
} else {
throw new Error("Provided Participant Email is invalid.");
}
} catch (err) {
throw new Error("Provided Participant Email is invalid.");
}
}
participants.push(participant);
});
if (participants.length > 0) {
json_body["participantSetsInfo"] = participants;
var participants_obj = {};
participants_obj['participantSetsInfo'] = participants;
outputs.participants_info = JSON.stringify(participants_obj);
}
}
if (!inputs.agreement_record) {} else if (inputs.agreement_record && (inputs.agreement_record.status == "") || (inputs.agreement_record.status == "DRAFT")) {

} else {
throw new Error("Provided agreement status should be DRAFT or null.");
}
json_body["message"] = inputs.message;
json_body["signatureType"] = "ESIGN";
json_body["state"] = inputs.state;
json_body["externalId"] = "API_SERVICENOW_SPOKE";

if (inputs.suppress_notification == true) {
json_body["emailOption"] = {};
json_body["emailOption"]["sendOptions"] = {};
json_body["emailOption"]["sendOptions"]["completionEmails"] = "NONE";
json_body["emailOption"]["sendOptions"]["inFlightEmails"] = "NONE";
json_body["emailOption"]["sendOptions"]["initEmails"] = "NONE";
}

outputs.group_record = new AdobeSignUtils().validateGroupRecord(inputs.group_record);
outputs.json_body = JSON.stringify(json_body);
var connection_sys_id = inputs.connection_sys_id;
outputs.adobe_version = new AdobeSignUtils().validateAdobeVersion(connection_sys_id);
}(inputs, outputs));'}, connection_alias=StringValue{fValue=''}, application=StringValue{fValue='35aa573fd7802200bdbaee5b5e610375'}, required_run_time=StringValue{fValue='instance'}, host=StringValue{fValue=''}, group_record=FlowReference{value=s3o.record}, name=FlowReference{value=c6i.name}, state=FlowReference{value=c6i.state.__result}, suppress_notification=BooleanValue{value=false}, agreement_record=StringValue{fValue=''}}, output={json_body=s5o.json_body, name=s5o.name, group_record=s5o.group_record, participants_info=s5o.participants_info, __step_status__=s5o.__step_status__, adobe_version=s5o.adobe_version}, defaultOutput={}}, name=707db6de43702110f588ead67bb8f219, parentActionId=94983e9a43702110f588ead67bb8f2d2}
com.snc.process_flow.exception.OpException: Error: Mandatory inputs cannot be empty (sys_script_include.e3b33778db4227003e146583ca961970.script; line 117)
at com.snc.process_flow.operation.script.ScriptOperationBase.handleScriptResult(ScriptOperationBase.java:83)
at com.snc.process_flow.operation.script.ScriptOperationBase.runScript(ScriptOperationBase.java:57)
at com.snc.process_flow.operation.script.ScriptOperation.run(ScriptOperation.java:75)
at com.snc.process_flow.engine.Operation.run2(Operation.java:90)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:74)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:68)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.runOperation(OpInstructionExecutor.java:62)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:29)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:16)
at com.snc.process_flow.v2.exec.EngineV2.executeInstructions(EngineV2.java:183)
at com.snc.process_flow.v2.exec.EngineV2.run(EngineV2.java:107)
at com.snc.process_flow.v2.exec.EngineRunner.lambda$run$0(EngineRunner.java:28)
at com.snc.process_flow.v2.exec.enviroment.execution.ExecutionScopeRunner.run(ExecutionScopeRunner.java:37)
at com.snc.process_flow.v2.exec.EngineRunner.run(EngineRunner.java:28)
at com.snc.process_flow.engine.EngineV2Adapter.runWithExecutionResult(EngineV2Adapter.java:160)
at com.snc.process_flow.engine.EngineV2Adapter.resumeWithExecutionResult(EngineV2Adapter.java:443)
at com.snc.process_flow.engine.ProcessAutomation.runWithExecutionResult(ProcessAutomation.java:107)
at com.snc.process_flow.engine.ProcessAutomation.run(ProcessAutomation.java:70)
at com.snc.process_flow.engine.GlideProcessAutomation.runSync(GlideProcessAutomation.java:254)
at com.snc.process_flow.engine.GlideProcessAutomation.runWithDomain(GlideProcessAutomation.java:375)
at com.snc.process_flow.engine.GlideProcessAutomation.lambda$runAsUserSync$2(GlideProcessAutomation.java:349)
at com.snc.process_flow.engine.PFSessionClone.run(PFSessionClone.java:50)
at com.snc.process_flow.engine.GlidePFSession.run(GlidePFSession.java:72)
at com.snc.process_flow.engine.GlideProcessAutomation.runAsUserSync(GlideProcessAutomation.java:348)
at com.snc.process_flow.engine.GlideProcessAutomation.messageFlow(GlideProcessAutomation.java:445)
at com.snc.process_flow.engine.GlideProcessAutomation._start(GlideProcessAutomation.java:753)
at com.snc.process_flow.engine.GlideProcessAutomation$StartBuilder.start(GlideProcessAutomation.java:1872)
at com.snc.process_flow.engine.subflow.StartFlowOperation.run(StartFlowOperation.java:435)
at com.snc.process_flow.engine.Operation.run2(Operation.java:90)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:74)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:68)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.runOperation(OpInstructionExecutor.java:62)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:29)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:16)
at com.snc.process_flow.v2.exec.EngineV2.executeInstructions(EngineV2.java:183)
at com.snc.process_flow.v2.exec.EngineV2.run(EngineV2.java:107)
at com.snc.process_flow.v2.exec.EngineRunner.lambda$run$0(EngineRunner.java:28)
at com.snc.process_flow.v2.exec.enviroment.execution.ExecutionScopeRunner.run(ExecutionScopeRunner.java:37)
at com.snc.process_flow.v2.exec.EngineRunner.run(EngineRunner.java:28)
at com.snc.process_flow.engine.EngineV2Adapter.runWithExecutionResult(EngineV2Adapter.java:160)
at com.snc.process_flow.engine.EngineV2Adapter.resumeWithExecutionResult(EngineV2Adapter.java:443)
at com.snc.process_flow.engine.ProcessAutomation.runWithExecutionResult(ProcessAutomation.java:107)
at com.snc.process_flow.engine.ProcessAutomation.run(ProcessAutomation.java:70)
at com.snc.process_flow.engine.GlideProcessAutomation.runSync(GlideProcessAutomation.java:254)
at com.snc.process_flow.engine.GlideProcessAutomation.runWithDomain(GlideProcessAutomation.java:375)
at com.snc.process_flow.engine.GlideProcessAutomation.lambda$runAsUserSync$2(GlideProcessAutomation.java:349)
at com.snc.process_flow.engine.PFSessionClone.run(PFSessionClone.java:50)
at com.snc.process_flow.engine.GlidePFSession.run(GlidePFSession.java:72)
at com.snc.process_flow.engine.GlideProcessAutomation.runAsUserSync(GlideProcessAutomation.java:348)
at com.snc.process_flow.engine.GlideProcessAutomation.messageFlow(GlideProcessAutomation.java:445)
at com.snc.process_flow.engine.GlideProcessAutomation._start(GlideProcessAutomation.java:753)
at com.snc.process_flow.engine.GlideProcessAutomation$StartBuilder.start(GlideProcessAutomation.java:1872)
at com.glide.plan.runners.FlowObjectExecutor.startFlowObjectSynchronously(FlowObjectExecutor.java:340)
at com.glide.plan.runners.FlowObjectExecutor.executeToCompletion(FlowObjectExecutor.java:198)
at com.glide.plan.runners.FlowObjectExecutor.executeInternalWithResponse(FlowObjectExecutor.java:188)
at com.glide.plan.runners.FlowObjectExecutor.lambda$executeWithResponse$0(FlowObjectExecutor.java:135)
at com.glide.sys.Transaction.executeWithTimeout(Transaction.java:4018)
at com.glide.plan.runners.FlowObjectExecutor.executeWithResponse(FlowObjectExecutor.java:135)
at com.glide.plan.runners.FlowObjectAPI.executeWithResponse(FlowObjectAPI.java:109)
at com.glide.plan.runners.FlowRunner.run(FlowRunner.java:355)
at com.glide.plan.runners.scriptable.ScriptableFlowRunner.jsFunction_run(ScriptableFlowRunner.java:206)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.base/java.lang.reflect.Method.invoke(Method.java:569)
at org.mozilla.javascript.MemberBox.invoke(MemberBox.java:229)
at org.mozilla.javascript.FunctionObject.doInvoke(FunctionObject.java:693)
at org.mozilla.javascript.FunctionObject.call(FunctionObject.java:622)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3194)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_35cb33ee476a21105da8ff58936d4390_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_64e874a243f52110ff9fead67bb8f2b4_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_53b5af02849a2110fa9bfd0f8bd07b03_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_cd6e5e7d84922110fa9bfd0f8bd07bdb_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ContextFactory.doTopCall(ContextFactory.java:722)
at org.mozilla.javascript.ScriptRuntime.doTopCall(ScriptRuntime.java:4812)
at org.mozilla.javascript.InterpretedFunction.exec(InterpretedFunction.java:141)
at com.glide.script.ScriptCompiler.executeAndPublishMetric(ScriptCompiler.java:83)
at com.glide.script.ScriptEvaluator.execute(ScriptEvaluator.java:552)
at com.glide.script.ScriptEvaluator.evaluate(ScriptEvaluator.java:254)
at com.glide.script.ScriptEvaluator.evaluateString(ScriptEvaluator.java:442)
at com.glide.script.ScriptEvaluator.evaluateString(ScriptEvaluator.java:166)
at com.glide.service_portal.SPScriptEvaluator.eval(SPScriptEvaluator.java:72)
at com.glide.service_portal.SPScriptEvaluator.eval(SPScriptEvaluator.java:87)
at com.glide.service_portal.widget.SPScriptRunner.eval(SPScriptRunner.java:260)
at com.glide.service_portal.widget.SPScriptRunner.runScript(SPScriptRunner.java:132)
at com.glide.service_portal.widget.SPWidget.get(SPWidget.java:122)
at com.glide.service_portal.widget.SPWidget.handleWidget(SPWidget.java:569)
at com.glide.service_portal.SPRestService.widget(SPRestService.java:162)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.base/java.lang.reflect.Method.invoke(Method.java:569)
at com.glide.rest.handler.impl.ServiceHandlerImpl.invokeService(ServiceHandlerImpl.java:45)
at com.glide.rest.processors.RESTAPIProcessor.invokeService(RESTAPIProcessor.java:508)
at com.glide.rest.processors.RESTAPIProcessor.processInternal(RESTAPIProcessor.java:490)
at com.glide.rest.processors.RESTAPIProcessor.process(RESTAPIProcessor.java:475)
at com.glide.processors.AProcessor.runProcessor(AProcessor.java:919)
at com.glide.processors.AProcessor.processTransaction(AProcessor.java:345)
at com.glide.processors.ProcessorRegistry.process0(ProcessorRegistry.java:200)
at com.glide.processors.ProcessorRegistry.process(ProcessorRegistry.java:188)
at com.glide.ui.GlideServletTransaction.process(GlideServletTransaction.java:62)
at com.glide.sys.Transaction.run(Transaction.java:3248)
at com.glide.ui.HTTPTransaction.run(HTTPTransaction.java:44)
at com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runTransaction(SemaphoreQueueThreadPool.java:338)
at com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runThreadImpl(SemaphoreQueueThreadPool.java:303)
at com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runThread(SemaphoreQueueThreadPool.java:150)
at java.base/java.lang.Thread.run(Thread.java:841)
2026-10-28 15:20:32

Contract Core: Electronic Signature Adob...	Error	Encountered error executing instruction: OpInstruction{id=71, opClass=com.snc.process_flow.operation.script.ScriptOperation, io=ReadOnlyDefaultOutputsIo{input={mid_selection=StringValue{fValue='auto_select'}, capabilities=StringValue{fValue=''}, agreement=FlowReference{value=s10o.agreement_record}, connection_alias=StringValue{fValue=''}, application=StringValue{fValue='35aa573fd7802200bdbaee5b5e610375'}, mid_server=StringValue{fValue=''}, mid_cluster=StringValue{fValue=''}, required_run_time=StringValue{fValue='instance'}, host=StringValue{fValue=''}, context=FlowReference{value=in.context_object}, mid_selection_type=StringValue{fValue='any'}, script=StringValue{fValue='(function execute(inputs, outputs) {
var extension = ContractIntegrations.getProviderExtensionPoint("esign_adobe_sign");
if (!gs.nil(extension)) {
extension.handleAgreement(inputs.agreement, inputs.context);
}
})(inputs, outputs);
'}}, output={__step_status__=s11o.__step_status__}, defaultOutput={}}, name=9467b28847cbe510b3247b48946d436a, parentActionId=bae57e0847cbe510b3247b48946d43ae}
com.snc.process_flow.exception.OpException: Error: Cannot convert null to an object.,Detail: Cannot convert null to an object.
at com.snc.process_flow.operation.script.ScriptOperationBase.handleScriptResult(ScriptOperationBase.java:83)
at com.snc.process_flow.operation.script.ScriptOperationBase.runScript(ScriptOperationBase.java:57)
at com.snc.process_flow.operation.script.ScriptOperation.run(ScriptOperation.java:75)
at com.snc.process_flow.engine.Operation.run2(Operation.java:90)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:74)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:68)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.runOperation(OpInstructionExecutor.java:62)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:29)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:16)
at com.snc.process_flow.v2.exec.EngineV2.executeInstructions(EngineV2.java:183)
at com.snc.process_flow.v2.exec.EngineV2.run(EngineV2.java:107)
at com.snc.process_flow.v2.exec.EngineRunner.lambda$run$0(EngineRunner.java:28)
at com.snc.process_flow.v2.exec.enviroment.execution.ExecutionScopeRunner.run(ExecutionScopeRunner.java:37)
at com.snc.process_flow.v2.exec.EngineRunner.run(EngineRunner.java:28)
at com.snc.process_flow.engine.EngineV2Adapter.runWithExecutionResult(EngineV2Adapter.java:160)
at com.snc.process_flow.engine.EngineV2Adapter.resumeWithExecutionResult(EngineV2Adapter.java:443)
at com.snc.process_flow.engine.ProcessAutomation.runWithExecutionResult(ProcessAutomation.java:107)
at com.snc.process_flow.engine.ProcessAutomation.run(ProcessAutomation.java:70)
at com.snc.process_flow.engine.GlideProcessAutomation.runSync(GlideProcessAutomation.java:254)
at com.snc.process_flow.engine.GlideProcessAutomation.runWithDomain(GlideProcessAutomation.java:375)
at com.snc.process_flow.engine.GlideProcessAutomation.lambda$runAsUserSync$2(GlideProcessAutomation.java:349)
at com.snc.process_flow.engine.PFSessionClone.run(PFSessionClone.java:50)
at com.snc.process_flow.engine.GlidePFSession.run(GlidePFSession.java:72)
at com.snc.process_flow.engine.GlideProcessAutomation.runAsUserSync(GlideProcessAutomation.java:348)
at com.snc.process_flow.engine.GlideProcessAutomation.messageFlow(GlideProcessAutomation.java:445)
at com.snc.process_flow.engine.GlideProcessAutomation._start(GlideProcessAutomation.java:753)
at com.snc.process_flow.engine.GlideProcessAutomation$StartBuilder.start(GlideProcessAutomation.java:1872)
at com.snc.process_flow.engine.subflow.StartFlowOperation.run(StartFlowOperation.java:435)
at com.snc.process_flow.engine.Operation.run2(Operation.java:90)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:74)
at com.snc.process_flow.v2.exec.OpRunner.runOperation(OpRunner.java:68)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.runOperation(OpInstructionExecutor.java:62)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:29)
at com.snc.process_flow.v2.exec.OpInstructionExecutor.execute(OpInstructionExecutor.java:16)
at com.snc.process_flow.v2.exec.EngineV2.executeInstructions(EngineV2.java:183)
at com.snc.process_flow.v2.exec.EngineV2.run(EngineV2.java:107)
at com.snc.process_flow.v2.exec.EngineRunner.lambda$run$0(EngineRunner.java:28)
at com.snc.process_flow.v2.exec.enviroment.execution.ExecutionScopeRunner.run(ExecutionScopeRunner.java:37)
at com.snc.process_flow.v2.exec.EngineRunner.run(EngineRunner.java:28)
at com.snc.process_flow.engine.EngineV2Adapter.runWithExecutionResult(EngineV2Adapter.java:160)
at com.snc.process_flow.engine.EngineV2Adapter.resumeWithExecutionResult(EngineV2Adapter.java:443)
at com.snc.process_flow.engine.ProcessAutomation.runWithExecutionResult(ProcessAutomation.java:107)
at com.snc.process_flow.engine.ProcessAutomation.run(ProcessAutomation.java:70)
at com.snc.process_flow.engine.GlideProcessAutomation.runSync(GlideProcessAutomation.java:254)
at com.snc.process_flow.engine.GlideProcessAutomation.runWithDomain(GlideProcessAutomation.java:375)
at com.snc.process_flow.engine.GlideProcessAutomation.lambda$runAsUserSync$2(GlideProcessAutomation.java:349)
at com.snc.process_flow.engine.PFSessionClone.run(PFSessionClone.java:50)
at com.snc.process_flow.engine.GlidePFSession.run(GlidePFSession.java:72)
at com.snc.process_flow.engine.GlideProcessAutomation.runAsUserSync(GlideProcessAutomation.java:348)
at com.snc.process_flow.engine.GlideProcessAutomation.messageFlow(GlideProcessAutomation.java:445)
at com.snc.process_flow.engine.GlideProcessAutomation._start(GlideProcessAutomation.java:753)
at com.snc.process_flow.engine.GlideProcessAutomation$StartBuilder.start(GlideProcessAutomation.java:1872)
at com.glide.plan.runners.FlowObjectExecutor.startFlowObjectSynchronously(FlowObjectExecutor.java:340)
at com.glide.plan.runners.FlowObjectExecutor.executeToCompletion(FlowObjectExecutor.java:198)
at com.glide.plan.runners.FlowObjectExecutor.executeInternalWithResponse(FlowObjectExecutor.java:188)
at com.glide.plan.runners.FlowObjectExecutor.lambda$executeWithResponse$0(FlowObjectExecutor.java:135)
at com.glide.sys.Transaction.executeWithTimeout(Transaction.java:4018)
at com.glide.plan.runners.FlowObjectExecutor.executeWithResponse(FlowObjectExecutor.java:135)
at com.glide.plan.runners.FlowObjectAPI.executeWithResponse(FlowObjectAPI.java:109)
at com.glide.plan.runners.FlowRunner.run(FlowRunner.java:355)
at com.glide.plan.runners.scriptable.ScriptableFlowRunner.jsFunction_run(ScriptableFlowRunner.java:206)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.base/java.lang.reflect.Method.invoke(Method.java:569)
at org.mozilla.javascript.MemberBox.invoke(MemberBox.java:229)
at org.mozilla.javascript.FunctionObject.doInvoke(FunctionObject.java:693)
at org.mozilla.javascript.FunctionObject.call(FunctionObject.java:622)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3194)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_35cb33ee476a21105da8ff58936d4390_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_64e874a243f52110ff9fead67bb8f2b4_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_53b5af02849a2110fa9bfd0f8bd07b03_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sys_script_include_cd6e5e7d84922110fa9bfd0f8bd07bdb_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ScriptRuntime.doCall2(ScriptRuntime.java:3296)
at org.mozilla.javascript.ScriptRuntime.doCall(ScriptRuntime.java:3204)
at org.mozilla.javascript.Interpreter.interpretLoop(Interpreter.java:1968)
at org.mozilla.javascript.Interpreter.interpret(Interpreter.java:940)
at org.mozilla.javascript.InterpretedFunction.lambda$call$0(InterpretedFunction.java:127)
at com.glide.caller.gen.sp_widget_1ebf13f7e0a63010fa9b460b5b232860_script.call(Unknown Source)
at com.glide.script.ScriptCaller.call(ScriptCaller.java:22)
at org.mozilla.javascript.InterpretedFunction.call(InterpretedFunction.java:125)
at org.mozilla.javascript.ContextFactory.doTopCall(ContextFactory.java:722)
at org.mozilla.javascript.ScriptRuntime.doTopCall(ScriptRuntime.java:4812)
at org.mozilla.javascript.InterpretedFunction.exec(InterpretedFunction.java:141)
at com.glide.script.ScriptCompiler.executeAndPublishMetric(ScriptCompiler.java:83)
at com.glide.script.ScriptEvaluator.execute(ScriptEvaluator.java:552)
at com.glide.script.ScriptEvaluator.evaluate(ScriptEvaluator.java:254)
at com.glide.script.ScriptEvaluator.evaluateString(ScriptEvaluator.java:442)
at com.glide.script.ScriptEvaluator.evaluateString(ScriptEvaluator.java:166)
at com.glide.service_portal.SPScriptEvaluator.eval(SPScriptEvaluator.java:72)
at com.glide.service_portal.SPScriptEvaluator.eval(SPScriptEvaluator.java:87)
at com.glide.service_portal.widget.SPScriptRunner.eval(SPScriptRunner.java:260)
at com.glide.service_portal.widget.SPScriptRunner.runScript(SPScriptRunner.java:132)
at com.glide.service_portal.widget.SPWidget.get(SPWidget.java:122)
at com.glide.service_portal.widget.SPWidget.handleWidget(SPWidget.java:569)
at com.glide.service_portal.SPRestService.widget(SPRestService.java:162)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.base/java.lang.reflect.Method.invoke(Method.java:569)
at com.glide.rest.handler.impl.ServiceHandlerImpl.invokeService(ServiceHandlerImpl.java:45)
at com.glide.rest.processors.RESTAPIProcessor.invokeService(RESTAPIProcessor.java:508)
at com.glide.rest.processors.RESTAPIProcessor.processInternal(RESTAPIProcessor.java:490)
at com.glide.rest.processors.RESTAPIProcessor.process(RESTAPIProcessor.java:475)
at com.glide.processors.AProcessor.runProcessor(AProcessor.java:919)
at com.glide.processors.AProcessor.processTransaction(AProcessor.java:345)
at com.glide.processors.ProcessorRegistry.process0(ProcessorRegistry.java:200)
at com.glide.processors.ProcessorRegistry.process(ProcessorRegistry.java:188)
at com.glide.ui.GlideServletTransaction.process(GlideServletTransaction.java:62)
at com.glide.sys.Transaction.run(Transaction.java:3248)
at com.glide.ui.HTTPTransaction.run(HTTPTransaction.java:44)
at com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runTransaction(SemaphoreQueueThreadPool.java:338)
at com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runThreadImpl(SemaphoreQueueThreadPool.java:303)
at com.glide.sys.util.sema.SemaphoreQueueThreadPool$Semaphore.runThread(SemaphoreQueueThreadPool.java:150)
at java.base/java.lang.Thread.run(Thread.java:841)
2026-10-28 15:20:32
Encountered error executing instruction: ActionErrorEvalInstruction{id=73, conditions=[], statusKey=c10o.__action_status__, dontTreatAsErrorKey=c10o.__dont_treat_as_error__}, errorMessage:Error: Cannot convert null to an object.,Detail: Cannot convert null to an object., errorCode:1

Contract Core: Electronic Signature Adob...	Error	Encountered error executing instruction: ActionErrorEvalInstruction{id=73, conditions=[], statusKey=c10o.__action_status__, dontTreatAsErrorKey=c10o.__dont_treat_as_error__}, errorMessage:Error: Cannot convert null to an object.,Detail: Cannot convert null to an object., errorCode:1
