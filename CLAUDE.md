# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Rapid Agent Prototyping Platform (RAPP)** - A self-evolving AI assistant framework built on Azure Functions that enables dynamic capability acquisition without code deployment. The system features runtime agent loading from Azure File Storage, a dual-tier memory architecture (shared + user-specific contexts), and conversational programming capabilities.

## Development Commands

### Local Development
```bash
# Start the function (automatically activates virtual environment)
./run.sh

# Or manually
source .venv/bin/activate  # Unix/macOS
func start

# Install dependencies
pip install -r requirements.txt
```

Local API endpoint: `http://localhost:7071/api/businessinsightbot_function`

### Testing the API
```bash
# Basic interaction
curl -X POST http://localhost:7071/api/businessinsightbot_function \
  -H "Content-Type: application/json" \
  -d '{"user_input": "Hello", "conversation_history": []}'

# With user GUID for context
curl -X POST http://localhost:7071/api/businessinsightbot_function \
  -H "Content-Type: application/json" \
  -d '{"user_input": "Hello", "conversation_history": [], "user_guid": "your-guid-here"}'
```

### Azure Deployment
Deploy via ARM template: `azuredeploy.json` generates platform-specific setup scripts automatically. Click "Deploy to Azure" button in README.md.

## Architecture

### Core Request Flow (function_app.py)

1. **HTTP Request Entry** (function_app.py:621)
   - CORS handling for cross-origin requests
   - JSON payload validation: requires `user_input` and `conversation_history`
   - Optional `user_guid` parameter for user-specific memory context

2. **Dynamic Agent Loading** (function_app.py:85-206)
   - Scans local `agents/` folder for agent classes
   - Loads agents from Azure File Storage `agents/` directory at runtime
   - Loads multi-agent workflows from `multi_agents/` directory
   - All agents must inherit from `BasicAgent` and have `metadata` property

3. **Assistant Initialization** (function_app.py:208-246)
   - Azure OpenAI client setup using environment variables
   - Memory context initialization (defaults to `c0p110t0-aaaa-bbbb-cccc-123456789abc`)
   - Loads both shared and user-specific memories with `full_recall=True`

4. **Response Generation** (function_app.py:479-617)
   - GUID extraction from prompt or history (function_app.py:248-262, 303-322)
   - Memory context switching if new GUID detected
   - Conversation history limited to last 20 messages to prevent memory issues
   - Azure OpenAI function calling for agent invocation
   - Automatic retry mechanism (max 3 retries with 2-second delay)

### Memory System (utils/azure_file_storage.py)

**Two-Tier Architecture:**
- **Shared Memory**: `shared_memories/memory.json` - accessible to all users
- **User-Specific Memory**: `memory/{guid}/user_memory.json` - isolated per GUID

**Key Operations:**
- `set_memory_context(guid)` - switches between shared and user-specific storage
- `read_json()` / `write_json(data)` - handles current context automatically
- Memory format: UUID-keyed dictionaries with `{conversation_id, session_id, message, mood, theme, date, time}`

**Context Switching:**
- GUID validation: `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`
- Falls back to shared memory on invalid GUID or errors
- Directories created lazily only when valid GUID provided

### Agent System

**Base Class Pattern (agents/basic_agent.py):**
```python
class YourAgent(BasicAgent):
    def __init__(self):
        self.name = 'YourAgentName'
        self.metadata = {
            "name": self.name,
            "description": "What this agent does",
            "parameters": {
                "type": "object",
                "properties": {
                    "param_name": {
                        "type": "string",
                        "description": "Parameter description"
                    }
                },
                "required": ["param_name"]
            }
        }
        super().__init__(self.name, self.metadata)

    def perform(self, **kwargs):
        # Implementation
        return result_string
```

**Built-in Agents:**
- `ContextMemory` - Retrieves stored memories with filtering options
- `ManageMemory` - Stores new memories with type categorization

