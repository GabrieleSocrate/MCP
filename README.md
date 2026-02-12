In this md file you'll find the theory to understand MCP

# Model Context Protocol (MCP) - Core concepts

**MCP (Model Context Protocol)** is a standardized protocol for connecting AI applications to external tools and data sources

## Why MCP?
Without a standard, integrating:
- **M AI applications**
- with **N tools/data sources**
requires  **MxN custom integrations

<img width="897" height="373" alt="1a" src="https://github.com/user-attachments/assets/da92a087-3c06-4693-a577-8a8fd723a8f7" />

MCP reduces this to **M+N**:
- Each AI app implements MCP once (client side)
- Each tool implements MCP once (server side)
--> lower complexity, better scalability
  
<img width="1114" height="373" alt="2" src="https://github.com/user-attachments/assets/bf3dcbdf-8714-4b4b-9264-22cfc3809b29" />


## Architecture 
MCP follows a **client–server architecture** composed of three components:
1) **Host**: The **user-facing AI application**.
Examples:
- Chat applications (e.g., Claude, ChatGPT)
- AI-enhanced IDEs (e.g., Cursor)
- Custom AI agents built with frameworks like LangChain

The Host:
- Manages user interaction and permissions
- Uses the LLM to interpret user requests
- Decides which external capabilities are needed
- Orchestrates the overall workflow
- Presents results back to the user

2) **Client**: A component inside the Host responsible for communicating with **one specific MCP Server**.
- Maintains a 1:1 connection
- Handles protocol-level communication
- Discovers available capabilities
- Sends execution requests to the Server
Think of it as a structured communication bridge between the Host and a Server

3) **Server**: An external program or service exposing capabilities through MCP.
A Server:
- Wraps tools, data sources, or services
- Can run locally or remotely
- Exposes capabilities in a standardized format

## How MCP Works (Interaction Flow)
A typical MCP workflow looks like this:
1) **User Interaction** -> The user sends a request to the Host.
2) **LLM Processing** -> The Host interprets the request and determines if external capabilities are needed.
3) **Client Connection** -> The Host activates the appropriate Client.
4) **Capability Discovery** -> The Client queries the Server to see what it offers.
5) **Capability Invocation** -> The Host instructs the Client to invoke a Tool/Resource/Prompt.
6) **Server Execution** -> The Server executes the request.
7) **Result Integration** -> Results return to the Host and are incorporated into the final response.

A single Host can connect to **multiple Servers simultaneously**, enabling modular and composable AI systems.

## Core Capabilities 
An MCP Server can expose:
- **Tools**: Executable functions (actions, computations)
- **Resources**: Read only contextual data
- **Prompts**: Predefined templates/workflows
- **Sampling**: Server initiated LLM calls (eg., Code review after generation)

## Communication Protocol
MCP defines a standardized communication protocol that allows Clients and Servers to exchange messages in a predictable way.
At its core, MCP uses **JSON-RPC 2.0** as the message format.
In this way it standardizes how requests and responses are structured between Client and Server.

### Message Types
MCP defines three types of messages:
1) **Requests**
Sent from **Client -> Server** to invoke an operation.
A request includes:
- id (unique identifier)
- method (e.g., tools/call)
- params (arguments)
Example (Tool call):
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "weather",
    "arguments": {
      "location": "San Francisco"
    }
  }
}

2) **Responses**
Sent from **Server → Client** as a reply to a Request.
Contains:
- Same id
- Either a result (success) or error (failure)

3) **Notifications**
One-way messages (no response required).
Typically used for:
- Progress updates
- Status events

### Transport Mechanisms
JSON-RPC defines the **message structure**, but MCP also defines how messages are transported.
1) **stdio (Local communication)**
Used when Client and Server run on the **same machine**.
- Host launches Server as a subprocess
- Communication via stdin / stdout
Use cases:
- Local file system access
- Running local scripts
Advantages:
- Simple
- No network setup

