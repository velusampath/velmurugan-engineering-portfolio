# VisionBill — Suggestion and V1 Plan

This file is the product suggestion and V1 plan only. No application code yet.

**Working name:** VisionBill  
**Feature name:** Bulk Vision Billing (AI Bulk Checkout)  
**Positioning:** computer-vision-powered automated checkout and billing — not another barcode-only POS.

---

## My suggestion

Build this. Do not build it as a normal barcode-billing app only.

The bigger product is:

> Put many products on a tray → scan the tray once → identify each product → automatically create the bill.

Keep your stack:

| Layer | Choice |
| --- | --- |
| Backend / API | Python + FastAPI |
| Database | PostgreSQL |
| Web dashboard | React.js + TypeScript |
| Mobile / POS | Flutter |

Add a separate **AI / computer-vision layer**. Do not put heavy detection inside every FastAPI request.

Use **two billing modes**:

1. **Mode A — Normal barcode billing** (V1)  
   Barcode → product lookup → add to cart → bill.
2. **Mode B — AI bulk billing** (V2/V3)  
   Camera → detect many products → barcode / OCR / visual match → quantities → cart → invoice.

Example of Mode B:

A rack has 5 × Coke 500ml, 3 × Pepsi 500ml, 4 × Lays, 2 × biscuits.

Instead of 14 individual scans:

```
Tray of mixed products
    → Camera scan
    → AI detects 14 products on the tray
    → Coke × 5, Pepsi × 3, Lays × 4, Biscuits × 2
    → ₹ total
```

**Do not start with “our AI recognizes any supermarket product.”**  
Start with a **controlled merchant catalog of 100–500 SKUs** (grocery, FMCG, beverages, snacks, personal care). The merchant uploads name, SKU, barcode, brand, category, MRP, selling price, and product images. AI only searches that catalog.

**Do not trust AI blindly.** Use a confidence system:

- High confidence → auto-add to cart  
- Medium confidence → cashier confirms  
- Unknown → **AI Review Queue** (do not discard the crop; it becomes training data)

Identification order (never AI-only):

1. Barcode (most reliable)  
2. OCR / packaging text  
3. Product image matching (embedding + pgvector)  
4. AI classification  
5. Human confirmation  

**Phased build (this is the realistic path):**

1. POS + inventory + barcode billing  
2. Camera → many barcodes in one image → automatic cart  
3. Camera → detect products → recognize known catalog items  
4. Quantity detection + confidence confirmation  
5. Fully automated checkout station  
6. Multi-store SaaS + analytics  

Phase 2 (multi-barcode from one photo) is the first real differentiator. Full rack recognition is Phase 3+, not the MVP.

**If we do not want a custom camera product:** do not invent vision. Use software that already exists. Tray + many products is already solved in shops as **barcode POS + scanner**. Identity is always: barcode number → product master → bill. The software names are below.

---

## Existing software products (check first — no custom camera)

You do **not** need to build a camera AI product to bill a tray of items. That software already exists in three layers.

### 1. Billing / POS software (this is the product shops already use)

These identify a product **without a camera**: USB/Bluetooth barcode gun (or phone barcode SDK) → software looks up SKU → adds the line → GST bill → stock out.

Put 14 items on a tray. Cashier beeps each barcode. Software builds Coke × 5, Pepsi × 3, etc. as the same codes are scanned again (qty +1).

| Software | What it is | How it identifies the product |
| --- | --- | --- |
| **Vyapar** | India SMB POS + GST + inventory | Barcode scan / search → product master |
| **BUSY** | India retail billing + accounting | Barcode billing, variants, multi-store |
| **GoFrugal** (RetailEasy, GoBill) | India retail / supermarket POS | USB, Bluetooth, or phone barcode → cart |
| **Marg ERP** | India GST billing + inventory + barcode labels | Scan or pick item, auto price + GST |
| **Petpooja / Posist** | India restaurant POS | Menu item tap / KOTs (food tray is usually not barcode) |
| **Shopify POS / Square** | Global retail POS | Barcode to variant, payments, stock |

These are the **software products** for “tray has many products → make a bill.” They already do product master, barcode, cart, invoice, inventory. Building another Vyapar-class app is a POS play, not a camera play.

