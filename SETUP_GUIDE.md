# Setup Guide — Weather Station Project

## 1. Requirements

- PHP 8.0 or higher
- MySQL / MariaDB
- Apache or Nginx web server
- Composer (for PHPMailer)

---

## 2. Install PHPMailer

Open a terminal in the project root and run:

```bash
composer require phpmailer/phpmailer
```

**If Composer is unavailable** (e.g. on UniServer without internet):

1. Download PHPMailer from https://github.com/PHPMailer/PHPMailer/releases (choose the latest `.zip`)
2. Extract it — you will find a `src/` folder inside
3. Create the folder structure: `vendor/phpmailer/phpmailer/src/`
4. Copy all files from the archive's `src/` into that folder
5. Open `config/mailtrap.php` and replace the three `require` lines at the top with:
   ```php
   require __DIR__ . '/../vendor/phpmailer/phpmailer/src/Exception.php';
   require __DIR__ . '/../vendor/phpmailer/phpmailer/src/PHPMailer.php';
   require __DIR__ . '/../vendor/phpmailer/phpmailer/src/SMTP.php';
   ```
   Remove or comment out the `require __DIR__ . '/../vendor/autoload.php';` line.

---

## 3. Configure the Database

1. Open **phpMyAdmin**
2. Select your database (`weather_station_db`) — or create it if it does not exist
3. Click **Import**
4. Import the **base schema first**: `weather_station_db-db_2026-04-20 (1).sql`
5. Then import **`database_updates.sql`** — this adds the 8 new tables and 5 demo products

---

## 4. Configure the Database Connection

Open `includes/db_connection.php` and update the four variables at the top:

```php
$host   = 'localhost';
$dbname = 'weather_station_db';
$dbuser = 'root';    // your MySQL username
$dbpass = '';        // your MySQL password (blank by default on UniServer)
```

---

## 5. Configure Mailtrap (Email Testing)

All emails (verification links, OTP codes, password resets, welcome credentials) go through Mailtrap so no real email is sent during development.

1. Sign up free at **https://mailtrap.io**
2. Go to **Email Testing → Inboxes → SMTP Settings**
3. Open `config/mailtrap.php` and replace the placeholder values:
   ```php
   $mail->Username = 'YOUR_MAILTRAP_USERNAME';  // ← paste here
   $mail->Password = 'YOUR_MAILTRAP_PASSWORD';  // ← paste here
   ```

All emails will appear in your Mailtrap inbox instantly.

---

## 6. Set the Correct Base URL

In `auth/forgot_password.php` and `includes/register.inc.php`, the reset/verification links are built using `http://localhost/`. If your server runs on a different port or hostname, update those lines.

For UniServer the default is usually `http://localhost/` so no change is needed.

---

## 7. File Permissions (Linux / UniServer)

If the server returns permission errors, run in the project directory:

```bash
chmod -R 755 .
chmod -R 777 vendor/
```

---

## 8. Test Each Feature

| Feature | How to test |
|---|---|
| Registration | Go to `/auth/register.php`, fill in the form, check Mailtrap for the verification email, click the link |
| Email verification | The link in the Mailtrap email goes to `/auth/verify.php?token=...` — page says "Verified!" on success |
| Login OTP | Log in with a verified account; check Mailtrap for the 6-digit code; enter it on the verification page |
| Password reset | Click "Forgot password?" on login page; check Mailtrap for the reset link; set a new password |
| Admin creates user | Log in as admin → Users → Create; check Mailtrap for the welcome email with credentials |
| Shop | Log in as a regular user → Shop → add items → Cart → Checkout → Order History |
| Chat | Add a friend (Friends page), accept the request from another account, click "Chat" |
| Support | Go to Support, submit a ticket; log in as admin → Support to reply |
| Notifications | Receive a chat message or support reply; the bell icon in the navbar shows a red badge |
| Measurement graph | Go to My Measurements; apply date/station filters; Chart.js graph appears below |
| Admin dashboard stats | Log in as admin; the dashboard shows user count, station count, measurements today, ticket count |

---

## 9. Default Admin Account

The admin account comes from the original base schema. If it does not exist, insert one via phpMyAdmin:

```sql
INSERT INTO user (pk_username, firstName, lastName, email, password, role, status, mustChangePassword)
VALUES ('admin', 'Admin', 'User', 'admin@example.com',
        '$2y$10$REPLACE_WITH_REAL_HASH', 'Admin', 'verified', 0);
```

Generate the password hash in PHP:

```php
<?php echo password_hash('YourChosenPassword', PASSWORD_DEFAULT); ?>
```

Run this snippet once via your browser (e.g. save as `hash.php` in the project root, visit it, copy the hash, then delete the file).

---

## 10. Project Directory Structure

```
project-root/
├── index.php                    ← redirect-only entry point
├── add_measurement.php          ← sensor API (receives POST data from hardware)
├── database_updates.sql         ← import this AFTER the base schema
├── weather_station_db-....sql   ← base schema (import first)
│
├── auth/
│   ├── login.php
│   ├── register.php
│   ├── account.php
│   ├── forgot_password.php
│   ├── reset_password.php
│   ├── verify.php
│   └── verify_otp.php
│
├── user/
│   ├── dashboard.php
│   ├── user_stations.php
│   ├── user_measurements.php    ← filters + Chart.js graph
│   ├── user_friends.php         ← friend requests + Chat link
│   ├── user_collections.php
│   ├── user_sharing.php
│   ├── chat.php
│   ├── shop.php
│   ├── cart.php
│   ├── order_history.php
│   ├── notifications.php
│   ├── support.php
│   └── support_ticket.php
│
├── admin/
│   ├── dashboard.php            ← stat cards + nav tiles
│   ├── admin_users.php          ← emails credentials on create
│   ├── admin_stations.php
│   ├── admin_collections.php
│   ├── admin_collection_details.php
│   ├── admin_measurements.php
│   ├── admin_measurements_export.php
│   ├── admin_access_rights.php
│   ├── admin_products.php
│   ├── admin_support.php
│   └── admin_support_ticket.php
│
├── includes/
│   ├── db_connection.php        ← provides $link (MySQLi) + $pdo (PDO)
│   ├── header.php               ← navbar with bell + cart badges
│   ├── login.inc.php            ← OTP login processor
│   ├── register.inc.php         ← token-based registration processor
│   ├── logout.inc.php
│   ├── force_change_password.php
│   ├── cart.inc.php
│   ├── checkout.inc.php
│   └── poll_messages.php        ← JSON endpoint for chat polling
│
├── config/
│   ├── lang.php                 ← EN/DE translations
│   └── mailtrap.php             ← PHPMailer factory (update credentials here)
│
├── assets/
│   └── css/
│       ├── layout.css
│       └── sidebars.css
│
└── vendor/                      ← created by Composer (PHPMailer lives here)
```
