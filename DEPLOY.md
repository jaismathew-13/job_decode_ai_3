# JobDecode AI - Cloud Deployment Guide

This guide provides step-by-step instructions to deploy JobDecode AI to various cloud platforms.

## Prerequisites

- **Docker** installed on your local machine
- **Cloud provider account** (Google Cloud, AWS, or Azure)
- **GROQ API Key** - Get one from [https://console.groq.com/](https://console.groq.com/)
- **Git** for version control

## Project Structure

After cleanup, your project should have:
```
job-decode-ai/
├── app.py              # FastAPI application
├── rag_chain.py        # RAG chain logic
├── ingest.py           # Ingestion utility
├── requirements.txt    # Python dependencies
├── Dockerfile          # Docker configuration
├── .dockerignore       # Docker ignore rules
├── Procfile            # Heroku/Render process file
├── .env.example        # Environment variables template
├── .gitignore          # Git ignore rules
├── templates/          # HTML templates
└── DEPLOY.md          # This file
```

## Step 1: Environment Setup

### 1.1 Create `.env` file
```bash
# Copy the example file
cp .env.example .env

# Edit .env and add your GROQ API key
GROQ_API_KEY=your_actual_groq_api_key_here
```

### 1.2 Test locally (Optional)
```bash
# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run the application
uvicorn app:app --host 0.0.0.0 --port 8000 --reload
```

Test the API at `http://localhost:8000/docs`

## Step 2: Docker Deployment

### 2.1 Build Docker Image
```bash
docker build -t jobdecode-ai .
```

### 2.2 Run Docker Container Locally
```bash
docker run -d -p 8080:8080 --env-file .env jobdecode-ai
```

Test at `http://localhost:8080/docs`

## Step 3: Cloud Deployment Options

### Option A: Google Cloud Run (Recommended)

#### 3A.1 Install Google Cloud SDK
```bash
# Download and install from: https://cloud.google.com/sdk/docs/install
```

#### 3A.2 Authenticate
```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
```

#### 3A.3 Build and Push to Google Artifact Registry
```bash
# Enable Cloud Build API
gcloud services enable cloudbuild.googleapis.com

# Submit build to Cloud Build
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/jobdecode-ai
```

#### 3A.4 Deploy to Cloud Run
```bash
gcloud run deploy jobdecode-ai \
  --image gcr.io/YOUR_PROJECT_ID/jobdecode-ai \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars GROQ_API_KEY=your_groq_api_key
```

Your service will be available at the URL shown in the output.

#### 3A.5 Set Environment Variables (If needed)
```bash
gcloud run services update jobdecode-ai \
  --region us-central1 \
  --set-env-vars GROQ_API_KEY=your_groq_api_key
```

---

### Option B: AWS App Runner

#### 3B.1 Push Docker Image to Amazon ECR
```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Create repository (if not exists)
aws ecr create-repository --repository-name jobdecode-ai

# Tag and push image
docker tag jobdecode-ai:latest YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/jobdecode-ai:latest
docker push YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/jobdecode-ai:latest
```

#### 3B.2 Deploy to App Runner
1. Go to AWS Console → App Runner
2. Click "Create service"
3. Select "Container image"
4. Choose your ECR image
5. Add environment variable: `GROQ_API_KEY`
6. Deploy

---

### Option C: Render (Simplest)

#### 3C.1 Push to GitHub
```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/job-decode-ai.git
git push -u origin main
```

#### 3C.2 Deploy to Render
1. Go to [https://render.com](https://render.com)
2. Click "New +" → "Web Service"
3. Connect your GitHub repository
4. Select "Docker" as runtime
5. Add environment variable: `GROQ_API_KEY`
6. Click "Deploy Web Service"

---

### Option D: Railway

#### 3D.1 Deploy via CLI
```bash
# Install Railway CLI
npm install -g @railway/cli

# Login
railway login

# Initialize project
railway init

# Add environment variable
railway variables set GROQ_API_KEY=your_groq_api_key

# Deploy
railway up
```

---

### Option E: Heroku

#### 3E.1 Install Heroku CLI
```bash
# Download from: https://devcenter.heroku.com/articles/heroku-cli
```

#### 3E.2 Login and Create App
```bash
heroku login
heroku create jobdecode-ai
```

#### 3E.3 Set Environment Variables
```bash
heroku config:set GROQ_API_KEY=your_groq_api_key
```

#### 3E.4 Deploy
```bash
heroku container:login
heroku container:push web -a jobdecode-ai
heroku container:release web -a jobdecode-ai
```

---

## Step 4: Verify Deployment

### 4.1 Health Check
```bash
curl https://your-service-url/health
```

Expected response:
```json
{"status": "ok", "service": "jobdecode-ai"}
```

### 4.2 Test API Endpoints
```bash
# Load a job description
curl -X POST https://your-service-url/load \
  -H "Content-Type: application/json" \
  -d '{"raw_text": "Senior Software Engineer position..."}'

# Ask a question
curl -X POST https://your-service-url/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What are the required skills?"}'
```

## Step 5: Monitoring and Logs

### Google Cloud Run
```bash
gcloud logs tail jobdecode-ai --region us-central1
```

### AWS App Runner
View logs in CloudWatch Logs via AWS Console

### Render
View logs in Render Dashboard

## Troubleshooting

### Issue: "ModuleNotFoundError"
**Solution:** Ensure `requirements.txt` includes all dependencies and Docker build completed successfully.

### Issue: "GROQ_API_KEY not found"
**Solution:** Set the environment variable in your cloud platform's configuration.

### Issue: FAISS index errors
**Solution:** The FAISS index is created at runtime. Ensure the container has write permissions to `/app/faiss_index`.

### Issue: CORS errors
**Solution:** The app allows all origins (`*`). If you need to restrict, modify `CORSMiddleware` in `app.py`.

## Cost Optimization

- **Google Cloud Run:** Pay only when requests are served (scale-to-zero)
- **AWS App Runner:** Automatic scaling, pay per use
- **Render:** Free tier available for small projects
- **Railway:** Free tier with $5 credit

## Security Best Practices

1. **Never commit `.env` file** to version control
2. **Use secret management** provided by your cloud platform
3. **Enable HTTPS** (most platforms provide this by default)
4. **Restrict CORS origins** in production if needed
5. **Monitor logs** for suspicious activity

## API Endpoints

- `GET /` - Health check
- `GET /health` - Service status
- `POST /load` - Load and index job description
- `POST /ask` - Ask questions about loaded job description
- `POST /reset` - Clear the index

## Support

For issues or questions:
- Check the logs in your cloud platform
- Review the API documentation at `/docs` endpoint
- Ensure all environment variables are set correctly
