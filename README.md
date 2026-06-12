# MedRAG Setup & Deployment Guide

## Quick Setup (5 minutes)

### 1. Clone & Install
```bash
git clone https://github.com/rika1089/medrag_multiagent_evaluation.git
cd medrag_multiagent_evaluation
pip install -r requirements.txt
```

### 2. Configure API Keys
Create `.env` file:
```bash
# Required API Keys
OPENROUTER_API_KEY=sk-or-v1-xxxxx
SERPER_API_KEY=xxxx
QDRANT_API_KEY=xxxx

# Optional
OPENROUTER_MODEL="claude-3-opus-20240229"
LOG_LEVEL=INFO
```

### 3. Run Backend
```bash
uvicorn app.main:app --reload --port 8000
```

### 4. Run Frontend
```bash
streamlit run ui/dashboard.py
```

✅ System ready at: http://localhost:8501

---

## Detailed Installation

### Prerequisites
- **Python 3.9+**
- **pip** or **conda**
- **Git**

### Step 1: Clone Repository
```bash
git clone https://github.com/rika1089/medrag_multiagent_evaluation.git
cd medrag_multiagent_evaluation
```

### Step 2: Create Virtual Environment (Recommended)

**Using venv:**
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

**Using conda:**
```bash
conda create -n medrag python=3.9
conda activate medrag
```

### Step 3: Install Dependencies
```bash
pip install -r requirements.txt
```

**Key Dependencies:**
- `fastapi==0.104.1` – REST API framework
- `streamlit==1.28.0` – Web dashboard
- `crewai==0.1.0` – Multi-agent orchestration
- `qdrant-client==2.7.0` – Vector database client
- `sentence-transformers==2.2.2` – Embeddings
- `pydantic==2.5.0` – Data validation
- `openai==1.3.5` – OpenAI/OpenRouter clients
- `google-search-results==2.4.3` – Serper API
- `altair==5.1.0` – Visualization (Streamlit)

### Step 4: API Keys & Configuration

#### Get OpenRouter API Key
1. Visit https://openrouter.ai
2. Sign up (free tier available)
3. Generate API key from dashboard
4. Choose model: `claude-3-opus-20240229`

#### Get Serper API Key
1. Visit https://serper.dev
2. Sign up for free tier (100 requests/month)
3. Copy API key

#### Get Qdrant API Key
1. **Option A (Cloud):** https://qdrant.tech
   - Create account
   - Create cluster
   - Copy API key
2. **Option B (Local):** Skip if using local Qdrant Docker
   ```bash
   docker run -p 6333:6333 qdrant/qdrant
   ```

#### Create `.env` File
```bash
cat > .env << EOF
# Core API Keys
OPENROUTER_API_KEY=sk-or-v1-your-key-here
SERPER_API_KEY=your-serper-key-here
QDRANT_API_KEY=your-qdrant-key-here

# Optional Configuration
OPENROUTER_MODEL=claude-3-opus-20240229
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_URL=http://localhost:6333  # For local
LOG_LEVEL=INFO
DEBUG=false
EOF
```

### Step 5: Initialize Data & Models

```bash
# Download embedding model (auto on first use, ~100MB)
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"

# Load evaluation dataset (50 USMLE questions)
ls data/medqa_eval_50.jsonl

# (Optional) Build Qdrant index
python scripts/build_qdrant_index.py --source data/pubmed_sample.jsonl
```

---

## Running the System

### Option 1: Local Development

**Terminal 1 - Backend:**
```bash
uvicorn app.main:app --reload --port 8000 --host 0.0.0.0
```

**Terminal 2 - Frontend:**
```bash
streamlit run ui/dashboard.py
```

**Access:**
- API Docs: http://localhost:8000/docs
- Dashboard: http://localhost:8501

### Option 2: Docker Deployment

**Build Docker image:**
```bash
docker build -t medrag:latest .
```

**Run container:**
```bash
docker run -d \
  -p 8000:8000 \
  -p 8501:8501 \
  -e OPENROUTER_API_KEY=$OPENROUTER_API_KEY \
  -e SERPER_API_KEY=$SERPER_API_KEY \
  -e QDRANT_API_KEY=$QDRANT_API_KEY \
  medrag:latest
```

### Option 3: Cloud Deployment (Render)

**1. Push to GitHub:**
```bash
git remote add origin https://github.com/YOUR_USERNAME/medrag.git
git push -u origin main
```

**2. Connect to Render:**
- Go to https://render.com
- Click "New +"
- Select "Web Service"
- Connect GitHub repo
- Use `render.yaml` configuration

**3. Environment Variables (Render Dashboard):**
- Add OPENROUTER_API_KEY
- Add SERPER_API_KEY
- Add QDRANT_API_KEY

**4. Deploy:**
- Click "Create Web Service"
- Wait for deployment (~5 minutes)
- Access via Render URL

---

## Configuration

### Backend Config (`config/agents.yaml`)

```yaml
# Agent Settings
agents:
  fusion:
    model: "claude-3-opus-20240229"
    temperature: 0.7
    max_tokens: 1500
    top_p: 0.95
    timeout_seconds: 30
    
  evidence_scanner:
    enabled: true
    activation_threshold: 0.65  # Trigger when confidence < this
    max_searches: 3
    timeout_seconds: 5
    
  optimizer:
    max_iterations: 3
    convergence_threshold: 0.80
    temperature: 0.5  # Lower for stability
    
  evaluator:
    scoring_model: "claude-3-sonnet"
    rubric_version: "v2.1"

# RAG Settings
rag:
  retriever:
    type: "qdrant"
    collection: "medical_knowledge"
    top_k: 5
    similarity_threshold: 0.60
    rerank: true
    
  embeddings:
    model: "all-MiniLM-L6-v2"
    cache: true
    
  evidence_search:
    enabled: true
    provider: "serper"
    timeout_seconds: 5
    result_limit: 5

# System Settings
system:
  log_level: "INFO"
  cache_ttl_seconds: 3600
  max_workers: 4
```

