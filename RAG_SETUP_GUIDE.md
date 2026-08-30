# RAG Setup with Supabase & pgvector

## Prerequisites
- Google AI API Key (already configured)
- Supabase account (free tier available at supabase.com)
- Node.js 18+

## Step 1: Set Up Supabase Project

1. Go to https://supabase.com and sign in/create account
2. Create a new project
3. Wait for project initialization (2-5 mins)
4. Go to **Project Settings** → **API** and copy:
   - Project URL → `SUPABASE_URL`
   - anon key → `SUPABASE_KEY`
   - service_role key → `SUPABASE_SERVICE_KEY`

## Step 2: Run Database Migration

1. In Supabase, go to **SQL Editor**
2. Click **New Query**
3. Copy contents of `supabase_migration.sql`
4. Paste into the editor and run
5. Verify: You should see 2 new tables: `documents` and `document_chunks`

## Step 3: Update Environment Variables

Update `.env` in server directory:
```env
AI_API_KEY=your-google-ai-key
AI_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai/
EMBEDDING_MODEL=text-embedding-004
CHAT_MODEL=gemini-1.5-flash

# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-role-key
```

## Step 4: Install Dependencies

```bash
cd server
npm install
```

## Step 5: Integrate RAG Routes

Add this to your main `src/index.js` file (after other imports):

```javascript
import ragRouter from "./routes/rag.js";

// ... existing code ...

// Add before app.listen()
app.post("/rag/upload", upload.single("file"), auth, async (req, res) => {
  try {
    const ragResponse = await fetch("/rag/upload", {
      method: "POST",
      body: req.body,
    });
    res.json(await ragResponse.json());
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Mount RAG routes (all routes require auth)
app.use("/rag", auth, ragRouter);
```

## Step 6: Test the Setup

### Upload a Document
```bash
curl -X POST http://localhost:4000/rag/upload \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -F "file=@document.pdf" \
  -F "documentName=My Document"
```

### Query Documents (RAG)
```bash
curl -X POST http://localhost:4000/rag/query \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "What is in this document?"}'
```

### Delete a Document
```bash
curl -X DELETE http://localhost:4000/rag/documents/1 \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

## API Endpoints

### POST /rag/upload
Upload a PDF or text document for RAG
- **Required**: `file` (multipart), `documentName` (string)
- **Response**: `{ documentId, chunkCount, message }`

### POST /rag/query
Query uploaded documents using RAG
- **Required**: `query` (string, min 3 chars)
- **Response**: `{ answer, sources }`
- **Sources**: Array of chunks used to generate answer with similarity scores

### DELETE /rag/documents/:documentId
Delete a document and all its chunks
- **Response**: `{ success, message }`

## How It Works

1. **Upload**: Document → Extract Text → Split into Chunks → Generate Embeddings → Store in Supabase
2. **Query**: User Query → Generate Query Embedding → Vector Search → Retrieve Similar Chunks → Generate Answer with LLM
3. **Delete**: Remove document and all associated chunks from database

## Embedding Model
- Google's `embedding-001` (768 dimensions)
- Free tier supported by Google AI Studio
- Each embedding costs minimal tokens

## Vector Search
- Uses pgvector's cosine similarity in Supabase
- Configurable `match_threshold` (default 0.25)
- Returns top 5 most similar chunks by default

## Performance Tips
- Use PDF documents for better text extraction
- Keep chunk size around 500 characters for optimal embedding quality
- Create indexes (done in migration) for faster searches
- Batch insert chunks to handle large documents

## Troubleshooting

**pgvector not found**: Ensure pgvector extension is enabled in Supabase
**Slow searches**: Check indexes with: `EXPLAIN ANALYZE SELECT ...`
**Wrong embeddings**: Verify Google AI API key and embedding model name
**Auth errors**: Check JWT token validity and user exists in system
