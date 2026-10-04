# WhatsApp Webhook → Firebase → Gemini AI

> How patient messages on WhatsApp are received, validated, and processed by a Gemini-powered AI agent, with all state stored in Firestore.

---

## Overview

```
Patient sends WhatsApp message
        ↓
Meta WhatsApp Cloud API
        ↓
Firebase Cloud Function (onRequest) — HTTPS webhook
        ↓
Signature verification (HMAC-SHA256)
        ↓
Inbound pipeline (idempotency → patient lookup → rate limit)
        ↓
AI Orchestrator (orchestrator.ts)
        ↓
Gemini API (function-calling tools)
        ↓
Firestore (state + message log)
        ↓
WhatsApp reply sent to patient
        ↓
Dashboard updated in real-time (Firestore stream)
```

---

## 1. Webhook Entry Point (Firebase Cloud Function)

The webhook is an `onRequest` Gen 2 Cloud Function that handles:

- **GET** — Meta webhook verification (hub challenge)
- **POST** — Inbound messages and delivery status updates

```typescript
// backend/functions/src/webhooks/core/whatsapp.ts

import { onRequest } from "firebase-functions/v2/https";
import { defineSecret } from "firebase-functions/params";

// Secrets loaded from Firebase Secret Manager — never hardcoded
const WHATSAPP_VERIFY_TOKEN = defineSecret("WHATSAPP_VERIFY_TOKEN");
const WHATSAPP_APP_SECRET   = defineSecret("WHATSAPP_APP_SECRET");
const WHATSAPP_TOKEN        = defineSecret("WHATSAPP_TOKEN");

export const whatsappWebhook = onRequest(
  {
    secrets: [WHATSAPP_VERIFY_TOKEN, WHATSAPP_APP_SECRET, WHATSAPP_TOKEN],
    timeoutSeconds: 60,
    memory: "512MiB",
  },
  async (req, res) => {
    // ── GET: Meta webhook verification ─────────────────────────────────────
    if (req.method === "GET") {
      const mode      = req.query["hub.mode"];
      const token     = req.query["hub.verify_token"];
      const challenge = req.query["hub.challenge"];

      if (mode === "subscribe" && timingSafeTokenMatch(token, WHATSAPP_VERIFY_TOKEN.value())) {
        res.status(200).send(challenge);
      } else {
        res.sendStatus(403);
      }
      return;
    }

    // ── POST: Inbound message ───────────────────────────────────────────────
    if (req.method === "POST") {
      // Always return 200 immediately — Meta retries on anything else.
      res.sendStatus(200);

      // Signature verification happens in middleware before we get here.
      await handleInboundWebhook(req.body);
    }
  }
);
```

---

## 2. Signature Verification (Security Middleware)

Every POST is verified using HMAC-SHA256 over the **raw body** before any processing starts. Invalid signatures return 403 immediately.

```typescript
// backend/functions/src/webhooks/middleware/signature.ts

import * as crypto from "crypto";

export function verifyWhatsAppSignature(
  rawBody: Buffer,
  signatureHeader: string,
  appSecret: string,
): boolean {
  const expectedSig = "sha256=" +
    crypto.createHmac("sha256", appSecret)
          .update(rawBody)
          .digest("hex");

  // Constant-time comparison prevents timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(signatureHeader),
    Buffer.from(expectedSig),
  );
}
```

---

## 3. Inbound Message Pipeline

After the webhook receives a valid message, it runs through a sequential pipeline. Each stage is a separate module — the pipeline file only orchestrates the order.

