# Setup Guide - Codex with Runtime Provider Overrides

This fork adds environment variable support for overriding provider configuration at runtime.

## Prerequisites

- **Rust toolchain** (1.70 or newer)
- **Node.js** 16+ (for the CLI wrapper)
- **Git**

Install Rust if needed:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

## Installation Steps

### 1. Clone the Fork

```bash
git clone https://github.com/daniel-salib/codex.git
cd codex
git checkout runtime-provider-overrides
```

Or if you already have it cloned:
```bash
cd /path/to/codex
git fetch origin
git checkout runtime-provider-overrides
git pull
```

### 2. Build the Rust Binary

```bash
cd codex-rs
cargo build --release
# Or if behind a corporate proxy:
# with-proxy cargo build --release
```

This will take several minutes on first build. The binary will be at:
`codex-rs/target/release/codex`

### 3. Install to Vendor Directory

Copy the compiled binary to where the CLI wrapper expects it:

```bash
# From the codex root directory
mkdir -p codex-cli/vendor/x86_64-unknown-linux-musl/codex
cp codex-rs/target/release/codex codex-cli/vendor/x86_64-unknown-linux-musl/codex/codex
chmod +x codex-cli/vendor/x86_64-unknown-linux-musl/codex/codex
```

### 4. Make CLI Wrapper Executable

The Node.js wrapper needs execute permissions:

```bash
# From the codex root directory
chmod +x codex-cli/bin/codex.js
```

### 5. Install Globally (Choose One Method)

#### Option A: Symlink (Recommended for Development)

```bash
# Make sure ~/bin exists and is in your PATH
mkdir -p ~/bin

# Create symlink
ln -sf "$(pwd)/codex-cli/bin/codex.js" ~/bin/codex

# Add to PATH if needed (add to ~/.bashrc or ~/.zshrc)
export PATH="$HOME/bin:$PATH"
```

### 6. Verify Installation

Test that codex is working:

```bash
codex --version
# Should output: codex-cli 0.0.0
```

If you get a "Permission denied" error, make sure you completed Step 4 (making codex.js executable).

### 7. Configure Codex for Local vLLM

Create a codex config file that defines a vLLM provider:

```bash
mkdir -p ~/.codex

cat > ~/.codex/config.toml << 'EOF'
# Default model name - vLLM typically exposes models as "default"
model = "default"

# Default provider to use
model_provider = "vllm-local"

[model_providers.vllm-local]
name = "Local vLLM"
# Fallback values - these can be overridden by environment variables (see next step)
base_url = "http://localhost:8000/v1"
wire_api = "responses"
env_key = "OPENAI_API_KEY"
EOF
```

### 8. Set Up Environment Variables

If you want to easily switch between different vLLM servers without editing config files:

```bash
# Add to ~/.bashrc or ~/.zshrc
export OPENAI_API_KEY="dummy-key-for-local-vllm"

# Runtime provider overrides - these override config.toml values!
export CODEX_PROVIDER_VLLM_LOCAL_BASE_URL="http://localhost:8000/v1"
export CODEX_PROVIDER_VLLM_LOCAL_WIRE_API="responses"

export PATH="$HOME/bin:$PATH"
```

Then reload your shell config:

```bash
source ~/.bashrc  # or source ~/.zshrc
```

### 9. Test Your Setup

Make sure your vLLM server is running on port 8000, then test codex:

```bash
# Simple test
codex exec --dangerously-bypass-approvals-and-sandbox "what's 2+2?"

# You should see output like:
# --------
# workdir: /your/current/directory
# model: default
# provider: vllm-local
# approval: never
# sandbox: danger-full-access
# --------
```

### 10. Changing vLLM Server Ports (Optional)

**If you used Option A (Simple Setup):**

Edit your config file to change the port:
```bash
nano ~/.codex/config.toml
# Change: base_url = "http://localhost:9000/v1"
```

**If you used Option B (Runtime Overrides):**

Just update the environment variable (no config file editing needed!):
```bash
export CODEX_PROVIDER_VLLM_LOCAL_BASE_URL="http://localhost:9000/v1"
# That's it! This demonstrates the power of runtime overrides.
```

## Usage

### Environment Variable Overrides (This Fork's Feature)

This fork allows you to override provider settings via environment variables **without modifying config files**:

```bash
# Format: CODEX_PROVIDER_<PROVIDER_ID>_BASE_URL and CODEX_PROVIDER_<PROVIDER_ID>_WIRE_API
# The PROVIDER_ID is the kebab-case provider name converted to SCREAMING_SNAKE_CASE
# Example: "vllm-local" becomes "VLLM_LOCAL"

export CODEX_PROVIDER_VLLM_LOCAL_BASE_URL="http://localhost:8000/v1"
export CODEX_PROVIDER_VLLM_LOCAL_WIRE_API="responses"

# Now any codex command automatically uses these overrides
codex exec --dangerously-bypass-approvals-and-sandbox "your prompt here"
```

**Benefits:**
- Change provider URL without editing config files
- Switch between different vLLM servers easily
- Great for testing and development

**Note:** The provider must already exist in your `~/.codex/config.toml`. These environment variables only override the `base_url` and `wire_api` fields.

### Alternative: Config Overrides with -c Flag

You can also override settings directly on the command line (works with any codex version):

```bash
# Override specific fields (must be on one line, no line breaks in the TOML)
codex exec -c "model_providers.vllm-local.base_url=http://localhost:9000/v1" -c "model_providers.vllm-local.wire_api=responses" --dangerously-bypass-approvals-and-sandbox "what's 2+2?"

# Or define a completely new provider inline (all on one line!)
codex exec -c "model_providers.my-vllm={ name = 'My vLLM', base_url = 'http://localhost:8000/v1', wire_api = 'responses', env_key = 'OPENAI_API_KEY' }" -c model_provider="my-vllm" -c model="default" --dangerously-bypass-approvals-and-sandbox "what's 2+2?"
```

**Important:** When using `-c` with inline TOML tables (`{ ... }`), the entire command must be on **one line** with no line breaks inside the TOML string, or the parser will fail.

## Troubleshooting

### Error: "Missing environment variable: OPENAI_API_KEY"
Set a dummy value:
```bash
export OPENAI_API_KEY="dummy-key-for-local-vllm"
```

### Error: "The model `gpt-5-codex` does not exist"
Update your config to use the correct model name (usually "default" for vLLM):
```bash
# Add to ~/.codex/config.toml
model = "default"
```

### Error: "unexpected status 404 Not Found"
Check that your vLLM server is running and accessible:
```bash
curl http://localhost:8000/v1/models
```

### Error: "invalid type: string ... expected struct ModelProviderInfo"
You have line breaks in your `-c` TOML inline table. Put the entire command on one line.
