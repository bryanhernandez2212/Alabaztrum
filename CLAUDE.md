# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ALABAZTRUM is a Flask-based e-commerce web app for a fragrance/perfume retailer (Mexico). It uses server-side Jinja2 templates with client-side Firebase for auth, database, and storage, plus Stripe for payments.

## Running the App

```bash
# Development
python app.py          # Runs on http://127.0.0.1:5001

# Production (Railway via Procfile)
gunicorn --config gunicorn_config.py app:app
```

No build step required. Python 3.12.0 (see `runtime.txt`).

## Architecture

### Backend (`app.py`)
Flask serves 40+ routes — all pages are server-rendered (returning HTML templates) except a few JSON API endpoints:
- `POST /api/create-payment-intent` — creates a Stripe PaymentIntent in MXN
- `POST /api/upload-transfer-proof` — saves uploaded file to `static/uploads/transfer_proofs/` with UUID filename

All data reads/writes happen client-side via Firebase SDK in the browser. The Flask backend does **not** query Firestore directly (except for the Firebase Admin SDK used for file uploads).

### Frontend Architecture
Client-side state is managed by a set of JS modules in `static/js/`:

- **`firebase-config.js`** — initializes Firebase app, exports `auth`, `db`, `storage` singletons
- **`auth-manager.js`** — singleton that listens to `onAuthStateChanged`, caches `currentUser`/`userRole`/`userData` in `sessionStorage`, and notifies observers. Every page that checks auth state imports from here.
- **`auth.js`** — sign-up/sign-in/sign-out functions
- **`cart.js`** — all cart logic: Firestore collection `carts/{userId}`, functions `getCart()`, `addToCart()`, `removeFromCart()`, `updateCartCount()`
- **`admin.js`** — dashboard stats and user-table loading; checks `role === 'administrador'` before showing admin UI

Templates include these scripts via `<script>` tags at the bottom of each page. There is no bundler — scripts are loaded individually from CDN or `static/js/`.

### Firestore Data Model
| Collection | Access | Key fields |
|---|---|---|
| `products` | Public read | name, brand, price, images[], stock, gender, fragranceType, isDecant |
| `brands` | Public read | name, logo |
| `fragranceTypes` | Public read | name |
| `users` | Owner read/write | fullName, email, role (`cliente`\|`administrador`) |
| `carts` | Owner only | items[{productId, name, price, quantity, image}], updatedAt |
| `orders` | Owner read, admin write | userId, items[], total, status, trackingInfo, createdAt |
| `favorites` | Owner only | userId, productId |
| `comments` | Owner read, public if approved | userId, productId, text, rating, approved |
| `siteReviews` | Owner read, public if approved | userId, text, rating, approved |
| `messages` | Write-only for users, admin read | status, content, createdAt |
| `config` | Admin only | payment config and site settings |

Security rules are in `firestore.rules`. Admin access is gated by the `isAdmin()` helper which checks `users/{uid}.role === 'administrador'`.

### Template Structure
- `templates/partials/` — shared navbar, admin navbar, breadcrumbs, footer (included via `{% include %}`)
- `templates/auth/` — login, register
- `templates/perfume/` — product catalog pages and detail view
- `templates/admin/` — 20 admin-only pages (dashboard, product CRUD, order management, POS, moderation)
- Top-level templates — carrito, checkout, profile, mis_compras, buscar

The breadcrumb data is generated in `app.py` and passed to templates as a `breadcrumbs` context variable.

## Key Dependencies

**Python** (`requirements.txt`): Flask 3.0, firebase-admin 6.4, stripe ≥8.0, gunicorn 21.2, python-dotenv  
**Frontend (CDN)**: Firebase SDK 10.7.1, Tailwind CSS 2.2.19, Font Awesome 6.4, Stripe.js

## Environment Variables

Defined in `.env` (not committed — contains Stripe test keys):
- `STRIPE_PUBLISHABLE_KEY`
- `STRIPE_SECRET_KEY`

Firebase credentials come from `serviceAccountKey.json` (also gitignored) or environment-injected config in `config.py`.
