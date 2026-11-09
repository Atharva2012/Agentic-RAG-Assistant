# AI-Research-Assistant-Agents on GKE

- Streamlit frontend
- LangGraph-based Agent service
- Translator service (FastAPI + googletrans)
- Milvus vector database on GKE
- 30 PDFs already categorized into AI, Security, Other, you can fine them on: https://cmu.box.com/s/73bv9vlxhcznyf38zmyie077yd0i3iwr
- Demo video shows below.



https://github.com/user-attachments/assets/25123b76-62b2-40e2-8b94-594c4e8c6edc



## Architecture Diagram

```mermaid
flowchart LR
    U[User] --> S[Streamlit Frontend]
    S --> A[LangGraph Agent Service]
    A --> T[Translator Service]
    A --> M[Milvus]
    S --> A
    A --> S
```

## 1) Project Structure

- `Docker-NLP/`: Translator service
- `agent-service/`: LangGraph orchestration + PDF ingestion + RAG
- `frontend/`: Streamlit UI
- `milvus-gke/`: Existing Terraform for Milvus deployment on GKE
- `infra/k8s/`: K8s manifests for translator/agent/frontend + HPA

## 2) Data/Schema Design

Milvus collection fields:

- `id` (Primary Key)
- `paper_title` (String)
- `domain` (String: `AI` / `Security` / `Other`)
- `language` (String: `en`, `es`, `fr`, `it`)
- `text_chunk` (String)
- `embedding` (FloatVector)

## 3) Important Parsing Rule (No References Indexing)

In `agent-service/app/pdf_utils.py`, the ingestion logic stops parsing once it sees a heading matching:

- `References`
- `Bibliography`

Everything after that heading is discarded before chunking/embedding.

## 4) GKE Deployment Roadmap

### Stage 1: Containerization + Artifact Registry

Build and push three images:

- `translator`
- `agent`
- `frontend`

Procedure:
1) Configure GCP and repository variables
```bash
export PROJECT_ID="your-gcp-project-id"
export REGION="us-central1"
export ZONE="us-central1-a"
export REPO_NAME="option1"
export CLUSTER_NAME="option1-gke"

gcloud auth login
gcloud config set project $PROJECT_ID
gcloud services enable container.googleapis.com artifactregistry.googleapis.com

gcloud artifacts repositories create $REPO_NAME \
  --repository-format=docker \
  --location=$REGION

gcloud auth configure-docker ${REGION}-docker.pkg.dev
```

### Stage 2: Infrastructure + HPA + Internal Services
First upload the entire folder (milvus-gke) to the Google Cloud CLI environment.
1. Deploy Milvus using your existing `milvus-gke/`.
```bash
cd milvus-gke
terraform init
terraform apply -var="project_id=$PROJECT_ID" -var="zone=$ZONE" -var="region=$REGION"

kubectl get svc -n milvus
kubectl get pods -n milvus
```
2. Build and push 3 images
Translator
```bash
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/translator:latest Docker-NLP
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/translator:latest
```
Agent
```bash
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/agent:latest agent-service
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/agent:latest
```
Streamlit
```bash
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/frontend:latest frontend
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/frontend:latest
```

3. Connect to the GKE cluster and deploy the application
```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

cd infra/k8s
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.example.yaml
kubectl apply -f milvus-alias-service.yaml
kubectl apply -f translator.yaml
kubectl apply -f agent.yaml
kubectl apply -f frontend.yaml

kubectl get pods -n option1
kubectl get svc -n option1
kubectl get hpa -n option1

kubectl logs -n option1 deploy/translator
kubectl logs -n option1 deploy/agent
kubectl logs -n option1 deploy/frontend
```
4.Test each service individually
Translator
```bash
kubectl port-forward -n option1 svc/translator-svc 8000:8000
```
Use another terminal：
```bash
curl http://127.0.0.1:8000/healthz
curl -X POST http://127.0.0.1:8000/detect-language \
  -H "Content-Type: application/json" \
  -d '{"text":"Hola, como estas?"}'
```
Agent
```bash
kubectl port-forward -n option1 svc/agent-svc 8080:8080
curl http://127.0.0.1:8080/healthz
```
Streamlit
```bash
kubectl get svc -n option1 frontend-svc
```

5. Internal DNS-based access:
   - Translator: `http://translator-svc.option1.svc.cluster.local:8000`
   - Milvus: `milvus-svc.option1.svc.cluster.local:19530` (via `ExternalName` alias)

6. HPA is configured for `translator` and `agent`:
   - min replicas: 1
   - max replicas: 5
   - CPU threshold: 70%

### Stage 3: Limitations and Design Decisions
Limitations
- `googletrans` may occasionally fail due to upstream limitations.
- PDF text extraction quality depends on document formatting.
- Current answer grounding is context-only prompting
- stronger hallucination controls can be added with citation-by-span checks.
- Benchmarks depend on current data volume and cluster resource state.

Design Decisions
- I implemented a LangGraph-based custom tool calling agent 
- Streamlit + Agent + Translator + Milvus: The system is split into independent services: Streamlit frontend, LangGraph-based agent, translator API, and Milvus vector database. This keeps responsibilities clear and simplifies deployment, scaling, and debugging on GKE.
- Agent as the single orchestrator: The frontend communicates only with the agent. The agent handles translation, retrieval, and answer generation. This centralizes workflow logic and enforces a consistent API contract.
- Multilingual workflow via translation pivot: User queries are detected/translated into English for retrieval and generation, then translated back to the user’s original language (en/es/fr/it). This improves retrieval consistency with a single embedding/query pipeline.
- Hallucination-aware generation strategy: The generation prompt instructs the model to answer only from retrieved context and explicitly acknowledge insufficient evidence. This reduces unsupported claims.
- Reference-section exclusion during ingestion: PDF parsing stops at References / Bibliography, and all subsequent content is discarded before chunking and embedding. This prevents bibliography-only chunks from polluting retrieval.
- Domain-aware retrieval with explicit metadata schema: Milvus stores id, paper_title, domain, language, text_chunk, and embedding. Queries apply a domain filter (AI/Security/Other) to improve relevance and align with project requirements.
- Kubernetes networking and exposure policy: translator and agent use internal ClusterIP services; frontend uses LoadBalancer. This minimizes public exposure while keeping user access simple.

