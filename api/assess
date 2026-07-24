// api/assess.js
// Vercel serverless function — this is the ONLY place the real Anthropic API key
// is ever used. The browser never sees it.
//
// Set ANTHROPIC_API_KEY in your Vercel project's Environment Variables
// (Project Settings -> Environment Variables), not in this file.

const MODEL = "claude-sonnet-5";       // swap to "claude-haiku-4-5-20251001" for a cheaper/faster option
const MAX_TOKENS = 1500;
const MAX_PAYLOAD_BYTES = 4_000_000;   // guard against oversized attachments (Vercel body limits apply too)

export default async function handler(req, res) {
  if (req.method !== "POST") {
    res.setHeader("Allow", "POST");
    return res.status(405).json({ error: "Method not allowed" });
  }

  const apiKey = process.env.ANTHROPIC_API_KEY;
  if (!apiKey) {
    return res.status(500).json({ error: "Server is missing ANTHROPIC_API_KEY. Set it in Vercel project settings." });
  }

  const { content } = req.body || {};
  if (!Array.isArray(content) || content.length === 0) {
    return res.status(400).json({ error: "Missing or invalid 'content' in request body." });
  }

  const approxSize = JSON.stringify(content).length;
  if (approxSize > MAX_PAYLOAD_BYTES) {
    return res.status(413).json({ error: "Attachment too large for this endpoint. Try a smaller PDF or a plain-text file." });
  }

  try {
    const anthropicRes = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "content-type": "application/json",
        "x-api-key": apiKey,
        "anthropic-version": "2023-06-01",
      },
      body: JSON.stringify({
        model: MODEL,
        max_tokens: MAX_TOKENS,
        // model and max_tokens are fixed server-side on purpose — the client
        // cannot request a different (more expensive) model or token budget.
        messages: [{ role: "user", content }],
      }),
    });

    if (!anthropicRes.ok) {
      const errText = await anthropicRes.text();
      return res.status(anthropicRes.status).json({ error: "Anthropic API error", detail: errText });
    }

    const data = await anthropicRes.json();
    const textBlocks = (data.content || [])
      .filter((b) => b.type === "text")
      .map((b) => b.text)
      .join("\n");

    const clean = textBlocks.replace(/```json|```/g, "").trim();

    let parsed;
    try {
      parsed = JSON.parse(clean);
    } catch (e) {
      return res.status(502).json({ error: "Could not parse the model's response as JSON.", raw: clean });
    }

    return res.status(200).json({ result: parsed });
  } catch (err) {
    return res.status(500).json({ error: err.message || "Unknown server error" });
  }
}
