---
title: "Extending SDF and CoAP for AI Agent Interaction with IoT Devices"
abbrev: "SDF CoAP Agents"
docname: draft-corneo-t2trg-sdf-coap-agents-latest
category: info
ipr: trust200902
submissiontype: IRTF
date: 2026

workgroup: Thing-to-Thing Research Group (T2TRG)

submissiontype: IRTF
number:
date:
v: 3

author:
  - name: Lorenzo Corneo
    org: Ericsson
    country: Finland
    email: lorenzo.corneo@ericsson.com
  - name: Jaime Jiménez
    org: Ericsson
    country: Finland
    email: jaime.jimenez@ericsson.com
  - name: Ari Keränen
    org: Ericsson
    country: Finland
    email: ari.keranen@ericsson.com

informative:
  RFC7252:
  RFC8613:
  RFC6690:
  RFC9176:
  RFC9421:
  RFC9880:
  I-D.meunier-web-bot-auth-architecture:
  I-D.ietf-wimse-workload-creds:
  I-D.ietf-wimse-wpt:

--- abstract

This document proposes extensions to the Semantic Definition Format
(SDF) and the Constrained Application Protocol (CoAP) to better support
AI agent interaction with Internet of Things (IoT) devices. It covers
two aspects: identifying AI agents in CoAP transactions, and making SDF
device descriptions readily consumable by large language models (LLMs)
and autonomous agents. The document is intended to stimulate discussion
within T2TRG on these topics.

--- middle

# Introduction

AI agents — software systems that combine large language models with
tool use and autonomous decision-making — are increasingly used to
interact with networked services on behalf of users. In the IoT domain,
devices expose their capabilities through machine-readable descriptions
such as SDF {{RFC9880}}. Agents that want to discover,
understand, and actuate IoT devices need to consume these descriptions
and communicate with the devices, often over CoAP {{RFC7252}}.

Two gaps exist in the current protocol landscape:

1. CoAP has no mechanism for a client to declare that it is an AI agent,
   nor to carry an authenticated agent identity. Servers cannot
   distinguish an autonomous agent from a human-operated application,
   limiting their ability to apply agent-specific policies.

2. SDF documents are JSON-based and machine-readable, but they lack
   explicit guidance for LLM consumption. Missing or terse descriptions
   can cause agents to misinterpret device capabilities, leading to
   incorrect actions — a problem that ranges from wasted API calls to
   safety hazards when physical actuators are involved.

This document outlines approaches to address both gaps. It is
informational and intended to frame discussion; it does not define
protocol extensions or registrations.

# Terminology

{::boilerplate bcp14-tagged}

SDF:
: Semantic Definition Format, a JSON-based language for describing IoT
  device capabilities {{RFC9880}}.

AI Agent:
: A software system that uses a large language model in a loop with
  external tools to achieve a goal autonomously.

LLM:
: Large Language Model; a neural network trained on text that generates
  token sequences given an input context.

Resource Directory (RD):
: A CoAP-based directory where endpoints register their resources
  {{RFC9176}}.

# AI Agent Identification for CoAP

On the web, work is underway to authenticate automated clients through
HTTP Message Signatures {{RFC9421}} and related mechanisms such as
{{I-D.meunier-web-bot-auth-architecture}}. These approaches let servers
verify what software is making a request and apply appropriate policies.

In parallel, the IETF WIMSE (Workload Identity in Multi-System
Environments) working group is defining identity credentials for
workloads, including autonomous agents. The Workload Identity Token
(WIT) {{I-D.ietf-wimse-workload-creds}} is a signed token that
represents a workload's identity, and the Workload Proof Token (WPT)
{{I-D.ietf-wimse-wpt}} lets a workload prove possession of the key bound
to its WIT on a per-request basis. Together these represent the current
state of the art for authenticated agent identity, but they are defined
for HTTP and JSON-based environments.

CoAP has no equivalent. The protocol authenticates devices via DTLS or
OSCORE {{RFC8613}}, but these mechanisms establish device identity, not
the nature of the client software. A CoAP server receiving a GET request
cannot tell whether the client is a firmware update agent, a user's
mobile app, or an autonomous AI agent exploring device capabilities.

## Approach

The approach proposed in this draft consists of a workflow in three steps.

### Step 1: Agent-ID CoAP Option

