# Deploying Feastory on Vercel (with MongoDB Atlas)

Two Vercel projects from one GitHub repo:

| Vercel project | Root Directory | What it runs |
|---|---|---|
| `feastory-api` | `backend` | FastAPI as a Python serverless function |
| `feastory` | `frontend` | Vite/React static site |

Database stays on MongoDB Atlas. Dish photos go to Vercel Blob.

## 1. Atlas
- **Rotate the DB password** if the old `.env.example` (with the real password) was ever pushed or shared: Database Access → Edit user → Edit Password.
- Network Access → Add IP Address → **Allow access from anywhere** (`0.0.0.0/0`). Vercel has no fixed IPs.
- Copy the connection string (Connect → Drivers). URL-encode special characters in the password.

## 2. GitHub
```bash
cd feastoryv3
git init && git add . && git commit -m "Feastory for Vercel"
git branch -M main
git remote add origin https://github.com/<you>/feastory.git
git push -u origin main
```
Check that no `.env` file got committed (`git ls-files | grep .env` should show only `.env.example`).

## 3. Backend project
1. vercel.com → Add New → Project → import the repo.
2. Project name `feastory-api`, **Root Directory → `backend`**. Framework preset: leave on auto-detect (FastAPI / Other). No build command needed.
3. Environment Variables:
   | Name | Value |
   |---|---|
   | `MONGO_URL` | your Atlas string |
   | `MONGO_DB` | `feastory` |
   | `JWT_SECRET` | long random string (`python -c "import secrets;print(secrets.token_urlsafe(48))"`) |
   | `SEED_STAFF_PIN` | a new manager PIN (only used if no manager exists in the DB yet) |
4. Deploy. Open `https://feastory-api.vercel.app/api/health` → should show `"status":"ok","storage":"blob"`.

## 4. Blob store (dish photos)
1. In `feastory-api` → **Storage** tab → Create → **Blob** → access **Public** → Create & connect to the project (all environments).
2. Deployments → latest → ⋯ → **Redeploy** (so the new Blob variables are picked up).

## 5. Frontend project
1. Add New → Project → same repo again.
2. Project name `feastory`, **Root Directory → `frontend`**. Vite is auto-detected (build `npm run build`, output `dist`).
3. Environment Variable: `VITE_API_URL` = `https://feastory-api.vercel.app` (no trailing slash).
4. Deploy.

## 6. Connect them (CORS)
In `feastory-api` → Settings → Environment Variables → add
`CORS_ORIGINS` = `https://feastory.vercel.app` (comma-separate if you add a custom domain later) → Redeploy.

## 7. Test
- Register a student, add to cart, checkout, download the bill.
- `/canteenlogin` as the manager → move the order Pending → Ready → Served.
- Add a dish with a photo → it should appear on the menu with the image.

## Troubleshooting
| Symptom | Fix |
|---|---|
| Every API call fails, browser console says CORS | `CORS_ORIGINS` doesn't exactly match the frontend URL (https, no trailing slash) → fix and redeploy backend |
| Backend 500 on every request, logs mention `ServerSelectionTimeoutError` | Atlas IP allow-list not `0.0.0.0/0`, or wrong password in `MONGO_URL` |
| Logs: `Set the JWT_SECRET environment variable` | add `JWT_SECRET`, redeploy |
| "Image storage not set up" when adding a dish | Blob store not connected, or you didn't redeploy after connecting |
| Changing `VITE_API_URL` has no effect | it's baked in at build time → redeploy the frontend |
| First request after idle is slow (1–3 s) | normal serverless cold start (reconnects to Atlas) |
