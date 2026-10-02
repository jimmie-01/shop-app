# AGENTS.md

## Project

This repository contains a small Shopify-like e-commerce application.

The goal is to build a maintainable MVP using a simple monolithic architecture.

The application has two primary user types:

- Shop owners
- Shoppers

Do not introduce complexity that is not required by the current product requirements.

---

# 1. Technology Stack

## Frontend

- Next.js
- TypeScript
- React

## Backend

- Node.js
- Express.js
- TypeScript

## Database

- PostgreSQL
- Supabase

## Authentication

- Supabase Auth
- Google OAuth
- Google Cloud Console for OAuth configuration

## File Storage

- Supabase Storage

## Email

- Mailgun

## Testing

- Vitest
- Supertest

## Development

- Docker
- WSL

---

# 2. Architecture

Use a modular monolith.

Do NOT introduce microservices.

The application consists of:

frontend/
backend/

The frontend communicates with the backend through HTTP/JSON APIs.

The backend owns business logic and database access.

Preferred backend flow:

Route
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Database

Controllers must remain thin.

Business rules belong in services.

Database access belongs in repositories.

---

# 3. Frontend Responsibilities

Next.js is responsible for:

- Rendering the user interface
- Product browsing
- Product details
- Shopping cart UI
- Checkout UI
- Authentication UI
- Order history UI
- Shop owner dashboard UI

The frontend must not contain business-critical authorization logic.

The backend must enforce authorization.

---

# 4. Backend Responsibilities

Express is responsible for:

- HTTP API
- Authentication verification
- Authorization
- Validation
- Business logic
- Database access
- Order creation
- Email triggering
- Shop owner operations

The backend must never trust user IDs supplied by the client.

For authenticated operations, obtain the user identity from the verified authentication token.

---

# 5. Authentication

Use Supabase Auth.

Google OAuth is the only authentication provider required for the initial version.

Do NOT implement a custom authentication system.

Do NOT implement custom password authentication unless explicitly requested.

Do NOT create a separate JWT authentication system when Supabase Auth already provides authentication tokens.

The frontend obtains the authenticated Supabase session.

The backend verifies the Supabase access token.

The authenticated user's Supabase user ID is the source of truth for identity.

Never accept this pattern:

req.body.userId

for determining the authenticated user.

Use the verified authenticated identity instead.

---

# 6. Authentication Persistence

Users must be able to:

1. Sign in with Google.
2. View their orders.
3. Close the browser/page.
4. Return to the application.
5. Sign in again.
6. Still see their previous orders.

Orders must be associated with the persistent Supabase Auth user ID.

Signing out must not delete users or orders.

---

# 7. Authorization

There are two application roles:

- OWNER
- SHOPPER

Authorization must be enforced on the backend.

Only owners can:

- Update shop settings
- Upload/change the shop logo
- Create products
- Update products
- Delete/deactivate products
- View shoppers associated with their shop
- View orders belonging to their shop

Shoppers can:

- Browse products
- Add products to a cart
- Checkout
- View their own orders

A shopper must never be able to retrieve another shopper's orders.

An owner must only access data belonging to their shop.

---

# 8. Database

Use PostgreSQL through Supabase.

Keep the initial database schema small.

Initial entities:

- users
- shops
- products
- orders
- order_items

Do not introduce additional tables unless there is a clear requirement.

Orders must contain a user ID and shop ID.

Order items must store historical product information:

- product name
- unit price
- quantity
- subtotal

Do not rely exclusively on the current product price when displaying historical orders.

Prices must be stored using a safe representation.

Do not use floating-point arithmetic for money calculations.

Prefer integer minor units where appropriate.

Example:

NGN 5,000.00

may be stored as:

500000

when using kobo as the smallest unit.

---

# 9. Products

Products initially require:

- name
- description
- price
- image
- active/inactive status
- shop ID

Do not introduce:

- variants
- SKUs
- inventory reservation
- product bundles
- discounts
- coupons

unless explicitly requested.

---

# 10. Shop

A shop initially requires:

- name
- logo URL
- owner ID

Shop owners can upload a logo using Supabase Storage.

Do not store image binaries in PostgreSQL.

Store the resulting storage URL/path in the shop record.

---

# 11. Shopping Cart

The first version should keep cart state on the client.

Do not create persistent cart database tables unless a requirement makes them necessary.

The server must recalculate product prices when creating an order.

Never trust prices sent by the browser.

For example, do NOT calculate an order using:

req.body.price

The backend must retrieve the current product price from the database.

---

# 12. Checkout

Checkout should initially perform:

1. Validate authenticated user.
2. Validate requested products.
3. Retrieve products from the database.
4. Verify products are active.
5. Retrieve authoritative prices.
6. Calculate order totals on the server.
7. Create the order.
8. Create order items.
9. Send confirmation email.
10. Return the created order.

Do not trust totals calculated by the frontend.

The backend is the source of truth for order totals.

---

# 13. Order Status

Keep order status simple.

Initial statuses:

- PENDING
- CONFIRMED
- CANCELLED
- COMPLETED

Do not implement a complex fulfillment workflow unless explicitly required.

---

# 14. Email

Use Mailgun for transactional order confirmation emails.

Email sending belongs in a dedicated service/module.

Example:

services/email/