This approach leverages an elective CoAP option that carries agent
identification information. Being elective, servers that do not
understand it simply ignore it, preserving backward compatibility.

The option would carry:

- An agent identifier string (analogous to HTTP User-Agent but
  structured for constrained environments).
- A cryptographic signature over relevant message fields, allowing the
  server to verify the agent's claimed identity.
- A reference to where the agent's public key can be retrieved.

The signature and key material would use CBOR encoding to remain compact
for constrained links. An agent reading a sensor might then send:

~~~ coap
GET /sensors/temperature
Agent-ID: h'a4012667...' (COSE_Sign1, keyid "sig1")
Accept: 60 (application/cbor)

2.05 Content
Content-Format: 60 (application/cbor)

22.5
~~~

The `Agent-ID` value is a CBOR-encoded COSE_Sign1 structure; it is shown
here as a hexadecimal item with an inline comment. A server that does
not understand the elective option ignores it and answers as usual.

### Step 2: Key Discovery via Resource Directory

Rather than defining a new key distribution mechanism, agent public keys
can be registered in a CoAP Resource Directory {{RFC9176}}. The RD
already serves as a discovery mechanism for IoT endpoints; extending it
to host agent key material (or links to externally hosted keys) reuses
existing infrastructure.

A server receiving an Agent-ID option with an unknown key identifier
would query the RD once to retrieve the key, then cache it for
subsequent interactions with that agent.

### Step 3: Agent-Aware Server Behavior

Once a server has verified an agent's identity, it can adjust its
responses. Examples include:

- Returning richer descriptions alongside sensor data.
- Indicating content usage restrictions (e.g., data not to be used for
  model training).
- Applying rate limiting or access restrictions specific to automated
  clients.

These policy decisions remain simple — appropriate for constrained
devices with limited processing budgets.

# SDF Content Negotiation for AI Agents

SDF documents are designed for toolchains and developers. An AI agent
requesting an SDF model receives a JSON structure that it can parse, but
may not correctly interpret — particularly when optional description
fields are absent or terse. The agent has no way to signal that it would
prefer a representation optimized for language model consumption.

## Approach: Signaling Agent Intent

A content type such as `text/sdf` could indicate that the client wants a
natural-language rendering of the SDF model rather than the raw JSON.
In HTTP, this maps to the Accept header:

    GET /models/thermometer
    Accept: text/sdf

In CoAP, the same intent could be expressed through a Content-Format
option. SDF already has a registered CoAP Content-Format (434) for
`application/sdf+json` {{RFC9880}}, so a representation for agents would
be a new, second format. The intent could alternatively be expressed
through resource type filtering using the CoRE Link Format {{RFC6690}}:

    GET /sdf?rt=ai

In MQTT v5, the Content Type property in a SUBSCRIBE or PUBLISH packet
can carry the same media type string (e.g., `text/sdf`) to signal the
desired representation. An MQTT broker or bridge that also serves SDF
models can use this field to select the appropriate representation
before forwarding the payload to the subscriber.

A server (or intermediary) receiving such a request would return a
text representation assembled from the SDF model's descriptive content,
rather than the raw JSON document.

Requesting the registered format returns the JSON model:

~~~ coap
GET /sdf/thermometer
Accept: 434 (application/sdf+json)

2.05 Content
Content-Format: 434 (application/sdf+json)

{
  "sdfObject": {
    "thermometer": {
      "sdfProperty": {
        "temperature": { "type": "number", "unit": "Cel" }
      }
    }
  }
}
~~~

Requesting the agent-oriented format returns the natural-language
rendering. Its content-format has no assigned code point, shown here as
`TBD`:

~~~ coap
GET /sdf/thermometer
Accept: TBD (text/sdf)

2.05 Content
Content-Format: TBD (text/sdf)

Temperature sensor. Property @T is the reading in Celsius.
GET /temperature to read it.
~~~

# LLM-Friendly SDF Descriptions

Even when an AI agent successfully retrieves and parses an SDF document,
it may misinterpret the semantics. Tests show that agents make errors
when critical optional fields (like `description`) are missing, or when
the relationship between properties, actions, and events is not obvious
from the JSON structure alone.

## Approach 1: Developer-Defined Templates

A new SDF quality (working name: `sdfBot`) would allow model developers
to embed natural-language instructions directly in the SDF document.
These instructions tell agents how to interpret and use the described
capabilities.

