# NestWay — Complete Deployment Guide

Deploy NestWay live in 4 stages. Takes about 30–45 minutes total.
All services used are **free tier**.

---

## Stage 1 — MongoDB Atlas (Database)

### 1.1 Create a free cluster
1. Go to https://mongodb.com/atlas and sign up (free)
2. Click **Build a Database** → choose **Free (M0 Sandbox)**
3. Pick any region (Mumbai ap-south-1 is fastest for India)
4. Cluster name: `nestway` → click **Create**

### 1.2 Create a database user
1. In the left sidebar → **Database Access** → **Add New Database User**
2. Username: `nestway-user`
3. Password: generate a strong one, **copy it somewhere safe**
4. Role: **Read and write to any database**
5. Click **Add User**

### 1.3 Whitelist all IPs (for Render)
1. Left sidebar → **Network Access** → **Add IP Address**
2. Click **Allow Access from Anywhere** (0.0.0.0/0)
3. Click **Confirm**

### 1.4 Get your connection string
1. Left sidebar → **Database** → **Connect** on your cluster
2. Choose **Connect your application**
3. Driver: Node.js, Version: 4.1 or later
4. Copy the string — it looks like:
   ```
   mongodb+srv://nestway-user:<password>@nestway.xxxxx.mongodb.net/?retryWrites=true&w=majority
   ```
5. Replace `<password>` with your actual password
6. Add the database name before `?`:
   ```
   mongodb+srv://nestway-user:yourpassword@nestway.xxxxx.mongodb.net/nestway?retryWrites=true&w=majority
   ```
7. **Save this string** — you'll need it in Stage 2

---

## Stage 2 — Deploy Backend to Render

### 2.1 Push backend to GitHub
```bash
cd nestway-backend

git init
git add .
git commit -m "Initial NestWay backend"

# Create a new repo on github.com named "nestway-backend"
# Then run:
git remote add origin https://github.com/YOUR_USERNAME/nestway-backend.git
git branch -M main
git push -u origin main
```

### 2.2 Create Render web service
1. Go to https://render.com and sign up with GitHub
2. Click **New** → **Web Service**
3. Connect your `nestway-backend` GitHub repo
4. Fill in the settings:
   - **Name**: nestway-backend
   - **Runtime**: Node
   - **Build Command**: `npm install`
   - **Start Command**: `npm start`
   - **Instance Type**: Free

### 2.3 Set environment variables on Render
Click **Environment** tab and add these one by one:

| Key | Value |
|-----|-------|
| `NODE_ENV` | `production` |
| `PORT` | `10000` |
| `MONGO_URI` | your Atlas connection string from Stage 1 |
| `ADMIN_SECRET` | pick a long random string e.g. `nw-secret-abc123xyz` |
| `ADMIN_PASSWORD` | pick a strong password e.g. `NestWay@Admin2024` |
| `USE_MOCK_DATA` | `false` |
| `OPENWEATHER_API_KEY` | your key (optional — falls back to mock weather) |

Click **Create Web Service** → wait 2–3 minutes for first deploy.

### 2.4 Test your backend
Once deployed, Render gives you a URL like:
`https://nestway-backend.onrender.com`

Test it:
```
https://nestway-backend.onrender.com/
```
You should see the JSON health check response.

### 2.5 Seed the database
```bash
# Get an admin token first:
curl -X POST https://nestway-backend.onrender.com/api/admin/token \
  -H "Content-Type: application/json" \
  -d '{"password": "NestWay@Admin2024"}'

# Copy the token, then seed:
curl -X POST https://nestway-backend.onrender.com/api/admin/seed \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

You should see: `"Seeded 1 new cities from mock data."`

---

## Stage 3 — Deploy Frontend to Vercel

### 3.1 Update the backend URL in vercel.json
Open `nestway-frontend/vercel.json` and replace the placeholder:
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/" }],
  "env": {
    "REACT_APP_API_URL": "https://nestway-backend.onrender.com"
  }
}
```
Replace `nestway-backend` with your actual Render subdomain.

### 3.2 Push frontend to GitHub
```bash
cd nestway-frontend

git init
git add .
git commit -m "Initial NestWay frontend"

# Create a new repo on github.com named "nestway-frontend"
git remote add origin https://github.com/YOUR_USERNAME/nestway-frontend.git
git branch -M main
git push -u origin main
```

### 3.3 Deploy on Vercel
1. Go to https://vercel.com and sign up with GitHub
2. Click **Add New Project**
3. Import your `nestway-frontend` repo
4. Framework Preset: **Create React App** (auto-detected)
5. Add environment variable:
   - Key: `REACT_APP_API_URL`
   - Value: `https://nestway-backend.onrender.com` (your Render URL)
6. Click **Deploy** → wait ~2 minutes

Vercel gives you a live URL like:
`https://nestway-frontend.vercel.app`

---

## Stage 4 — Access Admin Panel

### 4.1 Open the admin panel
```
https://nestway-frontend.vercel.app/admin
```

### 4.2 Login
Enter the `ADMIN_PASSWORD` you set on Render.

### 4.3 What you can do
- **Seed DB** — imports Chandigarh from mock data (do this first)
- **Publish/unpublish** cities
- **Add cities** — creates skeleton, fill details via API or mockData.js
- **Add PG listings** — add rent listings to any city
- **Delete cities**

---

## Stage 5 — Add More Cities

### Option A: Add to mockData.js, re-seed
1. Add the new city to `nestway-backend/src/data/mockData.js`
2. Push to GitHub (Render auto-redeploys)
3. Hit the seed endpoint again — it skips existing cities

### Option B: Use Admin Panel
1. Go to `/admin` → **Add city** tab
2. Creates a skeleton city
3. Update the full data via PUT API (use Postman or curl):
```bash
curl -X PUT https://nestway-backend.onrender.com/api/city/pune \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "weather": {...}, "travel": {...}, ... }'
```

---

## URLs Summary (after deployment)

| What | URL |
|------|-----|
| Frontend (live app) | `https://nestway-frontend.vercel.app` |
| Backend API | `https://nestway-backend.onrender.com` |
| Admin panel | `https://nestway-frontend.vercel.app/admin` |
| Compare page | `https://nestway-frontend.vercel.app/compare` |
| API health | `https://nestway-backend.onrender.com/` |
| City data | `https://nestway-backend.onrender.com/api/city/chandigarh?mode=student` |

---

## Common Issues

**Render backend is slow on first request**
Free tier sleeps after 15 mins of inactivity. First request takes ~30 seconds to wake up.
Fix: use https://uptimerobot.com to ping your backend URL every 10 minutes (free).

**CORS error in browser**
Make sure your backend's CORS allows your Vercel domain.
In `src/index.js`, update:
```js
app.use(cors({ origin: ['https://nestway-frontend.vercel.app', 'http://localhost:3000'] }));
```

**MongoDB connection timeout**
Double-check that you whitelisted `0.0.0.0/0` in Atlas Network Access (Stage 1.3).

**Admin login says "wrong password"**
Verify `ADMIN_PASSWORD` in Render environment variables matches what you're typing.
Environment variable changes need a manual redeploy on Render.

---

## Local development (always works without internet)

```bash
# Terminal 1
cd nestway-backend
cp .env.example .env   # Set USE_MOCK_DATA=true
npm install && npm run dev

# Terminal 2
cd nestway-frontend
cp .env.example .env   # REACT_APP_API_URL=http://localhost:5000
npm install && npm start
```

App runs at http://localhost:3000 — fully functional with mock data,
no MongoDB or API keys needed.
Completed 
