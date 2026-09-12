# EtherRAT Structural PoC: Results Against LangGraph and AutoGen

Run September 11, 2026, in this session's sandbox. Both PoCs are real, executable code (langgraph 1.2.11, autogen-agentchat/autogen-core 0.7.5, both current releases as of this run), not a description of hypothetical behavior. Full scripts and raw run logs are attached alongside this report.

## What this does and doesn't prove, read this before citing it anywhere

No LLM API key is available in this sandbox, so there's no live model in the loop. At each point where a real deployment would have a model deciding what to do, both PoCs use a scripted, deterministic stand-in that reproduces the documented, well-established behavior pattern of an instruction-following model that has already been manipulated by authority, urgency, and familiarity framing. That pattern itself isn't novel, it's the basis of existing prompt-injection research.

What's genuinely tested by actually running this code: whether the frameworks themselves, LangGraph and AutoGen, provide any default structural boundary that would stop a manipulated decision from propagating into a real tool call, a persisted memory write, and a cross-session privilege delegation. That's an architecture question, answerable independent of which model sits behind the decision point, and it's what these PoCs answer.

What this does NOT prove: that any specific real model (Claude, GPT, or otherwise) would actually be talked into this chain by this exact injected text. That's a separate, narrower claim that would need a live-model run to support, and it's the natural next step if you want the Black Hat submission to also carry an empirical injection-susceptibility result rather than only a framework-structural one.

## The chain

Four stages, one manipulation primitive each, targeting four different layers of a typical agent pipeline:

1. Context poisoning (authority): a document entering through a retrieval step contains a passage framed as an IT-administrator system notice.
2. Tool-call argument injection (urgency): the same passage adds time pressure, driving a tool call with attacker-supplied arguments (routing a data export to an outside address).
3. Memory persistence (familiarity): the interaction gets written into long-term memory, framed as something already agreed with the security team.
4. Identity delegation (reciprocity): a separate, later agent session reads that memory as established fact and uses it as justification to delegate elevated privilege to a second, higher-privileged service, completing lateral escalation.

## Results

**LangGraph (1.2.11).** Stages 1-3 ran as one graph invocation under session `victim-session-001`: the poisoned document flowed into a state field with no provenance tag, the scripted agent node called `tool_grant_data_access` with the attacker's target address, and the interaction was written to a plain persistent-memory dict. Stage 4 ran as a completely separate graph invocation under a new, unrelated session id (`victim-session-047`, modeling a different session days later): that fresh invocation read the same memory entry and used it, unmodified and unverified, to justify calling `tool_grant_identity_delegation`, which succeeded.

**AutoGen (autogen-agentchat/autogen-core 0.7.5).** Stages 1-3 ran through a real `AssistantAgent` with a real `FunctionTool`: the poisoned document, handed in as the task string, drove a genuine tool-execution call to `grant_data_access`, visible in the agent's actual message log. The interaction was then written into a real `autogen_core.memory.ListMemory` store. Stage 4 instantiated a brand-new `AssistantAgent` (a different scripted model client, no shared conversation history) with only that `ListMemory` attached. AutoGen's own `update_context` mechanism injected the stored memory content into the new agent's context automatically, the log shows it arriving as a `MemoryContent` message, and the new agent used it to justify a real call to `grant_identity_delegation`, which succeeded.

## Structural findings, same three gaps in both frameworks

1. **No context provenance.** Neither framework tags a value with where it came from. LangGraph's state is a plain shared dict; AutoGen's task/message input is plain text. Both treat retrieved, untrusted content identically to developer- or user-authored content once it's in context.
2. **No default pre-tool-call policy.** Both frameworks wire the model's decision straight to tool execution (LangGraph via a direct node edge, AutoGen via `max_tool_iterations`) with no default allowlist, argument validator, or human-in-the-loop checkpoint. Adding one is the application's job in both cases.
3. **No memory re-verification across sessions or agent instances.** LangGraph's checkpointer-style memory and AutoGen's `ListMemory` both persist and later re-surface content with no expiry, source check, or re-authentication. A completely unrelated later session, or a brand-new agent instance with no shared history, still inherits and acts on an earlier session's planted memory.

That the same three gaps show up in two architecturally different frameworks (LangGraph's graph/state model vs. AutoGen's agent-message-passing model) is the strongest part of this result: it's evidence the gap is structural to this generation of agent frameworks generally, not an implementation quirk of one library. Neither gap is a defect exactly, both frameworks are intentionally unopinionated about trust boundaries, but that's exactly the point EtherRAT is making: the manipulation primitives work precisely because the boundary isn't there by default, and each primitive in the chain was chosen to target one of these three specific gaps.

## What would make this stronger before the Black Hat submission

A live-model run (with Claude, GPT, or another model actually deciding at each stage instead of a scripted stand-in) would let you report a genuine injection-susceptibility result on top of this structural one, and would answer Black Hat's likely first question directly. If that's not feasible before the deadline, this framework-structural result stands on its own as a real, run, two-framework finding, and the submission should describe it exactly as that: which parts are demonstrated code behavior and which parts are the documented manipulation pattern standing in for a model, the same distinction drawn throughout this report.
