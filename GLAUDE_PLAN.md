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
| **Tool Use** | Automatic file/system access | Core basic system tools | Full Godot API & custom OS tools |

**MVP UI Components:**
- **Chat Display:** A scrolling `RichTextLabel` with BBCode enabled. Basic formatting (bold, italics, code blocks) will be translated from the API's markdown responses into BBCode. Syntax highlighting will be kept to a minimum or handled via simple color tags to ensure the plugin remains lightweight.
- **Input Field:** A `TextEdit` at the bottom of the panel allowing multiline input (e.g., pasting code snippets).
- **Submit Button:** A button to send the prompt.
- **Loading Indicator:** A simple visual cue (e.g., spinning icon or text ellipsis) while waiting for the Gemini API response.

## 4. MVP Tool Set
To truly mimic Claude Code's capabilities, Glaude's MVP will support **core system tools** via the Gemini function calling API.

**Included MVP Tools (Replicating Real Claude capabilities):**
- `BashTool` (or `PowerShellTool` for Windows) - Executing OS shell commands via `OS.execute()`.
- `FileEditTool` - Modifying files using string replacements or diffs.
- `FileReadTool` - Reading files via Godot's `FileAccess`.
- `FileWriteTool` - Creating or overwriting files entirely.
- `GlobTool` / `GrepTool` - Listing files or searching file contents inside the project directory via `DirAccess` and Regex.
- `AskUserQuestionTool` - Pausing execution to explicitly prompt the developer for an answer.

**Excluded Tools (Anti-Distillation "Poison Pills"):**
Claude Code's internal implementation contains "fake" tools designed specifically to prevent model distillation (e.g. `TaskCreateTool`, `TeamDeleteTool`, `CronCreateTool`, `TestingPermissionTool`, `SleepTool`). These are explicitly **out of scope** and will not be mirrored in Glaude.

## 5. Implementation Steps (MVP)
1. **Plugin Initialization:**
   - Create the `plugin.cfg` and main `editor_plugin.gd` script.
   - Register the addon to add a control to the Script Editor bottom/side dock.
2. **Settings Management:**
   - Implement logic to add/read the `gemini_api_key` in `EditorInterface.get_editor_settings()`.
3. **UI Layout:**
   - Build a `.tscn` file for the chat interface (VBoxContainer containing a RichTextLabel, HBoxContainer with TextEdit and Button).
4. **API Integration & Function Calling:**
   - Create a `GeminiClient` script that manages an `HTTPRequest` node.
   - Construct the JSON payload for the Gemini API (e.g., `gemini-1.5-flash`), providing definitions for the MVP toolset array.
   - Parse tool call requests from the API response.
5. **Tool Execution:**
   - Map requested tools to GDScript implementations (e.g., reading a file via `FileAccess.get_file_as_string()` when `FileReadTool` is called) and append the tool response back into the chat history payload to send back to the API.

## 6. Future Phase: Godot Context (Out of Scope for MVP)
While basic file and system tools are in the MVP, deeper integrations specific to the engine will be reserved for future updates:
- **Godot Context Tools:** Tools specifically querying the current open script, the active scene tree in the editor, or the `EditorInterface` API itself.
- **Automated Engine Actions:** Allowing the AI to generate or attach nodes, configure Godot signals natively, or modify `.tscn` binary configurations safely.