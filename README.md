# MicrosoftChatbot

# AI CV Chatbot with Azure OpenAI  
Static website (GitHub Pages) + Azure Function backend

This guide reflects the **exact, working setup** discussed in this conversation, aligned with Azure’s current UI (2026), and mirrors the same architecture you previously used with AWS and Google Cloud.

---

## 1) Prerequisites (do once)

You need:

- A GitHub account  
- A Microsoft Azure account with an active subscription  
- A computer (Windows or macOS)

Prepare two texts (copy–paste later):

### CV Summary
A short, factual summary of:
- professional background  
- work experience  
- skills  
- projects and achievements  

### Assistant Rules (System Message)

Recommended example:

You are an AI assistant for my CV and portfolio.  
Be polite, concise, and factual.  
Answer in French or English depending on the user's language.  
If the question is ambiguous, ask exactly one short clarifying question.  
If the information is not in the CV summary, say you do not know and ask what information is needed.  
Do not invent details.

---

## 2) Create Azure OpenAI (AI model)

### Step 2.1 — Create the Azure OpenAI resource

1. Go to Azure Portal  
   https://portal.azure.com/#home

2. Search for **Azure OpenAI**

3. Click **Create**

4. You will see a choice between:
   - **Foundry (recommended)**  
   - **Azure OpenAI**

   👉 **Choose: Azure OpenAI**

5. Fill in the settings:

   Resource group  
   `cv-chatbot-rg`

   Instance name  
   `cv-chatbot-openai`

   Region  
   `France Central`

   Pricing  
   `Standard S0`

6. Click **Next** until creation (you can skip **Tags**, it is optional)

### Important note (expected behavior)

After creation, the instance page may show a warning like **“Content failed to load”**.  
This is **normal** because **no model is deployed yet**.

To confirm the resource exists:
- Go to **All resources**
- Verify that `cv-chatbot-openai` appears and is **Running**

---

### Step 2.2 — Deploy a model (VERY IMPORTANT)

1. Open Azure AI Foundry:  
   https://ai.azure.com/

2. Select your created instance:  
   `cv-chatbot-openai`

3. In the left menu, click **Model catalog**

4. From the catalog, select:  
   **gpt-4o-mini**

5. Click **Use this model**

6. Deployment settings:
   - Deployment type: **Standard**
   - Deployment name:  
     `gpt-4o-mini`

7. Click **Deploy**

---

### Step 2.3 — Write down these values (you will need them later)

Use the **exact format below**:

ENDPOINT  
https://cv-chatbot-openai.cognitiveservices.azure.com/

API KEY  
<your-key-1>

DEPLOYMENT NAME  
gpt-4o-mini

Important:
- The endpoint is the **base URL only**
- Do NOT include `/openai/deployments`
- Do NOT include `/chat/completions`
- Do NOT include `api-version`

---

## 3) Create the Azure Function (backend API)

This function is your backend API, equivalent to AWS Lambda or Google Cloud Function.

### Step 3.1 — Create Function App

1. Go to Azure Portal  
   https://portal.azure.com/#home

2. Search for **Function App**

3. Click **Create**

4. Select hosting option:
   - **Consumption** (serverless)

5. On the next screen:

   Resource group  
   `cv-chatbot-rg`

   Function App name  
   `cv-chatbot-api`

6. Runtime stack:
   - Node.js

7. All other settings:
   - Leave **default**
   - Do **not** configure GitHub deployment
   - Do **not** change authentication
   - Do **not** click through advanced tabs

8. Click **Review + Create**
9. Click **Create**

Wait until the Function App is **Running**

---

## 4) Create the chat function

### Step 4.1 — Create HTTP-triggered function

1. Open **Function App** → `cv-chatbot-api`
2. Go to **Functions**
3. Click **Create**
4. Choose **Create in Azure portal**
5. Select **HTTP trigger**
6. Configure:

   Authorization level  
   `Anonymous`

   Function name  
   `chat`

7. Click **Create**

Your backend endpoint will be:

https://cv-chatbot-api.azurewebsites.net/api/chat

---

### Step 4.2 — Paste backend code (copy exactly)

Go to:

Function App → Functions → chat → Code + Test  
Replace **everything** in `index.js` with:

