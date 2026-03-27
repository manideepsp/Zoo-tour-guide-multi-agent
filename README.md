# 🦁 Zoo Tour Guide — Multi-Agent AI Application

> An intelligent Zoo Tour Guide powered by **Google Agent Development Kit (ADK)** and **Gemini**. A multi-agent pipeline that greets visitors, researches animal information from Wikipedia, and delivers friendly, informative tour responses.

[![Python](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python)](https://www.python.org/)
[![Google ADK](https://img.shields.io/badge/Google%20ADK-1.14.0-orange)](https://pypi.org/project/google-adk/)
[![Cloud Run](https://img.shields.io/badge/Deploy-Cloud%20Run-4285F4?logo=googlecloud)](https://cloud.google.com/run)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

---

## 📖 Table of Contents

1. [Overview](#-overview)
2. [Architecture](#-architecture)
3. [Agent Pipeline Explained](#-agent-pipeline-explained)
4. [Project Structure](#-project-structure)
5. [Prerequisites](#-prerequisites)
6. [Local Setup from Scratch](#-local-setup-from-scratch)
7. [Running the Agent Locally](#-running-the-agent-locally)
8. [Deploying to Google Cloud Run](#-deploying-to-google-cloud-run)
9. [Environment Variables Reference](#-environment-variables-reference)
10. [Troubleshooting](#-troubleshooting)

---

## 🌟 Overview

The **Zoo Tour Guide** is a conversational AI application that acts as a knowledgeable and friendly zoo tour guide. When a visitor asks about an animal, the system:

1. **Greets** the visitor and captures their question
2. **Researches** the animal using Wikipedia for general knowledge (lifespan, diet, habitat, fun facts)
3. **Formats** all gathered information into a friendly, easy-to-read response

The application is built with [Google ADK](https://google.github.io/adk-docs/), which provides a robust framework for orchestrating multiple specialized AI agents as a sequential pipeline.

---

## 🏗️ Architecture

```
User Input
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                   root_agent (Greeter)               │
│  • Welcomes visitor                                  │
│  • Captures user prompt via `add_prompt_to_state`   │
│  • Delegates to tour_guide_workflow                  │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│            tour_guide_workflow (SequentialAgent)     │
│                                                      │
│  Step 1: comprehensive_researcher                    │
│    └─ Queries Wikipedia via LangchainTool            │
│    └─ Stores findings in state["research_data"]      │
│                                                      │
│  Step 2: response_formatter                          │
│    └─ Reads research_data from state                 │
│    └─ Produces a friendly, readable response         │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
              Final Response to User
```

### Technology Stack

| Component | Technology |
|---|---|
| Agent Framework | [Google ADK](https://google.github.io/adk-docs/) `1.14.0` |
| AI Model | Gemini (configurable via `MODEL` env var) |
| External Knowledge | Wikipedia via `langchain-community` + `wikipedia` |
| Logging | Google Cloud Logging |
| Deployment | Google Cloud Run |

---

## 🤖 Agent Pipeline Explained

The application defines agents in `agent.py`. Here is a detailed walkthrough:

### 1. State Management Tool — `add_prompt_to_state`

```python
def add_prompt_to_state(tool_context: ToolContext, prompt: str) -> dict[str, str]:
    """Saves the user's initial prompt to the state."""
    tool_context.state["PROMPT"] = prompt
    logging.info(f"[State updated] Added to PROMPT: {prompt}")
    return {"status": "success"}
```

This is a **custom tool** (a plain Python function) that stores the user's question in the ADK **session state** dictionary under the key `"PROMPT"`. The session state is a shared memory store accessible by all agents in the pipeline, enabling them to pass data between steps without explicit function calls.

---

### 2. Wikipedia Tool

```python
wikipedia_tool = LangchainTool(
    tool=WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())
)
```

ADK's `LangchainTool` wrapper adapts any LangChain tool into a native ADK tool. Here it wraps LangChain's `WikipediaQueryRun`, which searches and retrieves article summaries from Wikipedia. This gives the researcher agent access to a vast general knowledge base.

---

### 3. `comprehensive_researcher` — Step 1 of the Pipeline

```python
comprehensive_researcher = Agent(
    name="comprehensive_researcher",
    model=model_name,
    instruction="""...""",
    tools=[wikipedia_tool],
    output_key="research_data"
)
```

This agent is instructed to:
- Read the user's `{ PROMPT }` from session state (ADK automatically substitutes template variables)
- Decide whether to query Wikipedia, or both Wikipedia and internal zoo data, based on the question
- Store its synthesized findings into `output_key="research_data"` — ADK automatically writes this to `state["research_data"]`

---

### 4. `response_formatter` — Step 2 of the Pipeline

```python
response_formatter = Agent(
    name="response_formatter",
    model=model_name,
    instruction="""...\n    RESEARCH_DATA:\n    { research_data }"""
)
```

This agent receives `{ research_data }` (injected from state) and transforms it into a warm, conversational tour-guide response. It separates zoo-specific information from general Wikipedia facts and presents them in a friendly, engaging way.

---

### 5. `tour_guide_workflow` — Sequential Orchestrator

```python
tour_guide_workflow = SequentialAgent(
    name="tour_guide_workflow",
    sub_agents=[comprehensive_researcher, response_formatter]
)
```

`SequentialAgent` is an ADK built-in orchestrator that runs its `sub_agents` **in order**, one after the other. The output of each step is stored in session state so the next agent can consume it.

---

### 6. `root_agent` — Entry Point

```python
root_agent = Agent(
    name="greeter",
    model=model_name,
    tools=[add_prompt_to_state],
    sub_agents=[tour_guide_workflow]
)
```

ADK requires a single `root_agent` as the main entry point. The greeter:
1. Welcomes the user
2. When the user responds, calls `add_prompt_to_state` to save the question
3. Transfers control to `tour_guide_workflow` to run the research-and-format pipeline

---

## 📁 Project Structure

```
Zoo-tour-guide-multi-agent/
├── agent.py           # All agent definitions, tools, and pipeline logic
├── __init__.py        # Package entry point — imports the agent module
├── requirements.txt   # Python dependencies
└── README.md          # This file
```

---

## ✅ Prerequisites

Before you begin, make sure you have:

- **Python 3.11+** — [Download](https://www.python.org/downloads/)
- **Google Cloud Account** — [Create one free](https://cloud.google.com/free)
- **Google Cloud CLI (`gcloud`)** — [Install guide](https://cloud.google.com/sdk/docs/install)
- **Docker** (for Cloud Run deployment) — [Install](https://docs.docker.com/get-docker/)
- A **Google Cloud Project** with billing enabled
- A **Gemini API key** or Vertex AI access

---

## 🛠️ Local Setup from Scratch

### Step 1 — Clone the repository

```bash
git clone https://github.com/manideepsp/Zoo-tour-guide-multi-agent.git
cd Zoo-tour-guide-multi-agent
```

### Step 2 — Create and activate a virtual environment

```bash
# Create venv
python -m venv .venv

# Activate on macOS/Linux
source .venv/bin/activate

# Activate on Windows
.venv\Scripts\activate
```

### Step 3 — Install dependencies

```bash
pip install -r requirements.txt
```

This installs:

| Package | Version | Purpose |
|---|---|---|
| `google-adk` | `1.14.0` | ADK framework — agents, tools, orchestration |
| `langchain-community` | `0.3.27` | LangChain integrations (Wikipedia tool) |
| `wikipedia` | `1.4.0` | Wikipedia API client |

> **Note:** `python-dotenv` and `google-cloud-logging` are pulled in as transitive dependencies of `google-adk`.

### Step 4 — Authenticate with Google Cloud

```bash
gcloud auth login
gcloud auth application-default login
```

### Step 5 — Set your Google Cloud project

```bash
gcloud config set project YOUR_PROJECT_ID
```

### Step 6 — Configure environment variables

Create a `.env` file in the project root:

```bash
touch .env
```

Add the following:

```dotenv
# The Gemini model to use for all agents
# Examples: gemini-2.0-flash, gemini-1.5-pro, gemini-2.5-flash
MODEL=gemini-2.0-flash
```

> The `MODEL` variable is read at startup via `os.getenv("MODEL")` and used by all three agents.

### Step 7 — Enable required Google Cloud APIs

```bash
gcloud services enable \
  aiplatform.googleapis.com \
  logging.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
```

---

## 🚀 Running the Agent Locally

Google ADK ships with a built-in development server and web UI.

### Start the ADK web UI (recommended for development)

```bash
adk web
```

This starts a local server at `http://localhost:8080`. Open it in your browser to:
- Chat with the Zoo Tour Guide interactively
- Inspect the agent pipeline execution step by step
- View tool calls, state values, and model outputs in real time

### Run as a CLI conversation

```bash
adk run Zoo-tour-guide-multi-agent
```

This starts an interactive chat session in your terminal.

### Run the API server

```bash
adk api_server
```

This exposes the agent as a REST API at `http://localhost:8080/run`. You can send requests like:

```bash
curl -X POST http://localhost:8080/run \
  -H "Content-Type: application/json" \
  -d '{"app_name": "Zoo-tour-guide-multi-agent", "user_id": "user1", "session_id": "session1", "new_message": {"role": "user", "parts": [{"text": "Tell me about lions"}]}}'
```

---

## ☁️ Deploying to Google Cloud Run

Cloud Run is a fully managed serverless platform. ADK has built-in support for deploying agents to Cloud Run using the `adk deploy cloud_run` command.

### Step 1 — Set environment variables for deployment

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1          # Choose your preferred region
export SERVICE_NAME=zoo-tour-guide
```

### Step 2 — Deploy with ADK CLI

The simplest way is to use ADK's built-in deployment command, which handles Dockerization, image push, and Cloud Run service creation automatically:

```bash
adk deploy cloud_run \
  --project=$PROJECT_ID \
  --region=$REGION \
  --service_name=$SERVICE_NAME \
  --app_name=Zoo-tour-guide-multi-agent \
  --with_ui \
  Zoo-tour-guide-multi-agent
```

**What this command does:**
1. Packages your agent code into a container image
2. Pushes the image to **Google Artifact Registry**
3. Creates (or updates) a **Cloud Run service** with the specified name
4. Configures the service to serve both the agent API and the ADK web UI (via `--with_ui`)

> ⏱️ First deployment typically takes 3–5 minutes.

### Step 3 — Set the MODEL environment variable on Cloud Run

After deploying, set your model environment variable on the Cloud Run service:

```bash
gcloud run services update $SERVICE_NAME \
  --region=$REGION \
  --set-env-vars MODEL=gemini-2.0-flash
```

### Step 4 — Verify the deployment

```bash
# Get the service URL
gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --format="value(status.url)"
```

Open the URL in your browser — you'll see the ADK web UI, ready to chat with the Zoo Tour Guide.

---

### Manual Deployment (Alternative)

If you prefer full control, you can deploy manually using Docker and `gcloud`.

#### 1. Create a `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080

CMD ["adk", "api_server", "--host", "0.0.0.0", "--port", "8080"]
```

#### 2. Create an Artifact Registry repository

```bash
gcloud artifacts repositories create zoo-tour-guide-repo \
  --repository-format=docker \
  --location=$REGION \
  --description="Zoo Tour Guide container images"
```

#### 3. Build and push the container image

```bash
# Configure Docker authentication
gcloud auth configure-docker $REGION-docker.pkg.dev

# Build the image
docker build -t $REGION-docker.pkg.dev/$PROJECT_ID/zoo-tour-guide-repo/$SERVICE_NAME:latest .

# Push the image
docker push $REGION-docker.pkg.dev/$PROJECT_ID/zoo-tour-guide-repo/$SERVICE_NAME:latest
```

#### 4. Deploy to Cloud Run

```bash
gcloud run deploy $SERVICE_NAME \
  --image=$REGION-docker.pkg.dev/$PROJECT_ID/zoo-tour-guide-repo/$SERVICE_NAME:latest \
  --region=$REGION \
  --platform=managed \
  --allow-unauthenticated \
  --set-env-vars MODEL=gemini-2.0-flash \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=10
```

#### 5. Confirm the service is running

```bash
gcloud run services list --region=$REGION
```

---

### Cloud Run Service Configuration Reference

| Flag | Default | Description |
|---|---|---|
| `--allow-unauthenticated` | — | Allows public access without authentication |
| `--memory` | `512Mi` | Memory allocated per container instance |
| `--cpu` | `1` | CPU allocated per container instance |
| `--min-instances` | `0` | Minimum instances (0 = scale to zero, saves cost) |
| `--max-instances` | `10` | Maximum instances for auto-scaling |
| `--concurrency` | `80` | Max concurrent requests per instance |

---

## 🔐 Environment Variables Reference

| Variable | Required | Description | Example |
|---|---|---|---|
| `MODEL` | ✅ Yes | Gemini model name used by all agents | `gemini-2.0-flash` |

> **Where to find model names:** See [Gemini model overview](https://ai.google.dev/gemini-api/docs/models/gemini) for available model IDs.

---

## 🔍 Troubleshooting

### `google.auth.exceptions.DefaultCredentialsError`

You are not authenticated with Google Cloud.

```bash
gcloud auth application-default login
```

### `MODEL` environment variable is `None`

Make sure your `.env` file exists in the project root and contains `MODEL=...`. For Cloud Run, set it via:

```bash
gcloud run services update $SERVICE_NAME --region=$REGION --set-env-vars MODEL=gemini-2.0-flash
```

### Wikipedia tool returns no results

This is usually a network issue or a very specific query with no Wikipedia article. Try rephrasing the animal name (e.g., "African lion" instead of "lion at zoo").

### Cloud Run deployment fails with permission errors

Ensure your account has the required IAM roles:

```bash
# Grant Cloud Run Admin and Service Account User roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/iam.serviceAccountUser"
```

### View Cloud Run logs

```bash
gcloud run services logs read $SERVICE_NAME --region=$REGION --limit=50
```

Or view them in the [Google Cloud Console → Cloud Run → Logs](https://console.cloud.google.com/run).

---

## 📚 Resources

- [Google ADK Documentation](https://google.github.io/adk-docs/)
- [Google ADK GitHub](https://github.com/google/adk-python)
- [Google ADK Samples](https://github.com/google/adk-samples)
- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Gemini API Models](https://ai.google.dev/gemini-api/docs/models/gemini)
- [LangChain Community Tools](https://python.langchain.com/docs/integrations/tools/)

---

*Happy exploring! 🦒🐘🦁*