2) **HTTP + SSE (Remote communication)**
Used when Client and Server run on **different machines**.
- Communication over HTTP
- Server can stream updates via Server-Sent Events (SSE)
Use cases:
- Remote APIs
- Cloud services
- Shared resources
Advantages:
- Works across networks
- Web-compatible
- Suitable for serverless environments

## Core Capabilities
An MCP Server exposes four distinct primitives.
Each has a different control boundary, purpose, and safety level.
1) **Tools**
Executable functions the **LLM can invoke**.
- **Controlled by**: Model (it decides if and when to use them)
- **Direction: Client** -> Server
- **Side effects**: Yes (potentially)
- **User approval**: Typically required
Used for:
- API calls
- Sending messages
- Creating tickets
- Performing calculations
Tools can modify the external world, which is why they require explicit control.
2) **Resources**
Read-only access to data sources.
- **Controlled by**: Host application
- **Direction**: Client → Server
- **Side effects**: No
- **User approval**: Usually not required
Used for:
- Reading files
- Retrieving database records
- Accessing configuration or documentation
Resources are lightweight and safe (similar to GET endpoints).
3) **Prompts**
Predefined templates or workflows that structure interactions.
- **Controlled by**: User
- **Side effects**: No
Used for:
- Guided workflows
- Specialized task templates
- Structured interactions
Prompts define how the LLM should behave before processing begins.
4) **Sampling**
Allows the **Server to request LLM execution** from the Host.
**Controlled by**: Server (with Host mediation)
**Direction**: Server → Client → Server
**Side effects**: Indirect
**User approval**: Typically required
Used for:
- Multi-step tasks
- Agent-like behavior
- Recursive refinement
**Sampling Flow**:
- Server requests LLM sampling
- Client reviews/modifies request
- LLM generates output
- Client reviews result
- Result returned to Server
This ensures a **human-in-the-loop safety model**.
### How Capabilities Work Together
In a typical interaction:
- User selects a **Prompt**
- Host retrieves context via **Resources**
- LLM invokes **Tools** if needed
- Server may trigger **Sampling** for complex reasoning
This separation creates:
- Clear control boundaries
- Modular composition
- Safe and scalable AI-tool interaction


# Real Example: MCP in Practice
## Scenario — AI Assistant with Tools
Imagine building an AI web app that:
- Reads uploaded PDFs
- Queries a company database
- Creates Jira tickets
- Sends emails
Instead of hardcoding all integrations, you use MCP servers.

## Architecture in Action
User
  ↓
Host (App + LLM)
  ↓
MCP Client
  ↓
MCP Server(s)
  ↓
External APIs / DB / Filesystem
You may have multiple servers:
- pdf_server
- database_server
- jira_server
- email_server
Each exposes its own tools.
## Example Interaction
User:
"Open the uploaded contract. If it contains a termination clause, create a Jira ticket."
Flow:
1) **Host receives request**
2) **LLM analyzes intent**
3) **LLM proposes a tool call**:
{
  "tool_call": "read_pdf",
  "arguments": { "file_id": "uploaded_123" }
}
4) Host checks permissions
5) Client sends JSON-RPC request to pdf_server
6) Server returns extracted content
7) LLM analyzes result
8) If needed, proposes:
{
  "tool_call": "create_jira_ticket",
  "arguments": { ... }
}
9) Client contacts jira_server
10) Final response returned to user
The LLM never directly calls APIs.
It only proposes structured actions.

## Critical Clarification: LLM ≠ Client
This distinction is fundamental.
**The LLM**:
- Receives text
- Produces text
- Can propose tool calls
- Does **not** open network connections
- Does **not** speak MCP
- Does **not** execute code
It only generates structured outputs.
**The Client**:
- Maintains connection to MCP Servers
- Sends JSON-RPC requests
- Receives responses
- Implements protocol logic
It is real networking code.

**The LLM is part of the Host.**
