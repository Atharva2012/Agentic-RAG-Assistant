# AI Research Assistant Agents

An AI-powered research assistant for searching and asking questions about indexed research papers. The system accepts PDF papers, extracts and indexes their text in Milvus, retrieves relevant passages, and generates answers with a Gemini model. It also supports multilingual questions in English, Spanish, French, and Italian.

## Features

- Upload and index research papers in PDF format.
- Categorize papers into \`AI\`, \`Security\`, or \`Other\` domains.
- Ask questions using natural language.
- Automatically detect the question language or select it manually.
- Translate non-English questions into English for retrieval and generation.
- Translate generated answers back into the user’s language.
- Return relevant paper references and recommended papers.
- Exclude reference and bibliography sections during PDF ingestion.
- Deploy the services independently on Kubernetes with horizontal pod autoscaling.

## Architecture

\`\`\`mermaid
flowchart LR
    User --> Frontend[Streamlit frontend]
    Frontend --> Agent[FastAPI agent service]
    Agent --> Translator[Translation service]
    Agent --> Milvus[Milvus vector database]
    Agent --> Gemini[Gemini models]
\`\`\`

The project is split into four services:

- \`frontend/\` — Streamlit interface for uploading papers and asking questions.
- \`agent-service/\` — FastAPI service that handles PDF ingestion, retrieval-augmented generation, and orchestration with LangGraph.
- \`Docker-NLP/\` — FastAPI translation and language-detection service.
- Milvus — Vector database used to store paper chunks and embeddings.

## How the workflow works

1. A user uploads a PDF and assigns it a title, domain, and language.
2. The agent service extracts the text and stops before the \`References\` or \`Bibliography\` section.
3. The remaining text is split into overlapping chunks and converted into embeddings.
4. Chunks, metadata, and embeddings are stored in Milvus.
5. When a user asks a question, the system detects or uses the supplied language.
6. Non-English questions are translated into English before similarity search.
7. The most relevant chunks from the selected domain are passed to the generation model.
8. The answer is translated back to the original question language when necessary.

## Prerequisites

For the Kubernetes deployment, install and configure:

- Docker
- Google Cloud CLI
- \`kubectl\`
- Terraform
- Access to a Google Cloud project with permission to create a GKE cluster or use an existing one
- A Google AI API key with access to the configured Gemini models

The application currently supports these languages:

\`\`\`text
en, es, fr, it
\`\`\`

## Try it with Kubernetes on GKE

The repository includes Kubernetes manifests and Terraform configuration for deploying Milvus and the application services on Google Kubernetes Engine.

### 1. Configure deployment variables

Set values appropriate for your Google Cloud project:

\`\`\`bash
export PROJECT_ID="your-gcp-project-id"
export REGION="us-central1"
export ZONE="us-central1-a"
export REPO_NAME="research-assistant"
export CLUSTER_NAME="research-assistant-gke"
\`\`\`

Authenticate and enable the required services:

\`\`\`bash
gcloud auth login
gcloud config set project "\$PROJECT_ID"
gcloud services enable container.googleapis.com artifactregistry.googleapis.com
\`\`\`

Create a Docker Artifact Registry repository:

\`\`\`bash
gcloud artifacts repositories create "\$REPO_NAME" \\
  --repository-format=docker \\
  --location="\$REGION"

gcloud auth configure-docker "\${REGION}-docker.pkg.dev"
\`\`\`

### 2. Deploy Milvus

If using the included Terraform configuration:

\`\`\`bash
cd milvus-gke
terraform init
terraform apply \\
  -var="project_id=\$PROJECT_ID" \\
  -var="zone=\$ZONE" \\
  -var="region=\$REGION"
\`\`\`

Confirm that Milvus is running:

\`\`\`bash
kubectl get pods -n milvus
kubectl get svc -n milvus
\`\`\`

### 3. Build and push the application images

Run these commands from the project root:

\`\`\`bash
docker build -t "\${REGION}-docker.pkg.dev/\${PROJECT_ID}/\${REPO_NAME}/translator:latest" Docker-NLP
docker push "\${REGION}-docker.pkg.dev/\${PROJECT_ID}/\${REPO_NAME}/translator:latest"

docker build -t "\${REGION}-docker.pkg.dev/\${PROJECT_ID}/\${REPO_NAME}/agent:latest" agent-service
docker push "\${REGION}-docker.pkg.dev/\${PROJECT_ID}/\${REPO_NAME}/agent:latest"

docker build -t "\${REGION}-docker.pkg.dev/\${PROJECT_ID}/\${REPO_NAME}/frontend:latest" frontend
docker push "\${REGION}-docker.pkg.dev/\${PROJECT_ID}/\${REPO_NAME}/frontend:latest"
\`\`\`

Before applying the manifests, update the image values in \`infra/k8s/translator.yaml\`, \`infra/k8s/agent.yaml\`, and \`infra/k8s/frontend.yaml\` to point to the images you pushed.

### 4. Create the cluster namespace and secrets

Connect \`kubectl\` to the target cluster:

\`\`\`bash
gcloud container clusters get-credentials "\$CLUSTER_NAME" \\
  --zone "\$ZONE" \\
  --project "\$PROJECT_ID"
\`\`\`

Create the namespace and application configuration:

\`\`\`bash
kubectl apply -f infra/k8s/namespace.yaml
kubectl apply -f infra/k8s/configmap.yaml
\`\`\`

Create a secret file from the example and replace the placeholder API key with a valid key. Do not commit the populated secret file.

\`\`\`bash
cp infra/k8s/secret.example.yaml infra/k8s/secret.yaml
kubectl apply -f infra/k8s/secret.yaml
\`\`\`

### 5. Deploy the services

\`\`\`bash
kubectl apply -f infra/k8s/milvus-alias-service.yaml
kubectl apply -f infra/k8s/translator.yaml
kubectl apply -f infra/k8s/agent.yaml
kubectl apply -f infra/k8s/frontend.yaml
\`\`\`

Check the deployment:

\`\`\`bash
kubectl get pods -n option1
kubectl get svc -n option1
kubectl get hpa -n option1
\`\`\`

The frontend service is exposed through a Kubernetes \`LoadBalancer\`. Retrieve its address with:

\`\`\`bash
kubectl get svc frontend-svc -n option1
\`\`\`

Open the external address in a browser. Use the **Upload Papers** tab to index PDFs, then use **Ask Questions** to query the indexed papers.

## Test the APIs

### Translator service

\`\`\`bash
kubectl port-forward -n option1 svc/translator-svc 8000:8000
\`\`\`

In another terminal:

\`\`\`bash
curl http://127.0.0.1:8000/healthz

curl -X POST http://127.0.0.1:8000/detect-language \\
  -H "Content-Type: application/json" \\
  -d '{"text":"Hola, ¿cómo estás?"}'
\`\`\`

### Agent service

\`\`\`bash
kubectl port-forward -n option1 svc/agent-svc 8080:8080
curl http://127.0.0.1:8080/healthz
\`\`\`

Upload a paper:

\`\`\`bash
curl -X POST http://127.0.0.1:8080/ingest \\
  -F "file=@/path/to/paper.pdf" \\
  -F "paper_title=Example Research Paper" \\
  -F "domain=AI" \\
  -F "language=en"
\`\`\`

Ask a question:

\`\`\`bash
curl -X POST http://127.0.0.1:8080/ask \\
  -H "Content-Type: application/json" \\
  -d '{
    "question":"What problem does this paper address?",
    "domain":"AI",
    "source_language":"en"
  }'
\`\`\`

## Configuration

The main configuration is defined in \`agent-service/app/config.py\` and \`infra/k8s/configmap.yaml\`.

Important settings include:

- \`GOOGLE_API_KEY\` — API key used by the Gemini language and embedding models.
- \`GEMINI_LLM_MODEL\` — generation model.
- \`GEMINI_EMB_MODEL\` — embedding model.
- \`MILVUS_HOST\` and \`MILVUS_PORT\` — Milvus connection settings.
- \`MILVUS_COLLECTION\` — collection used for paper chunks.
- \`TOP_K\` — number of chunks retrieved for a question.
- \`CHUNK_SIZE\` and \`CHUNK_OVERLAP\` — text-splitting parameters.

## Project structure

\`\`\`text
.
├── agent-service/       # Agent API, PDF ingestion, RAG pipeline, and Milvus access
├── Docker-NLP/          # Translation API
├── frontend/            # Streamlit application
├── infra/k8s/           # Kubernetes manifests
├── milvus-gke/          # Terraform configuration for Milvus on GKE
└── README.md
\`\`\`

## Limitations

- PDF extraction quality depends on the structure and formatting of the source document.
- The translation service relies on \`googletrans\`, which may occasionally fail because it depends on an external translation backend.
- Answers are grounded through retrieved context, but the system does not currently perform citation-by-span verification.
- Search quality and response time depend on the indexed corpus and available cluster resources.
- Milvus must be running and reachable before the agent service can ingest or search papers.

## Security notes

- Keep API keys and populated Kubernetes secret files out of version control.
- Review the CORS policy before exposing the agent service beyond the cluster.
- Restrict access to the frontend and cluster according to the deployment environment.

