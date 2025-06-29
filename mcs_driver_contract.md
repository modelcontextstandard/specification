# MCS Driver Contract – Version 0.1

This document defines the minimal contract all MCS-compatible drivers must implement.  
The syntax is language-agnostic and intended to guide implementations across all runtimes.

```pseudo
struct DriverMeta {
    id              // UUID to uniquely identify the driver (used in registries or for code validation via SHA)
    name?           // Optional name to avoid function name collisions in tool descriptions
    version         // Semantic version of the driver implementation, e.g. "0.4.1"
    protocol        // "REST", "GraphQL", "EDI", …
    transport       // "HTTP", "AS2", "CAN", "MQTT", …
    spec_format     // "OpenAPI", "JSON-Schema", "WSDL", "Custom", …
    target_llms[]   // Wildcard "*" for generic prompts, or explicit list like ["gpt-4o"]
    capabilities[]  // Optional extensions or capabilities exposed to orchestrators or clients
}

abstract class MCSDriver {
    meta: DriverMeta

    get_function_description(model_name?) -> string
    get_driver_system_message(model_name?) -> string
    process_llm_response(llm_response: string) -> any  // raw result
}
```
