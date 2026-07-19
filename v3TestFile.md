POST http://localhost:8000/api/users/register
Headers: Content-Type: application/json
Body:
{
"name": "Admin Tester",
"email": "admin@test.com",
"password": "pass123"
}

Expect 201.

Step 2 — Log in, get a JWT
POST http://localhost:8000/api/users/login
Headers: Content-Type: application/json
Body:
{
"email": "admin@test.com",
"password": "pass123"
}

Expect 200, body {"token": "eyJ..."}. Copy this token — call it TOKEN1.
Step 3 — Create 3 categories
For each, in Postman: Headers tab → Content-Type: application/json and Authorization: Bearer TOKEN1.

POST http://localhost:8000/api/categories
Body: {"name": "Electronics", "description": "gadgets and accessories"}

POST http://localhost:8000/api/categories
Body: {"name": "Books", "description": "programming and self-help"}

POST http://localhost:8000/api/categories
Body: {"name": "Home & Kitchen", "description": "household items"}

Each returns 201 with an id — note all three ids (call them ELEC_ID, BOOKS_ID, HOME_ID).
Step 4 — Create 8 products (same headers as Step 3)

POST http://localhost:8000/api/products
Body: {"name": "Wireless Mouse", "description": "ergonomic", "price": 25.99, "stockQuantity": 50, "categoryId": ELEC_ID}

POST http://localhost:8000/api/products
Body: {"name": "Mechanical Keyboard", "description": "clicky switches", "price": 79.99, "stockQuantity": 30, "categoryId": ELEC_ID}

POST http://localhost:8000/api/products
Body: {"name": "USB-C Hub", "description": "7-in-1", "price": 34.99, "stockQuantity": 40, "categoryId": ELEC_ID}

POST http://localhost:8000/api/products
Body: {"name": "Bluetooth Speaker", "description": "portable", "price": 45.00, "stockQuantity": 25, "categoryId": ELEC_ID}

POST http://localhost:8000/api/products
Body: {"name": "Clean Code", "description": "by Robert Martin", "price": 32.50, "stockQuantity": 20, "categoryId": BOOKS_ID}

POST http://localhost:8000/api/products
Body: {"name": "Atomic Habits", "description": "by James Clear", "price": 18.00, "stockQuantity": 35, "categoryId": BOOKS_ID}

POST http://localhost:8000/api/products
Body: {"name": "Coffee Maker", "description": "drip style", "price": 55.00, "stockQuantity": 15, "categoryId": HOME_ID}

POST http://localhost:8000/api/products
Body: {"name": "Blender", "description": "500W", "price": 40.00, "stockQuantity": 20, "categoryId": HOME_ID}

Note each returned id. Call the first four MOUSE_ID, KEYBOARD_ID, HUB_ID, SPEAKER_ID.
Step 5 — Register a second user (for co-purchase data)

POST http://localhost:8000/api/users/register
Body: {"name": "Second User", "email": "user2@test.com", "password": "pass123"}

POST http://localhost:8000/api/users/login
Body: {"email": "user2@test.com", "password": "pass123"}

Copy the token → TOKEN2, and note the user's own id — call getCurrentUser if you need it:

GET http://localhost:8000/api/users/me
Headers: Authorization: Bearer TOKEN2

Response includes id — call it USER2_ID. Do the same for admin@test.com to get USER1_ID.
Step 6 — Generate real order history (needed for recommendations to have anything to work with)
As User1 — add mouse + keyboard to cart, checkout:

POST http://localhost:8000/api/orders/cart/USER1_ID/items
Headers: Authorization: Bearer TOKEN1
Body: {"productId": MOUSE_ID, "quantity": 1}

POST http://localhost:8000/api/orders/cart/USER1_ID/items
Headers: Authorization: Bearer TOKEN1
Body: {"productId": KEYBOARD_ID, "quantity": 1}

POST http://localhost:8000/api/orders/checkout/USER1_ID
Headers: Authorization: Bearer TOKEN1

s User2 — add mouse + hub to cart, checkout (this creates the co-occurrence signal: Mouse now links to both Keyboard and Hub

POST http://localhost:8000/api/orders/cart/USER2_ID/items
Headers: Authorization: Bearer TOKEN2
Body: {"productId": MOUSE_ID, "quantity": 1}

POST http://localhost:8000/api/orders/cart/USER2_ID/items
Headers: Authorization: Bearer TOKEN2
Body: {"productId": HUB_ID, "quantity": 1}

POST http://localhost:8000/api/orders/checkout/USER2_ID
Headers: Authorization: Bearer TOKEN2


Note its id → WATCH_ID. No orders will ever touch this one until you say so — it exists purely to test Rule 2 (category fallback).

Frontend testing steps
Do these in order, matching the verification checklist you were given, but now with real data behind it:

Logged out, product with history — open http://localhost:8080/products/MOUSE_ID. Scroll down, confirm "You might also like" shows Keyboard and/or Hub (the actual co-purchase result).
Logged out, homepage — open http://localhost:8080/. Confirm there's no "Recommended for you" section anywhere.
Log in as User1 (admin@test.com / pass123) through the browser. Revisit the homepage — confirm "Recommended for you" now appears.
Fallback test — visit http://localhost:8080/products/WATCH_ID (the Smart Watch, zero order history). Confirm "You might also like" still shows something — other Electronics products (Mouse, Keyboard, Hub, Speaker), via category fallback, not an empty section.
Fail-open test — stop recommendation-service. Reload the Mouse's product page and the (logged-in) homepage. Confirm both render completely normally, just without their recommendation sections — no error page, no stack trace. Restart recommendation-service and reload both pages again to confirm the sections come back.