```typescript
// backend/functions/src/webhooks/core/pipeline.ts (simplified)

/**
 * Pipeline stages:
 *  1. runPrecheck          — idempotency guard, patient lookup, rate limit
 *  2. interceptBookingConfirmation — button-tap handling (before AI)
 *  3. routeMedia           — audio / image / document / text routing
 *  4. sendUiUpdates        — read receipt + typing indicator
 *  5. runAi                — AI enablement check + orchestrator
 */
export async function processInboundMessage(
  db: Firestore,
  value: any,
  channel: WhatsAppChannel,
): Promise<void> {
  if (!value.messages || value.messages.length === 0) return;

  for (const messageObj of value.messages) {
    await processSingleMessage(db, value, channel, messageObj);
  }
}

async function processSingleMessage(db, value, channel, messageObj) {
  const messageId = messageObj.id;
  const from      = messageObj.from;       // patient phone
  const type      = messageObj.type;       // text | audio | image | document

  // ── Stage 1: Idempotency + Patient Resolution ────────────────────────────
  const precheck = await runPrecheck(db, messageId, from, channel);
  if (precheck.shouldAbort) return; // duplicate or rate-limited

  const { patientId, inboundText } = precheck;

  // ── Stage 2: Booking button intercept ───────────────────────────────────
  const bookingResult = await interceptBookingConfirmation(db, messageObj, patientId, channel);
  if (bookingResult.handled) return;

  // ── Stage 3: Route by media type ────────────────────────────────────────
  const media = await routeMedia(db, messageObj, patientId, channel);

  // ── Stage 4: UI updates (non-blocking) ───────────────────────────────────
  sendUiUpdates(db, patientId, channel); // fire-and-forget

  // ── Stage 5: Run AI ──────────────────────────────────────────────────────
  if (await shouldRunAi(db, patientId, channel.channelId)) {
    await runAgentOrchestrator(
      db, patientId, channel, from, messageId,
      media.text ?? inboundText, media.audioData, media.imageData,
    );
  }
}
```

---

## 4. Idempotency Guard

Meta may retry webhooks. The guard ensures each `messageId` is processed exactly once using a Firestore `processed_messages` document with a TTL.

```typescript
// backend/functions/src/webhooks/middleware/idempotency.ts

export async function checkIdempotency(
  db: Firestore,
  messageId: string,
): Promise<boolean> {
  const ref = db.collection("processed_messages").doc(messageId);
  const snap = await ref.get();
  if (snap.exists) return false; // already processed

  await ref.set({ processedAt: new Date() }); // mark as processed
  return true;
}
```

---

## 5. AI Orchestrator

The orchestrator is the brain. It is a **pure coordinator** — each step is implemented in its own module. The orchestrator only decides the order and stop conditions.

```typescript
// backend/functions/src/ai/orchestrator.ts (annotated)

export async function runAgentOrchestrator(
  db: Firestore,
  patientId: string,
  channel: WhatsAppChannel,
  from: string,
  messageId: string,
  inboundText: string,
  audioData?: { buffer: Buffer; mimeType: string },
  imageData?: ImageData,
): Promise<void> {

  // ── Step 1: Load conversation context (single Firestore batch read) ──────
  const ctx = await buildConversationContext(db, patientId, channel.channelId, profile);

  // ── Step 2: Human-in-loop guard ──────────────────────────────────────────
  if (ctx.mode === "Manual") return; // staff is replying — AI stays silent

  // ── Step 3: Audio transcription (Gemini audio → text) ────────────────────
  ({ inboundText, audioData } = await handleAudioTranscription(
    db, patientId, channel.channelId, messageId, inboundText, audioData, langHint,
  ));

  // ── Step 4: Language detection (Arabic / English) ────────────────────────
  const isEn = detectLanguage(inboundText, ctx.preferredLanguage);

  // ── Step 5: Emergency bypass ──────────────────────────────────────────────
  if (inboundText && isEmergency(inboundText)) {
    await triggerEmergencyProtocol(db, patientId, ...);
    return;
  }

  // ── Step 6: Pre-LLM deterministic routing (payment / staff / fail guard) ─
  if ((await handlePreLLMRouting(db, patientId, ...)).handled) return;

  // ── Step 7: Prescription image handling ──────────────────────────────────
  if (imageData && profile.supportsPrescriptionBooking) {
    const result = await handlePrescriptionImage(...);
    if (result.handled) return;
  }

  // ── Step 8: Active cancel flow (state machine) ────────────────────────────
  if (ctx.cancelState) {
    await handleActiveCancelFlow(db, ctx.cancelState, inboundText, patientId, ...);
    return;
  }

  // ── Step 9: Active booking flow (state machine) ───────────────────────────
  const bookingResult = await handleActiveBookingFlow(db, ctx.bookingState, ...);
  if (bookingResult.handled) return;

  // ── Step 10: Call Gemini LLM ──────────────────────────────────────────────
  const fullPrompt = [...ctx.history, { role: "user", parts: buildParts(inboundText, audioData) }];

  let llmResult;
  try {
    llmResult = await callLLM(fullPrompt, isEn, profile);
  } catch (e) {
    // Graceful degradation: send a polite unavailability message + handoff to staff
    await send(getReplyLlmUnavailable(isEn), `llm_error_${messageId}`);
    await handoffToStaff(db, patientId, channel.channelId);
    return;
  }

  // ── Step 11: Post-LLM routing (tool execution, intent parse) ──────────────
  const postResult = await handlePostLLMRouting(db, llmResult, fullPrompt, ...);
  if (postResult.handled) return;

  // ── Step 12: Log + Send in parallel ───────────────────────────────────────
  await Promise.all([
    writeAiLog(db, messageId, patientId, fullPrompt, postResult.finalReply!, ...),
    sendWithRetryFallback(db, patientId, channel.channelId, channel.phoneNumberId, from, postResult.finalReply!, ...),
  ]);
}
```