### 2. Ready barcode software (many codes at once, still not custom AI)

If the goal is “do not scan each pack by hand” **without building YOLO**, license or embed software that already reads many barcodes:

| Software | What it does |
| --- | --- |
| **Scandit** (MatrixScan Batch / Count) | One view, all visible barcodes; Flutter/Android/iOS SDK |
| **Dynamsoft Barcode Reader / Batch Scanner** | Many barcodes in one pass / many frames |
| **Google ML Kit Barcode Scanning** | Free on-device; can return multiple barcodes in a frame |
| **ZXing / ZBar / OpenCV** | Open-source decode; more DIY |

This software still uses a phone or scanner imager, but **you do not write camera AI**. You get a list of barcode strings, then your POS looks them up — same as Vyapar.

### 3. Counter hardware + any POS (supermarket tray / platter)

Shops that pass items over a glass tray already use **bioptic scanners**, not a custom app:

| Hardware (works with POS software) | What it does |
| --- | --- |
| **Datalogic Magellan** (9300i / 9600i / 9900i) | In-counter platter; reads barcode as items move over the tray |
| **Zebra / Honeywell** handheld or presentation scanners | Beep one code into whatever POS is open |

The scanner types numbers into the POS (like a keyboard). **Vyapar, Marg, GoFrugal, or our FastAPI** can all receive that. Identity is still barcode → product table.

### 4. Full “put the tray, don’t scan” products (closed systems)

These exist, but they **are** camera/vision kiosks, not a Python POS you assemble:

| Product | Note |
| --- | --- |
| **Mashgin** | Tray/kiosk; 3D cameras identify items; integrates with existing POS; not a DIY stack |
| **Amazon Just Walk Out, Grabango, Trigo** | Whole-store cashierless; huge install, not an SMB billing app |

Do not try to recreate Mashgin in V1.

### What this means for us

| Goal | Software to use / copy | Custom camera? |
| --- | --- | --- |
| Bill a tray of known barcode products | Vyapar / Busy / GoFrugal / Marg pattern, or our FastAPI POS + USB gun | No |
| Many barcodes in one go | Scandit or ML Kit → same product lookup | No custom model |
| Identify packs with no barcode | Mashgin-class, or later our catalog vision | Yes, later only |

**Recommendation:** treat **existing POS software** as the baseline. V1 should be that class of product (product master + barcode + tray bill + inventory). For many items on one tray, use a **barcode scanner** (or Scandit/ML Kit). Do not start by building a camera product — those software products already own the shop counter.


## Architecture I recommend

Do not make this only:

```
Flutter → Python API → PostgreSQL
```

Use this:

```
                    ┌─────────────────┐
                    │   React Admin   │
                    │    Dashboard    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Python API    │
                    │    FastAPI      │
                    └────────┬────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
┌──────▼──────┐      ┌───────▼───────┐      ┌──────▼──────┐
│ PostgreSQL  │      │  AI Service   │      │ Redis/Queue │
│ + pgvector  │      │ Detection     │      │ Jobs        │
│ Products    │      │ Recognition   │      │ Processing  │
│ Inventory   │      │ OCR           │      │             │
│ Customers   │      │ Barcode       │      │             │
│ Billing     │      │               │      │             │
└─────────────┘      └───────────────┘      └─────────────┘
                             ▲
                             │
                    ┌────────┴────────┐
                    │     Flutter     │
                    │ Mobile / POS    │
                    └─────────────────┘
```

FastAPI stays one ecosystem for V1:

```
FastAPI
├── Authentication
├── Products API
├── Inventory API
├── Billing API
├── Customer API
├── Payment API
├── Barcode API
├── AI scanning API
└── Reports API
```

Split the AI service out later. Use workers/queues for vision jobs. Do not run YOLO inside the billing request that creates an invoice.

### Cloud shape (later)

```
Internet → Load balancer → FastAPI
                              ├── PostgreSQL + pgvector
                              ├── Redis
                              └── AI workers / GPU
```

### Hardware path

1. **MVP:** phone camera (Flutter) + USB / Bluetooth scanner  
2. **Shop:** fixed counter / rack camera  
3. **Differentiator:** smart checkout station + edge agent (do not stream every frame to the cloud)

