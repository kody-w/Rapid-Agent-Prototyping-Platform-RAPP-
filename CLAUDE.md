# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **Copilot Agent 365** - an enterprise AI assistant platform built on Azure Functions that provides a modular agent system with persistent memory management. The system is designed for rapid deployment and agent prototyping with Azure OpenAI integration.

## Development Commands

### Running Locally
```bash
# Activate virtual environment and start the function
./run.sh

# Or manually:
source .venv/bin/activate  # Unix/macOS
# or
source .venv/Scripts/activate  # Windows/Git Bash
func start
```

The local API will be available at: `http://localhost:7071/api/businessinsightbot_function`

### Installing Dependencies
```bash
pip install -r requirements.txt
```

### Azure Deployment
The project uses ARM template deployment via `azuredeploy.json` and generates platform-specific setup scripts automatically.

## Architecture Overview

### Core Components

1. **Azure Function Entry Point** (`function_app.py`):
   - HTTP-triggered function with CORS support
   - Handles user requests with conversation history
   - Manages user GUIDs for personalized memory contexts
   - Uses a default GUID (`c0p110t0-aaaa-bbbb-cccc-123456789abc`) when no specific user is identified

2. **Agent System** (`agents/`):
   - Base class: `BasicAgent` - All agents inherit from this
   - Dynamic agent loading from both local `agents/` folder and Azure File Storage
   - Agents are registered with metadata describing their capabilities
   - Supports multi-agent workflows loaded from `multi_agents/` in Azure storage

3. **Memory Management** (`utils/azure_file_storage.py`):
   - `AzureFileStorageManager` handles persistent storage in Azure Files
   - Two-tier memory system:
     - Shared memories: Available to all users (`shared_memories/`)
     - User-specific memories: Isolated per GUID (`memory/{guid}/`)
   - Automatic memory context switching based on user GUID

4. **Assistant Core Logic**:
   - Integrates with Azure OpenAI for natural language processing
   - Function calling mechanism to invoke agents dynamically
   - Dual response format: formatted text and voice-optimized response
   - Memory synthesis from both shared and user-specific contexts

### Agent Development Pattern

New agents should follow this structure:
```python
from agents.basic_agent import BasicAgent

class YourAgent(BasicAgent):
    def __init__(self):
        self.name = 'YourAgentName'
        self.metadata = {
            "name": self.name,
            "description": "What this agent does",
            "parameters": {
                "type": "object",
                "properties": {
                    # Define parameters here
                },
                "required": []
            }
        }
        super().__init__(self.name, self.metadata)

    def perform(self, **kwargs):
        # Implementation logic
        return result
```

## Configuration Requirements

The following environment variables must be set (typically in `local.settings.json` for local development):

- `AZURE_OPENAI_API_KEY`: Azure OpenAI service key
- `AZURE_OPENAI_ENDPOINT`: Azure OpenAI endpoint URL
- `AZURE_OPENAI_DEPLOYMENT_NAME`: Deployment name (e.g., 'gpt-deployment')
- `AZURE_OPENAI_API_VERSION`: API version (default: '2024-02-01')
- `AzureWebJobsStorage`: Azure Storage connection string
- `AZURE_FILES_SHARE_NAME`: File share name for memory storage
- `ASSISTANT_NAME`: Bot's display name
- `CHARACTERISTIC_DESCRIPTION`: Bot's personality description

## Important Implementation Details

1. **GUID Handling**: The system uses GUIDs to maintain separate conversation contexts. If no GUID is provided, it defaults to a common GUID for shared context.

2. **Memory Initialization**: Memory contexts are loaded with `full_recall=True` to ensure complete context is available for conversations.

3. **Dynamic Agent Loading**: Agents can be deployed to Azure File Storage without redeploying the function app - they're loaded at runtime.

4. **Response Format**: All responses include both a formatted version (with markdown) and a voice-optimized version (plain text, 1-2 sentences).

5. **Error Handling**: Comprehensive error handling with retries for OpenAI API calls and graceful fallbacks for memory operations.