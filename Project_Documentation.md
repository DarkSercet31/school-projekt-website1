# Weather Station Project — Feature Documentation

## Introduction

This document explains all features added to the Weather Station website as part of the school project.
Each section describes what the feature does, how it works technically, and where to find more information.

The project is built with **PHP 8** and **MySQL**, uses **Bootstrap 5** for the interface, and
**PHPMailer** to send emails through Mailtrap (a testing inbox that does not send real emails).

---

## 1. User Registration with Email Verification

### What it does

New users fill out a registration form with their username, name, email address, and password.
After submitting, they receive an email with a verification link.
Their account is only activated after clicking the link — they cannot log in before this step.

### How it works

1. The registration form (at `/auth/register.php`) posts to `includes/register.inc.php`
2. The processor validates all inputs and checks that the username and email are not already taken
3. The password is hashed using PHP's `password_hash()` with the bcrypt algorithm — the plaintext password is never stored
4. A random 64-character token is generated using `bin2hex(random_bytes(32))`
5. The token is stored in the `email_tokens` table with `type = 'verify'` and an expiry of 24 hours
6. An email is sent via PHPMailer to the user's address with a link to `auth/verify.php?token=...`
7. When the user clicks the link, the token is validated, the user's `status` is updated to `'verified'`, and the token is deleted

### Files involved

- `auth/register.php` — registration form
- `includes/register.inc.php` — validation and insert logic
- `auth/verify.php` — token validation and account activation
- `config/mailtrap.php` — email sender factory

### Learn more

- PHP `password_hash`: https://www.php.net/manual/en/function.password-hash.php
- PHP `random_bytes`: https://www.php.net/manual/en/function.random-bytes.php
- PHPMailer basics: https://github.com/PHPMailer/PHPMailer
- YouTube — "PHP registration with email verification" by Dani Krossing

---

## 2. Password Reset via Email

### What it does

Users who forget their password can request a reset link by entering their email address.
The system sends them a secure link that expires in 1 hour.
Clicking the link lets them choose a new password.

### How it works

1. User submits their email on `auth/forgot_password.php`
2. The system looks up the email — but **always** shows the same message regardless of whether the email was found (this prevents an attacker from guessing which emails are registered)
3. If the email exists: a random token is generated and stored in `email_tokens` with `type = 'reset'` and a 1-hour expiry
4. A password reset email is sent with a link to `auth/reset_password.php?token=...`
5. On that page the token is validated, and the user can enter and confirm a new password
6. On success: the password is updated and hashed, the token is deleted, the user is redirected to login

### Files involved

- `auth/forgot_password.php` — email input form + token generation
- `auth/reset_password.php` — token validation + new password form

### Learn more

- YouTube — "PHP forgot password reset tutorial" by Dani Krossing
- OWASP — Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html

---

## 3. Login with Email OTP (Two-Factor Authentication)

### What it does

Every login requires two steps:
1. Username + password (something you know)
2. A 6-digit one-time code sent to your email (something you have)

This is a simplified form of **two-factor authentication (2FA)**, which makes accounts much harder to break into even if a password is stolen.

### How it works

1. User submits username + password to `includes/login.inc.php`
2. The processor verifies the password with `password_verify()`
3. It checks that the account status is `'verified'`
4. Instead of logging in immediately, a 6-digit OTP is generated with `str_pad(random_int(0, 999999), 6, '0', STR_PAD_LEFT)`
5. The OTP is stored in `email_tokens` with `type = 'otp_login'` and a 15-minute expiry
6. An email is sent with the code
7. The username is stored temporarily in `$_SESSION['otp_pending_user']` — no full session exists yet
8. The user is redirected to `auth/verify_otp.php` to enter the code
9. If the code matches and has not expired: the token is deleted, the full session is created, and the user is routed to their dashboard
10. If the user has `mustChangePassword = 1`, they are redirected to change their password before accessing anything else

### Files involved

- `auth/login.php` — login form
- `includes/login.inc.php` — password verify + OTP generation
- `auth/verify_otp.php` — OTP validation + session creation

### Learn more

- YouTube — "Two factor authentication PHP" — Traversy Media
- Wikipedia — Two-factor authentication: https://en.wikipedia.org/wiki/Multi-factor_authentication

---

## 4. Admin Creates Users and Emails Credentials

### What it does

Administrators can create new user accounts directly from the admin panel.
When a new user is created, the system:
- Generates a random 8-character temporary password
- Sets the account to `mustChangePassword = 1` so the user must change it on first login
- Sends the user an email with their username and temporary password

### How it works

1. Admin fills in the "Create User" form on `admin/admin_users.php`
2. A secure random password is generated using `random_int()` over a character set that excludes ambiguous characters (0, O, l, 1, I)
3. The password is hashed and the user row is inserted with `status = 'verified'` (admin-created accounts skip email verification) and `mustChangePassword = 1`
4. An email with the login credentials is sent via PHPMailer
5. On first login, after OTP verification, the user is redirected to `includes/force_change_password.php`

### Files involved