---

## How we identify a product with any hardware

Hardware does **not** identify the product. Hardware only **captures a signal**. Identification always happens in software against the merchant **product master**.

That is why one FastAPI pipeline can support a cheap USB scanner today and a rack camera later without changing billing.

```
ANY HARDWARE
    → capture signal (barcode number, photo, video, RFID, weight)
    → normalize to a Scan Event
    → Identification pipeline
          1. Barcode lookup
          2. OCR / packaging text
          3. Image embedding + pgvector
          4. AI classification
          5. Human confirmation
    → Product ID + qty + confidence
    → Cart → Invoice
```

### One Scan Event, many devices

Every device should produce the same payload shape:

```text
Scan Event
  source        usb_scanner | bluetooth_scanner | mobile_camera | counter_camera | scale | rfid | typed
  barcodes[]    decoded EAN/UPC/QR strings (may be empty)
  image_url     photo/frame if a camera was used
  crops[]       optional object crops from a rack/cart image
  weight_grams  optional
  rfid_tags[]   optional
  branch_id
  device_id
```

Flutter, React, or an edge box only captures. FastAPI / AI workers identify.

### Hardware options

| Hardware | What it captures | How we identify | Accuracy | When to use |
| --- | --- | --- | --- | --- |
| **USB / Bluetooth barcode scanner** | One barcode number per beep | `GET /products/barcode/{code}` | Highest | V1 POS, every shop |
| **Phone camera (Flutter)** | Photo or live frames | Decode barcode(s) in the image; if none, crops → visual match | High if barcode visible | MVP camera path |
| **Fixed counter / rack camera** | Wide photo or video of many items | Detect all barcodes first; leftover objects → detection + embedding | High for barcodes, medium for vision | Phase 2–3 shop install |
| **Smart checkout station** | Overhead/side cameras + optional scale | Same pipeline on an **edge PC** (low latency, works if internet drops) | High when hybrid | Phase 5 |
| **Electronic scale** | Weight | Confirm quantity / catch mismatch after ID (e.g. 5 cans vs 2) | Support signal, not ID | Later |
| **RFID reader** | Tag IDs on tagged items | Map tag → product | Very high, expensive tagging | Optional, not V1 |
| **Keyboard / search** | Typed SKU, name, or barcode | Normal product search | Human | Fallback always |

We can accept **any** of these because identification is not tied to one gadget. If a new camera or scanner appears, we only add a capture adapter. Product matching stays the same.

### What happens on each capture

**1. Handheld scanner (simplest, V1)**

```
Gun beep → 8901234567890
    → product_barcodes table
    → Coke 500ml, ₹40
    → add to cart (confidence 1.00)
```

No AI. This must work even if cameras fail.

**2. Phone or counter camera, barcodes visible (Phase 2)**

```
One photo of the rack/cart
    → barcode detector finds many codes in the frame
    → 890…790, 890…791, 890…792, …
    → group identical codes as quantity
    → lookup each code in product master
    → cart: Coke × 5, Pepsi × 3, Lays × 4
```

This is the first “whole rack without scanning each item by hand” feature. Still barcode identity, just many codes at once.

**3. Camera, barcode hidden or not facing us (Phase 3)**

```
Photo
    → object detector (YOLO / RT-DETR): 14 boxes
    → for each box:
          try barcode in that crop
          else OCR brand/size text
          else embedding → pgvector top 5
    → confidence:
          ≥ 0.90 auto-add
          0.60–0.90 cashier confirms
          < 0.60 unknown queue
```

Example UI: “Coca Cola 500ml — 97% — Qty 4 — [Confirm] [Change]”

### Rules that keep this reliable on any hardware

1. **Same catalog for every device.** Scanner, phone, and rack camera all look up the same `products` / `product_barcodes` / `product_images`.  
2. **Barcode wins.** If a device returns a valid barcode, do not override it with AI.  
3. **Camera is optional.** A shop with only a ₹2,000 USB scanner can still bill.  
4. **Edge for video.** Live rack cameras should decode on a local box; the API receives barcodes/crops, not 30 fps video.  
5. **Always allow manual search.** Hardware will miss items. Cashier can type or pick the SKU.  
6. **Unknown is a feature.** Unreadable packs go to the review queue instead of a wrong bill.

