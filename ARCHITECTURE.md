# Glaude: Architecture Document

This document outlines the internal implementation details of the Glaude Godot addon. The architecture avoids over-engineered, "Enterprise Java"-style abstractions in favor of simple, pragmatic, and idiomatic GDScript.

## 1. Philosophy & Design Principles
- **Idiomatic GDScript:** Simple, flat class structures. We rely on Godot's built-in nodes, signals, and native APIs (`FileAccess`, `OS`, `HTTPRequest`).
- **Single Responsibility:** Scripts handle one specific domain (UI, networking, tool execution) without requiring convoluted interfaces or deep inheritance trees.
- **Data-Driven State:** The core truth of the application is a simple array of dictionaries representing the conversation history.
- **Synchronous-like Flow:** While `HTTPRequest` is asynchronous, the user experience will feel synchronous. The UI locks, the request fires, tools execute, and the UI unlocks when the final text response arrives.

## 2. Core Modules
The implementation is split into a few focused files:

- **`plugin.gd`**: The entry point extending `EditorPlugin`. It registers the UI dock into the Godot Editor (bottom or side panel) and loads/unloads the components when the plugin is toggled.
- **`glaude_dock.gd` / `glaude_dock.tscn`**: The user interface. It handles user input, displays the chat history using BBCode, and manages the visual loading states.
- **`gemini_client.gd`**: A lightweight wrapper around `HTTPRequest`. It strictly handles authenticating, serializing the JSON request, and awaiting the HTTP response.
- **`chat_state.gd`**: Manages the current conversation array, context limits, and local disk persistence.
- **`tool_executor.gd`**: A pure logic script that maps requested tool names from the API directly to GDScript functions.

## 3. UI Node Hierarchy
The UI uses Godot's native Control nodes. No custom drawing or heavy external rendering libraries.

```text
VBoxContainer (Main Dock)
 ├── RichTextLabel (Chat Display)
 │    - bbcode_enabled = true
 │    - scroll_following = true (auto-scrolls to new messages)
 ├── HBoxContainer (Input Area)
 │    ├── TextEdit (User Input)
 │    │    - wrap_mode = line wrapping
 │    │    - custom logic to submit on Enter, newline on Shift+Enter
 │    └── Button (Submit)
```

## 4. State & Context Management

### 4.1 In-Memory State
The state is an array of `Dictionary` objects that mirrors the exact structure required by the Gemini API (`role` and `parts`). This avoids needing an intermediate "Message" class to translate back and forth.

### 4.2 Persistence
Chat history is saved to `.godot/glaude_history.json` (so it persists across editor restarts but is ignored by version control). It is loaded when the `plugin.gd` initializes.

### 4.3 Context Window Limits
To prevent exceeding Gemini's token limits over long sessions:
- The system prompt is always preserved.
- If the `chat_state` array exceeds a predefined threshold (e.g., 50 messages or a rough character count approximation), the oldest messages are sliced off before the payload is sent to `gemini_client.gd`.
- `history = history.slice(history.size() - MAX_MESSAGES)` ensures a fast, simple sliding window context.

## 5. Tool Execution Flow
The tool execution skips complex interface classes. It relies on a simple dispatcher pattern using Godot `Callable`s.

1. **User Input:** The user types a request. UI adds it to `chat_state` and calls `gemini_client.send()`.
2. **API Response:** The `gemini_client` parses the JSON. If it detects a `functionCall`, it pauses the text output.
3. **Dispatch:** It passes the tool name and arguments to `tool_executor.gd`.
4. **Execution:** `tool_executor.gd` uses a simple `match` statement or dictionary map to route the call:
   - `"BashTool"` -> `OS.execute("bash", ["-c", args.command], output)`
   - `"FileReadTool"` -> `FileAccess.get_file_as_string(args.path)`
5. **Callback:** The output string is returned to `gemini_client`.
6. **Re-prompt:** `gemini_client` packages the tool output into a `functionResponse` dictionary, appends it to `chat_state`, and immediately sends a *new* HTTP request back to Gemini.
7. **Completion:** When Gemini finally returns standard text (no function calls), the UI unlocks and displays the answer.

## 6. Gemini API JSON Structures
Because the architecture relies heavily on raw dictionaries for state, these are the structures the networking layer will parse and emit:

**Sending a Request (with Tool Definitions):**
```json
{
  "system_instruction": { "parts": { "text": "You are Glaude, an AI assistant for Godot..." } },
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

**Receiving a Tool Call:**
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

**Sending the Tool Result Back:**
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
*Note: Gemini requires the `functionResponse` to be sent under the `user` role.*
