---
layout: post
title: "We Used to Talk About Tor. Well We’ve Got LLM Agents"
date: 2026-04-14
categories: llm security
image: /assets/wbtor.png
---

![We Used to Talk About Tor. Well We’ve Got LLM Agents](/assets/wbtor.png)

There was a time when internet privacy debates had a relatively stable shape.

We talked about Tor, VPNs, encrypted email, browser fingerprinting, metadata retention, and traffic analysis. The underlying model was clear enough: a human user interacted with a networked environment, and the primary risk was that this interaction could be observed, recorded, correlated, and ultimately exploited. The goal of security and privacy technologies was therefore to reduce visibility, distribute trust, and make surveillance more expensive or less reliable.

Those concerns remain valid. But they are no longer sufficient to describe the current landscape.

What has changed is not only how data is transmitted or stored, but where action itself takes place. Increasingly, users are not acting alone. They are accompanied by software systems - LLM-based assistants, copilots, and agents - that read across data sources, interpret intent, retrieve context, and, in many cases, take action on their behalf. These systems do not simply protect or expose user activity. They participate in it.

This introduces a qualitatively different problem. The question is no longer only who can observe the user. It is also what can act for the user, what information that system must consume in order to do so, and how its decision-making process can be influenced or subverted.

Tor addressed concealment. Agents introduce delegated authority.

That distinction is not superficial. It alters the level at which security needs to be reasoned about.

## From protecting communication to governing execution

The traditional privacy stack focused largely on protecting communication paths. Systems like Tor obscured origin through layered routing; TLS secured content in transit; end-to-end encryption attempted to ensure that even service providers could not access message contents. Anti-tracking tools reduced the ability of platforms to correlate user behavior across contexts.

These mechanisms were designed for a world in which the user initiated discrete actions. The system's responsibility was to carry or protect those actions, not to originate them.

Agentic systems shift this boundary upward. They are not merely transporting user intent; they are interpreting it, extending it, and, in some cases, generating new actions that the user did not explicitly specify in detail. This moves the security problem away from transport and storage and toward interpretation, planning, and execution.

In practical terms, this means that the integrity of the system no longer depends only on whether data is encrypted or access-controlled, but on whether the system correctly understands what it is supposed to do and whose authority it is operating under.

## What "agent" means in real systems

The term "agent" is often used loosely, so it is useful to ground it in actual system design.

In most current implementations, an agent consists of a language model acting as a central planner within a control loop. It receives user input and contextual state, retrieves additional information through search or RAG pipelines, and has access to a set of tools - these might include APIs, file systems, browsers, databases, or code execution environments. Based on the combined context, the model proposes actions, which are executed by the system, and the results are fed back into the loop until some completion condition is reached.

This architecture effectively turns the model into a coordination layer across heterogeneous systems. It is not simply generating text; it is orchestrating operations. The critical point is that the same mechanism used to interpret natural language is now also responsible for selecting actions that have real side effects.

This coupling between interpretation and execution is what creates new risk.

A striking amount of the current ecosystem still builds agents in a way that, from a security perspective, is essentially equivalent to letting the model read everything, decide everything, and call everything. In pseudo-code, the unsafe version looks something like this:

```python
def run_agent(user_request: str):
    context = []
    context.append({"role": "user", "content": user_request})

    retrieved_docs = rag_search(user_request)
    for doc in retrieved_docs:
        context.append({"role": "system", "content": doc.text})

    while True:
        response = llm.generate(context, tools=ALL_TOOLS)

        if response.type == "tool_call":
            result = execute_tool(
                name=response.tool_name,
                args=response.tool_args,
            )
            context.append({"role": "tool", "content": str(result)})
        else:
            return response.content
```

At first glance this looks clean. It is also a compact summary of the problem. Retrieved documents are inserted into the same effective decision space as the user's request. The model is trusted to decide which tool to call. All tools are available in the same loop. There is no external policy layer, no trust separation, no approval boundary, and no explicit identity scoping. If one of the retrieved documents contains adversarial instructions, or if the model simply infers the wrong next step, the system has no meaningful brake.