### Recommended hardware by phase

| Phase | Device at the counter | Identity method |
| --- | --- | --- |
| V1 | USB/Bluetooth scanner + Flutter phone as backup | Single barcode lookup |
| V2 | Phone or cheap USB camera on a stand | Multi-barcode from one image |
| V3 | Fixed 1080p camera above the packing area | Barcode + object crops + visual match |
| V5 | Checkout station: camera(s) + optional scale + edge PC | Hybrid, mostly automatic |

Do not require special “AI cameras” in V1. A normal Android/iPhone camera and a normal barcode gun are enough to start.

---

## How we identify many products on one tray

The real billing scene is a **tray** (checkout tray / basket / flat plate) with **many mixed products** on it. One camera looks down at that tray. We still do not name the tray. We find **each product on the tray**, then count.

```
Customer / cashier puts items on the TRAY
            │
            ▼
     Camera above the tray (or phone)
            │
            ▼
   Find every product on the tray
            │
     14 packs = 14 boxes (or 14 barcodes)
            │
            ▼
   Identify EACH pack
     barcode → OCR → image match → confirm
            │
            ▼
   Same product on the tray → quantity
     Coke × 5, Pepsi × 3, Lays × 4, Biscuits × 2
            │
            ▼
          Bill
```

### Tray layout (what the camera sees)

```
              CAMERA
                 │
                 ▼
┌────────────────────────────────────────┐
│                 TRAY                   │
│                                        │
│   Coke   Coke   Pepsi   Lays   Lays    │
│   Coke   Pepsi  Lays    Biscuit        │
│   Coke   Pepsi  Lays    Biscuit        │
│   Coke                                 │
│                                        │
└────────────────────────────────────────┘
                 │
                 ▼
        one photo of the tray
```

Put packs **side by side, not stacked**. Leave a little gap so each pack is its own box. That is how one tray with 10–20 items can become a bill without scanning each barcode by hand.

This is **not** a supermarket aisle shelf. Aisle racks hide barcodes and stack depth. A checkout **tray** is the product we build first.

### Step 1 — Put products on the tray, take one photo

Hardware can be:

- Phone held above the tray (Flutter MVP)
- Cheap camera on a stand above the tray
- Later: fixed checkout station with a marked tray area

Flutter or the edge box sends **one tray image** (or a short burst) to the scan API. Billing does not care which camera it was.

### Step 2 — Find every product on the tray

| Finder | What it does on the tray | Phase |
| --- | --- | --- |
| **Multi-barcode** | Reads every barcode facing up | Phase 2 — first tray feature |
| **Object detector** | Draws a box on each pack (“14 products on this tray”) | Phase 3 |

Detection only answers: *how many things are on the tray, and where?* It does not yet say Coke or Pepsi.

### Step 3 — Identify each pack on the tray

For each box / barcode:

1. Barcode on that pack → product master (best)  
2. Else OCR (brand, 500ml)  
3. Else crop photo vs catalog images (pgvector)  
4. High score → add; medium → cashier confirms; low → unknown queue  

### Step 4 — Tray becomes quantity lines

Five Coke boxes on the same tray become **Coke 500ml × 5**, not five separate mystery items.

```
Tray boxes          →  Bill lines
Coke, Coke, Coke,
Coke, Coke          →  Coke 500ml     × 5
Pepsi, Pepsi, Pepsi →  Pepsi 500ml    × 3
Lays × 4            →  Lays           × 4
Biscuit × 2         →  Biscuits       × 2
                    →  ₹ Total
```

### Worked tray example

Tray has 14 products. One photo.

1. 10 barcodes face the camera → those 10 are identified immediately.  
2. Detector still sees 14 packs.  
3. Remaining 4: visual match (2 auto-add, 1 confirm, 1 unknown).  
4. Cashier confirms the weak one.  
5. Bill is ready. No 14 hand scans.

### Tray rules (so identity stays correct)

