# Running a Coding Agent Locally

## Requirements:
- Mac Mini M2
- 16GB RAM
- 256GB VRAM
- Package manager homebrew installed
## 🎯 Recommended Ollama Model
| Model                       | Size   | VRAM Usage | Best For                                                           |
| --------------------------- | ------ | ---------- | ------------------------------------------------------------------ |
| **`qwen2.5-coder:7b`**      | ~4.5GB | ~6-8GB     | **Primary recommendation** - Best quality-to-size ratio for coding |
| **`deepseek-coder-v2:16b`** | ~9GB   | ~10-12GB   | Alternative for complex reasoning (slower but smarter)             |
| **`phi3:3.8b`**             | ~2.3GB | ~4-5GB     | Ultra-fast fallback for simple completions                         |
## Steps
### 1. Install Ollama
    
```sh    
# Install Ollama natively (required for Metal GPU acceleration)
brew install ollama

# Or use the official installer
curl -fsSL https://ollama.com/install.sh | sh

# Start Ollama service
ollama serve

# Pull the recommended 7B coding model
ollama pull qwen2.5-coder:7b

# Optional: Pull smaller fallback model
ollama pull phi3:3.8b

# Test it works
ollama run qwen2.5-coder:7b "Write a Python function to validate email addresses"

# Check if Ollama is actually running
lsof -i :11434

# Then in another terminal, check if it's using GPU:
ollama ps

# If it shows 100% CPU instead of 100% GPU Metal acceleration is NOT working.
```
