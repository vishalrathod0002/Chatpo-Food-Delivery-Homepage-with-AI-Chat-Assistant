# Chatpo 🍽️

A single-page food delivery homepage (Zomato-style) with a live AI chat assistant powered by **n8n**, **Mistral**, **Pinecone (RAG)**, and a webhook-based chat widget.

**Live demo Like**
<img width="1897" height="911" alt="image" src="https://github.com/user-attachments/assets/febad4af-b1a4-4a19-8d84-71c4136ffa0f" />

---

## ✨ Features

- Responsive single-page homepage built with **Bootstrap 5** + custom CSS
- Restaurant listing cards, cuisine browsing, search bar, hero section
- Floating AI chat assistant (bottom-right bubble) that can:
  - Track orders
  - Handle refund requests
  - Recommend restaurants / cuisines
  - Escalate to human support
- Chat assistant is powered by an **n8n AI Agent** workflow with:
  - **Mistral** as the chat model
  - **Simple Memory** for session context
  - **HTTP Request tool** connected to the Chatpo backend API
  - **Pinecone vector store (RAG)** so answers about policies/FAQs are grounded in real documents, not guesses

---

## 📁 Project structure

```
chatpo-home.html   → the main homepage + embedded chat widget
README.md          → this file
```

This is a static site — no build step, no server-side code required for the frontend.

---

## 🚀 Deployment

### Quick deploy (Netlify Drop)
1. Go to https://app.netlify.com/drop
2. Drag `chatpo-home.html` into the browser window
3. Netlify gives you a live URL instantly (e.g. `https://your-site.netlify.app`)

### Other static hosts
Works the same on **Vercel**, **GitHub Pages**, **Cloudflare Pages**, or any static file host — there's nothing to build, just upload/serve the HTML file.

### GitHub Pages (if hosting from this repo)
1. Push this repo to GitHub
2. Repo **Settings → Pages** → set source branch to `main` (or `gh-pages`) and root folder
3. Rename `chatpo-home.html` to `index.html` if you want it served at the root URL
4. Your site will be live at `https://<username>.github.io/<repo-name>/`

---

## 🤖 Chat Assistant Setup (n8n)

The chat widget is powered by [`@n8n/chat`](https://www.npmjs.com/package/@n8n/chat), which connects directly to an n8n **Chat Trigger** webhook — no backend code needed on your end.

### n8n workflow structure
```
When chat message received (Chat Trigger)
        ↓
    AI Agent
        ├── Chat Model  → Mistral Cloud Chat Model
        ├── Memory      → Simple Memory
        └── Tools       ├── HTTP Request (Chatpo backend API)
                         └── Pinecone Vector Store (RAG retrieval tool)
```

### Retrieval-Augmented Generation (RAG) with Pinecone
The assistant uses a **Pinecone vector store** as a retrieval tool so it can answer questions grounded in Chatpo's actual policy/menu/FAQ documents (e.g. refund policy, delivery policy, restaurant details) instead of relying only on the model's general knowledge.

- Source documents (policies, FAQs, restaurant/menu data) are chunked and embedded, then upserted into a **Pinecone index**
- In n8n, a **Pinecone Vector Store** node is connected as a **Tool** on the AI Agent, alongside an embeddings model to encode incoming queries
- When a user asks something like *"what's your refund policy?"*, the Agent queries Pinecone for the most relevant chunks and uses them to ground its answer, rather than guessing

### Setting it up yourself
1. Create a workflow in n8n starting with a **Chat Trigger** node ("When chat message received")
2. Connect it to an **AI Agent** node
3. Attach a Chat Model (Mistral, Gemini, Groq, etc. — any provider with tool-calling support)
4. Attach **Simple Memory** so the assistant remembers context per session
5. Attach an **HTTP Request** tool node pointing to your backend API (order status, refunds, etc.)
6. Attach a **Pinecone Vector Store** tool node (with an embeddings model) for RAG-based answers over your policy/FAQ documents — see [Pinecone setup](#-pinecone-rag-setup) below
7. Inside the Chat Trigger node:
   - Toggle **"Make Chat Publicly Available"** ON
   - Copy the **Chat URL** shown there — this is your webhook URL
8. **Activate the workflow** using the toggle in the top-right of the editor (this is different from "Published" — the workflow must be Active for the public URL to work)

### 📚 Pinecone RAG setup
1. Create a free index at https://app.pinecone.io (dimension must match your embeddings model, e.g. 1536 for OpenAI `text-embedding-3-small`)
2. In n8n, add a **Pinecone** credential with your API key
3. Use a **Document Loader + Text Splitter** to chunk your source documents (e.g. `Chatpo_Policies.docx` content, FAQs, menu data)
4. Add an **Embeddings** node (OpenAI, Cohere, etc.) to convert chunks into vectors
5. Use the **Pinecone Vector Store** node in "Insert" mode to upsert the embedded chunks into your index (one-time or whenever documents change)
6. In your main chat workflow, add a **Pinecone Vector Store** node in "Retrieve as Tool" mode, connect it to the same embeddings model, and attach it to the AI Agent's **Tool** input
7. The Agent will now automatically query Pinecone whenever a user's question is best answered by retrieved documents (policies, FAQs, etc.)

### Wiring it into the HTML
In `chatpo-home.html`, find the `createChat()` call and set `webhookUrl` to your Chat URL:

```js
createChat({
  webhookUrl: 'https://YOUR-INSTANCE.app.n8n.cloud/webhook/YOUR-ID/chat',
  mode: 'window',
  showWelcomeScreen: true,
  initialMessages: [
    "Hi! I'm the Chatpo Assistant 👋",
    "I can help you track orders, request refunds, or find restaurants. What do you need?"
  ]
});
```

⚠️ **This URL can change** if you toggle "Make Chat Publicly Available" off/on or significantly edit the Chat Trigger node. If the assistant stops responding after making changes in n8n, check this first — re-copy the Chat URL from the node and update it here.

---

## 🧩 System prompt (AI Agent → System Message)

```
You are Chatpo Assistant, the AI support agent for Chatpo food delivery.

You help with order tracking, cancellations, refunds, and restaurant recommendations by calling Chatpo's HTTP API. Never guess order details — always fetch real data via the connected tools before answering.

Rules:
- Cancel free before restaurant accepts; after that, cancellation may not be possible.
- Refunds must be requested within 24 hours of delivery.
- If unsure or a tool call fails, say so and offer to escalate — never make up info.
- Keep replies short, friendly, and specific to the user's real order.
- Stay on topic: food, orders, delivery, refunds only.
```

---

## 🛠️ Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "Failed to receive response" | Webhook URL is stale/wrong | Re-copy the Chat URL from the Chat Trigger node and update `webhookUrl` in the HTML |
| `404 ... webhook is not registered` | Workflow is not **Active** | Toggle Active (top-right of n8n editor) — "Published" alone isn't enough |
| CORS error, `origin 'null'` | HTML opened directly from disk (`file://`) | Serve it from a real URL (Netlify, local server, GitHub Pages) |
| CORS error on a real domain | Origin not allowed by n8n | Add your domain in the Chat Trigger node's CORS/allowed origins field, or instance-level CORS settings |
| Model errors (503 / high demand) | Using a preview model (e.g. `gemini-3-flash-preview`) | Switch to a GA model (`gemini-2.0-flash`, stable Mistral models) or enable a fallback model |

---

## 📄 License

Demo project — for learning/prototyping purposes.
