# Kong AI Gateway — Enterprise AI Chat & DLP Firewall

This repository contains a Proof of Concept (PoC) demonstrating how to secure AI Chat systems using the **Kong AI Gateway** as an enterprise perimeter. Initially built for Volvo Cars, this project uses Kong to enforce strict Data Loss Prevention (DLP) rules (preventing PII leaks), log AI usage for GDPR, and intelligently route large language model requests.

## 🚀 Getting Started

1. Copy `.env.example` to `.env` and fill in your OpenRouter API Key and path to your Kong license file.
2. Run `./start.sh` to initialize the full environment (Docker containers, Kong configuration, and the Python backend).
3. Access the Chat Dashboard locally at `http://localhost:8080`.

**Note:** The `.env` file contains absolute secrets and is explicitly ignored in the `.gitignore` to prevent leaking credentials to GitHub. Only the `.env.example` file is tracked.

---

## 📂 Architecture Overview: How the Files Connect

The application functions via three interconnected layers: Infrastructure, Security Configuration, and the Backend/Frontend Application.

### 1. Infrastructure Layer
- **`docker-compose.yml`**
  - **Purpose:** Spins up the core infrastructure. It starts the Kong API Gateway and a PostgreSQL database (required by Kong Enterprise to store its policies). It mounts the required license file securely.
- **`start.sh`**
  - **Purpose:** The master orchestration script. 
  - **How it connects:** It executes the Docker up commands, verifies `.env` existence, installs Python dependencies from `requirements.txt`, runs `setup_kong.sh` to configure the gateway, and finally boots up the FastAPI web server (`main.py`).

### 2. Security & Gateway Configuration Layer
- **`setup_kong.sh`**
  - **Purpose:** Programmatically configures the Kong AI Gateway via its Admin API.
  - **How it connects:** It creates the routes routing traffic to OpenRouter (Gemini), injects your authentication keys, and attaches multiple core plugins: `rate-limiting`, `ai-prompt-guard` (the DLP Firewall), `ai-prompt-decorator` (System prompt injection), and `file-log`.
- **`dlp_test_cases.md`**
  - **Purpose:** A markdown file documenting various PII, Prompt Injection, and IP leak prompts used to validate the `ai-prompt-guard` firewall configurations injected by `setup_kong.sh`.

### 3. Application Layer
- **`main.py`**
  - **Purpose:** The FastAPI Python backend. it serves as the bridge between the user's browser and the Kong proxy.
  - **How it connects:** When a user types a chat message, this backend transforms it into an OpenAI compatible JSON payload and sends it to `http://localhost:8000/ai` (which is Kong). It then parses the response (or Kong's blocked 400 error) and feeds it back to the UI.
- **`static/index.html`**
  - **Purpose:** The sleek frontend Chat UI. 
  - **How it connects:** It renders the chat bubbles, captures the user's string prompt, and dynamically updates the **Live Request Journey Pipeline** at the bottom of the screen by polling the `/api/chat` route exposed by `main.py`.
- **`requirements.txt`**
  - **Purpose:** Declares the Python packages necessary (like `fastapi` and `uvicorn`) to run `main.py`.

---

## 🔒 Security Capabilities (Kong Plugins)

When a message is sent from the Chat UI (`index.html`), it flows through Kong's active plugins defined in `setup_kong.sh`:

1. **`ai-prompt-decorator`:** Automatically injects the enterprise system instruction so the AI responds as a helpful assistant without needing the frontend app to configure it.
2. **`ai-prompt-guard`:** The DLP bouncer. Using highly-optimized Regular Expressions, it instantly scans the `$..content` payload for Credit Cards, Swedish Identity Numbers, Volvo VINs, automotive IP ("ecu data dump"), and Prompt Injections ("ignore previous instructions"). If found, the HTTP request is dropped before entering the public internet.
3. **`rate-limiting`:** Prevents users from submitting more than 5 prompts per minute natively at the gateway.
4. **`file-log`:** Dumps successful traffic and dropped threats to `/tmp/kong-dns-tapir.log` for GDPR Article 30 auditing.

---

**Built as part of the Volvo DNS TAPIR internal evaluation for Securing AI Systems.**