Do not place Mailgun API calls directly inside controllers.

Order creation and email sending should be separated sufficiently that email failures do not corrupt order data.

If an order is successfully created but email delivery fails:

- Keep the order.
- Log the email failure.
- Return the appropriate response.
- Do not delete or roll back a successfully created order merely because email delivery failed.

Do not implement an email queue unless required later.

---

# 15. Validation

Use Zod for request validation.

Validate:

- request bodies
- query parameters
- route parameters where appropriate

Do not rely on TypeScript types alone for runtime validation.

---

# 16. Error Handling

Use centralized Express error handling.

Controllers should pass errors to the global error handler.

Use appropriate HTTP status codes.

Do not expose:

- database credentials
- stack traces
- internal implementation details
- secrets

in production responses.

---

# 17. Testing

Use Vitest for unit tests.

Use Supertest for backend integration tests.

Important areas requiring tests:

- Authentication middleware
- Authorization
- Product creation
- Product retrieval
- Product updates
- Product deletion/deactivation
- Cart/checkout calculations
- Order creation
- Order ownership
- Owner access restrictions
- Shopper access restrictions
- Invalid product IDs
- Inactive products
- Invalid quantities
- Email service failure handling

Every significant backend feature should include appropriate tests.

---

# 18. Security Rules

Never trust client-provided:

- user ID
- owner ID
- shop ID
- product price
- order total
- role

These values must be derived or verified server-side.

Never expose:

- Supabase service role key
- Mailgun API key
- Google client secret
- database credentials

to the browser.

Never commit `.env` files.

Only `.env.example` may be committed.

---

# 19. Environment Variables

Keep secrets in environment variables.

Example:

SUPABASE_URL=
SUPABASE_ANON_KEY=

SUPABASE_SERVICE_ROLE_KEY=

MAILGUN_API_KEY=
MAILGUN_DOMAIN=

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

DATABASE_URL=

Never hard-code credentials.

Never print secrets in logs.

---

# 20. API Design

Use REST.

Do not introduce GraphQL.

Use resource-oriented endpoints.

Example:

GET    /api/products
GET    /api/products/:id

POST   /api/products
PATCH  /api/products/:id
DELETE /api/products/:id

GET    /api/orders
GET    /api/orders/:id

POST   /api/orders

GET    /api/shop
PATCH  /api/shop

POST   /api/shop/logo

GET    /api/shop/shoppers

Use consistent JSON responses.

---

# 21. Code Quality

Use TypeScript strict mode.

Avoid `any`.

Prefer explicit types.

Keep functions small and focused.

Avoid unnecessary abstractions.

Do not create interfaces, services, repositories, utilities, hooks, or components unless they provide a clear benefit.

Do not duplicate business logic.

Prefer existing project patterns over introducing new patterns.

---

# 22. Dependency Rules

Before adding a dependency:

1. Check whether the functionality already exists in the project.
2. Check whether the functionality can be implemented simply without a dependency.
3. Prefer established libraries already used by the project.
4. Avoid dependencies that solve trivial problems.

Do not add a dependency merely because it is convenient.

---

# 23. Scope Control

The MVP should remain intentionally small.

Do NOT implement the following unless explicitly requested:

- Payments
- Shipping
- Taxes
- Coupons
- Discounts
- Reviews
- Wishlists
- Product variants
- Inventory management
- Notifications
- Analytics
- Search engine
- Recommendation engine
- Chat
- Real-time features
- WebSockets
- Redis
- Message queues
- Microservices
- GraphQL
- Kubernetes
- Event sourcing
- CQRS

When a new requirement appears, implement the smallest solution that satisfies it.

---

# 24. Changes and Refactoring

Before modifying code:

1. Inspect the existing implementation.
2. Understand the current architecture.
3. Identify the smallest set of files that need changing.
4. Explain the proposed approach when the change is significant.

Do not refactor unrelated code while implementing a feature.

Do not rewrite working code merely because another architecture is preferred.

Preserve existing conventions.

---

# 25. Database Changes

Database schema changes must be explicit.

Do not modify production data directly.

Use migrations for schema changes.

After changing the schema:

- update the application types/models
- update tests
- verify migrations
- verify the development database

Never silently change the schema as part of an unrelated task.

---

# 26. Git

Do not work directly on main for feature development.

Use feature branches.

Example:

feature/product-management
feature/google-auth
feature/checkout
feature/order-history

Before completing a task:

- inspect git diff
- run relevant tests
- run the TypeScript build
- run lint if configured

Do not commit generated files, secrets, node_modules, or build artifacts unless explicitly required.

---

# 27. Working With the Developer

The developer is using this project to learn full-stack software engineering.

When implementing a non-trivial feature:

1. Explain the architectural decision.
2. Explain important trade-offs.
3. Implement the smallest practical solution.
4. Explain why the implementation works.
5. Add tests.
6. Tell the developer how to verify it.

Do not hide important architectural decisions.

Do not unnecessarily generate large amounts of code at once.

---

# 28. Before Completing Any Task

Verify:

- The requested behavior works.
- Existing functionality has not been unnecessarily changed.
- Relevant tests pass.
- TypeScript compilation succeeds.
- No secrets were added.
- No unnecessary dependencies were added.
- No unrelated refactoring was introduced.

Prefer a small correct change over a large clever change.