The quality can appear at the document root (providing an overview) and
within individual affordances (providing usage instructions for specific
properties, actions, or events). A short reference mechanism allows the
root template to incorporate per-affordance descriptions by reference.

Example SDF fragment:

~~~ json
{
  "sdfBot": "Temperature sensor. @T is the reading in Celsius.",
  "sdfObject": {
    "thermometer": {
      "sdfProperty": {
        "temperature": {
          "description": "Measured temperature",
          "type": "number",
          "unit": "Cel",
          "botRef": "T",
          "sdfBot": "Read-only. GET /temperature (Celsius)."
        }
      }
    }
  }
}
~~~

When an agent requests the `text/sdf` representation, the server
assembles the `sdfBot` strings into a coherent natural-language
description. When a non-agent client requests `application/sdf+json`,
it receives the standard JSON model (with or without the `sdfBot`
fields, depending on implementation).

SDF already has a `description` field for human-readable text. The
`sdfBot` quality differs in intent: it targets automated consumers and
can include explicit usage instructions (endpoints to call, expected
value ranges, sequencing constraints) that would be unusual in a
human-facing description.

## Approach 2: Structured Examples for Language Models

LLMs can be guided to perform tasks more accurately when the prompt
includes concrete examples of similar interactions. By showing the model
what a successful request looks like, it can generalize to new tasks in
the same context.

This approach proposes a new SDF quality `sdfStructuredExamples` for including
such usage examples in the LLM-friendly representation. For simple cases, the
value of the quality is an array of JSON objects, each with three members:

- "task": describing the example task
- "action": the resulting action the agent would take to achieve the
  goals of the task
- "parameters": giving associated parameter values needed for the action

The "parameters" member can be omitted if the invocation of the
affordance does not need any parameters or they are obvious from the
context. Similarly, "action" can be omitted if the required interaction
is obvious from the context (e.g., reading the value of the property the
example describes).

For SDF documents, such examples can be on object level to describe the
use of the capabilities of the object in general, or at the affordance
level to show how the affordance can be used to achieve a goal.

Example of a structured example for a simple task:

~~~ json
{
  "sdfStructuredExamples": [
    {
      "task": "Get temperature notifications every 3s for 60s.",
      "action": "Execute sdfAction/startMonitor",
      "parameters": {
        "interval": 3,
        "duration": 60
      }
    }
  ]
}
~~~

For more complex interactions, a set of steps with actions and optional
parameters can be given. A step can also include a "readyWhen" member describing
the end state the agent should look for before considering the step (or the
overall task) complete. For example:

~~~ json
{
  "sdfStructuredExamples": [
    {
      "task": "Update firmware of the device to version 2",
      "steps": [
        {
          "action": "Set sdfProperty/firmware to v2 image",
          "parameters": "<new firmware file>"
        },
        {
          "action": "Execute sdfAction/startFirmwareUpdate"
        },
        {
          "action": "Observe sdfProperty/firmwareVersion",
          "readyWhen": "sdfProperty/firmwareVersion has version 2"
        }
      ]
    }
  ]
}
~~~

# Security Considerations

Agent identification introduces the question of trust: a server must
decide which agent identities to accept and what policies to associate
with them. The cryptographic signature in the Agent-ID option prevents
impersonation but does not establish authorization. Servers need a
policy framework to map verified agent identities to permitted actions.

The `sdfBot` and `sdfStructuredExamples` qualities expose instructions
that agents will follow. Malicious or poorly written content could cause
agents to perform unintended actions, and structured examples are
especially risky because they name concrete actions and parameters, so an
agent may execute them with little further reasoning. Developers should
treat both qualities as security-relevant, similar to API documentation
that automated systems act on without human review. A server or
intermediary that assembles the `text/sdf` representation should only
incorporate content from trusted SDF sources.

Content usage restrictions signaled via agent-aware responses are
advisory. Enforcement depends on the agent honoring them; a malicious
agent can ignore any declared restrictions. Servers holding sensitive
data should not rely solely on declared agent identity for access
control.

# IANA Considerations

This document requests no IANA actions. If the mechanisms discussed
here were pursued, they would require registrations for a CoAP option
number for Agent-ID, a media type and CoAP Content-Format for the
agent-oriented SDF representation, and a resource type value.

--- back
