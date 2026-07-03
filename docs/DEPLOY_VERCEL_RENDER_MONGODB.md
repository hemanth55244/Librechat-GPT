# Deploy LibreChat with Vercel, Render, and MongoDB Atlas

This setup keeps secrets out of GitHub:

- Vercel serves the built frontend from `client/dist`.
- Render runs the LibreChat API from `Dockerfile.multi`.
- Render runs `danny-avila/rag_api` for document retrieval and file RAG.
- MongoDB Atlas stores the database.
- Vercel rewrites `/api/*` and `/oauth/*` to the Render API so browser login cookies stay on the Vercel site.

## 1. Push this repo to GitHub

Use this repo:

```bash
git remote set-url origin https://github.com/hemanth55244/Librechat-GPT.git
git add .dockerignore .gitignore .env.production.example render.yaml vercel.json docs/DEPLOY_VERCEL_RENDER_MONGODB.md
git commit -m "Add Vercel Render MongoDB deployment config"
git push -u origin main
```

Do not commit `.env`, `docker-compose.override.yml`, API keys, passwords, or OAuth secrets.

## 2. Create MongoDB Atlas

1. Create a free MongoDB Atlas cluster.
2. Create a database user.
3. Allow network access from `0.0.0.0/0` for Render, or use stricter IPs after deployment.
4. Copy the connection string.
5. Use database name `LibreChat`, for example:

```text
mongodb+srv://USER:PASSWORD@CLUSTER.mongodb.net/LibreChat?retryWrites=true&w=majority
```

## 3. Deploy the backend on Render

1. Open Render.
2. New > Blueprint.
3. Connect `hemanth55244/Librechat-GPT`.
4. Render will read `render.yaml`.
5. Fill every `sync: false` environment variable in the Render dashboard.

Required values:

```text
MONGO_URI=your MongoDB Atlas connection string
JWT_SECRET=64 hex chars
JWT_REFRESH_SECRET=64 hex chars
CREDS_KEY=64 hex chars
CREDS_IV=32 hex chars
OPENAI_API_KEY=your provider key, only if you still use OpenAI
GROQ_API_KEY=your Groq key, if you want Groq models instead of OpenAI
GOOGLE_CLIENT_ID=your Google OAuth client id
GOOGLE_CLIENT_SECRET=your Google OAuth client secret
```

For the `librechat-rag-api` Render service, set:

```text
ATLAS_MONGO_DB_URI=the same MongoDB Atlas connection string
RAG_GOOGLE_API_KEY=your Google AI Studio / Gemini API key for embeddings
JWT_SECRET=the same JWT_SECRET used by librechat-gpt-api
```

The RAG API is not a chat model. It stores and retrieves document chunks for file/RAG workflows. You still need a chat model provider for normal answers.

The Render backend uses `CONFIG_PATH=/app/config/librechat.render.yaml`, which enables a Groq endpoint in LibreChat. Paste your real Groq key only into Render as `GROQ_API_KEY`; do not commit it.

Generate secrets locally with:

```bash
openssl rand -hex 32
openssl rand -hex 16
```

Your current Render backend URL is:

```text
https://librechat-gpt.onrender.com
```

If Render gives a different URL in a future deployment, update `DOMAIN_SERVER` in Render and update `vercel.json` rewrites before pushing again.

Your current RAG service URL is:

```text
https://librechat-rag-for-own.onrender.com
```

Set `RAG_API_URL` on the `librechat-gpt-api` Render service to that URL.

## 4. Enable MongoDB Atlas Vector Search for RAG

The RAG service is configured with:

```text
VECTOR_DB_TYPE=atlas-mongo
COLLECTION_NAME=rag_chunks
ATLAS_SEARCH_INDEX=vector_index
EMBEDDINGS_PROVIDER=google_genai
EMBEDDINGS_MODEL=gemini-embedding-001
```

In MongoDB Atlas, create a Vector Search index named:

```text
vector_index
```

Use it on the collection:

```text
rag_chunks
```

The exact index dimensions must match the embedding model you choose. If you change `EMBEDDINGS_MODEL`, recreate the Atlas vector index to match that model.

## 5. Deploy the frontend on Vercel

1. Open Vercel.
2. Add New Project.
3. Import `hemanth55244/Librechat-GPT`.
4. Keep root directory as the repo root.
5. Vercel will use `vercel.json`.
6. Deploy.

If Vercel gives a URL different from `https://librechat-gpt.vercel.app`, update `DOMAIN_CLIENT` in Render to the actual Vercel URL.

## 6. Configure Google OAuth

In Google Cloud > Auth Platform > Clients > your web client:

Authorized JavaScript origins:

```text
https://librechat-gpt.vercel.app
https://librechat-gpt.onrender.com
```

Authorized redirect URIs:

```text
https://librechat-gpt.vercel.app/oauth/google/callback
https://librechat-gpt.onrender.com/oauth/google/callback
```

If your Vercel or Render URLs are different, use your actual URLs.

## 7. Production notes

- Rotate any key that was pasted into chat, committed, shared, or shown on screen.
- Render free services sleep after inactivity, so the first request can be slow.
- This split now includes `rag_api`, but not Meilisearch or persistent upload storage. Durable file uploads should be added later with paid/private services or external storage.
- For the most stable LibreChat production deployment, a VPS with Docker Compose is still simpler than splitting frontend/backend across Vercel and Render.