- `admin/admin_users.php` — create/edit/delete user form
- `includes/force_change_password.php` — forced password change on first login
- `config/mailtrap.php` — email sender

---

## 5. Demo Shop

### What it does

Users can browse a product catalog, add items to a shopping cart, and complete a mock checkout.
After checkout, an order record is saved with a snapshot of the products and prices at the time of purchase.
No real payment is processed — this is entirely for demonstration.

### How it works

1. Products are stored in the `product` table and displayed as cards on `user/shop.php`
2. "Add to Cart" sends a POST to `includes/cart.inc.php` which either inserts a new cart row or increments the quantity (using `ON DUPLICATE KEY UPDATE`)
3. The cart is stored in the `cart` table (one row per user + product combination, with a UNIQUE constraint)
4. On `user/cart.php` the user can increase or decrease quantities, or remove items
5. The total is calculated in PHP: `SUM(price × quantity)` for all cart items
6. "Checkout" posts to `includes/checkout.inc.php` which:
   - Inserts a row into `orders` with the total amount
   - Inserts one row per product into `order_item`, copying the name and price at that moment (snapshot)
   - Deletes all cart rows for the user
7. The user is redirected to `user/order_history.php` which shows all past orders with their line items

### Why the price is copied into `order_item`

If the product price changes in the future, the order history should still show what the user paid. Copying the price at purchase time ("snapshotting") is standard practice in e-commerce systems.

### Files involved

- `user/shop.php` — product catalog
- `user/cart.php` — cart management
- `user/order_history.php` — order list and line items
- `includes/cart.inc.php` — add/change/remove cart processor
- `includes/checkout.inc.php` — create order processor

### Learn more

- YouTube — "PHP Shopping Cart Tutorial" — ProgrammingKnowledge
- MySQL `ON DUPLICATE KEY UPDATE`: https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html

---

## 6. Admin Product Management

### What it does

Administrators can add, edit, and delete products from the shop without touching the database directly.
All changes are made from a table in the admin panel.

### How it works

The page `admin/admin_products.php` displays all products in an inline-editable table.
- **Create**: a form at the top adds a new product row
- **Edit**: each row has its own Save button that sends the updated fields in a POST request
- **Delete**: each row has a delete button (with a confirmation prompt)

All operations use PDO prepared statements to prevent SQL injection.

### Files involved

- `admin/admin_products.php` — full product CRUD

---

## 7. Friends Chat

### What it does

Accepted friends can send messages to each other in real time.
The chat page automatically checks for new messages every 5 seconds without needing to refresh the page.

### How it works

1. On `user/user_friends.php`, each accepted friend has a "Chat" button
2. Clicking it opens `user/chat.php?with=<username>`
3. The page verifies that the target user is actually an accepted friend
4. Existing messages are loaded from the `message` table (last 200) and displayed in a scrollable thread
5. Messages sent by the current user appear on the right; received messages appear on the left
6. A small JavaScript `setInterval()` calls `includes/poll_messages.php` every 5 seconds
7. That endpoint returns any messages with a higher `pk_message_id` than the last seen ID, as a JSON array
8. New messages are appended to the thread without refreshing — this technique is called **short polling**
9. Incoming messages are marked as `is_read = 1` both on page load and during polling

### Files involved

- `user/chat.php` — message thread and send form
- `includes/poll_messages.php` — JSON polling endpoint
- `user/user_friends.php` — friends list with Chat buttons

### Learn more

- MDN — `setInterval`: https://developer.mozilla.org/en-US/docs/Web/API/setInterval
- MDN — `fetch()` API: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- YouTube — "Simple PHP chat application" — Dani Krossing

---

## 8. Support / Complaint Ticket System

### What it does

Users can submit a support request or complaint to the administrators.
Admins can view all tickets and reply to them.
Users receive a notification badge when a new reply arrives.

### How it works

**User side:**
1. User fills in a subject and description on `user/support.php`
2. The ticket is inserted into `support_ticket` with the user's username and timestamp
3. The ticket list shows the number of replies and a "new" badge for unread replies
4. Clicking "View" opens `user/support_ticket.php` which shows the ticket body and all admin replies
5. On load, all replies for that ticket are marked as `is_read = 1`

**Admin side:**
1. `admin/admin_support.php` lists all tickets from all users
2. Clicking "View" opens `admin/admin_support_ticket.php`
3. The admin can read the full ticket and submit a reply
4. The reply is inserted into `support_reply` with `is_read = 0`

### Files involved

- `user/support.php` — ticket list + new ticket form
- `user/support_ticket.php` — ticket detail (user view)
- `admin/admin_support.php` — all tickets (admin list)
- `admin/admin_support_ticket.php` — ticket detail + reply form

---

## 9. Notification Bell

### What it does

A bell icon in the top navigation bar shows a red badge with the total count of unread notifications.
Clicking it goes to the notifications page, which lists unread chat messages (grouped by sender) and unread support replies (grouped by ticket).

### How it works

1. On every page load, `includes/header.php` runs two COUNT queries:
   - Unread messages: rows in `message` where `fk_receiver = current user` and `is_read = 0`
   - Unread support replies: rows in `support_reply` joined to `support_ticket` where `fk_username = current user` and `is_read = 0`
