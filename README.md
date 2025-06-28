# Model Context Standard – *Specification*

**Status:** Draft v0.1 | **Audience:** Driver authors & SDK maintainers

This document is *language-agnostic*. Code blocks illustrate the contract in
pseudocode so it can be mapped 1-to-1 to Python, TypeScript, Go, ...

---

## 1 · MCS Motivation Overview

LLMs are token predictors. They consume and produce sequences of tokens that are rendered as text via a tokenizer. This means text is the only interface an LLM has to interact with its environment. It cannot execute anything directly.

To enable execution, you need a parser. The parser takes the LLM’s output, identifies structured instructions, and translates them into real-world actions.

> **Function Calling Recap**
> 
> Function calling enables LLMs to interact with external systems by generating structured outputs (typically JSON) that describe function invocations.
>
> The process involves:
> 1. Providing the LLM with function schemas/descriptions 
> 2. The LLM generates a structured call in response to user queries
> 3. A parser extracts and validates the function call
> 4. The system executes the call against the target API/service
> 5. Results are returned to continue the conversation
>
> This pattern allows LLMs to act as intelligent orchestrators using text alone. The exact format of the function description doesn't matter. What matters is that the LLM understands when and how to call a tool and how to format the output for the parser.

This concept was formalized in [TALM: Tool Augmented Language Models](http://arxiv.org/abs/2205.12255), which showed how LLMs can be extended with non-differentiable tools to solve real-world tasks.

Modern AI frameworks often provide parsers, but not standardized descriptions or callable functions. This leads to a fragmented landscape of custom tooling where developers repeatedly reinvent the same logic for each application.

MCP addressed this by introducing the first open standard to connect LLMs with external systems. OpenAI followed a similar idea with "Actions" for Custom GPTs long before that, but never published it as a general concept.

However, MCP added a full protocol stack, along with new complexity and security implications. Worse, much of the effort that followed focused on building wrappers around APIs that could already be used directly by LLMs, as demonstrated in the 2-minute MCS proof of concept.

Despite this, MCP succeeded, not because of elegance, but because it was the first real standard in this space. Ans a standard was needed.

Useful features like autostarting MCP servers were not design decisions, but emerged from practical needs when using the STDIO transport layer. Making a core benefit for some developers are side effect not a core of MCP itself.

MCS now distills this all down to the essentials, what is actually required to connect an LLM to external systems in a standardized and reusable way.

At the core, it's a driver problem.

A MCS driver must expose a function specification that LLMs can consume. Most modern LLMs can use these out of the box. But to ensure precision, the driver should also provide usage instructions and formatting hints so the output can be correctly parsed.

The parser is the other half of the equation. It bridges the LLM’s output to real-world execution by scanning for and dispatching structured calls.

Previously, function implementations were written from scratch for every use case. But with MCS generalization is key. If a REST call works for one service, it can be reused for all REST-over-HTTP services.

An MCS Driver does exactly that: it generalizes function calling for a given protocol over a specific transport layer.


## 2 · Core Idea

**MCS** defines a *thin glue layer* between a Language Model and any existing interface such as REST, GraphQL, CAN-Bus, EDI, or even filesystems.

Every MCS driver must implement **two mandatory capabilities**:

1. **Expose** a machine-readable *function description* via `get_function_description()`
   This enables the LLM to discover available tools and understand how to call them.
2. **Execute** structured LLM-emitted calls via `process_llm_response()`
   The driver parses the request, routes it through a bridge (HTTP, serial, message bus, etc.), and returns the raw result.

The complexity of a **MCS Driver** is mostly concentrated in the execution phase. Everything related to authentication, rate-limiting, retries, logging, or protocol-specific quirks is handled internally by the driver using existing transports and if available machine readable specs like OpenAPI.

Drivers are initialized with configuration parameters through the constructor. This makes it easy to inject dependencies or load configuration dynamically at runtime.

Optional or advanced functionality can be added modularly via *capabilities*, allowing drivers to remain lightweight by default.

The **client** acts as a coordinator. It retrieves the function specification from the driver, injects it into the LLM system message, and later passes the LLM’s output back to the driver for inspection, if an execution of a function is wanted by the LLM. 

Importantly, the client does not need to know how the driver works internally or which technology stack it uses.

---

### Phase A – Spec exposure

```
 Client ─── request spec ───▶  Driver                
  ▲                              │
  └─── Spec (OpenAPI …) ◄────────┘
```

The client first calls `get_function_description()` to retrieve a machine-readable function specification. How this spec is generated or retrieved—whether from a local file, HTTP endpoint, or generated dynamically—is left to the driver implementation.

The client may embed the spec into the LLM's system prompt or use it in other prompt injection strategies.

To simplify this process, drivers may also implement `get_driver_system_message()` which returns a complete, ready-to-use system prompt. This includes both the tool description and formatting guidance tailored for a LLM or for a specific LLM as well.

This is crucial because different LLMs may respond better to differently phrased prompts.


### Phase B – Call execution

```
LLM ──► JSON call ──► Driver/Parser ──► External API
 ▲                           │
 └─────────── Result ◄───────┘
```

Once the LLM emits a structured function call (typically as a JSON object in the text output), the client passes this to the driver’s `process_llm_response()` method.

The driver parses the call, dispatches it over its bridge (e.g. HTTP, CAN-Bus, AS2), and returns the raw result. The result can then be forwarded back into the conversation, either directly or via formatting logic handled elsewhere.



---

## 3 · Minimal Driver Contract

```pseudo
struct DriverMeta {
    id              // UUID to uniquely identify the driver (used in registries or for code validation via SHA)
    prefix          // Optional prefix to avoid function name collisions in tool descriptions
    protocol        // "REST", "GraphQL", "EDI", …
    transport       // "HTTP", "AS2", "CAN", "MQTT", …
    spec_format     // "OpenAPI", "JSON-Schema", "WSDL", "Custom", …
    target_llms[]   // Wildcard "*" for generic prompts, or explicit list like ["gpt-4o"] or both ["*", "t5-small"]
    capabilities[]  // Optional extensions or capabilities exposed to orchestrators or clients
}

abstract class MCSDriver {
    meta: DriverMeta

    get_function_description(model_name?) -> string
    get_driver_system_message(model_name?) -> string
    process_llm_response(llm_response: string) -> any  // raw result
}

```

*The exact signatures are up to each SDK. The semantics **must** match.*

### 3.1 `get_function_description(model_name?)`

Returns a static artifact that describes the available functions in a LLM-readable format.
The approach follows a standard-first principle. If an established specification format exists, it should be used.

Standard formats like:
* OpenAPI (JSON/YAML) – for RESTful APIs
* JSON Schema – for structured input/output validation, CLI tools, or message formats
* GraphQL SDL – for GraphQL-based APIs
* WSDL – for SOAP and legacy enterprise services
* gRPC / Protocol Buffers (proto) – for high-performance binary APIs
* OpenRPC – for JSON-RPC APIs
* EDIFACT/X12 schemas – for EDI-based B2B interfaces

If no standard is available, a custom function description has to be written.

Drivers may implement/use a dynamic description provider to tailor the spec based on the LLM’s capabilities. 
For example, instead of exposing a raw OpenAPI schema, the driver may generate a simplified and LLM-friendly 
representation that retains full fidelity but improves comprehension.

### 3.2 `get_driver_system_message(model_name?)`

Returns a complete system message containing or referencing the function description, 
crafted specifically for an LLM family.

This message guides the model to make valid and parseable tool calls. While the 
default behavior may simply inline get_function_description(), advanced drivers can 
define custom prompts tailored to different LLMs (e.g. OpenAI, Claude, Mistral), including:

* Format hints
* JSON schema constraints
* Few-shot examples
* Token budget control


### 3.3 `process_llm_response(llm_response)`

*Consumes* the output message generated by the LLM and executes the described operation. The method should:

1) Validate and parse the input (typically JSON or structured text)
2) Map the request to a bridge-compatible operation (e.g. HTTP call, MQTT message)
3) Return the raw result without postprocessing (the orchestrator or client will handle formatting or retries)

