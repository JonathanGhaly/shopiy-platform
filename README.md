# Shopiy: Order Management and Admin Dashboard 
**Candidate Assessment Architecture & System Design Document**

## Table of Contents 
1. [Requirements](#requirements)
2. [Database Design & Schema](#database-design-&-schema) 
3. [Backend Architecture](#backend-architecture) 
4. [System Design Scenario: Duplicate Order Prevention](#system-design-scenario-duplicate-order-prevention) 
5. [Frontend Architecture & CORS](#frontend-architecture--cors) 
6. [Docker Setup & CI/CD](#docker-setup--cicd) 
7. [Quality Assurance & Testing Plan](#quality-assurance--testing-plan)
8. [Logging & Observability Strategy](#logging--observability-strategy)

---

# Requirements

## User

### Functional

- **Browse Product Catalog:** Users can view a paginated list of all active products, with the ability to sort by price and creation date.

- **View Product Profiles:** Users can view a detailed page for any individual product, showcasing descriptions, dynamic metadata attributes (e.g., sizes, colors), and real-time inventory levels.

- **Checkout & Order Creation:** Authenticated users with verified emails can convert items into a formal order by providing verified shipping and billing address JSON structures.

- **Order Confirmation:** Upon successful payment or order entry, users receive an immutable confirmation summary containing their unique Order UUID, receipt calculations, and a delivery tracking status placeholder.

- **Purchase History:** Users can access a personal dashboard displaying all past orders linked to their account ID, along with detailed line-item breakdowns for each invoice.
### Non-Functional

- **Performance (Latency):** API endpoints for catalog browsing, searching, and viewing product details must respond in under 100ms (P95) to ensure a fluid, bounce-resistant shopping experience.
    
- **Scalability (Concurrency):** The checkout and inventory systems must be able to handle traffic spikes (e.g., flash sales, holiday traffic) without degrading performance or dropping connections.
    
- **Security (Data Protection):** * Customer passwords must be hashed using PBKDF2 (ASP.NET Identity default).
    
    - Authentication state must be maintained securely using short-lived JWTs and `HttpOnly`, `SameSite=Strict` refresh cookies to prevent XSS and CSRF attacks.
        
    - All data in transit must be encrypted via TLS 1.2/1.3.
        
- **Availability (Uptime):** The storefront API must maintain an uptime of 99.9%, ensuring customers can browse and place orders 24/7.
    
- **Usability & Accessibility:** The React frontend must be fully responsive (mobile, tablet, desktop) and adhere to WCAG 2.1 Level AA standards to ensure accessibility for users with disabilities (e.g., screen-reader support, keyboard navigation).
## Admin
### Functional

- **Secure Dashboard Authentication:** Authorized administrative or manager accounts can securely log in via specialized claim roles managed by ASP.NET Core Identity.

- **Global Order Oversight:** Administrators can view a comprehensive master table of all platform-wide customer orders, displaying real-time financial metrics and fulfillment statuses.

- **Status Pipeline Filtering:** Admins can quickly filter order volumes using specific status categories (`pending`, `paid`, `shipped`, `delivered`, `cancelled`, `refunded`) to manage active fulfillment workloads.

- **Order Profile Diagnostics:** Admins can drill down into any individual order ID to review absolute line-item prices, localized address records, customer checkout notes, and fulfillment historical timestamps.

- **Status Lifecycles Updates:** Admins can modify an order's fulfillment state. Transitioning an order status to `paid` triggers atomic inventory adjustments, while changing a status to `shipped` permanently restricts any subsequent cancellation requests.
### Non-Functional

- **Performance (Query Segregation):** Heavy administrative data queries (e.g., paginating thousands of global orders, generating financial summaries) must be routed to a **Database Read Replica**. This ensures admin tasks never consume resources needed by the primary database to process live customer checkouts.
    
- **Security (RBAC & Auditability):** * Admin API endpoints must strictly validate JWT claims for `Manager` or `Administrator` roles at the gateway layer before processing requests.
    
    - **Audit Trail:** All destructive or critical state changes (e.g., cancelling an order, manually overriding stock quantities, refunding payments) must be logged with the Admin's UUID, action taken, and an immutable UTC timestamp.
        
- **Data Integrity (Concurrency Controls):** Stock quantity updates and order status transitions must be executed as atomic database transactions (using PostgreSQL row-level locking via `SELECT ... FOR UPDATE`). This prevents race conditions if two admins attempt to process the same order simultaneously.
    
- **Session Management:** Admin dashboard sessions must enforce strict inactivity timeouts. For security compliance, the dashboard must automatically log out or lock the screen after 15 minutes of idle time.
    
- **Resilience (Bulk Operations):** If admins perform bulk actions (e.g., batch-updating the status of 100 orders), the API must handle the request asynchronously or via chunking to prevent browser timeouts and ensure partial successes are recorded if one item fails.

---
# Database Design & Schema
The system utilizes PostgreSQL 17.
## User Authentication Schema

**Use ASP.NET Core Identity APIs (Recommended)**
ASP.NET handle the Identity setup, it will automatically generate tables prefixed with `AspNet` (e.g., `AspNetUsers`, `AspNetRoles`).
### Business Rules:
- `email` is globally unique, matched case-insensitively, and serves as the primary username.
- `password_hash` is null for third-party OAuth users (e.g., Google, GitHub).
- `email_verified_at` must be set before a user can place an order or complete checkout.
- Users are never hard-deleted from the database, only deactivated via `deleted_at`.
- Password: 8-128 chars, requires uppercase + lowercase + digit + special character, hashed via ASP.NET Identity default (PBKDF2 with HMAC-SHA256, 100,000 iterations).
- Account locks automatically for 15 minutes after 5 consecutive failed login attempts.
- Access Token (JWT) is valid for exactly 15 minutes.
- Refresh Token must be stored in a secure, `HttpOnly`, `SameSite=Strict`, HTTPS-only cookie, expiring in 7 days.
- Refresh Tokens use automatic token rotation; reusing an old refresh token instantly revokes the entire token family.

---
## E-Commerce Schema

Product catalog, orders, and payment tracking.
### Design
#### Products:

| Column         | Type         | Nullable | Unique | Default           | Description                        |
| -------------- | ------------ | -------- | ------ | ----------------- | ---------------------------------- |
| id             | UUID         | No       | Yes    | gen_random_uuid() | Primary key, auto-generated        |
| name           | VARCHAR(300) | No       | No     | -                 | Display name of the organization   |
| slug           | VARCHAR(300) | No       | Yes    | -                 | URL-friendly identifier            |
| description    | TEXT         | No       | No     | -                 |                                    |
| price          | INTEGER      | No       | No     | -                 | Price in cents/piasters            |
| currency       | VARCHAR(3)   | No       | No     | 'EGP'             |                                    |
| sku            | VARCHAR(255) | Yes      | Yes    | -                 | Stock Keeping Unit                 |
| stock_quantity | INTEGER      | Yes      | No     | 0                 |                                    |
| is_active      | BOOLEAN      | Yes      | No     | true              |                                    |
| metadata       | JSONB        | No       | No     | '{}'              | Extra attributes (color, size)     |
| created_at     | TIMESTAMP    | No       | No     | CURRENT_TIMESTAMP | Record creation time (UTC)         |
| updated_at     | TIMESTAMP    | Yes      | No     | CURRENT_TIMESTAMP | Last modification time (UTC)       |
| deleted_at     | TIMESTAMP    | Yes      | No     | -                 | Soft delete marker (null = active) |
#### Categories:

| Column     | Type         | Nullable | Unique | Default           | Description                                  |
| ---------- | ------------ | -------- | ------ | ----------------- | -------------------------------------------- |
| id         | UUID         | No       | Yes    | gen_random_uuid() | Primary key, auto-generated                  |
| name       | VARCHAR(200) | No       | No     | -                 | Display name of the organization             |
| slug       | VARCHAR(200) | No       | Yes    | -                 |                                              |
| parent_id  | TEXT         | No       | No     | -                 | References categories(id) for sub-categories |
| created_by |              | No       | No     | -                 | References users(id)                         |
| created_at | TIMESTAMP    | No       | No     | CURRENT_TIMESTAMP | Record creation time (UTC)                   |
| updated_at | TIMESTAMP    | Yes      | No     | CURRENT_TIMESTAMP | Last modification time (UTC)                 |
| deleted_at | TIMESTAMP    | Yes      | No     | -                 | Soft delete marker (null = active)           |
| sort_order | INTEGER      | No       | No     | 0                 | Display order                                |
####  Product Category:

| Column      | Type | Nullable | Unique | Default | Description                      |
| ----------- | ---- | -------- | ------ | ------- | -------------------------------- |
| product_id  | UUID | No       | No     | -       | Foreign key, to products table   |
| category_id | UUID | No       | No     | -       | Foreign key, to categories table |

Primary key is a composite of (product_id, category_id)

#### Orders:

| Column           | Type        | Nullable | Unique | Default           | Description                                                        |
| ---------------- | ----------- | -------- | ------ | ----------------- | ------------------------------------------------------------------ |
| id               | UUID        | No       | Yes    | gen_random_uuid() | Primary key, auto-generated                                        |
| user_id          | UUID        | No       | No     | -                 | References users(id)                                               |
| status           | VARCHAR(20) | No       | No     | 'pending'         | 'pending', 'paid', 'shipped', 'delivered', 'cancelled', 'refunded' |
| subtotal         | INTEGER     | No       | No     | -                 | Sum of item totals                                                 |
| tax              | INTEGER     | No       | No     | 0                 |                                                                    |
| shipping         | INTEGER     | No       | No     | 0                 |                                                                    |
| total            | INTEGER     | No       | No     | -                 | subtotal + tax + shipping                                          |
| currency         | VARCHAR(3)  | No       | No     | 'EGP'             |                                                                    |
| shipping_address | JSONB       | No       | No     |                   | Snapshot of shipping details                                       |
| billing_address  | JSONB       | No       | No     |                   | Snapshot of billing details                                        |
| notes            | TEXT        | Yes      | No     |                   | Customer or admin notes                                            |
| placed_at        | TIMESTAMP   | No       | No     | CURRENT_TIMESTAMP |                                                                    |
| shipped_at       | TIMESTAMP   | Yes      | No     | -                 |                                                                    |
| delivered_at     | TIMESTAMP   | Yes      | No     | -                 |                                                                    |


#### Order Items:

| Column     | Type    | Nullable | Unique | Default           | Description                 |
| ---------- | ------- | -------- | ------ | ----------------- | --------------------------- |
| id         | UUID    | No       | Yes    | gen_random_uuid() | Primary key, auto-generated |
| order_id   | UUID    | No       | No     |                   | References orders(id)       |
| product_id | UUID    | No       | No     |                   | References products(id)     |
| quantity   | INTEGER | No       | No     | 1                 |                             |
| unit_price | INTEGER | No       | No     |                   | Price at time of purchase   |
| total      | INTEGER | No       | No     |                   | quantity * unit_price       |


--- 
### Schema

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(300) NOT NULL,
  slug VARCHAR(300) UNIQUE NOT NULL,
  description TEXT,
  price INTEGER NOT NULL CHECK (price >= 0),  -- Store as cents
  currency VARCHAR(3) DEFAULT 'EGP',
  sku VARCHAR(100) UNIQUE,
  stock_quantity INTEGER DEFAULT 0 CHECK (stock_quantity >= 0),
  is_active BOOLEAN DEFAULT true,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP 
);

CREATE TABLE categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(200) NOT NULL,
  slug VARCHAR(200) UNIQUE NOT NULL,
  parent_id UUID REFERENCES categories(id),  -- Self-referencing hierarchy
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  sort_order INTEGER DEFAULT 0
);

CREATE TABLE product_categories (
  product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
  PRIMARY KEY (product_id, category_id)
);

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  status VARCHAR(20) DEFAULT 'pending'
    CHECK (status IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled', 'refunded')),
  subtotal INTEGER NOT NULL,
  tax INTEGER NOT NULL DEFAULT 0,
  shipping INTEGER NOT NULL DEFAULT 0,
  total INTEGER NOT NULL,
  currency VARCHAR(3) DEFAULT 'EGP',
  shipping_address JSONB NOT NULL,
  billing_address JSONB NOT NULL,
  notes TEXT,
  placed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  shipped_at TIMESTAMP,
  delivered_at TIMESTAMP
);

CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  quantity INTEGER NOT NULL CHECK (quantity > 0),
  unit_price INTEGER NOT NULL,  -- Price at time of purchase
  total INTEGER NOT NULL
);

-- Product & Category relationships
CREATE INDEX idx_categories_parent_id ON categories(parent_id);
CREATE INDEX idx_product_categories_category_id ON product_categories(category_id);

-- Order lookups by User
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Line item lookups
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
-- For "Sort by Price: Low to High / High to Low" 
CREATE INDEX idx_products_price ON products(price); 
-- For "New Arrivals" sorting 
CREATE INDEX idx_products_created_at ON products(created_at DESC); 
-- For displaying categories in the correct order in your nav menu
CREATE INDEX idx_categories_sort_order ON categories(sort_order);

-- Only indexes products that are actually active, saving disk space and memory
CREATE INDEX idx_products_active ON products(id) WHERE is_active = true AND deleted_at IS NULL;

-- Instantly find orders that need to be fulfilled or paid
CREATE INDEX idx_orders_status_pending ON orders(status) WHERE status = 'pending';

-- Allows fast querying inside the JSONB object
CREATE INDEX idx_products_metadata_gin ON products USING GIN (metadata);
```

**Key business rules**: Store all monetary values as integers in cents to avoid floating-point rounding errors. `order_items.unit_price` captures the price at time of purchase - products may change price later. Decrement `stock_quantity` atomically when order status changes to 'paid'. Orders cannot be cancelled after status changes to 'shipped'. Categories support nesting via `parent_id` for hierarchical product navigation.
## Database Connection Configuration

In the local development environment, database connections utilize Docker-compose virtual bridge networks. Connection parameters are securely injected at runtime using environment variables.

- **Primary Connection String:** Maps dynamically inside the API container to: `Host=postgres; Port=5432; Database=${POSTGRES_DB}; Username=${POSTGRES_USER}; Password=${POSTGRES_PASSWORD}`
    
- **Topology:** The local stack targets a single-node PostgreSQL 17 engine instance (`shopiy-postgres`).
## Entity Relationships

```
users (AspNetUsers root)
  |
  |-- 1:N --> orders --> 1:N --> order_items --> N:1 --> products
  |                                                       ^
  |-- 1:N --> categories (created_by)                     |
                |                                         |
                |-- 1:N --> categories (sub-categories)   |
                |                                         |
                |-- 1:N --> product_categories <-- 1:N ---|
```

| Parent     | Child              | FK Column   | Cardinality | ON DELETE | Notes                                                                 |
| ---------- | ------------------ | ----------- | ----------- | --------- | --------------------------------------------------------------------- |
| users      | orders             | user_id     | 1:N         | RESTRICT  | Cannot delete a user if they have active or past orders.              |
| orders     | order_items        | order_id    | 1:N         | CASCADE   | Deleting an order purges its associated line items.                   |
| products   | order_items        | product_id  | 1:N         | RESTRICT  | Cannot delete a product that has been ordered (historical integrity). |
| products   | product_categories | product_id  | 1:N         | CASCADE   | Deleting a product removes its category assignments.                  |
| categories | product_categories | category_id | 1:N         | CASCADE   | Deleting a category breaks associations with its products.            |
| categories | categories         | parent_id   | 1:N         | RESTRICT  | Cannot delete a parent category containing active sub-categories.     |
| users      | categories         | created_by  | 1:N         | RESTRICT  | Prevents deletion of admin/staff accounts who managed categories.     |


## Index Inventory

| Index Name                           | Table              | Columns         | Type   | Partial Filter                                  | Optimizes                                                     |
| ------------------------------------ | ------------------ | --------------- | ------ | ----------------------------------------------- | ------------------------------------------------------------- |
| `idx_categories_parent_id`           | categories         | parent_id       | B-Tree | None                                            | Hierarchical sub-category lookups.                            |
| `idx_product_categories_category_id` | product_categories | category_id     | B-Tree | None                                            | Retrieving all products linked to a targeted category.        |
| `idx_orders_user_id`                 | orders             | user_id         | B-Tree | None                                            | Loading customer purchase history profiles.                   |
| `idx_order_items_order_id`           | order_items        | order_id        | B-Tree | None                                            | Line-item retrieval during invoice generation.                |
| `idx_order_items_product_id`         | order_items        | product_id      | B-Tree | None                                            | Internal product performance metrics and sales metrics.       |
| `idx_products_price`                 | products           | price           | B-Tree | None                                            | High-to-low and low-to-high price catalog sorting.            |
| `idx_products_created_at`            | products           | created_at DESC | B-Tree | None                                            | Frontend sorting for "New Arrivals".                          |
| `idx_categories_sort_order`          | categories         | sort_order      | B-Tree | None                                            | Sequential layout of the frontend navigation menu.            |
| `idx_products_active`                | products           | id              | B-Tree | `WHERE is_active = true AND deleted_at IS NULL` | Live marketplace product lookups (excl. hidden/soft-deleted). |
| `idx_orders_status_pending`          | orders             | status          | B-Tree | `WHERE status = 'pending'`                      | Merchant dashboard pipelines for unfulfilled orders.          |
| `idx_products_metadata_gin`          | products           | metadata        | GIN    | None                                            | Dynamic filtering by custom attributes (size, color, brand).  |


## Query Performance Targets

| Query Pattern                                                         | Target | Index Used                                               |
| --------------------------------------------------------------------- | ------ | -------------------------------------------------------- |
| Authenticate user & load roles by unique email                        | < 2ms  | Implicit primary/unique index on email                   |
| Fetch nested sub-categories for a parent category navigation          | < 2ms  | idx_categories_parent_id                                 |
| List active marketplace products assigned to a category               | < 5ms  | idx_product_categories_category_id + idx_products_active |
| Sort active product marketplace by ascending/descending price         | < 5ms  | idx_products_price                                       |
| Sort active product marketplace by newest additions                   | < 5ms  | idx_products_created_at                                  |
| Filter dynamic attributes (e.g., find all products where color = red) | < 10ms | idx_products_metadata_gin                                |
| Fetch historical list of orders placed by a specific user             | < 5ms  | idx_orders_user_id                                       |
| Load detailed summary of an order containing all line-items           | < 3ms  | idx_order_items_order_id                                 |
| Aggregate unfulfilled order pipeline for administrative processing    | < 5ms  | idx_orders_status_pending                                |

---
# API Endpoints

The API is fully documented via OpenAPI/Swagger 3.0.4. It is versioned (`/api/v1/`) and utilizes standard HTTP methods, status codes, and JWT Bearer token authentication.
### Authentication (`/api/v1/auth`)

Authentication is handled via ASP.NET Core Identity. All endpoints return standardized `ProblemDetails` (RFC 7807) on `400 Bad Request` or `401/403 Unauthorized` errors.

* **`POST /api/v1/auth/register`**: Registers a new user. 
	* **Payload:** `{ "fullName": "string", "email": "string", "password": "...", "confirmPassword": "..." }` 
	* **Response (201):** `AuthResponse` containing `accessToken`, `refreshToken`, `expiresAt`, and `UserDto`. 
* **`POST /api/v1/auth/login`**: Authenticates a user and issues a JWT. 
	* **Payload:** `{ "email": "string", "password": "string" }`
	* **Response (200):** `AuthResponse`. 
* **`POST /api/v1/auth/refresh`**: Refreshes an expired JWT using a valid refresh token (usually passed securely via cookies). 
	* **`POST /api/v1/auth/logout`**: Invalidates the current session/tokens (Returns `204 No Content`).
### Categories (`/api/v1/Categories`)
* **`GET /api/v1/Categories`**: Retrieves the category tree. 
* **`POST /api/v1/Categories`** *(Admin)*: Creates a new category. 
	* **Payload:** `{ "name": "string", "description": "string", "parentId": "uuid", "sortOrder": 0 }` 
* **`GET /api/v1/Categories/{slugOrId}`**: Retrieves a specific category by its UUID or URL-friendly slug.
* **`PUT /api/v1/Categories/{id}`** *(Admin)*: Updates a category. 
* **`DELETE /api/v1/Categories/{id}`** *(Admin)*: Deletes a category.
### Products (`/api/v1/Products`)
* **`GET /api/v1/Products`**: Retrieves a paginated list of products. 
	* **Query Parameters:** `page` (default: 1), `limit` (default: 20), `sort` (string), `categoryId` (uuid). 
* **`POST /api/v1/Products`** *(Admin)*: Adds a new product to the catalog. 
	* **Payload:** ```json { "name": "string", "description": "string", "price": 0.0, "stockQuantity": 0, "sku": "string", "currency": "EGP", "isActive": true, "metadata": {}, "categoryIds": ["uuid"] } ``` 
* **`GET /api/v1/Products/{slugOrId}`**: Retrieves comprehensive product details by UUID or slug. 
* **`PUT /api/v1/Products/{id}`** *(Admin)*: Updates an existing product's details and inventory. 
* **`DELETE /api/v1/Products/{id}`** *(Admin)*: Soft-deletes or removes a product.
### Orders & Checkout (`/api/v1/Orders`)

* **`GET /api/v1/Orders`**: 
	* *User Context:* Retrieves the authenticated user's order history. 
	* *Admin Context:* Retrieves a platform-wide paginated list of all orders. 
* **`POST /api/v1/Orders`**: Converts a user's cart into a formal order. Financial totals are calculated server-side. 
	* **Payload (`CreateOrderRequest`):** ```json { "items": [ { "productId": "uuid", "quantity": 1 } ], "shippingAddress": { "street": "string", "city": "string", "postalCode": "string", "country": "string" }, "billingAddress": { "street": "string", "city": "string", "postalCode": "string", "country": "string" }, "notes": "string" } ``` 
* **`GET /api/v1/Orders/{id}`**: Retrieves exact details, addresses, and line items for a specific order UUID. 
* **`PUT /api/v1/Orders/{id}/status`** *(Admin)*: Progresses an order through the fulfillment pipeline. 
	* **Payload:** `{ "status": "Confirmed" }` 
	* **Validation:** Status MUST be exactly one of: `Pending`, `Confirmed`, `Processing`, `Shipped`, `Delivered`, `Cancelled`.
### Standardized Error Responses
The API utilizes standard HTTP status codes and returns error details in the RFC 7807 `ProblemDetails` format. This ensures front-end clients can predictably parse and handle application failures.

| Code             | Status | When                             |
| ---------------- | ------ | -------------------------------- |
| AUTH_REQUIRED    | 401    | No token provided                |
| AUTH_INVALID     | 401    | Token expired or malformed       |
| FORBIDDEN        | 403    | Insufficient role/permissions    |
| NOT_FOUND        | 404    | Resource missing or no access    |
| VALIDATION_ERROR | 400    | Request failed schema validation |
| CONFLICT         | 409    | Duplicate resource               |
| RATE_LIMITED     | 429    | 100/min auth, 20/min login       |
| INTERNAL_ERROR   | 500    | Unexpected server error          |
|                  |        |                                  |

## Data Validation Rules
### 1. Authentication & User Management

|**Field**|**Validation Rule (API / FluentValidation)**|**Database Enforcement**|
|---|---|---|
|**Email**|Required. Must match RFC 5322 standard email regex. Max 255 characters. Normalized to lowercase.|`UNIQUE INDEX` case-insensitive.|
|**Password**|Required. Minimum 8, Maximum 128 characters. Must contain at least: 1 uppercase, 1 lowercase, 1 digit, and 1 special character (`!@#$%^&*`).|Hashed via PBKDF2. Raw password is **never** stored.|
|**Name**|Required. 2-200 characters. Letters, spaces, and hyphens only.|`VARCHAR(200) NOT NULL`|
|**User Role**|Must match predefined identity claims (e.g., `Admin`, `Manager`, `Customer`).|Managed via `AspNetRoles`.|

### 2. Product Catalog

|**Field**|**Validation Rule (API / FluentValidation)**|**Database Enforcement**|
|---|---|---|
|**Product Name**|Required. 3-300 characters. Stripped of leading/trailing whitespace.|`VARCHAR(300) NOT NULL`|
|**Product Slug**|Required. 3-300 characters. Regex: `^[a-z0-9-]+$`. Cannot start or end with a hyphen.|`VARCHAR(300) UNIQUE NOT NULL`|
|**SKU**|Optional. 3-100 characters. Uppercase alphanumeric and hyphens.|`VARCHAR(255) UNIQUE`|
|**Price**|Required. Must be an integer `>= 0`. Cannot accept decimals (must be submitted in cents/piasters).|`INTEGER NOT NULL CHECK (price >= 0)`|
|**Currency**|Defaults to `EGP`. Must be a valid 3-letter ISO 4217 code.|`VARCHAR(3) DEFAULT 'EGP'`|
|**Stock Quantity**|Required on creation. Integer `>= 0`. Cannot be negative.|`INTEGER DEFAULT 0 CHECK (stock_quantity >= 0)`|
|**Metadata**|Must be a valid JSON object. Max payload size: 100KB.|`JSONB`|

### 3. Category Management

|**Field**|**Validation Rule (API / FluentValidation)**|**Database Enforcement**|
|---|---|---|
|**Category Name**|Required. 2-200 characters.|`VARCHAR(200) NOT NULL`|
|**Category Slug**|Required. Regex: `^[a-z0-9-]+$`. Must be globally unique.|`VARCHAR(200) UNIQUE NOT NULL`|
|**Parent ID**|Optional. If provided, must be a valid `UUIDv4`. Cannot reference itself (circular dependency check at API level).|`UUID REFERENCES categories(id)`|
|**Sort Order**|Optional. Integer. Defaults to `0`.|`INTEGER DEFAULT 0`|

### 4. Order & Checkout Processing

| **Field**            | **Validation Rule (API / FluentValidation)**                                                                                       | **Database Enforcement**  |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| **Order Status**     | Must be strictly one of: `pending`, `paid`, `shipped`, `delivered`, `cancelled`, `refunded`.                                       | `CHECK (status IN (...))` |
| **Item Quantity**    | Required. Integer `> 0`. A cart cannot contain 0 or negative items.                                                                | `CHECK (quantity > 0)`    |
| **Financial Totals** | `subtotal`, `tax`, `shipping`, and `total` must all be mathematically valid integers `>= 0`.                                       | `INTEGER NOT NULL`        |
| **Addresses**        | `shipping_address` and `billing_address` must contain required properties: `street` (string), `city` (string), `country` (string). | `JSONB NOT NULL`          |
| **Order Notes**      | Optional. Max 1000 characters. HTML tags stripped to prevent XSS.                                                                  | `TEXT`                    |

## Cross-Cutting Security Rules

- **UUID Validation:** Any ID passed via route parameters (e.g., `/api/v1/products/{id}`) must strictly match the UUIDv4 regex pattern: `^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-4[0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$`. Requests failing this return a `400 Bad Request` before hitting the database.
    
- **Sanitization (XSS Prevention):** All rich-text inputs (like `Product.Description`) must be sanitized using a library like **HtmlSanitizer** on the ASP.NET Core backend to strip malicious `<script>` or `onload` tags before insertion.
    
- **Pagination Limits:** Query parameters for `limit` or `pageSize` must be strictly bounded between `1` and `100` to prevent database Denial of Service (DoS) attacks via massive data pulls.
    
- **Immutable Totals:** Order totals (`unit_price`, `subtotal`, `total`) submitted by the client must be completely ignored. The backend must independently recalculate these values by querying the `products` table directly during the checkout transaction.

## Query Monitoring

> **Prerequisite:** Finding slow queries requires the `pg_stat_statements` extension. 
> Run `CREATE EXTENSION pg_stat_statements;` and ensure it is added to `shared_preload_libraries` in your `postgresql.conf`.

```sql
-- Find queries slower than 100ms
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY total_exec_time DESC
LIMIT 20;

-- Table sizes including indexes (Monitor growth of Orders/Carts)
SELECT
  schemaname || '.' || tablename AS table_name,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
  pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS data_size
FROM pg_tables WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;

-- Active connections by state (Ensure ASP.NET Connection Pool isn't leaking)
SELECT state, COUNT(*) 
FROM pg_stat_activity
WHERE datname = 'your_ecommerce_db_name' -- CHANGE THIS to your actual DB name
GROUP BY state;
```
## Backup and Recovery
#### Using Azure Database for PostgreSQL

| Backup Type         | Schedule                | Retention | Method                                   |
| ------------------- | ----------------------- | --------- | ---------------------------------------- |
| Automated snapshots | Daily 3 AM UTC          | 30 days   | Azure native automated backups           |
| Transaction logs    | Continuous              | 30 days   | Point-in-time recovery (PITR) via WAL    |
| Manual snapshots    | Before major migrations | 90 days   | Azure manual restore points              |
| Logical backup      | Weekly                  | 12 months | `pg_dump` exported to Azure Blob Storage |

#### Hosting locally / [[Docker Setup]] / Bare VPS (Self-Managed) 

| Backup Type         | Schedule                | Retention | Method                                                    |
| :------------------ | :---------------------- | :-------- | :-------------------------------------------------------- |
| Automated snapshots | Daily 3 AM UTC          | 30 days   | Cron job running `pg_dumpall` to external drive           |
| Transaction logs    | Continuous              | 30 days   | PostgreSQL `pg_receivewal` to backup server               |
| Manual snapshots    | Before major migrations | 90 days   | Manual pg_dump file                                       |
| Logical backup      | Weekly                  | 12 months | `pg_dump` compressed and uploaded to secure cloud storage |

--- 
##  "Index Health" Query 
Because e-commerce databases handle a high volume of concurrent inserts and updates (especially in `shopping_cart` and `cart_items`), indexes can become "bloated" over time, slowing down your React frontend's API responses. Consider adding this fourth query to your monitoring toolset to find missing indexes on your foreign keys:
```sql 
-- Find missing indexes on Foreign Keys that might cause slow JOINs 
SELECT 
	tc.table_name, 
	kcu.column_name, 
	ccu.table_name AS foreign_table_name, 
	ccu.column_name AS foreign_column_name 
FROM 
	information_schema.table_constraints AS tc 
	JOIN information_schema.key_column_usage AS kcu 
	ON tc.constraint_name = kcu.constraint_name 
	JOIN information_schema.constraint_column_usage AS ccu 
	ON ccu.constraint_name = tc.constraint_name 
WHERE 
	tc.constraint_type = 'FOREIGN KEY' 
AND tc.table_schema = 'public';
```

--- 


# Backend Architecture

## Technology Stack

* **Framework:** ASP.NET Core
* **Database:** PostgreSQL (with Entity Framework Core)
* **Caching:** Redis
* **Design Paradigm:** Clean Architecture (Onion Architecture)

To ensure seamless scalability and maintainability, the backend strictly adheres to **Clean Architecture**. This is combined with the **CQRS (Command Query Responsibility Segregation)** pattern via **MediatR**, effectively decoupling core business workflows from Entity Framework Core data-access logic and network routing.

## Directory Structure

The solution is divided into four distinct projects to enforce dependency rules:

```text
Solution: EcommercePlatform/
  ├── 1. Shopiy.Domain/           # Enterprise logic: Entities, Value Objects, Domain Exceptions
  ├── 2. Shopiy.Application/      # Use Cases: CQRS (Commands/Queries), DTOs, FluentValidation
  ├── 3. Shopiy.Infrastructure/   # Data Access: ApplicationDbContext (Identity), JWT, Repositories
  └── 4. Shopiy.Api/           # Presentation: Controllers, Middleware, SignalR, Program.cs

```

## Architectural Layer Responsibilities

### 1. Domain Layer (Zero Dependencies)

The innermost layer. It contains the raw, fundamental business data logic and strictly has no dependencies on any external libraries or frameworks.

* **Entities:** Core mutable objects with distinct identities (`Product`, `Category`, `Order`, `OrderItem`). The `ApplicationUser` inherits from `IdentityUser<Guid>`.
* **Value Objects:** Immutable structured data types (e.g., an `Address` object used to map the `shipping_address` and `billing_address` JSONB columns).

### 2. Application Layer (Depends *only* on Domain)

Houses all business operational commands and use cases. It defines *what* the application does without worrying about *how* data is stored or presented.

* **CQRS Framework (MediatR):** Segregates writing data from reading data.
* *Commands (Write):* State-altering operations (e.g., `CreateOrderCommand`, `UpdateProductStockCommand`) that force atomic database transactions.
* *Queries (Read):* Side-effect-free data retrieval (e.g., `GetActiveProductsQuery`), heavily optimized for read replicas or lean projections.


* **Validation Pipeline:** Utilizes `FluentValidation` to run strict input logic across request schemas before they ever reach the handlers.

### 3. Infrastructure Layer (Depends on Application & Domain)

Handles the physical data serialization, third-party integrations, and external I/O.

* **Identity & Persistence:** The core data access layer inheriting from `IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>`.
* **Read/Write DB Splitting:** Implements a factory pattern to configure distinct connection pools—routing write operations to the Primary RDS and offloading tracking-free read queries to the Replica.
* **Services:** Implementations for JWT generation, caching mechanisms (Redis), and email delivery.

### 4. API Layer (The Entry Point)

The outermost presentation layer. It acts as the gateway, translating network transport envelopes (HTTP requests) into application-layer structures.

* **Controllers:** Lean routing endpoints that do little more than execute standard `MediatR.Send()` operations and return HTTP status codes.

* **Custom Middleware:** Centralized Global Exception Handlers that intercept unexpected errors and map them down into the platform's standardized JSON `Error Response Format`.
---
## System Design Scenario: Duplicate Order Prevention

To satisfy the non-functional reliability requirements and guarantee absolute processing safety if a client triggers duplicate requests (e.g., double-clicking checkout), the architecture enforces an **Idempotency Strategy** using Redis.

### 1. The Idempotency Contract

- Every checkout request must attach a unique V4 UUID passed via headers: `X-Idempotency-Key`.
- The API utilizes an `[IdempotentRequest]` Action Filter to intercept the request before execution.

### 2. High-Performance Atomic Locking

- The filter executes a single, atomic Redis command (`StringSetAsync` with `When.NotExists`) to acquire a processing lock with a 5-minute TTL.
    
- **Cache Hit (Duplicate):** If the key exists, the request is intercepted. If the previous request is still processing, a `409 Conflict` is returned. If the previous request completed successfully, the filter **replays the cached JSON response** directly to the client.
### 3. Downstream Failure Recovery

- If the downstream database fails, throws an exception, or returns a 4xx/5xx error, the filter catches the failure and **evicts the Redis key**. This prevents the "Failure Lockout Trap" and allows the user to correct their input and retry immediately.
---
# Frontend Architecture
> [!WARNING]
> **CORS Security Alignment:** To satisfy the Non-Functional Requirement mandating secure `HttpOnly` cookie-based authentication, the API presentation layer must be refactored to remove `.AllowAnyOrigin()` from `Program.cs`. It must explicitly list your frontend client URL (e.g., `http://localhost:8081` from your frontend port configurations) and include `.AllowCredentials()` to allow secure browser cookie transmission.
## Technology Stack

* **Core Framework:** React with TypeScript
* **Build Tool:** Vite
* **Styling & UI:** TailwindCSS, Lucide React (Icons)
* **Server State & Data Fetching:** TanStack Query (React Query), Axios
* **Client State Management:** Zustand
* **Routing:** React Router DOM

To maintain a scalable and highly modular codebase, the frontend strictly implements a **Feature-Based (Domain-Driven) Architecture**. Instead of globally grouping all application hooks or all API calls into massive shared folders, code is isolated into self-contained business domains. This closely mirrors the separation of concerns found in your backend's Clean Architecture.

## Directory Structure

```text
src/
├── components/          # Shared global UI components (MainLayout, ProductCard, CartDrawer)
├── config/              # Global library configurations (axios.ts, queryClient.ts, env.ts)
├── features/            # Self-contained business domains
│   ├── admin/           # Dashboard, product/category forms, and order oversight
│   ├── auth/            # Login, registration, and session management
│   ├── catalog/         # Product listing, search, and category browsing
│   └── checkout/        # Cart management, order creation, and purchase history
├── router/              # AppRouter orchestration and ProtectedRoute logic
├── services/            # Base API service definitions
├── store/               # Zustand global state slices (authStore, cartStore, uiStore)
├── types/               # Global TypeScript definitions (api.ts, common.ts)
└── utils/               # Pure utility functions (jwt.ts, formatDate.ts, validators.ts)

```

## Feature Module Anatomy

Each domain inside the `features/` directory acts as its own micro-application. For example, looking inside `src/features/catalog/`, the domain is completely encapsulated:

* `api/`: Axios network calls strictly related to the catalog (e.g., `getProducts.ts`, `searchProducts.ts`).
* `hooks/`: Custom React Query hooks encapsulating those API calls for UI consumption (e.g., `useProducts.ts`).
* `pages/`: Route-level component views (e.g., `CatalogPage.tsx`, `ProductPage.tsx`).
* `types.ts`: TypeScript interfaces strictly related to the catalog models.

## Key Frontend Implementation Patterns

* **Server State vs. Client State:** The architecture enforces a strict boundary between UI state and server data. **TanStack Query** (`queryClient.ts`) handles all asynchronous caching, background synchronization, and loading states for backend data. **Zustand** (`src/store/`) is reserved purely for synchronous, global client state (e.g., toggling the UI state in `uiStore.ts` or managing the active session in `authStore.ts`).
* **Secure Interceptors:** The `src/config/axios.ts` configuration acts as the network backbone, automatically attaching JWTs to outgoing requests and seamlessly handling token rotation using your backend's refresh token endpoints.
* **Route Protection & RBAC:** The `router/ProtectedRoute.tsx` component acts as the gateway layer. It integrates with your authentication utilities (`jwt.ts` and `authStore.ts`) to evaluate claims—blocking unauthenticated users from the `checkout` routes and completely securing the `admin` feature module from standard customer accounts.
* **Optimized Tooling:** By leveraging **Vite** alongside esbuild, the application benefits from instant Hot Module Replacement (HMR) during development and highly optimized, chunk-split static assets for production deployment via Nginx.
---

# Continuous Integration & Continuous Deployment (CI/CD)

Our deployment lifecycle is fully automated using **GitHub Actions**. To ensure optimal execution times and strict separation of concerns, the pipeline is split into distinct workflows for the frontend and backend. Every pull request or push targeting the `main` or `master` branches triggers these pipelines. Code cannot be safely merged unless all checks pass.

## 1. CI Pipeline: Backend Validation (`ci.yml`)

The backend pipeline is designed to build and thoroughly test the .NET 10 application in an environment that closely mirrors production.

* **Ephemeral Service Containers:** Before testing begins, the pipeline automatically spins up dedicated `postgres:17` and `redis:7-alpine` Docker containers. This ensures integration tests have access to real data stores rather than relying solely on in-memory mocks.


* **Build & Restore:** Utilizes the modern `Shopiy.slnx` solution file to restore dependencies and compile the application in `Release` mode.


* **Automated Testing:** Executes all unit and integration tests across the solution to verify business logic and data persistence.

```yaml
# .github/workflows/ci.yml
name: Backend CI Pipeline
on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
jobs:
  build-and-test:
    name: Build & Test (.NET Core)
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_DB: shopiy_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.x'

      - name: Restore Dependencies
        run: dotnet restore Shopiy.slnx

      - name: Build Solution
        run: dotnet build Shopiy.slnx --configuration Release --no-restore

      - name: Run Unit Tests
        run: dotnet test Shopiy.slnx --configuration Release --no-build --verbosity normal

```

## 2. CI Pipeline: Frontend Validation (`ci-frontend.yml`)

The frontend pipeline guarantees that the React application remains structurally sound and that there are no compilation errors during the bundling process.

* **Environment Setup:** Provisions a Node.js v20 environment and leverages `npm` caching via `package-lock.json` to significantly speed up dependency installation.

* **Clean Installation:** Runs `npm ci` to strictly adhere to the lockfile, preventing unexpected package version drifts.

* **Build Verification:** Executes `npm run build` (via Vite) to compile the TypeScript code and verify that the production bundle can be generated without errors.

```yaml
# .github/workflows/ci-frontend.yml
name: Frontend CI Pipeline

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

jobs:
  build-and-verify:
    name: Build & Verify (Node/Vite)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: 'package-lock.json'

      - name: Install Dependencies
        run: npm ci

      - name: Compile and Verify Build
        run: npm run build

```

## 3. CD Pipeline: Production Deployment (Planned Architecture)

*(Note: Currently orchestrated manually or via external triggers, with plans to integrate directly into GitHub Actions.)*

When code is successfully verified and merged into the `main` branch, the Continuous Deployment strategy executes the following sequence:

1. **Docker Build & Push:** Compiles the `ecommerce-api` and `ecommerce-frontend` Docker images and pushes them to a secure container registry.
2. **Database Migrations:** Applies any pending Entity Framework migrations against the production PostgreSQL database.
3. **Rolling Update:** Instructs the host server to pull the latest images and restart the containers with zero downtime.
---
# Docker Setup
This multi-container setup mirrors your production ecosystem locally. It configures the ASP.NET Core Web API, the React frontend (compiled via Node and served over Nginx), and a PostgreSQL database instance.

---
## docker-compose.yml
``` yaml
services:

  postgres:
    image: postgres:17
    container_name: shopiy-postgres
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: shopiy
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: shopiy-redis
    restart: always
    ports:

      - "6379:6379"

  

  seq:
    image: datalust/seq:latest
    container_name: shopiy-seq
    restart: always
    ports:
      - "5341:80"
    environment:
      SEQ_FIRSTRUN_ADMINPASSWORD: ${SEQ_ADMIN_PASSWORD}
      ACCEPT_EULA: Y
    volumes:
      - seq_data:/data
  shopiy-api:
    build:
      context: .
      dockerfile: src/Shopiy.Api/Dockerfile
    container_name: shopiy-api
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
      seq:
        condition: service_started
    ports:
      - "5000:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT}
      ConnectionStrings__DefaultConnection: >
        Host=postgres; Port=5432; Database=${POSTGRES_DB}; Username=${POSTGRES_USER}; Password=${POSTGRES_PASSWORD}
      ConnectionStrings__Redis: redis:6379
      Seq__ServerUrl: http://seq:5341
      Jwt__Issuer: Shopiy.Api
      Jwt__Audience: Shopiy.Client
      Jwt__Secret: ${JWT_SECRET}

volumes:
  postgres_data:
  seq_data:
```

--- 
## Backend Multi-Stage `Dockerfile` (ASP.NET Core)

``` DockerFile
FROM mcr.microsoft.com/dotnet/sdk:10.0-preview AS build
WORKDIR /src
COPY . .
WORKDIR /src/src/Shopiy.Api

RUN dotnet restore Shopiy.Api.csproj

RUN dotnet publish Shopiy.Api.csproj \
    -c Release \
    -o /app/publish \
    --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0-preview
WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 8080

ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet","Shopiy.Api.dll"]
```

---
## Frontend Multi-Stage `Dockerfile` (React + Vite + Nginx)

``` DockerFile
# Build Stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install
COPY . .
RUN pnpm build

# Production Runtime Stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# Copy custom nginx routing config to handle React Router SPA routing
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
## docker-compose.yml
``` yaml
services:
  shopiy-frontend:
    build:
      context: .
      args:
        VITE_API_URL: ${VITE_API_URL:-http://localhost:5000}
        VITE_APP_NAME: ${VITE_APP_NAME:-Shopiy}
    image: shopiy-frontend:latest
    ports:
      - "${SHOPIY_FRONTEND_PORT:-8081}:80"
    restart: unless-stopped
```

---
# Quality Assurance & Testing Plan

To ensure a stable shopping experience and prevent financial or data-loss bugs, the project follows the **Test Pyramid** strategy.

---
## 1. Testing Pyramid Breakdown

| **Test Level**       | **Scope & Purpose**                                                                                          | **Tools Used**                               | **Target Coverage** |
| -------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------- | ------------------- |
| **End-to-End (E2E)** | Simulates real user browser flows across the full stack (Frontend + API + DB).                               | Playwright                                   | Critical Paths Only |
| **Integration**      | Tests how the API interacts with the database (e.g., checking if an order actually saves to PostgreSQL).     | xUnit, WebApplicationFactory, Testcontainers | ~60%                |
| **Unit**             | Tests isolated business logic (e.g., calculating cart totals, validating JWT rules, UI component rendering). | xUnit, Moq, Vitest, React Testing Library    | > 80%               |

---
## 2. Backend Testing Strategy (ASP.NET Core)

- **Domain Unit Tests:** Pure C# tests using `xUnit` and `FluentAssertions`. We test core entities without database connections. _Example: Asserting that `Order.TransitionToShipped()` throws an exception if the order is still `pending`._
    
- **API Integration Tests:** We use `Microsoft.AspNetCore.Mvc.Testing` (`WebApplicationFactory`) combined with **Testcontainers**. Testcontainers automatically spins up a real, temporary PostgreSQL Docker container for the test run, applies migrations, runs the API request (like `POST /api/v1/orders`), asserts the 200 OK response, and then destroys the database.
    

---
## 3. Frontend Testing Strategy (React)

- **Component Testing:** We use `Vitest` and `React Testing Library` to mount components in isolation. _Example: Rendering the `AddToCartButton` and ensuring the `onClick` handler fires the correct Zustand store action._
    
- **Mocking APIs:** `MSW` (Mock Service Worker) is used to intercept Axios network requests during tests, allowing us to simulate server errors (500s) and test the UI's error states without needing the real ASP.NET backend running.
    
---
## 4. Critical Path E2E Scenarios

E2E tests take the longest to run, so they are reserved exclusively for "make or break" business functions. The following Playwright scripts must pass on every deployment:

1. **The Guest Checkout Flow:** User browses catalog $\rightarrow$ Adds item to cart $\rightarrow$ Registers for account $\rightarrow$ Completes checkout $\rightarrow$ Views Order Confirmation.
    
2. **The Admin Fulfillment Flow:** Admin logs in $\rightarrow$ Views pending orders $\rightarrow$ Updates an order status to 'shipped' $\rightarrow$ Verifies stock quantity decreases.
    
3. **Authentication Security:** Verifies that a locked-out user (5 failed attempts) sees the correct error message and cannot log in.
---
# Logging & Observability Strategy

The system relies on an integrated combination of structured semantic logging and standard system health middleware probes to maintain real-time visibility.

#### 1. Structured Logging Philosophy (Serilog + Seq)
Rather than producing flat text log files, the API emits machine-readable structured JSON event packets. This permits rapid diagnostics and log aggregation.
- **Log Ingestion Engine:** Logs are securely forwarded to an instance of **Seq** exposed locally at `http://localhost:5341` (configured via `Seq__ServerUrl` in the backend service).
- **Security Control:** Administrative console access is regulated via the secure variable `${SEQ_ADMIN_PASSWORD}`.
- **Developer Implementation Rule:** Avoid C# string interpolation inside logs to ensure property indexing functions correctly within Seq.
  * *Bad:* `Log.Information($"Host started at {DateTime.UtcNow}");`
  * *Good:* `Log.Information("Starting web host...");`

#### 2. System Readiness & Health Checks
Automated container health checking and live platform monitoring are exposed natively through the `/health` endpoint. This routes internal probes through two critical evaluators:
- **Database Health Probe (`database`):** Executes an asynchronous connection viability check (`CanConnectAsync()`) against the primary PostgreSQL container to confirm pool availability.
- **Distributed Cache Probe (`redis`):** Resolves the registered `IConnectionMultiplexer` instance and runs an internal active ping command to guarantee caching layer responsiveness.
