# Glaude (Godot Claude Code Implementation) - Product Development Plan

## 1. Overview & Architecture
**Project Name:** Glaude
**Concept:** A "Claude Code" style AI assistant directly integrated into the Godot Editor.
**Engine Version:** Godot 4.x (Latest)
**Language:** Pure GDScript
**Architecture:** The project will be implemented as a Godot `EditorPlugin`. It will add a new dock/panel to the Script Editor workspace to provide an interactive chat interface, allowing developers to communicate with an AI assistant.

## 2. API & Configuration
**AI Provider:** Google Gemini API (using HTTPRequest for REST API communication).
**Authentication:**
- The user's Gemini API key will be managed securely via Godot's `EditorSettings`.
- This ensures the key is stored locally and never accidentally committed to the project's version control.
- The addon will automatically prompt the user to configure their key in `Project -> Editor Settings -> Addons -> Glaude` upon activation.

## 3. UI/UX Design
The interface will be designed to be lightweight, avoiding heavy external dependencies like full Markdown parsers.

**Comparison with Claude Code CLI:**
| Feature | Claude Code CLI | Glaude (MVP) | Glaude (Future) |
| :--- | :--- | :--- | :--- |
| **Interface** | Terminal/CLI | Godot Script Editor Side Panel | - |
| **Rendering** | Rich Terminal Text/Markdown | `RichTextLabel` with BBCode | Custom syntax highlighting |
| **History** | Persistent thread history | Session-based chat history | Persistent local chat history |
| **Input** | Terminal prompt | `TextEdit` / `LineEdit` at the bottom | Multiline input with auto-resize |
| **Tool Use** | Automatic file/system access | None (out of scope for MVP) | Full Godot API & OS tool access |

**MVP UI Components:**
- **Chat Display:** A scrolling `RichTextLabel` with BBCode enabled. Basic formatting (bold, italics, code blocks) will be translated from the API's markdown responses into BBCode. Syntax highlighting will be kept to a minimum or handled via simple color tags to ensure the plugin remains lightweight.
- **Input Field:** A `TextEdit` at the bottom of the panel allowing multiline input (e.g., pasting code snippets).
- **Submit Button:** A button to send the prompt.
- **Loading Indicator:** A simple visual cue (e.g., spinning icon or text ellipsis) while waiting for the Gemini API response.

## 4. Implementation Steps (MVP)
1. **Plugin Initialization:**
   - Create the `plugin.cfg` and main `editor_plugin.gd` script.
   - Register the addon to add a control to the Script Editor bottom/side dock.
2. **Settings Management:**
   - Implement logic to add/read the `gemini_api_key` in `EditorInterface.get_editor_settings()`.
3. **UI Layout:**
   - Build a `.tscn` file for the chat interface (VBoxContainer containing a RichTextLabel, HBoxContainer with TextEdit and Button).
4. **API Integration:**
   - Create a `GeminiClient` script that manages an `HTTPRequest` node.
   - Construct the JSON payload required by the Gemini API (e.g., `gemini-1.5-flash` or `gemini-1.5-pro`).
   - Parse the JSON response and extract the text.
5. **Message Formatting:**
   - Write a lightweight parser to convert basic Markdown (like `**bold**` or ```code```) into Godot's BBCode equivalents (`[b]bold[/b]`, `[code]code[/code]`).

## 5. Future Phase: Tool Use (Out of Scope for MVP)
To truly match "Claude Code" capabilities, future iterations will introduce Gemini Function Calling (Tools).
- **File System Access:** Tools to read/write/list files using Godot's `FileAccess` and `DirAccess`.
- **System Commands:** A tool to execute bash/cmd scripts via `OS.execute()`.
- **Godot Context:** Tools to query the current open script, active scene, or project structure (`EditorInterface` API).
- **Automated Actions:** Allowing the AI to write scripts and create nodes automatically, requiring user confirmation for security.