This separation ensures that drivers focus on interfacing with the external system, while clients and orchestrators remain agnostic of internal logic and implementation.

---

## 4 · Configuration & Instantiation

Drivers MAY expose constructor parameters (or getters/setters) for:

| Purpose            | Examples                                               |
| ------------------ | ------------------------------------------------------ |
| Bridge endpoint(s) | list of URLs, broker topic, serial port …              |
| Auth / security    | API key, OAuth token, TLS verification flag …          |
| Prompt overrides   | *custom_system_message*, *custom_tool_description* |
| Runtime hints      | timeout, retry policy, proxy settings …                |

**Rule of thumb:** *Never* add a mandatory parameter that is *transport
agnostic*. Keep the universal interface small, push specifics into
driver-level config.

---

## 5 · Optional Capabilities

MCS keeps the base contract tiny.  Optional behaviour is signalled via
**capability flags** in `DriverMeta`:

| Capability       | Flag            | Suggested Mix-in / interface |
| ---------------- | --------------- | ---------------------------- |
| Health check     | `healthcheck`   | `def healthcheck() -> dict`  |
| Resource preload | `cache`         | `def warmup() -> None`       |
| Status & metrics | `status`        | `def get_status() -> dict`   |
| Streaming        | `stream`        | `def stream_call(…)`         |

