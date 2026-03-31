# 🔐 AI-Powered KYC Verification System

> **Full-stack identity verification platform for the BFSI sector** — classifies documents, extracts fields via OCR, and detects fraud using Graph Neural Networks. Deployed on AWS (S3 + EC2) with a React frontend, Node.js backend, and a Python Flask ML service.

<br/>

## 📋 Table of Contents

- [Overview](#overview)
- [Live Demo Architecture](#live-demo-architecture)
- [Tech Stack](#tech-stack)
- [ML Pipeline](#ml-pipeline)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Deployment](#deployment)
- [API Reference](#api-reference)
- [Screenshots](#screenshots)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Known Limitations](#known-limitations)
- [Team](#team)

<br/>

---

## Overview

This system automates KYC (Know Your Customer) document verification — a process typically done manually by BFSI institutions. A user uploads an identity document and within seconds receives a verification decision backed by ML inference.

**Supported documents:** Aadhaar Card · PAN Card · Passport

**What it does end-to-end:**

1. Classifies the document type using a TFLite CNN model
2. Extracts text fields (name, DOB, document number) using EasyOCR
3. Structures the raw OCR text into clean JSON using the Groq LLM API
4. Computes an anomaly score via a GNN comparing the document against a database of real records
5. Returns a verdict — **Approved**, **Suspicious**, or **Non-KYC** — and stores the result in MongoDB Atlas
6. Displays results in a React dashboard with a 5-stage progress pipeline UI

<br/>

---

## Live Demo Architecture

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                        REQUEST FLOW                             │
 │                                                                 │
 │  [Browser]  ──────────▶  [AWS S3 Frontend]                      │
 │                 HTTPS         React (Vite)                      │
 │                                    │                            │
 │                    POST /api/kyc/verify (multipart)             │
 │                                    ▼                            │
 │                         [Backend EC2 :5000]                     │
 │                         Node.js + Express                       │
 │                         JWT Auth · Multer                       │
 │                                    │                            │
 │                    POST /api/ml/classify (form-data)            │
 │                                    ▼                            │
 │                         [ML Service EC2 :5001]                  │
 │                         Flask · TFLite · PyTorch GNN            │
 │                         EasyOCR · Groq API                      │
 │                                    │                            │
 │              { document_type, ocr_data, anomaly_score }         │
 │                                    ▼                            │
 │                         [MongoDB Atlas]                         │
 │                         Verification record saved               │
 │                                    │                            │
 │                                    ▼                            │
 │                    Response back to browser                     │
 └─────────────────────────────────────────────────────────────────┘
```

| Layer | Service | Hosted On |
|-------|---------|-----------|
| Frontend | React (Vite) | AWS S3 Static Hosting |
| Backend API | Node.js + Express | AWS EC2 (t3.micro) |
| ML Service | Python Flask | AWS EC2 (m7i-flex.large) |
| Database | MongoDB Atlas | Cloud (managed) |
| Process Manager | PM2 | Both EC2 instances |

<br/>

---

## Tech Stack

**Frontend**
- React 18 (Vite)
- React Router
- Deployed on AWS S3 with static website hosting

**Backend**
- Node.js + Express
- JWT authentication (jsonwebtoken + bcryptjs)
- Multer for multipart file upload handling
- Axios for ML service communication
- Mongoose + MongoDB Atlas

**ML Service**
- Python 3.12 + Flask
- TFLite (CNN document classifier)
- PyTorch + PyTorch Geometric (GNN anomaly detection)
- EasyOCR (text extraction)
- Sentence Transformers — `all-MiniLM-L6-v2` (embedding)
- Groq API / LLaMA (structured field parsing from raw OCR)
- Scikit-learn (scaling), NetworkX (graph construction)

**Infrastructure**
- AWS EC2 (x2), AWS S3
- PM2 (process persistence)
- MongoDB Atlas

<br/>

---

## ML Pipeline

Each document upload goes through five sequential stages:

### Stage 1 — Document Classification
A quantised TFLite CNN model classifies the image as one of:
`Aadhaar Card` · `PAN Card` · `Passport` · `Non-KYC Document`

> **Heuristic fallback:** The TFLite model has a known bias misclassifying some PAN cards as Non-KYC. When this happens, the system runs an OCR-based override: regex `[A-Z]{5}[0-9]{4}[A-Z]` detects PAN numbers directly; keyword matching catches `INCOME TAX DEPARTMENT`, `AADHAAR`, `PASSPORT`. If OCR returns fewer than 5 characters (blurry/low-res image), the document is still handled gracefully.

### Stage 2 — OCR Extraction
EasyOCR runs on every image regardless of classification result, extracting raw text. This ensures the heuristic fallback always has text to work with.

### Stage 3 — Structured Parsing (Groq LLM)
Raw OCR text is sent to the Groq API with a document-specific prompt. The LLM returns structured JSON:
```json
{
  "Full Name": "...",
  "Date of Birth": "...",
  "Aadhaar Number": "..."
}
```
If the LLM call fails or returns invalid JSON, the system falls back to `{ "raw_text": "" }` — no crash.

### Stage 4 — GNN Anomaly Detection
Three independent PyTorch GNN models (one per document type) are loaded at startup. The extracted fields are embedded via SentenceTransformer and compared against a pre-computed database of real records:

- **Aadhaar** — 414 records
- **PAN Card** — 536 records
- **Passport** — 200 records

The top-5 similarity scores are computed. An anomaly score is derived:
- Score **> 2.0** → `Suspicious`
- Score **≤ 2.0** → `Approved`
- Non-KYC document → `Non-KYC` (no GNN inference)

### Stage 5 — Response
The ML service returns a flat JSON object (all fields guaranteed non-null):
```json
{
  "document_type": "Aadhaar Card",
  "confidence": 97.3,
  "ocr_data": { "Full Name": "...", "DOB": "..." },
  "anomaly_score": 1.82,
  "similar_records": [{ "similarity": 0.91 }],
  "fraud_detection": { "status": "Approved" }
}
```

<br/>

---

## Project Structure

```
project-root/
├── frontend/                   # React (Vite)
│   ├── src/
│   │   ├── components/
│   │   │   ├── Upload.jsx       # File upload + 5-stage progress bar
│   │   │   ├── VerificationResult.jsx
│   │   │   ├── DashboardLayout.jsx
│   │   │   ├── Login.jsx
│   │   │   ├── Register.jsx
│   │   │   └── ChatHistory.jsx
│   │   └── services/
│   │       └── api.js           # All backend API calls
│   ├── .env                     # VITE_API_BASE
│   └── vite.config.js
│
├── backend/                     # Node.js + Express
│   ├── controllers/
│   │   ├── kyc.js               # Verification logic + ML orchestration
│   │   └── auth.js
│   ├── routes/
│   │   ├── kyc.js
│   │   └── auth.js
│   ├── models/
│   │   ├── Verification.js      # Mongoose schema
│   │   └── User.js
│   ├── middleware/
│   │   └── auth.js              # JWT protect middleware
│   ├── .env                     # MONGODB_URI, JWT_SECRET, ML_SERVICE_URL
│   └── server.js
│
├── ml-service/                  # Flask ML service
│   ├── app.py                   # All 5 pipeline stages
│   ├── requirements.txt
│   └── .env                     # GROQ_API_KEY
│
└── trained_models/              # Pre-trained model artifacts
    ├── aadhaar_gnn_model.pth
    ├── aadhaar_embeddings.pt
    ├── aadhaar_scaler.pkl
    ├── aadhaar_records.pkl
    ├── pan_gnn_model.pth
    ├── pan_embeddings.pt
    ├── pan_scaler.pkl
    ├── pan_records.pkl
    ├── passport_gnn_model.pth
    ├── passport_embeddings.pt
    ├── passport_scaler.pkl
    ├── passport_records.pkl
    └── kyc_classifier.tflite
```

> ⚠️ `trained_models/` is not committed to the repo. Download separately and place at project root before deploying.

<br/>

---

## Getting Started

### Prerequisites

- Node.js v18+
- Python 3.12
- MongoDB Atlas cluster (free tier works)
- Groq API key — [console.groq.com](https://console.groq.com)
- AWS account (for EC2 + S3 deployment)

---

### Local Development

**1. Clone the repo**
```bash
git clone https://github.com/SannidhiSriram-06/AI-Powered-Identity-Verification-and-Fraud-Detection-for-KYC.git
cd AI-Powered-Identity-Verification-and-Fraud-Detection-for-KYC
```

**2. Start the ML service**
```bash
cd ml-service
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt   # ~10–20 mins first time
# Create ml-service/.env (see Environment Variables section)
python app.py
# Expected: Running on http://0.0.0.0:5001
```

**3. Start the backend**
```bash
cd backend
npm install
# Create backend/.env (see Environment Variables section)
node server.js
# Expected: Server running on port 5000 | MongoDB Connected
```

**4. Start the frontend**
```bash
cd frontend
npm install
# Create frontend/.env (see Environment Variables section)
npm run dev
# Open: http://localhost:5173
```

> Start services in this order: **ML → Backend → Frontend**

<br/>

---

## Environment Variables

### `backend/.env`
```env
PORT=5000
MONGODB_URI=mongodb+srv://<user>:<password>@<cluster>.mongodb.net/?appName=<app>
ML_SERVICE_URL=http://localhost:5001
JWT_SECRET=<your_random_secret_string>
NODE_ENV=development
```

### `frontend/.env`
```env
VITE_API_BASE=http://localhost:5000/api
```

### `ml-service/.env`
```env
GROQ_API_KEY=<your_groq_api_key>
```

> For production, replace `localhost` URLs with your EC2 public/private IPs.

<br/>

---

## Deployment

Full deployment runs on two EC2 instances + one S3 bucket.

### Infrastructure Overview

| Component | Instance Type | Port | Notes |
|-----------|--------------|------|-------|
| Backend EC2 | t3.micro | 5000 | Node.js + PM2 |
| ML EC2 | m7i-flex.large | 5001 | Flask + PM2 + venv |
| S3 Bucket | — | — | Static frontend hosting |

---

### Backend EC2

```bash
# SSH in
ssh -i ~/path/to/key.pem ubuntu@<BACKEND_EC2_IP>

# Install Node.js
sudo apt update && sudo apt install nodejs npm -y

# Upload and install
# (run from local machine)
scp -i ~/key.pem -r ./backend ubuntu@<BACKEND_EC2_IP>:~

# On EC2
cd ~/backend
npm install
# Create .env with production values
nano .env

# Start with PM2
sudo npm install -g pm2
pm2 start server.js --name backend
pm2 save && pm2 startup
```

---

### ML Service EC2

```bash
# SSH in
ssh -i ~/path/to/key.pem ubuntu@<ML_EC2_IP>

# System deps
sudo apt update && sudo apt install -y python3-pip python3-venv libgl1

# Upload files (from local machine)
scp -i ~/key.pem -r ./ml-service ubuntu@<ML_EC2_IP>:~
scp -i ~/key.pem -r ./trained_models ubuntu@<ML_EC2_IP>:~
scp -i ~/key.pem -r ./extracted_data_records ubuntu@<ML_EC2_IP>:~

# On EC2: set up venv
cd ~/ml-service
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
# Create .env with GROQ_API_KEY
nano .env

# Start with PM2 (must use venv interpreter explicitly)
sudo npm install -g pm2
pm2 start app.py --name ml-service \
    --interpreter /home/ubuntu/ml-service/venv/bin/python
pm2 save && pm2 startup
```

> ⚠️ **Important:** Always use the explicit venv interpreter path with PM2. Using `python3` directly will launch with system Python which has no ML dependencies installed.

---

### Frontend (S3)

```bash
# Update frontend/.env with production backend IP
echo "VITE_API_BASE=http://<BACKEND_EC2_IP>:5000/api" > frontend/.env

# Build
cd frontend && npm run build
```

Then in AWS Console:
1. Create S3 bucket → uncheck **Block all public access**
2. **Properties** → Static website hosting → Enable → `index.html` for both index and error document
3. **Permissions** → Bucket Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
  }]
}
```
4. Upload all contents of `dist/` to the bucket root
5. Add the S3 URL to backend CORS config and `pm2 restart backend`

---

### AWS Security Groups

| Instance | Port | Source |
|----------|------|--------|
| Backend EC2 | 22 (SSH) | Your IP only |
| Backend EC2 | 80 (HTTP) | Anywhere |
| Backend EC2 | 5000 (API) | Anywhere |
| ML EC2 | 22 (SSH) | Your IP only |
| ML EC2 | 5001 (ML API) | Anywhere |

<br/>

---

## API Reference

### Auth

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/register` | None | Register new user |
| POST | `/api/auth/login` | None | Login, returns JWT |

**Register / Login body:**
```json
{ "name": "...", "email": "...", "password": "..." }
```

---

### KYC Verification

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/kyc/verify` | JWT | Upload document for verification |
| GET | `/api/kyc/history` | JWT | Get user's verification history |
| GET | `/api/kyc/verifications` | JWT | Get all verifications (admin) |
| GET | `/api/kyc/verifications/stats` | JWT | Get summary statistics |
| POST | `/api/kyc/verifications/manual-decision` | JWT | Save Approve/Reject decision |

**POST `/api/kyc/verify`** — multipart/form-data:
```
identity: <file>       (required — identity document image)
supporting: <file>     (optional — supporting documents)
```

**Response:**
```json
{
  "success": true,
  "data": {
    "_id": "...",
    "document_type": "Aadhaar Card",
    "status": "Approved",
    "anomaly_score": 1.82,
    "extracted_data": { "Full Name": "...", "DOB": "..." },
    "similar_records": [...],
    "submitted_date": "2026-03-27T..."
  }
}
```

---

### ML Service (internal)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/ml/health` | Health check — confirms models loaded |
| POST | `/api/ml/classify` | Full pipeline: classify → OCR → parse → GNN |

**POST `/api/ml/classify`** — multipart/form-data:
```
file: <image>
```

<br/>

---

## Screenshots

> _Dashboard, verification result, and progress bar UI — see `/docs/screenshots/` if included in repo._

| Screen | Description |
|--------|-------------|
| Landing Page | Public-facing entry with login/register |
| Upload Page | Drag-and-drop document upload with 5-stage progress bar |
| Verification Result | Extracted fields, anomaly score, fraud status, similar records |
| Dashboard | Verification history, stats, manual decision controls |

<br/>

---

## Key Engineering Decisions

### Why EC2 over Lambda / SageMaker?

| Option | Reason Rejected |
|--------|----------------|
| AWS Lambda | 512MB memory limit — can't load TFLite + PyTorch + EasyOCR + Transformers simultaneously. 15-min timeout also incompatible with model initialisation. |
| AWS SageMaker | ~$0.096/hr minimum per inference endpoint — too expensive for internship project scope. Significant setup overhead. |
| Azure ML | Adds Azure vendor lock-in to an otherwise AWS-first stack. Per-token LLM costs add up fast. |
| **EC2 (chosen)** | Models loaded once at startup, stay in memory. No cold starts. Full Python env control. Cost-predictable. |

### Why Two Separate EC2 Instances?

The ML service runs heavy dependencies (PyTorch, TFLite, Transformers, EasyOCR) that alone require 2–4GB RAM and significant CPU. Running both Node.js and Flask on a single instance would make a t3.micro unstable. Separating them gives:
- Independent scaling (can upgrade ML EC2 without touching backend)
- Isolated failure domains — if ML crashes, backend still serves auth/history
- Cleaner microservice architecture

### Heuristic Fallback for PAN Classification

The TFLite model was trained on a relatively small dataset and shows strong bias, often classifying PAN cards as Non-KYC at 99%+ confidence. Rather than retraining, a two-layer heuristic override was added:
1. Regex `[A-Z]{5}[0-9]{4}[A-Z]` on OCR text detects PAN numbers directly
2. Keyword detection catches `INCOME TAX DEPARTMENT` in extracted text
3. If OCR returns fewer than 5 characters (blurry image), system defaults to PAN and processes gracefully

This keeps the system functional for real-world image quality variation without requiring model retraining.

<br/>

---

## Known Limitations

- **PAN OCR quality** — EasyOCR struggles with very low-resolution or heavily compressed PAN card images. The heuristic fallback handles these gracefully but extracted fields may be incomplete.
- **CPU-only inference** — All ML runs on CPU (no GPU on EC2). First request after model load is slower (~2–5s). Subsequent requests are faster due to warm models.
- **No file cleanup** — Uploaded files in `backend/uploads/` are not automatically purged. In production, add a cleanup cron or use S3 for temporary file storage.
- **Demo auth** — If no JWT is provided, the system falls back to `user_id: 'demo-user'`. This is intentional for demo purposes — production deployments should enforce auth on all routes.
- **Single-region** — Currently deployed in a single AWS region. No cross-region redundancy or auto-scaling.

<br/>

---

## 👥 Contributors

| Contributor | Role | Focus Area |
|-------------|------|------------|
| [Gireesh Kumar Gowd](https://github.com/Gireesh-Kumar-Gowd) | AI & ML Lead | GNN models, document classification, fraud detection pipeline |
| [Sriram Sannidhi](https://github.com/SannidhiSriram-06) | Cloud & Deployment Lead | AWS EC2/S3 deployment, backend integration, ML service orchestration |
| [Ganesh Jalkote](https://github.com/ganeshjalkote932) | AI & ML | Model training, dataset processing |
| [Navya Kedhari](https://github.com/Navyakedhari) | Web Development | Frontend React UI, component design |
| [Ankush](https://github.com/ankushsans) | Web Development | Backend Node.js, API routes |
| [Sai Pragna](https://github.com/SaiPragnaK) | Web Development | Frontend development, UI/UX |
| [Akhila](https://github.com/akhila607) | Web Development | Frontend development |

**Organisation:** Infosys Springboard — BFSI Sector Cloud Architecture Cohort

**Certifications (Deployment Lead):**
Microsoft AZ-900 · AI-900 · Oracle OCI DevOps Professional · OCI Data Science Professional · OCI Generative AI Professional · OCI Observability Professional · OCI Multicloud Architect Professional

<br/>

---

## License

This project was developed as part of the Infosys Springboard internship program. Not licensed for commercial use.

---

<div align="center">
  <sub>Built with Node.js · Flask · PyTorch · AWS · MongoDB Atlas</sub>
</div>
