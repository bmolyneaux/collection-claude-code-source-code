# Glaude: Architecture Document (Advanced Agent Design)

This document outlines the internal implementation details of the Glaude Godot addon. The architecture avoids over-engineered "Enterprise Java" abstractions in favor of idiomatic GDScript, but is explicitly designed to support the advanced autonomous loop, reflection, and context management required by a "Claude Code"-style coding agent.

## 1. Philosophy & Design Principles
- **Autonomous Agent Loop:** The core is an event-driven reflection loop. The AI evaluates its own tool outputs and iterates autonomously until a task is complete.
- **Idiomatic GDScript:** Simple, flat class structures. We rely on Godot's native APIs (`FileAccess`, `OS`, `HTTPRequest`, Signals).
- **Security First:** Destructive actions (writing files, executing shell scripts) pause the autonomous loop and demand user approval.
- **Data-Driven State:** The core truth of the application is a dictionary array representing the conversation history, augmented by runtime context (Git state, Godot editor state).

## 2. Core Modules
- **`glaude_plugin.gd`**: The entry point. Registers the UI dock and bootstraps the environment.
- **`agent_orchestrator.gd`**: The brain of the operation. Manages the autonomous loop, deciding when to hit the API, when to run tools, and when to pause for user input.
- **`gemini_client.gd`**: The HTTP wrapper. Serializes requests and parses JSON tool-calls.
- **`context_manager.gd`**: Manages the dynamic system prompt, token estimation, and context compression (summarization).
- **`tool_executor.gd`**: Maps requested tools to GDScript logic, handles execution errors gracefully, and returns robust states to the LLM.
- **`permission_hook.gd`**: Intercepts tool calls and pauses execution to request user confirmation for sensitive actions.

## 3. The Autonomous Agent Loop (Orchestration)
Unlike a simple chatbot, Glaude operates in a "ReAct" (Reasoning and Acting) loop.

1. **Task Initialization:** The user inputs a prompt (e.g., "Fix the jumping bug in the player script").
2. **Context Assembly:** `context_manager.gd` dynamically builds the system prompt, injecting the current Git branch, Godot engine version, and the active scene tree.
3. **API Request:** `gemini_client` sends the payload.
4. **Tool Call Parsing:** The API responds. If the model chooses to use a tool (e.g., `GrepTool` to find "jump"):
   - The loop pauses.
   - `tool_executor` runs the grep command.
   - The output is appended to the history.
   - **Crucially:** The loop immediately triggers step 3 again without user intervention. The AI "reflects" on the tool output and takes its next step.
5. **Termination:** The loop breaks only when the AI returns standard text without a tool call (or uses a specific `TaskCompleteTool`), signaling it has finished the task.

## 4. Context & Token Management
A simple sliding window is insufficient for a coding agent, as it forgets initial instructions.

- **Token Estimation:** GDScript will approximate tokens (e.g., string length / 4) to monitor the context window.
- **Dynamic Context Compression:** When nearing the token limit, `context_manager.gd` doesn't just slice the array. It drops the oldest tool outputs first, preserving the user's original prompt, the AI's high-level reasoning steps, and the system prompt.
- **System Prompt Assembly:** Before every API request, the system prompt is regenerated to reflect the immediate physical state of the workspace (e.g., "Current Dir: res://scripts", "Uncommitted Git files: player.gd").

## 5. Tool Execution & Error Handling
Robust tool execution is vital. If a tool fails, the agent must know *why* so it can self-correct.

- **Execution Interception:** When the API calls `BashTool`, the `tool_executor` runs `OS.execute()`.
- **Error Injection:** If the bash command returns a non-zero exit code, `tool_executor` catches this and returns a formatted error string directly back to the AI (e.g., `ERROR: command failed with code 1: directory not found`). The AI sees this and can retry with a different path.
- **Permissions:** If the AI requests `FileWriteTool`, `permission_hook.gd` emits a signal to the UI. The UI displays an "Approve/Reject" modal. The agent loop yields (using `await`) until the user resolves the modal.

## 6. UI Node Hierarchy
The UI uses Godot's native Control nodes.

```text
VBoxContainer (Main Dock)
 ├── RichTextLabel (Chat & Reasoning Display)
 │    - bbcode_enabled = true
 ├── PanelContainer (Permission Modal - Hidden by default)
 │    ├── Label (Tool details: "Agent wants to overwrite player.gd")
 │    └── HBoxContainer (Approve/Reject buttons)
 ├── HBoxContainer (Input Area)
 │    ├── TextEdit (User Input)
 │    └── Button (Submit)
```

## 7. API Payload Structures
Because the architecture relies heavily on raw dictionaries for state, these are the structures the networking layer will parse and emit:

**Sending a Request (with Context and Tools):**
```json
{
  "system_instruction": { "parts": { "text": "You are Glaude. Current branch: main. Uncommitted files: none." } },
  "contents": [
    { "role": "user", "parts": [{ "text": "Find the player script and read it." }] }
  ],
  "tools": [{
    "functionDeclarations": [
      {
        "name": "FileReadTool",
        "description": "Reads a file from the project directory.",
        "parameters": {
          "type": "OBJECT",
          "properties": { "path": { "type": "STRING" } }
        }
      }
    ]
  }]
}
```

**Receiving a Tool Call (The AI decides to act):**
```json
{
  "candidates": [{
    "content": {
      "role": "model",
      "parts": [{
        "functionCall": {
          "name": "FileReadTool",
          "args": { "path": "res://player.gd" }
        }
      }]
    }
  }]
}
```

**Sending the Tool Result Back (Continuing the Loop):**
```json
{
  "contents": [
    { "role": "user", "parts": [{ "text": "Find the player script and read it." }] },
    { "role": "model", "parts": [{ "functionCall": { "name": "FileReadTool", "args": {"path": "res://player.gd"} } }] },
    {
      "role": "user",
      "parts": [{
        "functionResponse": {
          "name": "FileReadTool",
          "response": { "output": "extends CharacterBody3D\n\nfunc _physics_process(delta):\n..." }
        }
      }]
    }
  ]
}
```
*Note: Gemini requires the `functionResponse` to be sent under the `user` role. After this is sent, the loop begins again automatically.*