Consumers must *feature-detect* before they invoke an optional method.

---

## 6 · Autostart Convention

Autostart is explicitly out of scope for the core driver contract. However, a companion 
component such as an AutoStarter may handle the execution layer by:

1) Inspecting the driver’s metadata
2) Resolving a container image or executable reference
3) Launching and sandboxing the driver (e.g. via Docker, Podman, Firecracker)
4) Returning the resulting bridge endpoint(s) to the caller

This approach keeps drivers stateless and portable, while still enabling a plug-and-play 
experience similar to MCP’s implicit STDIO servers.

If a driver provides a standard Docker configuration, the AutoStarter can abstract away the full runtime setup. 
In this case, the user only needs to reference the image name or alias. Removing the need to define a complete 
execution environment or custom entrypoint.

This dramatically lowers the barrier to usage and encourages rapid prototyping and deployment.

---

## 7 · LLM Prompt Patterns (Informative)

While outside the formal spec, most drivers follow one of two prompt styles:

1. **Inline-spec** – Embed the *entire* function description as markdown code
   block.  Simple, but token-heavy on big specs.
2. **Schema-only** – Provide a condensed *argument schema* and keep the
   detailed spec in the background (LLM fetches on demand).  Saves tokens,
   requires retrieval mechanism.

Driver authors should document which approach they use.

Every time someone writes a client (for example, for MCP), they need to come up with a prompt for tool use. MCS offers a way to collect all best practices and give the client the ability to choose the right prompt for the right model.

This makes the effort to find a really good prompt for a use case a one-time deal.

The standard way to check and test drivers by comparing input and output makes it a good fit for prompt optimizers like DSPy.

For every use case or model, train the perfect prompt once and it can be used for many other scenarios where this driver comes into play.


---

## 8 · Relation to MCP

MCS does **not** compete with MCP – it generalises the same idea without
imposing a new wire protocol.  An *MCP-over-STDIO* or *MCP-over-SSE* driver is
perfectly valid: the driver treats the MCP server as its **bridge** and
exposes MCP's tool list as its **spec**.

---

## 9 · Versioning & Compatibility

* **Semantic Versioning** (`MAJOR.MINOR.PATCH`) for the spec doc itself.
* Drivers must declare which spec version they implement.
* Deprecations require +1 MINOR with a clear migration note.

---

## 10 · Next Steps

* Finalise JSON-Schema for `DriverMeta` and capability flags.
* Provide language-specific reference interfaces in
  `python-sdk` / `typescript-sdk`.
* Draft an *autostart recommendation* (container labels, health endpoints …).
* Collect community feedback.
* Decide if sync / async drivers are needed

---

*Feel free to open a GitHub Discussion for clarifications or proposals.*

