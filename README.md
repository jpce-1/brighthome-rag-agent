# BrightHome RAG Agent

A beginner-friendly **RAG (Retrieval-Augmented Generation)** customer support agent, built in **n8n**.

It answers questions about a fictional company, **BrightHome Store** (an online home-appliance shop in Lagos), using only the information found in documents stored in Google Drive. If the answer isn't in the documents, it says so instead of guessing.

> **Think of RAG as an open-book exam for AI.** Instead of answering from memory, the agent first looks up the relevant notes in your documents, then answers using them.

Write-up: _[https://medium.com/@jpchie7/i-built-an-ai-that-answers-from-your-own-documents-and-the-first-thing-that-broke-was-not-the-ai-bf2b5e3062d8]_

## Demo video

Watch the demo on X(https://x.com/Johnpaul_JPCE/status/2123155306591277452?a=V6AGngL09vjaZAXNVSg&s=20)

---

## How it works

Two flows live in one n8n workflow.

**1. Load the knowledge (run once, or when documents change)**

```
Google Drive docs -> Download file -> Text splitter (chunks) -> Gemini embeddings -> Pinecone
```

**2. Answer a question (runs on every chat message)**

```
Chat message -> AI Agent -> searches Pinecone for the closest chunks -> Gemini chat model writes a grounded answer
```

The agent also has simple memory, so follow-up questions work.

## Tech stack

| Tool | What it does here |
|---|---|
| n8n (Cloud) | Connects everything visually |
| Google Drive | Holds the 5 company documents |
| Gemini (`gemini-embedding-001`) | Turns text into embeddings (3072 dimensions) |
| Gemini chat model | Writes the final answers |
| Pinecone | Stores and searches the embeddings (index: `class-rag`, cosine) |

## Repo structure

```
rag-agent/
├── README.md
├── .gitignore
├── workflow/rag-agent.json        (exported n8n workflow)
├── docs/                          (the 5 BrightHome documents)
├── prompts/system-prompt.md       (the agent's instructions)
├── tests/test-questions.md        (test questions, expected answers, results)
```

## Setup

1. **Create the documents.** Make a Google Drive folder named `RAG-Class` and add the 5 documents from `docs/` as Google Docs.
2. **Get a Gemini API key** from Google AI Studio.
3. **Choose the embedding model first.** Use `gemini-embedding-001(this is the one i used)`, which produces 3072-number embeddings. Your Pinecone index must match this exactly.
4. **Create the Pinecone index** (see the next section).
5. **Import the workflow.** In n8n: new workflow, then the three-dot menu, then Import from file, and pick `workflow/rag-agent.json`.
6. **Reconnect credentials.** n8n does not export secrets, so re-add Gemini, Pinecone and Google Drive credentials and select them in each node.
7. **Load the knowledge.** Run the top flow once. Do not run it twice, or Pinecone will hold duplicate chunks.
8. **Paste the system prompt** from `prompts/system-prompt.md` into the AI Agent node (Add Option, then System Message).
9. **Chat.** Open the chat and ask questions from `tests/test-questions.md`.

### Pinecone index: the one-time HTTP method

The Pinecone dashboard only offered a document-style index, which n8n's Pinecone Vector Store node could not write to. The fix was to create a compatible dense-vector index with a single, temporary **HTTP Request** node in n8n:

- Method: `POST`
- URL: `https://api.pinecone.io/indexes`
- Authentication: Generic Credential Type, then Header Auth (header name `Api-Key`, value is your Pinecone key, stored in n8n Credentials only)
- Headers: `Content-Type: application/json` and `X-Pinecone-API-Version: 2025-10`
- Body:

```json
{
  "name": "class-rag",
  "dimension": 3072,
  "metric": "cosine",
  "vector_type": "dense",
  "spec": {
    "serverless": {
      "cloud": "aws",
      "region": "us-east-1"
    }
  },
  "deletion_protection": "disabled"
}
```

Run it **once**, wait until the index shows Ready in Pinecone, then disable or delete the node. It is a setup tool, not part of the normal workflow.

**The one rule:** the index dimension must match the embedding model's output. The same embedding model must be used to store documents and to search them.

## Testing

All test questions, expected answers and results are in [`tests/test-questions.md`](tests/test-questions.md). They cover simple lookups, multi-document questions, math across documents, a question the documents cannot answer, and a prompt-injection attempt.

### A finding worth noting: retrieval depth

I asked: *"I complained 3 days ago and nobody has replied. What should I do?"*

- **First attempt:** a sensible answer, but it missed that serious complaints are escalated to a manager within 48 hours.
- **Diagnosis:** asking *"Is there an escalation process for serious complaints?"* returned the right answer, so the information was in Pinecone. The agent was retrieving too few chunks.
- **Fix:** I increased the number of chunks the agent retrieves per search(it was raised to 5).
- **Result:** the complaint question now includes the 48-hour manager escalation.

I also noticed the agent once added details that are not in the source documents ("automatically", "if the issue is not resolved"). RAG reduces made-up answers, but it does not remove them, so answers still need checking against the source.

## Lessons learned

1. RAG is an open-book exam: the AI isn't smarter, it just gets to look things up.
2. Set up the storage carefully. The Pinecone index took more effort than the AI.
3. Embedding model and index dimension must match.
4. Always test with questions the documents cannot answer.
5. When an answer is incomplete, check retrieval first.
6. Record your tests: a bug and its fix tell a better story than a perfect run.

## Security notes

- No API keys, tokens or credentials are in this repo. n8n credentials are not part of the exported workflow.
- Before sharing any workflow export, search it for stray keys, email addresses, Drive IDs and pinned test data.
- All documents are fictional. Do not upload client secrets or private records to a practice knowledge base.

## Roadmap

- Run the harder test set and record results (math across documents, missing-information questions, prompt injection).
- Tighten the system prompt based on failures.
- Try a different chunk size and compare answer quality.
- Rebuild the same agent for a second fictional business.

---

Built by Johnpaul Ehiemere as part of an AI automation internship practical.