'''
module.exports = async function (context, req) {
  const corsHeaders = {
    "Access-Control-Allow-Origin": "https://yourusername.github.io",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type"
  };

  if (req.method === "OPTIONS") {
    context.res = { status: 204, headers: corsHeaders };
    return;
  }

  try {
    const userMessage = (req.body && req.body.message) ? String(req.body.message) : "";
    if (!userMessage) {
      context.res = {
        status: 400,
        headers: corsHeaders,
        body: { error: "Missing message" }
      };
      return;
    }

    const endpoint = process.env.AZURE_OPENAI_ENDPOINT;
    const apiKey = process.env.AZURE_OPENAI_API_KEY;
    const deployment = process.env.AZURE_OPENAI_DEPLOYMENT;
    const system = process.env.SYSTEM_MESSAGE || "You are a helpful CV assistant.";

    if (!endpoint || !apiKey || !deployment) {
      context.res = {
        status: 500,
        headers: corsHeaders,
        body: { error: "Missing Azure OpenAI configuration" }
      };
      return;
    }

    const apiVersion = "2024-06-01";
    const url = `${endpoint.replace(/\/$/, "")}/openai/deployments/${deployment}/chat/completions?api-version=${apiVersion}`;

    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "api-key": apiKey
      },
      body: JSON.stringify({
        temperature: 0.2,
        max_tokens: 400,
        messages: [
          { role: "system", content: system },
          { role: "user", content: userMessage }
        ]
      })
    });

    const text = await response.text();
    if (!response.ok) {
      context.res = {
        status: 500,
        headers: corsHeaders,
        body: { error: text }
      };
      return;
    }

    const data = JSON.parse(text);
    const reply = data?.choices?.[0]?.message?.content || "";

    context.res = {
      status: 200,
      headers: corsHeaders,
      body: { reply }
    };

  } catch (err) {
    context.log(err);
    context.res = {
      status: 500,
      headers: corsHeaders,
      body: { error: err.message }
    };
  }
};
'''

Click **Save**  
Save = Deploy (no package.json required)

---

## 5) Add environment variables (VERY IMPORTANT)

Secrets live here. Never put them in HTML.

Go to:

Function App → Settings → Environment variables → App settings

Add:

Name  
AZURE_OPENAI_ENDPOINT  
Value  
https://cv-chatbot-openai.cognitiveservices.azure.com/

Name  
AZURE_OPENAI_API_KEY  
Value  
<your-key-1>

Name  
AZURE_OPENAI_DEPLOYMENT  
Value  
gpt-4o-mini

Name  
SYSTEM_MESSAGE  
Value  
(Assistant rules + CV summary)

Notes:
- Do NOT check **Deployment slot setting**
- Names are case-sensitive

Click **Save**  
Restart the Function App

---

## 6) Enable CORS (critical step)

Because GitHub Pages ≠ Azure domain.

Go to:

Function App → API → CORS

Add allowed origin:

https://yourusername.github.io

Do NOT:
- Enable Access-Control-Allow-Credentials
- Use *

Click **Save**

---

## 7) Get your API URL

In your function:

Functions → chat → Get Function URL

Copy:

https://cv-chatbot-api.azurewebsites.net/api/chat

This is your permanent backend endpoint.

---

## 8) Create the frontend (index.html)

'''
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>AI CV Assistant</title>
</head>
<body>
  <h2>AI CV Assistant</h2>

  <textarea id="q" rows="4" cols="60"></textarea><br>
  <button onclick="ask()">Ask</button>

  <pre id="a"></pre>

<script>
const API_URL = "https://cv-chatbot-api.azurewebsites.net/api/chat";

async function ask() {
  const q = document.getElementById("q").value;
  const res = await fetch(API_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: q })
  });
  const data = await res.json();
  document.getElementById("a").textContent = data.reply || data.error;
}
</script>
</body>
</html>
'''

---

## 9) Publish on GitHub Pages

1. Create a GitHub repository
2. Upload **only** `index.html`
3. Go to **Settings → Pages**
4. Enable GitHub Pages
5. Open the public URL

---

## 10) What this setup gives you

- Same pattern as AWS / Google Cloud
- Static frontend only
- Azure-hosted backend
- Secrets securely stored
- No Copilot Studio
- No SDK or package.json
- Save = deploy
- Production-ready architecture

🎉 Done
