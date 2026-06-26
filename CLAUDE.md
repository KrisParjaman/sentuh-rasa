# Sentuh Rasa (SBSR) — WA Bridge Architecture

## System Overview

```
WhatsApp -> Meta Cloud API -> server.js (Express) -> OpenClaw container (LLM)
                ^                     |                      |
                |                     +-- admin.js (panel)   |
                |                     +-- catalog-map.json   |
                |                     +-- llm-router.js      |
                |                     +-- lib/*.cjs (util)   |
                +---------------------+                      |
                                                             |
               Biteship API <-- server.js <------------------+
```

**Server:** DigitalOcean droplet biks-droplet (206.189.34.228)
**Runtime:** Node.js (CommonJS), Express, PM2 (wa-bridge-sbsr)
**LLM Backend:** OpenClaw container (127.0.0.1:45891)

## File Structure

```
server.js              Main bridge (10,596 lines)
                        Express server, WA webhook handler,
                        intent router, order logic, quote, invoice
admin.js               Admin panel + chat history (58KB)
llm-router.js          LLM-first routing before deterministic logic
catalog-map.json       Product retailer_id to display name mapping
products.json          Static product catalog (addon fallback)
lib/
  rate-limiter.cjs       Per-phone rate limiting
  prompt-sanitizer.cjs   Input sanitization
  courier-choice-parser.cjs  Parse kurir selection from LLM
  draft-policy.cjs       Order draft policy
  cost-guard.cjs         Cost tracking guard
scripts/
  read-receipt.cjs       OCR receipt via Gemini API
  llm-addr.cjs           LLM address parser
.env.example             Template env
```

## Message Flow

### 1. Inbound Message

```
Meta Webhook POST /webhook
  -> verifySignature()
  -> handleMessage(msg, contacts)
    -> #1 Security: killswitch (SBSR_PAUSE) + rate-limit
    -> #2 Idempotent dedup (60s window for duplicate message_id)
    -> #3 Draft load + timestamp stamp
    -> #4 Route: llmFirstRouter() or deterministic handlers
       - tryHandleUseCaseRouter()
       - tryHandleAddonReply()
       - tryHandleNameCapture()
       - tryHandleAddressAndQuote()
       - tryHandleBareMapsUrl()
       - tryHandlePinConfirm()
       - tryHandleInvoiceOk()
       - tryHandleBuktiOcrFailedManualReview()
       - tryHandleCatalogRequest()
       - tryHandleAdminCmd()
       - tryHandleFrozenCourierChoice()
       - tryHandleFaq()
       and more...
    -> #5 OpenClaw LLM call (if LLM route)
    -> #6 Send response via WhatsApp Cloud API
```

### 2. LLM Routing (llm-router.js)

```
llmFirstRouter(from, text, draft)
  -> buildLlmContext()
     - current state, cart items, customer name
     - product catalog (live from Meta API)
     - order draft
  -> POST to OpenClaw API (127.0.0.1:45891)
  -> Parse JSON response: { intent, reply, data }
  -> If confidence >= 0.6: execute intent + return reply
  -> If confidence < 0.6: fallback to deterministic handler
```

The LLM prompt instructs the model to act as "Mintu", the CS rep for Sentuh Rasa.

### 3. Catalog System

```
Startup:
  -> load catalog-map.json (static mapping)
  -> refreshCatalogFromAPI() -- fetch from Meta Graph API
  -> setInterval() -- refresh every 5 minutes

Data stores:
  - catalogMap: retailer_id -> product name
  - catalogPrices: retailer_id -> price in IDR
  - catalogAvailability: retailer_id -> stock status

Key functions:
  - lookupProductName(rid)
  - lookupProductPrice(rid)
  - lookupProductAvailability(rid)
  - formatCatalogForLLM() -> system prompt context
  - formatSbsrFullMenuText() -> customer-facing menu text
```

### 4. Checkout State Machine

```
null/initial
    |  (customer says "beli" / "pesan")
    v
awaiting_usecase
    |  (pickup or delivery)
    v
awaiting_product_selection
    |  (select product + quantity)
    v
awaiting_addon_reply
    |  (yes/add-on/no)
    v
awaiting_delivery_method
    |  (courier or pickup)
    v
awaiting_name
    |  (customer name)
    v
awaiting_address
    |  (address + maps pin)
    v
awaiting_pin_confirmation
    |  (confirm location pin)
    v
awaiting_invoice_confirm
    |  (OK/YA)
    v
awaiting_proof
    |  (upload transfer receipt)
    v
[DONE] -> order saved, Biteship shipping
```

### 5. Biteship Shipping

```
Checkout final:
  -> create Biteship order with courier
  -> save order_id in draft
  -> Biteship sends webhook POST /biteship-webhook
     - event: order.status
     - status: "delivered" -> notify customer
     - order lookup by: metadata.order_id, biteship_order_id, waybill_id
```

## Key Endpoints

| Endpoint | Method | Function |
|----------|--------|----------|
| /webhook | GET | Meta webhook verification |
| /webhook | POST | Inbound WhatsApp messages |
| /biteship-webhook | POST | Biteship delivery status |
| /send | POST | Manual send (admin) |
| /send-pdf | POST | Send invoice PDF |
| /health | GET | Health check |
| /admin/* | Various | Admin panel routes |

## External Dependencies

| Service | Purpose | Connection |
|---------|---------|------------|
| Meta WhatsApp Cloud API | Send/receive WA messages | HTTPS API |
| OpenClaw | LLM processing container | HTTP 127.0.0.1:45891 |
| OpenRouter | LLM API for address parsing | HTTPS API |
| Google Geocoding API | Address to coordinates | HTTPS API |
| Nominatim | Geocoding fallback | HTTPS API |
| Biteship API | Shipping/courier | HTTPS API |
| Meta Catalog API | Product catalog sync | Graph API v22.0 |
| ImgBB | Image hosting for receipts | HTTPS API |

## Notes

- CommonJS (require), NOT ES modules
- PM2 process name: wa-bridge-sbsr
- Working directory: /docker/wa-webhook-sbsr/
- Single-process, non-blocking async architecture
- Chat data stored as individual JSON files in chats/
- Orders stored in orders.json (OpenClaw workspace)
- Catalog prices auto-refresh from Meta every 5 minutes
