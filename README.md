# KYC & AML Backend

This is the backend for the AI-Powered KYC & AML Compliance project.

## Technologies
- Node.js
- Express
- MongoDB (Mongoose)
- JWT (Authentication)
- Multer (File Uploads)

## Setup

1. **Install Dependencies**
   ```bash
   cd backend
   npm install
   ```

2. **Environment Variables**
   The `.env` file should contain:
   ```env
   PORT=5000
   MONGODB_URI=mongodb://localhost:27017/kyc_db
   JWT_SECRET=your_super_secret_key_123
   NODE_ENV=development
   ```

3. **Run the Server**
   ```bash
   npm run dev
   ```

## API Endpoints

### Authentication
- `POST /api/auth/register` - Create a new account
- `POST /api/auth/login` - Login and get token

### KYC Verification
- `POST /api/kyc/verify` - Upload documents for AI verification (Requires Auth)
- `GET /api/kyc/history` - Get history of verifications (Requires Auth)

## ML Integration
Currently, the verification logic in `controllers/kyc.js` is mocked to simulate ML processing. To integrate the real ML service, replace the mock logic with calls to your Python ML service (e.g., using `axios` to call a Gradio endpoint or a Flask/FastAPI service).