This is the architectural equivalent of saying: "Here is a probabilistic parser of ambiguous language. Let it sit in the middle of our infrastructure."

## The expansion of the attack surface into context

Traditional software systems treat input as data that must be validated before use. Agentic systems, by design, ingest large volumes of heterogeneous input and treat it as part of the reasoning process.

This input may include webpages, documents, emails, chat messages, code, logs, and prior outputs from the system itself. Importantly, there is no inherent distinction between "data" and "instructions" in natural language. Once incorporated into the model's context window, any piece of text can influence subsequent decisions.

This is the essence of prompt injection, but it is better understood as a broader class of semantic attacks. A malicious document does not need to exploit a memory vulnerability if it can alter the model's understanding of the task. A webpage does not need to execute code if it can persuade the system that a particular action is necessary or authorized.

In classical systems, we work hard to separate code from data. In agentic systems, that separation is blurred by design. The model must interpret meaning across inputs, and meaning in natural language often carries implicit instructions.

This makes context itself an attack surface.

## Retrieval as a security boundary

Retrieval-augmented generation is commonly framed as a technique for improving accuracy by grounding the model in external knowledge. In an agentic setting, however, retrieval becomes a critical security boundary.

When external or semi-trusted content is introduced into the model's working context, it gains the ability to influence decision-making. If all retrieved content is treated equally, then untrusted sources may acquire the same effective authority as system policies or user instructions.

A robust design therefore needs to treat different classes of input differently. User intent, system policy, structured internal state, and externally retrieved content should not be merged into a single undifferentiated prompt. Each should carry metadata about its origin, trust level, and permissible influence.

Without such separation, the model is left to infer authority relationships from patterns in text, which is not a reliable basis for security-critical decisions.

This is where safer designs begin to look less like chat wrappers and more like security middleware. A more defensible control loop usually has a very different shape:

```python
def run_agent(user_request: str, user_identity: Identity):
    plan_context = {
        "user_request": user_request,
        "trusted_policy": load_policy_bundle(user_identity),
        "structured_state": load_structured_state(user_identity),
        "retrieved_untrusted": retrieve_untrusted_context(user_request),
    }

    proposed_action = llm_plan(plan_context, tool_catalog=SAFE_TOOL_SCHEMAS)

    decision = policy_engine.evaluate(
        actor=user_identity,
        action=proposed_action,
        trust_context=plan_context,
    )

    if not decision.allowed:
        return deny(decision.reason)

    if decision.requires_approval:
        approved = request_human_approval(
            actor=user_identity,
            action=proposed_action,
            reason=decision.reason,
        )
        if not approved:
            return "Action cancelled."

    result = execute_tool_as_principal(
        principal=user_identity,
        tool=decision.tool_name,
        args=decision.filtered_args,
        scope=decision.scope,
    )

    write_audit_log(
        actor=user_identity,
        action=decision.tool_name,
        args=decision.filtered_args,
        provenance=plan_context,
        result_summary=summarize(result),
    )

    return result
```

The difference between these two designs is not stylistic. It is the difference between using an LLM as a helpful component inside a controlled system and using it as the system itself.

This is still only pseudo-code, but it reflects a radically different philosophy. The model proposes; it does not authorize. Retrieved content is not silently merged with policy. Tool access is scoped. Identity is explicit. Arguments can be filtered before execution. High-risk actions can be routed through human approval. The system is designed around the assumption that the model may be manipulated, confused, or simply wrong.

That is the mindset agentic systems require.

## Tool use and the materialization of errors

The introduction of tool use fundamentally changes the impact of model errors.

In a purely conversational system, a hallucination is often limited to incorrect text. In an agentic system, the same misinterpretation can result in a concrete action: an email sent to the wrong recipient, a file deleted, a database query executed, or sensitive data exported.

