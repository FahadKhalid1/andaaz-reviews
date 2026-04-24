# Andaaz Review System — Project Documentation

## Overview

A mobile-first review generation system for **Andaaz**, a Franco-Indian fusion restaurant in Paris 17 (near Arc de Triomphe). The goal is to make it as easy as possible for happy customers to leave a 5-star Google Maps review — without violating Google's policies.

The system works in three steps:
1. A customer scans a QR code on the table
2. They land on a branded page with an AI-assembled review ready to copy
3. They paste it into Google Maps and submit

Everything is tracked per waiter so management can see who is driving the most reviews.

---

## How It Works

```
Customer scans QR code
        ↓
Opens andaaz.step-upai.com?w=waiter_name
        ↓
Page loads → fires "scan" event to n8n webhook → logged to Google Sheet
        ↓
Customer picks mood (food / ambiance / service / family / fusion / everything)
        ↓
Page generates a unique, GEO-optimised review in French or English
        ↓
Customer copies the review → fires "copy" event → logged to Google Sheet
        ↓
Customer taps "Écrire un avis Google" → fires "submit" event → Google Maps opens
        ↓
Customer pastes review and submits on Google Maps
```

---

## Files

### `index.html` — The Review Page
**URL:** https://andaaz.step-upai.com

The page customers land on after scanning the QR code. Fully self-contained — no backend, no framework, plain HTML/CSS/JS.

**Features:**
- Dark gold/cream brand theme
- Bilingual: French (default) and English toggle
- 6 mood buttons the customer picks from before generating a review
- Review assembled on the fly from keyword/fragment pools (never the same twice)
- Editable text field — customer can personalise before copying
- Regenerate button for a fresh review
- Copy button with clipboard API
- Google Maps button that opens the review form directly
- Per-waiter event tracking via `?w=` URL parameter

**Review generation engine:**
Reviews are built client-side by combining random fragments from pools:
- `seoKeywords` — location tags, cuisine labels, superlatives, dish names
- `fragments` — mood-specific openers, middles, and closers
- `moodAddOns` — short extra sentences when multiple moods are selected

The result is a natural-sounding, GEO-optimised review that mentions specific dishes, Paris 17 location signals, and superlatives like "meilleur restaurant indien à Paris". A hash-deduplication set ensures the same review is never generated twice in a session.

**SEO/GEO keywords baked in:**
- Location: Paris 17, Arc de Triomphe, Porte Maillot, Neuilly-sur-Seine, Wagram, La Défense
- Dishes: dum biryani, butter chicken, chicken tandoori masala, cheese naan, truffle naan, nihari bourguignon, mixed grill, and more
- Labels: meilleur restaurant indien à Paris, halal fine dining, Indo-French fusion, etc.

---

### `qr.html` — The QR Code Generator
**URL:** https://andaaz.step-upai.com/qr.html

An internal tool for staff/management to generate print-ready QR codes. Not customer-facing.

**Features:**
- Generates a QR code pointing to `https://andaaz.step-upai.com`
- Waiter name field — appends `?w=waiter_name` to the URL so scans are tracked per waiter
- QR card and Print/Download buttons are **disabled until a waiter name is entered**
- Waiter name is automatically logged to Google Sheet 1 second after typing stops
- Print and Download also log a `qr_printed` / `qr_downloaded` event
- 3 size options: Small (table tent), Medium (A6), Large (A5)
- Print button (hides controls, prints just the card)
- Download QR as PNG
- Andaaz logo loaded from `andaaz.fr/logo.png`
- Copper/cream print aesthetic matching Andaaz brand

**Waiter QR workflow:**
1. Open `qr.html`
2. Type waiter's name (e.g. `fahad`)
3. QR code activates and updates to `https://andaaz.step-upai.com?w=fahad`
4. Waiter name is automatically logged to the Google Sheet (`qr_created` event)
5. Print or download the card
6. Place it on that waiter's tables

---

## Tracking System

### How events are fired

Tracking uses a **GET request with URL parameters** via `new Image().src = url`. This approach is completely exempt from CORS restrictions — no preflight, no blocking.