2. If the total is greater than zero, a red Bootstrap badge is shown on the bell icon using CSS classes `position-absolute` and `badge`
3. A cart icon with a blue badge shows the total quantity of items in the user's cart
4. On `user/notifications.php`, the user can see all sources of unread notifications with direct links and mark everything as read

### Files involved

- `includes/header.php` — badge query + icon rendering
- `user/notifications.php` — full notification list

### Learn more

- Bootstrap badges: https://getbootstrap.com/docs/5.3/components/badge/
- Bootstrap Icons: https://icons.getbootstrap.com/

---

## 10. Measurement Graph and Improved Filtering

### What it does

The measurements page now has two additions:
1. **Filters** — users can select a specific station and/or a date range to narrow down the results
2. **Chart** — a line graph shows temperature and humidity over time for the filtered data set

### How it works

**Filters:**
1. A form at the top sends GET parameters: `station`, `date_from`, `date_to`
2. PHP builds a parameterized SQL query dynamically, adding `WHERE` conditions only for the parameters that were provided
3. All parameters are passed through PDO prepared statements — no risk of SQL injection

**Chart:**
1. The first 50 results (in chronological order) are prepared as PHP arrays
2. The arrays are encoded to JSON and embedded into a `<script>` block using `json_encode()`
3. Chart.js reads those arrays and renders a line chart with two datasets: temperature (blue) and humidity (purple)
4. Chart.js is loaded from the jsDelivr CDN — no installation needed

### Files involved

- `user/user_measurements.php` — filter form, dynamic query, table, chart

### Learn more

- Chart.js documentation: https://www.chartjs.org/docs/latest/
- YouTube — "Chart.js tutorial for beginners" — Traversy Media
- PHP `json_encode`: https://www.php.net/manual/en/function.json-encode.php

---

## 11. Admin Dashboard Improvements

### What it does

The admin dashboard now shows four live statistics at the top of the page:
- Total registered users
- Total registered stations
- Total measurements recorded today
- Total support tickets submitted

Two new navigation tiles were also added for **Products** and **Support**.

### How it works

Each stat is a single `SELECT COUNT(*)` SQL query run when the page loads.
The results are stored in the `$stats` array and displayed in glass-style stat cards.
The measurements-today query uses `WHERE DATE(timestamp) = CURDATE()` which compares only the date portion.

### Files involved

- `admin/dashboard.php` — stat queries + stat card HTML + new nav tiles

---

## Appendix A — Database Tables

### Original Tables (base schema)

| Table | Description |
|---|---|
| `user` | User accounts (`pk_username`, `password`, `role`, `status`, `mustChangePassword`) |
| `station` | Weather stations (`pk_serialNumber`, `fk_user_owns`, `name`) |
| `measurement` | Sensor readings (`timestamp`, `temperature`, `humidity`, `pressure`, `light`, `gas`) |
| `collection` | Named groups of measurements |
| `ismember` | Measurements belonging to a collection |
| `isfriend` | Friend relationships (`status`: pending / accepted / declined) |
| `issharing` | Stations shared between users |

### New Tables (database_updates.sql)

| Table | Description |
|---|---|
| `email_tokens` | Stores tokens for email verify / password reset / OTP login |
| `product` | Shop products (name, description, price, stock, image_url) |
| `cart` | User cart (unique per user + product, stores quantity) |
| `orders` | Order header (total, status, timestamp) |
| `order_item` | Snapshot of product name + price at checkout time |
| `message` | Chat messages between users (is_read flag) |
| `support_ticket` | User support requests (subject, body) |
| `support_reply` | Admin replies to tickets (is_read flag) |

---

## Appendix B — Technology Stack

| Technology | Version | Purpose |
|---|---|---|
| PHP | 8.0+ | Server-side logic |
| MySQL / MariaDB | 8.0+ | Database |
| Bootstrap | 5.3.2 (CDN) | Responsive UI layout and components |
| Bootstrap Icons | 1.11.3 (CDN) | Bell and cart icons in the navbar |
| Chart.js | 4.4.0 (CDN) | Line graph for measurement data |
| PHPMailer | Latest (Composer) | Sending emails via SMTP (Mailtrap) |
| Mailtrap | Free tier | Email testing inbox — catches all outgoing emails |

---

## Appendix C — Security Notes

- All passwords are hashed with `password_hash()` using bcrypt — no plaintext passwords are stored
- All SQL queries use PDO or MySQLi prepared statements — SQL injection is not possible
- All user-supplied output is escaped with `htmlspecialchars()` — XSS is prevented
- Reset and verification tokens use `random_bytes()` which is cryptographically secure
- OTP tokens expire after 15 minutes; verification tokens expire after 24 hours; reset tokens expire after 1 hour
- Admin pages check `$_SESSION['role'] === 'Admin'` before loading — unauthorised access redirects to the user dashboard
- The "Forgot password" page always shows the same response regardless of whether the email was found — this prevents user enumeration attacks