The design of tools therefore becomes central to system safety. Tools should be narrowly scoped, with well-defined schemas and constrained capabilities. They should enforce least privilege, require explicit confirmation for sensitive operations, and ideally operate in sandboxed environments. Importantly, the system should treat model outputs as proposals rather than authoritative commands.

A secure architecture places policy enforcement outside the model. The model may suggest an action, but a separate control layer should determine whether that action is permitted given the current context, identity, and risk profile.

## The reappearance of the confused deputy

The confused deputy problem provides a useful lens for understanding many of these risks. In that scenario, a system with legitimate authority is tricked into misusing that authority on behalf of an unauthorized party.

Agents are particularly susceptible to this pattern because they aggregate multiple sources of input and operate across multiple systems. They may receive instructions from users, colleagues, documents, and external content, and must continuously decide which signals are authoritative.

If an agent misattributes authority - for example, by treating a statement in a retrieved document as a valid instruction - it may perform actions that appear legitimate from a technical perspective but are semantically unauthorized.

The challenge is that this is not a traditional exploit. It is a failure of interpretation under conditions of ambiguity and adversarial input.

## Identity, privilege, and execution context

Another area where agentic systems introduce complexity is identity management.

Actions may be executed under different identities: the user's account, a service account, an API key, or an authenticated browser session. If these identities are not clearly separated and bound to specific scopes, the system may inadvertently escalate privileges.

For example, an agent might fulfill a request using a backend API with broader access than the user's own permissions, simply because that path is available. From the system's perspective, this is efficient. From a security perspective, it violates the principle of least privilege.

Each tool invocation should therefore be explicitly associated with an identity, a scope, and a justification. These bindings should be visible, auditable, and enforceable.

## Memory as a persistent risk surface

Memory is often presented as a feature that enhances usability by allowing systems to retain context across interactions. From a security standpoint, however, memory is also a form of persistent state that can accumulate sensitive information, stale assumptions, or adversarial inputs.

Different types of memory - short-term conversational context, task-level state, and long-term user profiles - have different risk profiles, but all require lifecycle management. Systems should define what can be stored, how it is validated, how long it is retained, and how it can be inspected or deleted.

Without such controls, memory can become both a source of leakage and a vector for long-lived manipulation.

## Natural language as a control interface

One of the more subtle challenges in agent design is the reliance on natural language as a control interface.

Natural language is inherently ambiguous, context-dependent, and open to interpretation. While this flexibility is what makes it attractive for user interaction, it is also what makes it difficult to use safely for high-authority operations.

In traditional systems, commands are expressed in structured formats with well-defined semantics. In agentic systems, similar levels of authority may be triggered by loosely phrased instructions whose exact meaning depends on context.

This places a significant burden on the system to correctly interpret intent and distinguish between instructions, suggestions, and irrelevant information. It also creates opportunities for adversarial inputs to exploit ambiguity.

## The need for external policy enforcement

Given these challenges, it is not sufficient to rely on the model itself to enforce all constraints.

A robust agent architecture should include external policy mechanisms that evaluate proposed actions before execution. These mechanisms can enforce rules related to access control, data sensitivity, action scope, and risk thresholds.

This separation ensures that even if the model is misled or makes an incorrect inference, the system as a whole can prevent unsafe actions from being carried out.

## A shift in the core question

The technologies we built around Tor and related systems addressed a fundamental question: how can users interact with digital systems without exposing themselves unnecessarily to observation and control?

That question remains important. But agentic systems introduce a second, equally important question: how can users retain control over systems that act on their behalf?

This is not merely an extension of the original problem. It is a shift in focus from visibility to authority, from communication to execution, and from protecting data to governing action.

If we fail to recognize this shift, we risk applying the wrong solutions to the wrong layer of the system.

We used to worry about who could see us.

We now need to worry about what can act for us, how it makes decisions, and how easily those decisions can be influenced.

That is a more complex problem, and one that will require more than incremental adjustments to existing security models.

It will require treating agentic systems not as enhanced interfaces, but as intermediaries with real power - systems whose design must be constrained, audited, and governed accordingly.