---

## 6. Gemini Function-Calling Tools

The LLM is **constrained** — it can only act through the 8 registered tools. No free-form actions are possible.

```typescript
// backend/functions/src/ai/core/llm-call.ts (simplified)

import { GoogleGenAI, Type } from "@google/genai";

const TOOLS = [
  {
    name: "check_availability",
    description: "Query available appointment slots for a given service and date.",
    parameters: {
      type: Type.OBJECT,
      properties: {
        service: { type: Type.STRING },
        date:    { type: Type.STRING, description: "YYYY-MM-DD" },
      },
      required: ["service", "date"],
    },
  },
  {
    name: "create_booking",
    description: "Create a confirmed appointment booking.",
    parameters: { /* ... patientName, service, date, time, etc. */ },
  },
  {
    name: "cancel_booking",
    description: "Cancel an existing booking.",
    parameters: { type: Type.OBJECT, properties: { bookingId: { type: Type.STRING } }, required: ["bookingId"] },
  },
  {
    name: "join_waitlist",
    description: "Register the patient on the waitlist for a given date.",
    parameters: { /* ... */ },
  },
  {
    name: "answer_faq",
    description: "Answer a common clinic FAQ. Never hallucinate — use only the provided list.",
    parameters: { type: Type.OBJECT, properties: { question_key: { type: Type.STRING } }, required: ["question_key"] },
  },
  {
    name: "escalate_to_staff",
    description: "Transfer the conversation to a human staff member.",
    parameters: {},
  },
  {
    name: "trigger_emergency_protocol",
    description: "Fire the emergency protocol for critical medical situations.",
    parameters: {},
  },
  {
    name: "offer_mtm_consultation",
    description: "Offer a Medication Therapy Management consultation (drug interactions).",
    parameters: {},
  },
];

export async function callLLM(
  history: Content[],
  isEn: boolean,
  profile: ChannelProfile,
): Promise<GenerateContentResult> {
  const ai = new GoogleGenAI({ apiKey: GEMINI_API_KEY.value() });

  return ai.models.generateContent({
    model: "gemini-2.0-flash-lite",
    contents: history,
    config: {
      systemInstruction: buildSystemPrompt(isEn, profile),
      tools: [{ functionDeclarations: TOOLS }],
      toolConfig: { functionCallingConfig: { mode: "AUTO" } },
    },
  });
}
```

---

## 7. Tool Execution + Firestore Write-Back

When Gemini returns a tool call, the orchestrator executes it. Write tools (`create_booking`, `cancel_booking`, `join_waitlist`) receive extra validation.