1. One layer on the tray — do not pile packs on top of each other.  
2. Same catalog for every tray scan (100–500 enrolled SKUs).  
3. Barcode wins when it is visible on that pack.  
4. Each line has its own confidence, not one score for the whole tray.  
5. Cashier can always add/remove a line if a pack was hidden at the edge.

First tray feature: **one tray photo → many barcodes → grouped bill**.  
Then: packs on the tray with no visible barcode → crop → visual match.

---

## Billing workflow

```
START BILL
    → Select customer (or walk-in)
    → Scan products
          ├── Barcode (single)
          └── AI / bulk scan
                → Detect objects
                → Identify products
                → Calculate quantity
                → Confidence > threshold?
                      YES → add item
                      NO  → manual confirm
    → CART
    → PAYMENT
    → INVOICE
    → Reduce inventory (ledger)
```

---

## PostgreSQL schema (V1)

Multi-tenant from day one. Every business table has `tenant_id`. Store operations also have `branch_id`.

A tenant is the company / SaaS account. A branch is a store (Chennai, Bangalore, Madurai).

### Tenancy

```text
tenants
  id, name, slug, status, created_at, updated_at

branches
  id, tenant_id, name, code, city, address, status, created_at, updated_at

users
  id, tenant_id, branch_id (nullable), email, full_name,
  hashed_password, role, is_active, created_at, updated_at
```

Roles: `owner` | `manager` | `cashier` | `admin`.

### Product master (must be strong)

```text
categories
  id, tenant_id, name, created_at

brands
  id, tenant_id, name, created_at

products
  id
  tenant_id
  sku
  name
  brand_id
  category_id
  unit
  mrp
  selling_price
  purchase_price
  tax_id / tax_rate
  hsn_code
  reorder_level
  status
  created_at
  updated_at

product_barcodes
  id, tenant_id, product_id, barcode, is_primary
  UNIQUE (tenant_id, barcode)

product_images
  id, product_id, image_url, image_type, embedding, created_at
```

A real store often has **several barcodes for one SKU**. Support that from V1.

`product_images.embedding` is for later visual matching with **pgvector**. Image types: `front`, `side`, `back`, `packaging`.

Visual match later:

```
Detected crop → embedding → pgvector similarity → top 5 SKUs
  1. Coca Cola 500ml  94%
  2. Coca Cola 750ml  82%
  3. Pepsi 500ml      63%
→ UI: “Is this Coca Cola 500ml?”
```

That is better than one giant classification model.

### Inventory ledger (do not do `stock = stock - 1` only)

```text
inventory_balances
  id, tenant_id, branch_id, product_id, quantity, updated_at
  UNIQUE (tenant_id, branch_id, product_id)

inventory_transactions
  id, tenant_id, branch_id, product_id
  type              -- purchase | sale | return | adjustment | damage | stock_in | stock_out
  quantity_delta    -- signed
  unit_cost
  reference_type, reference_id
  notes, created_by, created_at
```

Current stock = balance row, auditable as the sum of deltas.

### Customers and billing

```text
customers
  id, tenant_id, name, phone, email, gstin, address, created_at, updated_at

sales
  id, tenant_id, branch_id, bill_number, customer_id, cashier_id,
  scan_session_id, status,          -- draft | confirmed | paid | void
  subtotal, discount_amount, tax_amount, total,
  notes, created_at, updated_at, paid_at

sale_items
  id, sale_id, product_id, barcode, name_snapshot,
  quantity, unit_price, tax_rate, line_total,
  identification_method,   -- barcode | ocr | visual | classification | manual
  confidence, confirmed

payments
  id, sale_id, method, amount, reference, created_at
```

### Scanning (AI-ready in V1, vision later)

```text
scan_sessions
  id, tenant_id, branch_id, sale_id
  mode      -- barcode | bulk_barcode | ai_vision
  status    -- processing | review | applied | cancelled
  source    -- typed | scanner | mobile_camera | counter_camera | upload
  image_url, created_by, created_at, updated_at

scan_results
  id, session_id, detected_barcode, product_id, product_name,
  quantity, confidence, identification_method,
  status            -- auto_added | needs_confirm | unknown
  crop_image_url, candidates_json

ai_review_items     -- Unknown Product Queue
  id, tenant_id, scan_result_id, crop_image_url,
  suggested_matches_json,
  status            -- pending | confirmed | created | dismissed
  resolved_product_id, created_at, resolved_at
```

