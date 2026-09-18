# PsyAgent: Structural PoC Against LangGraph and AutoGen

PsyAgent is a four-stage attack chain, one classical social-engineering primitive per stage, that targets four common weak points in an agentic AI pipeline: context/retrieval, tool-call arguments, persistent memory, and cross-session identity delegation. It comes out of the manipulation-primitive framework in "AI Security From PsyOps to CyberOps" (SecuredAI.org).

This repo contains two independent implementations of the chain, one against [LangGraph](https://github.com/langchain-ai/langgraph), one against [AutoGen](https://github.com/microsoft/autogen) (the autogen-agentchat/autogen-core lineage), plus the raw output from running both.

## Read this before the results below mean anything

Both scripts run real, current releases of their respective frameworks end to end. Nothing about the framework behavior shown here is simulated.

What is not real: at each point where a production deployment would have a live LLM deciding what to do, both scripts use a scripted, deterministic stand-in (`ScriptedModelClient` in the AutoGen script; a plain Python function in the LangGraph script) that reproduces the well-documented, already-known behavior pattern of a model that has been successfully manipulated by authority, urgency, and familiarity framing. No live model API was available when these were built.

So: this validates that neither framework provides a default structural boundary stopping a manipulated decision from reaching a real tool call, a real persisted memory write, and a real cross-session privilege delegation, the same three gaps in both frameworks despite very different architectures. This does NOT validate that any specific real model (Claude, GPT, or otherwise) would actually be manipulated by the injected text used here. That would need a live-model run, which is the natural next step for anyone extending this.

Full write-up: [`FINDINGS.md`](./FINDINGS.md).

## The chain

1. **Context poisoning (authority).** A document entering through a retrieval step is framed as an IT-administrator system notice.
2. **Tool-call argument injection (urgency).** Time-pressure framing in the same document drives a tool call with attacker-supplied arguments.
3. **Memory persistence (familiarity).** The interaction is written to long-term memory, framed as something already agreed with the security team.
4. **Identity delegation (reciprocity).** A separate, later agent session reads that memory as established fact and uses it to justify delegating elevated privilege to a second, higher-privileged service.

## Running it

```bash
pip install -r requirements.txt
python3 etherrat_langgraph_poc.py
python3 etherrat_autogen_poc.py
```

Both scripts are self-contained and print a full execution trace plus a structural-findings summary. `run_logs_langgraph.txt` and `run_logs_autogen.txt` are the raw output from the runs described in `FINDINGS.md`.

## Structural findings, summarized

The same three gaps showed up in both frameworks, despite LangGraph's graph/state architecture and AutoGen's agent-message-passing architecture being fundamentally different:

1. No default separation between untrusted, retrieved content and developer- or user-authored content once either is in context.
2. No default policy checkpoint between a model's decision and tool execution.
3. No default re-verification of values read from persistent memory across sessions or agent instances.

None of this is a defect in either framework; both are intentionally unopinionated about trust boundaries. The point of EtherRAT is that these three gaps are exactly what the four manipulation primitives are built to exploit, and that closing them is an application-level architectural decision, not something either framework provides by default.

## License

MIT. See [LICENSE](./LICENSE).

## Related

- Book: "AI Security From PsyOps to CyberOps" (SecuredAI.org)
- This chain and its findings also underpin a Black Hat Asia 2027 submission and a companion governance-policy-builder Arsenal submission.