```typescript
// backend/functions/src/ai/flows/post-llm-routing.ts (simplified)

export async function handlePostLLMRouting(
  db: Firestore,
  llmResult: GenerateContentResult,
  history: Content[],
  patientId: string,
  ...
) {
  const candidate = llmResult.candidates?.[0];
  const toolCall  = candidate?.content?.parts?.find(p => p.functionCall);

  if (!toolCall?.functionCall) {
    // Plain text reply — extract and return
    return { handled: false, finalReply: extractText(candidate) };
  }

  const { name: toolName, args: toolArgs } = toolCall.functionCall;

  switch (toolName) {
    case "create_booking":
      // Runs the booking state machine — writes to Firestore
      await executeCreateBooking(db, patientId, toolArgs, channel, ...);
      return { handled: true };

    case "cancel_booking":
      await executeCancelBooking(db, patientId, toolArgs.bookingId, ...);
      return { handled: true };

    case "escalate_to_staff":
      // Writes activeIntent = 'staff_handoff' to Firestore
      // → Dashboard inbox shows a "Needs Staff" badge in real-time
      await handoffToStaff(db, patientId, channelId);
      return { handled: true };

    case "trigger_emergency_protocol":
      // Writes an EmergencyAlert document to Firestore
      // → Dashboard shows a red banner immediately
      await triggerEmergencyProtocol(db, patientId, channelId, ...);
      return { handled: true };

    case "answer_faq":
      const reply = getFaqAnswer(toolArgs.question_key, isEn, profile);
      return { handled: false, finalReply: reply, toolName, toolArgs };

    // ... other tools
  }
}
```

---

## 8. Dashboard Real-Time Update (Firestore → Flutter)

After the AI writes to Firestore, the Flutter dashboard updates automatically via a Firestore real-time stream — no polling needed.

```dart
// dashboard/lib/features/inbox/data/datasources/remote/inbox_remote_data_source_impl.dart

@override
Stream<List<ConversationPreviewModel>> watchConversations({
  int limit = 30,
  String? channelId,
}) {
  Query<Map<String, dynamic>> query = _firestore
      .collection('Conversations')
      .orderBy('lastMessageAt', descending: true)
      .limit(limit);

  if (channelId != null) {
    query = query.where('channelId', isEqualTo: channelId);
  }

  return query.snapshots().map((snap) =>
    snap.docs
        .map((doc) => ConversationPreviewModel.fromFirestore(doc))
        .toList(),
  );
}
```

The stream flows through:
```
Firestore snapshot → DataSource → RepositoryImpl → UseCase → Cubit → Widget rebuilds
```

---

## 9. Delivery Status Updates

Meta sends delivery status webhooks (sent → delivered → read). The handler updates the matching message document in Firestore so the inbox shows live delivery ticks (✓ → ✓✓ → ✓✓ blue).

```typescript
// backend/functions/src/webhooks/handlers/message-status.ts (simplified)

export async function handleMessageStatus(value: any, db: Firestore, channel: WhatsAppChannel) {
  if (!value.statuses?.length) return;

  const { id: messageId, status, timestamp, recipient_id } = value.statuses[0];

  try {
    // Preferred: match by WhatsApp message ID (wamid)
    let docRef = await findByWamid(db, messageId);

    // Fallback: resolve by recipient phone → find most recent outbound message
    if (!docRef && recipient_id) {
      docRef = await findByRecipient(db, messageId, recipient_id);
    }

    if (docRef) {
      await docRef.set(
        { delivery_status: status, metaMessageId: messageId, updatedAt: new Date(timestamp * 1000) },
        { merge: true },
      );
    }
  } catch (e) {
    // Swallowed: a missing delivery tick must never break the webhook
    console.error("Failed to store message status:", e);
  }
}
```

---

## Secrets Setup (Firebase Secret Manager)

```bash
# All secrets are set via CLI — never hardcoded or committed
firebase functions:secrets:set GEMINI_API_KEY
firebase functions:secrets:set WHATSAPP_TOKEN
firebase functions:secrets:set WHATSAPP_APP_SECRET
firebase functions:secrets:set WHATSAPP_VERIFY_TOKEN
firebase functions:secrets:set WHATSAPP_PHONE_NUMBER_ID

# Optional: second WhatsApp Business number
firebase functions:secrets:set WHATSAPP_PHONE_NUMBER_ID_2
```

The webhook declares which secrets it needs at deployment time:
```typescript
secrets: ["WHATSAPP_VERIFY_TOKEN", "WHATSAPP_APP_SECRET", "WHATSAPP_TOKEN"]
```