### Frontend Config (`ui/config.toml`)

```toml
[theme]
primaryColor = "#0066CC"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F5F5F5"
textColor = "#262730"
font = "sans serif"

[client]
toolbarMode = "minimal"
showSidebarNavigation = true

[logger]
level = "info"
```

---

## Troubleshooting

### API Connection Issues

**Error: "Failed to connect to OpenRouter"**
```bash
# Check API key
echo $OPENROUTER_API_KEY

# Test connectivity
curl -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  https://openrouter.ai/api/v1/models

# Alternative: Use Claude API directly (faster)
export OPENAI_API_KEY=sk-...
# Update agents.yaml: model: gpt-4
```

**Error: "Qdrant connection refused"**
```bash
# Check if Qdrant is running
curl http://localhost:6333/health

# If using cloud:
export QDRANT_URL=https://xxxxx.qdrant.io
export QDRANT_API_KEY=xxx
```

**Error: "Serper API rate limit exceeded"**
```bash
# Check quota: https://serper.dev/dashboard
# Upgrade plan or reduce searches
# Fallback: Use web scraping (slower alternative)
```

### Performance Issues

**High latency (>15s per case)**
```bash
# 1. Check LLM model response time
# Backend logs should show latency per stage

# 2. Reduce top_k for retrieval
# config/agents.yaml: top_k: 3 (was 5)

# 3. Disable evidence scanner
# config/agents.yaml: evidence_scanner.enabled: false

# 4. Use faster model
# model: "claude-3-sonnet" (instead of opus)
```

**Out of memory**
```bash
# Reduce batch size
python scripts/evaluate_batch.py --batch_size 5

# Use smaller embedding model
# all-MiniLM-L6-v2 → all-MiniLM-L6-v2 (already small!)
```

### Dashboard Issues

**"Port 8501 already in use"**
```bash
# Find and kill process
lsof -i :8501
kill -9 <PID>

# Or use different port
streamlit run ui/dashboard.py --server.port 8502
```

**"Module not found: app"**
```bash
# Add project to Python path
export PYTHONPATH="${PYTHONPATH}:$(pwd)"
streamlit run ui/dashboard.py
```

**Graphs not rendering**
```bash
# Clear Streamlit cache
rm -rf ~/.streamlit/
streamlit run ui/dashboard.py
```

---

## Testing

### Unit Tests
```bash
# Run all tests
pytest tests/ -v

# Specific test file
pytest tests/test_agents.py -v

# With coverage
pytest tests/ --cov=app --cov-report=html
```

### Integration Tests
```bash
# Full pipeline test
pytest tests/test_integration.py -v -s

# Single case test
python -m pytest tests/test_integration.py::test_single_question -v
```

### Manual Testing

**Test single question:**
```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{
    "question": "A 45-year-old male with DM2, HTN presents with acute substernal chest pain...",
    "options": ["A) Unstable angina", "B) Pulmonary embolism", "C) Aortic dissection", "D) Esophageal spasm"],
    "ground_truth": "A"
  }'
```

**Test batch evaluation:**
```bash
curl -X POST http://localhost:8000/batch_evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "dataset_path": "data/medqa_eval_50.jsonl",
    "output_dir": "results/",
    "parallel": false
  }'
```

---

## Production Deployment Checklist

- [ ] All API keys configured & tested
- [ ] Vector database (Qdrant) deployed & accessible
- [ ] SSL/TLS certificates installed (HTTPS only)
- [ ] Rate limiting configured
- [ ] Monitoring (Grafana/DataDog) set up
- [ ] Logs aggregated (CloudWatch/Splunk)
- [ ] Backup strategy for vector database
- [ ] HIPAA compliance verified (if medical use)
- [ ] Load testing completed (target: 10 req/s)
- [ ] Incident response plan documented
- [ ] Documentation deployed (README visible)
- [ ] Team trained on system
- [ ] A/B testing framework ready
- [ ] Metrics dashboard live
- [ ] Automated testing in CI/CD

---

## Monitoring & Maintenance

### Health Check
```bash
curl http://localhost:8000/health
```

Response:
```json
{
  "status": "healthy",
  "components": {
    "qdrant": "connected",
    "serper": "connected",
    "llm_backend": "available"
  }
}
```

### View Logs
```bash
# Backend
tail -f logs/backend.log

# Frontend
streamlit run ui/dashboard.py  # Logs in terminal
```

### Monitor Metrics
```bash
curl http://localhost:8000/metrics
```

### Update Models
```bash
# Update embedding model
python scripts/update_embeddings.py --model all-mpnet-base-v2

# Rebuild Qdrant index
python scripts/rebuild_index.py --source data/new_knowledge_base.jsonl
```

---

## Support & Resources

- **Documentation:** See [README.md](README.md) and [ARCHITECTURE.md](ARCHITECTURE.md)
- **Issues:** GitHub Issues
- **Discussions:** GitHub Discussions
- **Contact:** rika1089@github.com

---

**Last Updated:** January 2025  
**Version:** 1.0.0-beta
