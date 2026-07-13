# Semantic Kernel

**Semantic Kernel (SK)** is Microsoft's engine for enterprise-grade AI orchestration. It remains the primary bridge for organizations committed to the **Azure/Microsoft ecosystem** and **C#/.NET** architectures, though much of its forward momentum now ships inside the **Microsoft Agent Framework** (the consolidated successor to AutoGen + SK, RC 1.0 February 2026, GA Q2 2026).

## Table of Contents

- [Enterprise DNA](#dna)
- [Plugins and Planners](#plugins)
- [Memory and Connectors](#memory)
- [Multi-Language Support (C# vs. Python)](#multi-language)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Enterprise DNA

While LangChain is favored by startups, Semantic Kernel is favored by **Banks and Fortune 500s**.
- **Dependency Injection**: SK follows standard enterprise design patterns.
- **Strong Typing**: First-class support for C# types makes it highly reliable in large-scale mission-critical systems.
- **Security**: Deep integration with Azure Active Directory (Microsoft Entra ID) and Managed Identities.

---

## Plugins and Planners

1. **Kernel Functions**: The basic unit of logic (Native code or LLM prompts).
2. **Plugins**: A collection of functions (e.g., a "GitHub Plugin" or an "SQL Plugin").
3. **Planners**: SK's planners have evolved from simple ReAct to **Hierarchical Planners** that can coordinate long-running business processes across multiple days.

---

## Memory and Connectors

Semantic Kernel uses **Connectors** to abstract away the underlying infrastructure.
- **Universal Connectors**: One interface for OpenAI, Mistral, and local Onyx models.
- **Vector Store Abstraction**: Seamlessly switch between Azure AI Search, Pinecone, and Qdrant without changing the core business logic.

---

## Multi-Language Support

SK is one of the few major frameworks that treats C# and Python as equals.
- **The Pattern**: Develop and prototype in Python; deploy the core orchestration in C# for performance and type-safety.
- **Logic Sharing**: Shared prompt templates (.yaml) that work across both languages.

---

## Interview Questions

### Q: Why would a Staff Engineer choose Semantic Kernel over LangChain?

**Strong answer:**
**Architectural Alignment**. If an organization is already built on the .NET/Azure stack, Semantic Kernel fits into their existing CI/CD, monitoring (App Insights), and security (Entra ID) pipelines. LangChain often feels like an "external" piece of tech. Furthermore, SK's **Strong Typing** and **Dependency Injection** patterns prevent the "spaghetti code" that often plagues large LangChain projects. For an enterprise handling sensitive financial data, the **Native Azure integration** for security and auditing is the deciding factor.

### Q: What is the "Function Calling" abstraction in Semantic Kernel?

**Strong answer:**
SK uses a **Plugin-based model**. Every function (native C# or LLM-based) is registered with the Kernel. When the LLM decides it needs a tool, the Kernel looks up the function in the Plugin registry, validates the parameters, and executes it. SK now supports **Automatic Intent Detection**: the Kernel can proactively suggest which Plugin a user might need before they even ask, based on the current context window.

---

## References
- Microsoft Learn. "Semantic Kernel Documentation" (2025)
- Azure Architecture Center. "AI Design Patterns with Semantic Kernel" (2025)
- Build 2025. "The Future of Copilots with SK" (2025 Conference Recap)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Semantic Kernel (SK)** | Microsoft's open-source AI orchestration SDK that integrates LLMs into enterprise .NET and Python applications. | Provides the primary path for organizations on the Azure/Microsoft stack to add AI capabilities to existing systems. |
| **Microsoft Agent Framework** | The consolidated successor to both AutoGen and Semantic Kernel, targeting unified .NET and Python agent development. | Merges two previously separate Microsoft AI frameworks into one coherent platform with graph-based workflows. |
| **Kernel Function** | The basic unit of logic in Semantic Kernel, implemented as either a native code method or an LLM prompt template. | Gives the agent a callable unit of capability that the planner can discover and sequence. |
| **Plugin** | A named collection of related Kernel Functions grouped by domain, such as a GitHub Plugin or an SQL Plugin. | Organizes capabilities into discoverable, reusable modules the agent can select from based on the task at hand. |
| **Planner** | A Semantic Kernel component that automatically sequences Kernel Functions to accomplish a user's goal. | Frees developers from hand-coding the order of tool calls, letting the LLM derive the execution plan. |
| **Hierarchical Planner** | An advanced SK planner that can coordinate multi-step, long-running business processes across time and services. | Enables enterprise automation scenarios that span multiple days, systems, or approval stages. |
| **Connector** | A Semantic Kernel abstraction that provides a uniform interface to external services such as vector stores or LLM providers. | Makes it possible to swap the underlying provider (e.g., switch from Azure AI Search to Pinecone) without changing business logic. |
| **Vector Store Abstraction** | SK's unified interface for interacting with different vector databases using the same API. | Prevents vendor lock-in at the retrieval layer and simplifies migration between vector database providers. |
| **Dependency Injection (DI)** | A design pattern where a component's dependencies are supplied from outside rather than created internally. | Makes SK components independently testable and follows the standard enterprise software architecture patterns that .NET developers already know. |
| **Strong Typing** | A programming discipline where every variable and function parameter has a declared type enforced at compile time. | Catches entire classes of bugs at development time rather than at runtime, which is especially important in mission-critical financial and healthcare systems. |
| **Azure Active Directory (Microsoft Entra ID)** | Microsoft's cloud-based identity and access management service. | Provides enterprise-grade authentication and authorization so SK applications can control who can invoke which functions. |
| **Managed Identity** | An Azure feature that gives a service a cloud-managed credential without requiring the developer to handle secrets manually. | Eliminates the risk of hard-coded API keys or credentials in AI application code. |
| **ReAct** | A reasoning pattern where the agent alternates between generating a thought and executing an action (tool call). | Provides a principled loop for agents to decompose tasks and verify results before proceeding. |
| **Automatic Intent Detection** | A Semantic Kernel capability where the system proactively identifies which Plugin a user is likely to need based on context. | Reduces friction by surfacing relevant capabilities before the user has to explicitly ask for them. |
| **Function Calling** | A model capability where the LLM signals which external function to call and with what arguments, rather than producing free-form text. | Enables reliable, structured tool use without brittle text parsing. |
| **Universal Connector** | A SK connector that presents a single interface for multiple model providers (OpenAI, Mistral, local models). | Prevents provider lock-in at the model layer and simplifies switching between models for cost or capability reasons. |
| **App Insights (Application Insights)** | Microsoft Azure's application performance monitoring service. | Provides the observability layer for SK-based applications running in Azure environments. |
| **CI/CD** | Continuous Integration and Continuous Deployment — the automation pipeline for building, testing, and releasing software. | Ensures AI-enhanced applications are tested and deployed with the same rigor as traditional software. |

*Next: [AutoGen and CrewAI](07-autogen-crewai.md)*