**Agent Parameters:**
- `user_guid` automatically injected for `ManageMemory` and `ContextMemory` agents (function_app.py:555-556)
- Parameters sanitized to convert `None` to empty strings (function_app.py:546-551)
- Results must be strings or convertible to strings

### Response Format (function_app.py:391-422)

All assistant responses must contain two parts separated by `|||VOICE|||`:

1. **Formatted Response** (before delimiter):
   - Full markdown formatting with **bold**, `code`, headings, lists
   - Detailed information and structure

2. **Voice Response** (after delimiter):
   - 1-2 sentences maximum
   - Plain text, no formatting
   - Conversational tone for audio playback

Example: `"**Result:** Success\n\n|||VOICE|||\nThe operation completed successfully."`

### Runtime Agent Loading

**Discovery Mechanism:**
- Local: `agents/*.py` (excluding `__init__.py` and `basic_agent.py`)
- Azure Storage: `agents/*_agent.py`
- Multi-agents: `multi_agents/*_agent.py`

**Hot-Loading Process:**
1. Download agent code from Azure File Storage
2. Write to `/tmp/agents/` or `/tmp/multi_agents/`
3. Dynamically import using `importlib.util.spec_from_file_location`
4. Instantiate and register agent
5. Clean up temporary file

**No restarts required** - agents are loaded on each function invocation.

## Configuration

Required environment variables in `local.settings.json`:

```json
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "AzureWebJobsStorage": "DefaultEndpointsProtocol=https;...",
    "AZURE_FILES_SHARE_NAME": "your-share-name",
    "AZURE_OPENAI_API_KEY": "your-key",
    "AZURE_OPENAI_ENDPOINT": "https://your-endpoint.openai.azure.com/",
    "AZURE_OPENAI_DEPLOYMENT_NAME": "your-deployment",
    "AZURE_OPENAI_API_VERSION": "2024-02-01",
    "ASSISTANT_NAME": "Your Assistant Name",
    "CHARACTERISTIC_DESCRIPTION": "helpful assistant"
  }
}
```

**NEVER commit `local.settings.json`** - it's in `.gitignore` and contains secrets.

## Important Implementation Details

### GUID Handling
- Default GUID: `c0p110t0-aaaa-bbbb-cccc-123456789abc` (function_app.py:19)
- GUID-only messages trigger memory context switching without response generation
- GUIDs extracted from first message in history or current prompt
- Format: `guid: your-guid-here` or just the GUID alone

### Memory Context Persistence
- Each Assistant instance starts with default GUID memory loaded
- Context switches when new GUID detected in request
- `full_recall=True` ensures complete memory loads during initialization
- Shared and user memories loaded simultaneously for comprehensive context

### Function Calling Flow
1. Azure OpenAI generates function call with agent name and parameters
2. Agent lookup in `known_agents` dictionary
3. Parameter sanitization (None → empty string)
4. Agent execution via `perform(**kwargs)`
5. Result appended to messages as `{"role": "function", "name": agent_name, "content": result}`
6. Follow-up call to OpenAI for natural language response
7. Response parsed into formatted and voice components

### Error Handling
- API retries: 3 attempts with 2-second delays
- Memory operation failures fall back to shared memory
- Conversation history trimming prevents memory exhaustion
- Comprehensive logging via Python `logging` module

### Python Version
- **Use Python 3.9-3.11** (3.11 recommended)
- **Avoid Python 3.13+** due to Azure Functions compatibility issues

## Testing

### Web Interface
Open `index.html` in a browser for a chat interface to the local or deployed function.

### Direct API Testing
Use curl, Postman, or any HTTP client with:
- Method: `POST`
- Headers: `Content-Type: application/json`
- Body: `{"user_input": "your message", "conversation_history": []}`

### Agent Development
1. Create new agent class in `agents/` or upload to Azure Storage `agents/`
2. Inherit from `BasicAgent`
3. Define `name` and `metadata` in `__init__`
4. Implement `perform(**kwargs)` method
5. Return string results
6. Test by invoking function (no restart needed for Azure Storage agents)