### Confidence rules

| Signal | Score | Action |
| --- | --- | --- |
| Exact barcode match | 1.00 | Auto-add |
| Embedding top-1 ≥ 0.90 | model | Auto-add |
| 0.60–0.90 | model | Ask cashier |
| < 0.60 or no match | — | Unknown queue |

Config: `AUTO_ADD_CONFIDENCE` (default 0.90), `CONFIRM_CONFIDENCE` (default 0.60).

---

## FastAPI folder structure (V1)

```text
backend/
  app/
    main.py
    api/
      auth.py
      products.py
      billing.py
      inventory.py
      customers.py
      stores.py
      scanning.py
      review_queue.py
      reports.py
    models/
      identity.py
      catalog.py
      inventory.py
      billing.py
      scanning.py
    schemas/
    services/
      billing_service.py
      inventory_service.py
      barcode_service.py
      recognition_service.py    # barcode now; OCR/vision later
    repositories/
    core/
      config.py
      security.py
      database.py
  tests/
```

Keep `recognition_service.py` as a pipeline with stages 1–5. V1 implements barcode only. Later stages plug in without rebuilding cart or invoice.

---

## API map (V1)

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/auth/login` | JWT + refresh |
| POST | `/auth/refresh` | Rotate access token |
| GET | `/auth/me` | User, tenant, branch |
| GET | `/dashboard/summary` | Today sales, bills, low stock, pending reviews |
| GET/POST | `/products` | Product master |
| GET | `/products/barcode/{code}` | Single-barcode lookup |
| GET/POST | `/inventory`, `/inventory/adjust` | Balances + ledger |
| GET/POST | `/customers` | Customer master |
| POST | `/bills` | Create draft bill |
| POST | `/bills/{id}/items` | Add line (barcode or product id) |
| POST | `/bills/{id}/pay` | Payment + complete + stock out |
| GET | `/bills` | History / invoice |
| POST | `/scans` | Start session; optional barcode list |
| POST | `/scans/{id}/barcodes` | Bulk barcodes (Phase 1/2 stand-in for one rack photo) |
| POST | `/scans/{id}/image` | Phase 2: multi-barcode from photo (not V1) |
| POST | `/scans/{id}/apply` | Push confirmed lines into the draft bill |
| GET | `/review-queue` | Unknown products |
| POST | `/review-queue/{id}/confirm` | Link to existing SKU |
| POST | `/review-queue/{id}/create-product` | Create SKU from unknown crop |

---

## React admin pages (V1)

React is the business / admin side. V1 can also demo billing here so the product is usable before Flutter is compiled.

```text
Login
Dashboard
│
├── Sales
│   ├── Today
│   ├── New Bill
│   ├── Bills
│   ├── Returns
│   └── Payments
│
├── Products
│   ├── Product Master
│   ├── Categories
│   ├── Brands
│   ├── Barcode
│   └── Product Images
│
├── Inventory
│   ├── Stock
│   ├── Stock In
│   ├── Stock Out
│   └── Stock Adjustment
│
├── AI Scanner
│   ├── Bulk Scan
│   ├── Training Data
│   ├── Product Matching
│   ├── Accuracy
│   └── Unrecognized Products   ← Unknown Product Queue
│
├── Customers
├── Suppliers                   ← stub in V1
├── Reports
└── Settings
    ├── Branch
    └── Users
```

**New Bill:** walk-in or named customer → add by barcode → optional bulk-scan session → pay cash / UPI / card → invoice.

**Bulk Scan:** paste or scan many barcodes (simulates one rack photo). Group quantities. Flag unknowns.

**Unknown Product Queue:**

```text
Unknown Product
[ crop image ]
Possible matches:
  Coca Cola 500ml   93%
  Coca Cola 750ml   81%
  Pepsi 500ml       64%
[ Confirm ]  [ Create Product ]
```

Every correction becomes catalog / training data.

---

## Flutter screens (V1)

Flutter is the **store staff / POS** app, not only a billing screen.

```text
Login
Dashboard
New Bill
  ├── Barcode Scan
  ├── AI Bulk Scan
  └── Cart