#### Events from `index.html` (customer-facing)

| Event | When |
|-------|------|
| `scan` | Page loads — customer opened the review page |
| `copy` | Customer clicks "Copy Review" |
| `submit` | Customer clicks "Écrire un avis Google" |

#### Events from `qr.html` (staff-facing)

| Event | When |
|-------|------|
| `qr_created` | Waiter name typed and QR generated (1s debounce, fires once per name) |
| `qr_printed` | Print button clicked |
| `qr_downloaded` | Download QR PNG button clicked |

> **Coming soon:** `qr_deactivated` — fired when a waiter's name is deleted, which will block their QR code from working.

### n8n Webhook
**URL:** `https://fk92.app.n8n.cloud/webhook/andaaz-waiter-tracker`

**Workflow:** `Andaaz — Waiter Review Tracker` (ID: `ZOyRscCdfPvgLUYZ`)

The workflow has 4 nodes:
1. **Track Event Webhook** — receives GET request from the review/QR page
2. **Respond OK** — immediately returns 200 with CORS headers
3. **Normalize Event** — cleans/standardises all fields from query params
4. **Append to Google Sheet** — writes a row to the Google Sheet via the Sheets API

### Google Sheet
**Spreadsheet:** `Andaaz Waiter Tracker`
**ID:** `1Fjx4sdDJMVQSRNh9n3H49lsTaXH0d4q2-86-Xa_VvZQ`
**Tab:** `events`
**Link:** https://docs.google.com/spreadsheets/d/1Fjx4sdDJMVQSRNh9n3H49lsTaXH0d4q2-86-Xa_VvZQ/edit

Columns:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| timestamp | waiter | event | lang | moods | userAgent |

Every action — scan, copy, submit, QR created, printed, downloaded — appears as a separate row, giving a full funnel view per waiter.

---

## Key Configuration Values

| Setting | Value |
|---------|-------|
| Review page URL | https://andaaz.step-upai.com |
| QR generator URL | https://andaaz.step-upai.com/qr.html |
| Google Maps review link | https://g.page/r/CWRCrOTozj1eEBM/review |
| n8n tracker webhook | https://fk92.app.n8n.cloud/webhook/andaaz-waiter-tracker |
| Google Sheet ID | 1Fjx4sdDJMVQSRNh9n3H49lsTaXH0d4q2-86-Xa_VvZQ |
| n8n tracker workflow ID | ZOyRscCdfPvgLUYZ |

---

## How to Add a New Waiter

1. Go to https://andaaz.step-upai.com/qr.html
2. Type the waiter's name in the **Waiter Name** field
3. The QR code activates automatically
4. The waiter's name is logged to the Google Sheet within 1 second
5. Click **Print** or **Download QR PNG**
6. Place the printed card on their assigned tables

That waiter's scans, copies, and submits will now appear in the Google Sheet with their name in the `waiter` column.

---

## Planned: Waiter Deactivation

When built, this will allow disabling a specific waiter's QR code without reprinting others:

1. Go to `qr.html`, type the waiter's name, then clear the field
2. A `qr_deactivated` event fires automatically
3. A new n8n status API marks the waiter as inactive
4. Anyone scanning that waiter's QR code sees a "This code is no longer active" message
5. To reactivate: type the name again — a new `qr_created` event re-enables them

---

## Why This Approach Is Legitimate

Google does not allow automated review posting. This system stays within policy because:
- The customer manually copies the text
- The customer manually pastes and submits on Google Maps
- The AI-assembled review is a starting point they can edit
- No Google account access is requested or required

---

## Possible Future Improvements

- **Waiter deactivation** — disable a QR code without reprinting (in progress)
- **Dashboard** — analytics page showing scans/copies/submits per waiter over time
- **Conversion funnel** — see scan → copy → submit drop-off per waiter
- **A/B testing** — rotate different review styles and measure which gets more submits
- **SMS follow-up** — send the review link to customers who gave their number
- **Dynamic keyword updates** — pull trending local search terms automatically for fresher GEO optimisation
- **Multi-location** — extend the system to other Andaaz locations with separate tracking