Payment
Invoice
Returns
Stock
  ├── Check Stock
  ├── Stock Count
  ├── Stock Transfer
  └── Stock Adjustment
Customers
Sales History
Profile
```

Later: customer mode — customer scans the cart → sees the bill → UPI / card → checkout.

Later still: owner app — today’s sales, profit, top products, low stock, store comparison.

Offline queue (SQLite sync) is important for real shops. **Not V1.** Design APIs so a local queue can replay bills later.

---

## AI stack (when we reach it)

There are three different problems. Do not mix them.

1. **Barcode detection** — find `8901234567890` in the frame → product master. Easiest and most reliable. This is Phase 2.  
2. **Product detection** — YOLO / RT-DETR: “there are 5 objects.”  
3. **Product recognition** — “these are Coke 500ml.” Hard. Use embeddings + pgvector against the merchant catalog, not a global classifier.

OCR: PaddleOCR or equivalent.  
Barcode: ZXing / ML Kit / OpenCV.  
Do not send every camera frame to the cloud. Prefer an edge agent for the counter camera.

---

## Recommended stack

| Component | Recommendation |
| --- | --- |
| Backend | Python + FastAPI |
| ORM | SQLAlchemy |
| Database | PostgreSQL |
| Vector | pgvector |
| Cache | Redis |
| Queue | Celery / RQ |
| Web | React.js + TypeScript |
| Mobile | Flutter + Dart |
| Detection | YOLO-class / RT-DETR (Phase 3) |
| OCR | PaddleOCR (Phase 3) |
| Barcode | ZXing / ML Kit / OpenCV (Phase 2) |
| Images | S3-compatible storage (not BLOBs in Postgres) |
| Auth | JWT + refresh token |
| API docs | OpenAPI / Swagger |
| Deploy | Docker |
| CI | GitHub Actions |
| Monitoring | Sentry + metrics |

---

## V1 scope vs later

### Build in V1

- Login, tenant, branch, users  
- Product master, barcodes, images (upload URL / placeholder)  
- Inventory ledger  
- Cart, billing, payment, invoice  
- Single barcode lookup  
- Bulk barcode **list** → grouped cart (API ready for a camera)  
- Confidence fields + unknown-product queue (even if only unmatched barcodes hit it)  
- React admin pages listed above  
- Flutter screen map (implement after API is stable)

### Do not build in V1

- Full AI product recognition  
- YOLO / GPU workers  
- RFID  
- Self-checkout kiosk  
- Loyalty, payroll, accounting ERP  
- Advanced GST automation  
- Many payment gateways  
- Offline sync  
- “Any product in any supermarket”

---

## Roadmap

```text
                    PRODUCT
                       │
          ┌────────────┴────────────┐
          │                         │
       POS / Billing            AI Checkout
          │                         │
     Barcode Scan             Bulk barcode scan
          │                         │
     Inventory                Object detection
          │                         │
     Customers                Product recognition
          │                         │
     Payments                 Quantity detection
          │                         │
     Reports                  Confidence engine
          │                         │
          └────────────┬────────────┘
                       │
                  SaaS platform
                       │
            Retail · Grocery · Supermarket
```

| Phase | Ship |
| --- | --- |
| 1 | POS + inventory + barcode billing |
| 2 | Camera → multiple barcodes → automatic cart |
| 3 | Detect + recognize known catalog products when barcode is hidden |
| 4 | Quantity + confidence confirmation UX |
| 5 | Automated checkout station |
| 6 | Multi-store SaaS + analytics |

---

## What to do next (after this plan)

When you want implementation, do it in this order:

1. FastAPI + PostgreSQL schema + seed catalog (100 demo SKUs is enough to think in)  
2. React: login, products, new bill, barcode add, pay  
3. Bulk barcode session + unknown queue UI  
4. Flutter: login, new bill, barcode scan, cart, pay  
5. Only then: photo → multi-barcode decode  
6. Only then: embeddings + pgvector + detector

That is the full V1 design (architecture, schema, API layout, React pages, Flutter screens) before coding.
