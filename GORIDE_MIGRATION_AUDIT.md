# GoRide — Production Migration Audit (old CyberPanel → new VPS)

- **Audit date:** 2026-11-21
- **Mode:** read-only static audit of local code, config and git history. No SSH, DNS changes, database access, live API calls, builds, migrations or scripts were run.
- **Secrets policy:** no password, key, token, private key, signing material or customer data is reproduced here. Credential files are cited by **path and type only**.
- **Old server:** 88.222.221.162 (CyberPanel, shared with unrelated sites). **New server:** 72.61.83.92 (Ubuntu 26.04.1, CyberPanel/OpenLiteSpeed, PHP CLI 8.5.4, MariaDB 11.8.6, Redis). **New domain:** goride1.com.

**Evidence tags:**

- **[VERIFIED]**: seen in local source, config or git at the cited line.
- **[INFERRED]**: strong evidence, but not proven.
- **[UNKNOWN]**: can't be settled locally. Section N lists the evidence that would settle it.

**Path prefixes used below:**

| Prefix | Absolute path | Git state at audit time |
|---|---|---|
| `CUST/` | `/Users/ge/goride-customer` | `main` @ `ff2b67d` (2026-09-10); uncommitted edits to `android/*.gradle`, `gradle.properties` and `pubspec.yaml`, which bump the version 4.0.9+24 → 4.0.10+25 |
| `DRV/` | `/Users/ge/goride-driver` | `main` @ `d3a2cb8` (2026-09-10); uncommitted edits to `android/app/build.gradle` and `pubspec.yaml` |
| `ADM/` | `/Users/ge/Herd/goride` | `master` @ `16a2885a` (2026-11-10); clean apart from `.DS_Store` |

---

## A. Executive summary

1. **The two mobile apps don't use the old server at all.** [VERIFIED in current source and in every commit of both app repos]
   - Neither Flutter app contains a REST or WebSocket base URL, an `/api` prefix, a server IP, `goride.cloud` or `goride1.com`.
   - All backend traffic goes to:
     - Firebase project `goride-a9d8f`: Auth, Firestore, Storage, FCM, Remote Config, and the Cloud Functions callables `sendNotification`, `bankilyPayment` and `bankilyCheckTransaction`.
     - Google Maps web services.
     - The public OSRM demo router.
     - `wa.me`.
   - So the "shared backend/API" of both apps is **Firebase, which is hosted by Google and does not move**.
   - **What this means for the published builds:** current App Store and Google Play builds should keep working with no change. This is [INFERRED] for the published binaries:
     - Both repos start at `version 1.0.0+1` (2026-02-19) and no server hostname appears anywhere in their history (checked with `git log -S`).
     - The store binaries themselves still need checking (N.17), together with the old server's access logs (N.11).
   - **No app release is needed for the migration.** Remote Config only drives force-update. There is no API URL to switch, and none is needed.

2. **What GoRide actually runs on the old server is the Laravel 10 admin dashboard.** The code is [VERIFIED]; the production hostname is [UNKNOWN]. It consists of:
   - Laravel 10.48.29.
   - A MySQL-protocol database holding admin users, roles and permissions, coupons, coupon usage, driver bans and the wallet-transaction audit ledger.
   - A scheduler that runs **6 jobs writing to production Firestore every 1–5 minutes**.
   - An optional Node.js script for scheduled-ride reminders.
   - The public pages `/privacy-policy` and `/terms-and-conditions`.
   - A server-side FCM sender.
   - Legacy template payment endpoints that the current apps never call.

3. **Top risks** (full list in H):
   - **Duplicate scheduler.** If cron runs on both servers, both write to the same Firestore: presence sweeps, auto-cancel, stale-order release, meter-total sync, and duplicate scheduled-ride pushes. `withoutOverlapping()` locks through the per-server file cache, so it can't stop two servers overlapping.
   - **PHP 8.5 is not compatible with the locked dependencies.** Six locked packages cap PHP at 8.4: kreait/firebase-php 7.18.0, kreait/laravel-firebase 5.10.0, kreait/firebase-tokens 5.2.1, lcobucci/jwt 5.5.0, beste/json and beste/in-memory-cache. Laravel 10 officially supports PHP 8.1–8.3. Run GoRide on **lsphp 8.3** (8.2 at minimum) on the new server.
   - **Security exposures that must not be copied to the new server:**
     - **3 distinct Firebase Admin service-account private keys** committed to git; one of them sits in the web root `public/`.
     - Adminer (`public/padmin.php`) in the web root.
     - `.env` tracked in git.
     - An **unauthenticated endpoint that mints a Firebase custom token** for a fixed admin UID.
     - An endpoint that overwrites the server credential file from a value the caller supplies.
   - **Unknown production details.** The admin hostname, cron line, PHP handler, `NODE_PATH` and database engine/version can't be determined locally.

4. **Downtime expectations:**
   - **Riders and drivers: effectively zero downtime.** They never touch the VPS. Only the admin-driven background sweeps pause during the freeze; Cloud Functions' own auto-cancel keeps running.
   - **Admin dashboard: near-zero, not guaranteed zero.** Plan a 10–20 minute write freeze, because the MySQL ledger must have a single writer, plus DNS propagation.

5. **Recommended path (fastest safe route):**
   1. Build the new site on lsphp 8.3 with `docRoot = <app>/public`.
   2. Rehearse with the restored backup on an access-restricted `goride1.com` staging host, with cron **OFF** and read-only tests. The restore also proves the backup is recoverable.
   3. Cut over in a short window:
      1. On the old server: `php artisan down` and disable cron.
      2. Take the final database dump and import it on the new server.
      3. Enable cron on the new server.
      4. Point the existing admin hostname (TTL lowered in advance) to 72.61.83.92.
   4. Keep the old GoRide site intact, in maintenance mode and with no cron, for a 7-day rollback window.
   5. Then rotate the exposed keys and decommission **only** the GoRide site on the old server.

---

## B. Workspace project inventory

| # | Project | Path | Framework / runtime | Entry points | Dependency files | Deployment / config files |
|---|---|---|---|---|---|---|
| 1 | **Customer app** | `CUST/` | Flutter, Dart `>=3.4.0 <4.0.0`; GetX, Provider, flutter_bloc; package `customer`, version `4.0.10+25` (`CUST/pubspec.yaml:19`) | `CUST/lib/main.dart` | `CUST/pubspec.yaml`, `CUST/pubspec.lock` | Android `applicationId "cloud.goride.customer"` (`CUST/android/app/build.gradle:54`); iOS bundle `cloud.goride.customer` (Xcode project); `CUST/lib/firebase_options.dart`, `CUST/android/app/google-services.json`, `CUST/ios/Runner/GoogleService-Info.plist`; release signing reads `key.properties` (not tracked) |
| 2 | **Driver app** | `DRV/` | Flutter; package `driver`, version `5.0.17+33` (`DRV/pubspec.yaml:19`) | `DRV/lib/main.dart` | `DRV/pubspec.yaml`, `DRV/pubspec.lock` | `applicationId "cloud.goride.driver"` (`DRV/android/app/build.gradle:54`); CI at `DRV/.github/workflows/ci.yml` (analyze, test, build APK; no deploy step) |
| 3 | **Admin dashboard** | `ADM/` | Laravel **10.48.29** (`ADM/vendor/composer/installed.php`); PHP `^8.1` (`ADM/composer.json:8`), resolved for PHP **8.2.0** (`ADM/composer.json:66-67`); Blade and jQuery with the Firebase JS SDK; Laravel Mix assets are **prebuilt and committed** in `ADM/public/` | `ADM/public/index.php`; root `ADM/server.php` plus root `ADM/.htaccess`, a hack that makes the project root the document root | `ADM/composer.json`, `ADM/composer.lock` (**`vendor/` is committed**, 39,041 files); `ADM/package.json` (`firebase-admin` is the runtime dependency of the Node script; no `node_modules` locally) | `ADM/.env.example`; `ADM/.env` (**tracked in git**; it's a local copy, `APP_URL=http://goride.test`); root `ADM/php.ini` (`memory_limit = 4096M`) |
| 4 | **Cloud Functions (v1)** | `CUST/Firebase Cloud Functions/` | Node **20**, firebase-functions v7 | `functions/index.js` (471 lines) | `functions/package.json` | `.firebaserc` default project `goride-a9d8f` |
| 5 | **Cloud Functions (v2)** | `ADM/Firebase Cloud Functions/` | Node **22**, firebase-functions v7 | `functions/index.js` (2,311 lines) plus `commission.js`, `dispatch_eligibility.js`, `presence.js` | `functions/package.json` | `.firebaserc` default project `goride-a9d8f`; **a service-account key is tracked here** |
| 6 | Firestore tooling and exports | `CUST/` and `ADM/Firebase Import Export Collections/`, `Firebase Indexing/` | Node scripts, JSON exports (`collections.json`, 4.7 MB) | — | `package.json` | Contains service-account key files (see H2). `collections.json` was **not opened** because it may contain customer data |

**There is no separate API backend project.** [VERIFIED] The "backend" is Firebase (Firestore plus Cloud Functions) together with the Laravel admin. `ADM/routes/api.php:16-23` defines only `GET /api/user` (Sanctum) and `GET /api/timezone`.

**Missing or unverifiable source:**

- **Deployed admin code** [UNKNOWN]: whether the old server runs `ADM@16a2885a`. There is no deploy script, and the deploy method is unknown.
- **Production runtime files** [UNKNOWN]: `.env`, `storage/app/firebase/credentials.json` (absent locally and created at runtime), the cron entry, and the vhost config.
- **Deployed Cloud Functions** [UNKNOWN]: there are **two divergent sources for the same project** (rows 4 and 5); see F.
- **Deployed Firestore and Storage rules** [UNKNOWN]: `CUST/` holds 3 rules variants (`firestore.rules`, `firestore_rules_simple.rules`, `firestore_rules_updated.rules`).
- **Other GoRide sites on the old server** [UNKNOWN]: any landing or marketing site on goride.cloud is not in the workspace.
- **Release provenance** [UNKNOWN]: the exact commit or working tree each published store build was made from. Neither app has flavors, `.env` files or URL dart-defines [VERIFIED]. The only `bool.fromEnvironment` flags are phone-auth debug switches (`CUST/lib/controller/login_controller.dart:143-147`, `DRV/lib/controller/login_controller.dart:144-148`).
- **Workspace file references:** `DRV/goride-driver.code-workspace` points to `../goride`, which does not exist. The admin lives at `/Users/ge/Herd/goride`.

---

## C. Production API hostname inventory

### C.1 Hosts contacted by the mobile apps (all [VERIFIED])

| Host / service | Purpose | Customer app reference | Driver app reference | On the old VPS? |
|---|---|---|---|---|
| Firebase project `goride-a9d8f` (Firestore, Auth, Storage, FCM, Remote Config via SDK) | All data, auth, push and file storage | `CUST/lib/firebase_options.dart` (RTDB URL `:57`, `:66`); `CUST/android/app/google-services.json`; `CUST/ios/Runner/GoogleService-Info.plist` | Same files under `DRV/` (`DRV/lib/firebase_options.dart:57,66`) | **No** |
| Cloud Functions `us-central1` callables: `sendNotification`, `bankilyPayment`, `bankilyCheckTransaction` | Push fan-out, Bankily payments | `CUST/lib/constant/send_notification.dart:8,27`; `CUST/lib/payment/bankily/bankily_service.dart:24,43-45,92-94` | `DRV/lib/constant/send_notification.dart:8,30,72` | **No** |
| `maps.googleapis.com` (Distance Matrix, Geocoding, Places Autocomplete, Details, Nearby) | ETA, reverse geocoding, place search | `CUST/lib/constant/constant.dart:313`; `CUST/lib/services/location_resolver.dart:309,365`; `CUST/lib/widget/google_map_search_place.dart:130,168` | `DRV/lib/constant/constant.dart:274`; `DRV/lib/controller/incoming_trip_request_controller.dart:194-206`; `DRV/lib/widget/google_map_search_place.dart:95,124` | No |
| `router.project-osrm.org` (public demo server, no SLA) | Route fallback | `CUST/lib/constant/constant.dart:344` | — | No |
| `firebasestorage.googleapis.com` placeholder image (deprecated, returns 404) | Avatar placeholder | `CUST/lib/constant/constant.dart:106-107` | `DRV/lib/constant/constant.dart:99-100` | No |
| `wa.me`, `whatsapp://`, `tel:` | Support contact | `CUST/lib/ui/contact_us/contact_us_screen.dart:79`, `CUST/lib/ui/settings_screen/setting_screen.dart:154`, `CUST/lib/utils/url_launcher_utils.dart:50` | `DRV/lib/utils/url_launcher_utils.dart:50`, `DRV/lib/ui/settings_screen/setting_screen.dart:264` | No |
| `play.google.com` | Force-update fallback store link | Remote Config keys | `DRV/lib/services/force_update_service.dart:34` | No |
| `goride.siswebapp.com` (template vendor) | `Constant.globalUrl`: **declared but never referenced** | `CUST/lib/constant/constant.dart:98-99` (the value is **corrupted**: Arabic text was pasted inside the URL) | `DRV/lib/constant/constant.dart:92` | No |

**Searched for and not found** in either app's source or in any commit of their history (`git log --all -S`):

- `goride.cloud`, `goride1`, `88.222.221.162`, `72.61.83.92`
- The template payment endpoints (`stripepaymentintent`, `getpaytmchecksum`, `paypalclientid`)
- Any Dio, WebSocket, Pusher or Socket.IO client. `dio` is declared in `CUST/pubspec.yaml` but never imported.
- WebView usage.

[VERIFIED]

### C.2 Can API URLs change remotely, or do they need a store release?

- There is **no API URL** to change. [VERIFIED]
- **Remote Config** is used only for force and optional updates:
  - Customer (`CUST/lib/services/force_update_service.dart:33-199`): `force_update_enabled`, `force_update_customer_version`, `latest_customer_version`, `play_store_url`, `app_store_url`, and the message keys.
  - Driver (`DRV/lib/services/force_update_service.dart:58-256`): `force_update_driver_enabled`, `force_update_driver_version`, `latest_driver_version`, `driver_play_store_url`, `driver_app_store_url`, and the message keys.
- **Google Maps key** is read from the Firestore `settings` collection, with a hard-coded fallback key in the customer app (`CUST/lib/utils/fire_store_utils.dart:157-169`; `DRV/lib/utils/fire_store_utils.dart:233-240`).
- **Privacy and terms content** shown in the apps comes from Firestore `settings` fields (`CUST/lib/utils/fire_store_utils.dart:219-228`, `DRV/lib/utils/fire_store_utils.dart:321-330`). No server URL is involved.
- **Social links** on the About screen come from Firestore data (`CUST/lib/ui/about_us/about_us_screen.dart:56-67`). [UNKNOWN] whether any live Firestore value points at the admin host (N.16).

### C.3 Deep links, callbacks, pinning and CORS (both apps)

- **No App Links or Universal Links.** There are no `autoVerify` intent filters and no `associated-domains` entitlement (`CUST/ios/Runner/Runner.entitlements`, `DRV/ios/Runner/Runner.entitlements`). [VERIFIED] **No domain dependency.**
- **Custom URL schemes:**
  - `${applicationId}.braintree`, a template leftover (`CUST/android/app/src/main/AndroidManifest.xml:89`, `DRV/android/app/src/main/AndroidManifest.xml:97`).
  - The Google Sign-In reversed-client-ID scheme (iOS `Info.plist`).
- **No certificate pinning** and no TLS overrides (no `badCertificateCallback`, `SecurityContext` or `HttpOverrides`).
- **Cleartext is disabled:** `usesCleartextTraffic="false"` (`CUST/android/app/src/main/AndroidManifest.xml:33`, `DRV/android/app/src/main/AndroidManifest.xml:54`). The customer app sets iOS ATS `NSAllowsArbitraryLoads=false`.
- **CORS** doesn't apply because the clients are native HTTP.

### C.4 Actual app flows and their backend

| Flow | Mechanism | Depends on VPS? |
|---|---|---|
| Authentication / OTP | Firebase Auth `verifyPhoneNumber`, Google Sign-In, Sign in with Apple (`signInWithCredential`) | No |
| Ride request and dispatch | App writes `orders` in Firestore and calls the `sendNotification` callable. Cloud Functions triggers in the ADM source include `monitorOrderRideRequests`, `advanceRideAfterDriverRejection` and `setOrderAutoCancelAt` | No |
| Driver accept | Firestore transaction (first driver wins) | No |
| Live tracking | Firestore streams on the order and driver documents | No |
| Driver availability | Driver app writes presence and location to Firestore. **The Laravel `presence:expire-stale --mode=enforce` job flips stale drivers offline every minute** | **Auxiliary only.** If paused, stale drivers stay "online" a little longer; nothing breaks |
| Payments | Cash and wallet through Firestore plus Cloud Functions settlement triggers (ADM source); Bankily through Cloud Functions → eBankily | No |
| Wallet | App and Cloud Functions write Firestore. Admin adjustments go through Laravel, which writes MySQL and mirrors to Firestore | Admin adjustments only |
| Notifications | `sendNotification` callable (apps); Laravel FCM v1 for admin manual and broadcast pushes; Laravel + Node for scheduled-ride reminders | Admin and scheduled pushes only |
| Chat / admin chat | Firestore (`.ref('messages')` appears only in a pagination widget docstring) | No |
| Uploads | Firebase Storage (e.g. `DRV/lib/constant/constant.dart:310,477,492,514,540`) | No |
| FCM topics | `goRide_customer`, `goRide_driver` | No |

### C.5 Admin dashboard hostnames

- **`APP_URL`** [UNKNOWN in production; the local value is `http://goride.test`]. It is used by:
  - The Node reminder script, which calls `${APP_URL}/api/timezone` (`ADM/storage/app/firebase/scheduleNotification.js:45-46`).
  - Password-reset links and console URL generation.
- **Same-origin endpoints:** the admin's browser JavaScript calls relative paths (`/get-firebase-token` and `/store-firebase-service` at `ADM/public/js/jquery.validate.js:43,86`), so it is **hostname-agnostic**.
- **Server-side outbound traffic:**
  - `firestore.googleapis.com` (`ADM/app/Services/Firebase/FirestoreRestClient.php:16`)
  - `oauth2.googleapis.com` (`FirestoreRestClient.php:559-564`)
  - `fcm.googleapis.com` (`ADM/app/Http/Controllers/NotificationController.php:82,275`)
  - A download of the service-account JSON through `file_get_contents` (`ADM/app/Http/Controllers/HomeController.php:100-101`, `ADM/app/Providers/FirebaseAuthService.php:16`), which requires `allow_url_fopen`
  - The SMTP host [UNKNOWN]
- **Browser-side CDNs:**
  - Libraries: gstatic (Firebase JS SDK), unpkg, jsdelivr, cdnjs, DataTables, Google Fonts.
  - Map services: the Google Maps JS API, with its key read from Firestore `globalKey.googleMapKey` (`ADM/resources/views/layouts/app.blade.php:488,506`); Nominatim; OSRM.
- **Vendor-hosted assets, unaffected by the migration:**
  - `goride.siswebapp.com/images/default_user.png` (`ADM/resources/views/drivers/view.blade.php:30`, `ADM/resources/views/users/view.blade.php:32`)
  - `goride-landing.siswebapp.com` (`ADM/resources/views/settings/landingpage_template.blade.php:28`)
- **No GoRide production domain or IP is hard-coded** anywhere in the admin source. [VERIFIED]
- **goride.cloud:** [INFERRED] owned by the business (the bundle IDs are `cloud.goride.*`) and listed on the old CyberPanel. [UNKNOWN] whether the admin is served on the apex or on a subdomain.
- **goride1.com:** referenced nowhere. [VERIFIED]

---

## D. Backend availability and architecture

### D.1 Real architecture

```
Customer/Driver apps ──Firebase SDK──► Firestore · Auth · Storage · FCM · Remote Config   (Google, project goride-a9d8f)
        │                         └──► Cloud Functions us-central1 (callables + triggers + schedulers) ──► eBankily API
        └──HTTPS──► Google Maps web services, OSRM demo router

Admin browser ──Firebase JS SDK (custom token)──► Firestore · Storage
        └──HTTPS──► Laravel admin  (OLD VPS 88.222.221.162)
                         ├── MySQL/MariaDB: admin users/roles/permissions, coupons, driver_bans, wallet_transactions (ledger)
                         ├── Firestore REST + FCM v1 (service-account file)
                         └── cron → artisan schedule:run → 6 jobs → Firestore   (+ optional Node script → FCM)
```

### D.2 Laravel admin: backend facts

**Runtime** [VERIFIED]

- Laravel 10.48.29.
- `vendor/composer/platform_check.php:7` requires PHP >= 8.2.
- `ADM/.htaccess:21` sets `AddHandler application/x-httpd-php82`, so production most likely runs **PHP 8.2** [INFERRED].

**Routes** (`ADM/routes/web.php`, 532 lines)

- `Auth::routes(['register' => false])` (`:20`).
- Public pages: `/privacy-policy` and `/terms-and-conditions` (`:23-29`). They render Firestore `settings/global` content client-side (`ADM/resources/views/public/privacy_policy.blade.php:10`).
- Permission-gated admin modules.
- Legacy "API Url for app" payment routes, exempt from CSRF through `payments/*` (`ADM/app/Http/Middleware/VerifyCsrfToken.php:14-17`):
  - `payments/razorpay/createorder` (`:43`)
  - Paytm, Braintree, Stripe and the `payment/success|failed|pending` pages (`:519-529`)
- **Firebase token endpoints** `store-firebase-service` and `get-firebase-token`, with no auth middleware (`:531-532`).

**Authentication**

- Laravel session authentication.
- `CheckUserRoleMiddleware` in the `web` group.
- The `permission` alias → `PermissionMiddleware` (`ADM/app/Http/Kernel.php`).
- The browser signs in to Firebase with a **custom token minted for a hard-coded admin UID** (`ADM/app/Providers/FirebaseAuthService.php:20-34`).
  - The token is fetched by `ADM/public/js/jquery.validate.js:36-60`. **Despite its name, this file is custom Firebase bootstrap code**, not the jQuery Validate library.
  - The file is loaded on the **login page** (`ADM/resources/views/auth/login.blade.php:442`) and in the main layout (`ADM/resources/views/layouts/app.blade.php:130`).

**Database**

- `mysql` driver with `utf8mb4` / `utf8mb4_unicode_ci` and `strict` mode (`ADM/config/database.php:55-59`).
- The migrations (`ADM/database/migrations/`, 12 files) create: users, password_resets, failed_jobs, personal_access_tokens, the Spatie permission tables, coupons, coupon_usage_history, driver_bans and wallet_transactions, plus `user_type` and user `status` columns.
- `2026_04_06_190100_align_role_permission_schema.php` reconciles a legacy roles/permissions shape. This suggests the **production schema came from a vendor SQL dump, not from migrations** [INFERRED]. Migrate with a logical dump, and **never** by running `php artisan migrate` on an empty database.
- Column types in use: `decimal` ×8, `enum` ×3, `json` ×1. All are MariaDB-compatible.

**Dual writes, MySQL plus a Firestore mirror** [VERIFIED]

- Wallet: `ADM/app/Services/WalletService.php:55-61,137-193`. The MySQL row is the authoritative audit record; the Firestore copy is `wallet_transaction/{id}` plus the balance.
- Coupons: `ADM/app/Http/Controllers/CouponController.php:172-195`.
- Driver bans: `ADM/app/Http/Controllers/DriverController.php:87`.

**Cache, queue, session and broadcast**

- The defaults are `file` / `sync` / `file` / `log` (`ADM/config/cache.php:18`, `ADM/config/queue.php:16`, `ADM/config/session.php:21`), and the local `.env` matches.
- **There are no `ShouldQueue` jobs**, so no queue worker is needed. [VERIFIED]
- **There is no Redis or WebSocket usage in code.** The Pusher and Redis variables exist only as `.env` names from the template.

**Uploads**

- All uploads go from the browser to **Firebase Storage**: 30 Blade views use `storageRef.child(...)`. [VERIFIED]
- There are no server-side upload handlers, and no `storage:link` (there is no `public/storage`).

**Scheduler** (`ADM/app/Console/Kernel.php`)

| Line | Command | Frequency | Writes |
|---|---|---|---|
| `:36` | `ride:auto-cancel` | every minute | Firestore `orders`. This duplicates the Cloud Function `autoCancelStaleRideRequests`, which also runs every minute |
| `:37` | `app:send-scheduled-ride-notification` | every minute | `shell_exec` of a Node script (`ADM/app/Console/Commands/SendScheduledRideNotification.php:28,35`) that sends FCM reminders. No idempotency marker was found, so a double run means **duplicate pushes** [INFERRED] |
| `:48` | `presence:expire-stale --mode=enforce` | every minute, `withoutOverlapping(10)` | `driver_users` (`isOnline`, `offlineReason`), `presence_recovery_state`, `presence_sweep_audit` |
| `:55` | `ops:health-check` | every 5 min | Log, optional email (`ADM/config/ops.php:19-21`), file-cache state |
| `:61` | `drivers:release-stale-orders` | every 5 min | `driver_users.currentOrderId` |
| `:69` | `orders:sync-meter-totals --apply` | every 5 min | `orders` totals (CAS-guarded) |

- None of these jobs use `evenInMaintenanceMode`, and the framework skips scheduled events while the app is down (`ADM/vendor/laravel/framework/src/Illuminate/Console/Scheduling/Event.php:361`). **`php artisan down` therefore freezes both web writes and the scheduler.** [VERIFIED]

**Node dependency**

- `NODE_PATH` is read with `env()` (`SendScheduledRideNotification.php:28`), which **breaks under `config:cache`**.
- The project root must have a `node_modules` containing `firebase-admin` and `axios`. **`axios` is only a devDependency** in `ADM/package.json`, so `npm ci --omit=dev` would break the script.
- `shell_exec` must be enabled.
- `APP_URL` must be in the process environment.
- `ADM/app/Console/Commands/AutoCancelRide.php:29-33` notes that **`NODE_PATH` was unset in the past**, so the reminder job may currently do nothing in production [INFERRED; N.7].

**Credentials**

- `storage/app/firebase/credentials.json` is used by:
  - `FirestoreRestClient.php:25-32`
  - `NotificationController.php:40-41`
  - `FirebaseAuthService.php:29`
  - The Node scripts (`ADM/storage/app/firebase/scheduleNotification.js:2`)
  
  It is **not in git**. **On every admin page load** it is **overwritten unconditionally** with whatever `file_get_contents()` returns for Firestore `settings/notification_setting.serviceJson`. [VERIFIED] The chain is: `ADM/public/js/jquery.validate.js:65` → `POST /store-firebase-service` (`ADM/routes/web.php:531`) → `ADM/app/Providers/FirebaseAuthService.php:14-16`. (`HomeController.php:99-102` contains a write-if-missing copy of this method, but no route points to it.) Because the fetch must return the key JSON, `serviceJson` is presumably a URL to that file [INFERRED].
  - **Copy the file to the new server before enabling cron**, because the scheduler needs it before any admin visits the site.
  - **A rotated key placed on disk will be overwritten by the old key on the next admin page load** unless `serviceJson` is changed too (H2).
- `public/firebase_credentials.json` (service account, **tracked**) is referenced only by `ADM/config/firebase.php:50` (`public_path(env('FIREBASE_CREDENTIALS'))`), for kreait's container services.
  - kreait is otherwise used only through `new Factory` with the storage credential file (`FirebaseAuthService.php:29`); no code resolves the container services. [VERIFIED]
  - The file is therefore **probably unnecessary** [INFERRED; confirm during the rehearsal].

**`env()` outside config files** [VERIFIED]

- Locations: `ADM/resources/views/layouts/app.blade.php:119-127,615` and `SendScheduledRideNotification.php:28`.
- **Do not run `php artisan config:cache`** on the new server: it would blank the Firebase web config, which then falls back to a hard-coded config, and `NODE_PATH`.

**Timezone**

- `ADM/config/app.php:70` is `Asia/Kolkata`, a template default; the business is in Mauritania, UTC+0. This is a pre-existing issue that affects reminder time calculations. Keep it unchanged during the migration.

**Required environment variable names** (no values)

- **Core:** `APP_NAME`, `APP_ENV`, `APP_KEY`, `APP_DEBUG`, `APP_URL`
- **Logging:** `LOG_CHANNEL`, `LOG_DEPRECATIONS_CHANNEL`, `LOG_LEVEL`
- **Database:** `DB_CONNECTION`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`
- **Drivers:** `BROADCAST_DRIVER`, `CACHE_DRIVER`, `FILESYSTEM_DRIVER`, `QUEUE_CONNECTION`, `SESSION_DRIVER`, `SESSION_LIFETIME`
- **Mail:** `MAIL_MAILER`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_ENCRYPTION`, `MAIL_FROM_ADDRESS`, `MAIL_FROM_NAME`
- **Firebase:** `FIREBASE_APIKEY`, `FIREBASE_AUTH_DOMAIN`, `FIREBASE_DATABASE_URL`, `FIREBASE_PROJECT_ID`, `FIREBASE_STORAGE_BUCKET`, `FIREBASE_MESSAAGING_SENDER_ID` (misspelling is in the code), `FIREBASE_APP_ID`, `FIREBASE_MEASUREMENT_ID`, `FIREBASE_CREDENTIALS`
- **Read by code but absent from the local `.env`:** `NODE_PATH`, `OPS_MIN_AVAILABLE_DRIVERS`, `OPS_ALERT_EMAIL`, `OPS_ALERT_COOLDOWN_MINUTES`
- **Template leftovers, unused:** `MEMCACHED_HOST`, `REDIS_*`, `AWS_*`, `PUSHER_*`, `MIX_PUSHER_*`

### D.3 What needs consistent migration of live data

- **Live ride data: nothing on the VPS.** It all lives in Firestore and Cloud Functions, which don't move. [VERIFIED]
- **The MySQL admin ledger:**
  - Tables: `wallet_transactions`, `coupons`, `coupon_usage_history`, `driver_bans`, `users`, `roles`, `permissions`, `password_resets`, `personal_access_tokens`, `failed_jobs`.
  - Its only writer is Laravel web actions, so a **controlled write freeze** is enough (J).
- **The scheduler singleton:** exactly one active cron across both servers.
- **Server-local files:**
  - `credentials.json`: carry it over, or replace it with a rotated key.
  - File cache (the ops alert state and the access-token cache): regenerated automatically.
  - Sessions: admins log in again.
  - **Reuse the production `APP_KEY`.**

### D.4 If you want certainty about the backend on the old server, inspect

The deployed code revision, the production `.env` (names and non-secret values only), the cron line, the PHP handler version and extensions, `NODE_PATH` and `node_modules`, the database engine/version/size/collations, the presence of credential files, and the access logs. Section N gives exact commands.

---

## E. Admin requirements on the new server

| Item | Requirement | Evidence |
|---|---|---|
| PHP | **lsphp 8.3** (8.2 only short-term, because PHP 8.2's security support ends 2026-12-31). **Not 8.5** for this migration. Use the same binary for cron, artisan and composer | `ADM/composer.lock` (see G); `ADM/.htaccess:21` |
| PHP extensions | ctype, curl, dom, fileinfo, filter, hash, json, libxml, mbstring, openssl, pcre, session, sodium, tokenizer, xmlwriter (union of `ext-*` required by the locked packages), plus **pdo_mysql**. Enable OPcache. gRPC is **not** needed (Firestore is accessed over REST) | `ADM/composer.lock`, `FirestoreRestClient.php:16` |
| php.ini | `allow_url_fopen=On`. `shell_exec` must not be disabled if the Node reminder script is used. A sane `memory_limit` (the repo ships a root `php.ini` with 4096M). Upload limits don't matter because uploads go to Firebase | `FirebaseAuthService.php:16`, `SendScheduledRideNotification.php:35`, `ADM/php.ini` |
| Web server | OpenLiteSpeed vhost with `docRoot = <app>/public`. Rewrite to `index.php`, either via the "auto-load .htaccess" option or in vhost-level rules. **Do not replicate** the root `.htaccess` + `server.php` hack. HTTPS with an HTTP→HTTPS redirect | `ADM/.htaccess:1-25`, `ADM/public/.htaccess` |
| Database | A MariaDB 11.8 database plus a dedicated user with privileges on that database only. `utf8mb4` / `utf8mb4_unicode_ci` | `ADM/config/database.php:55-59` |
| Cron | One line, **enabled only at cutover**: `* * * * * /usr/local/lsws/lsphp83/bin/php <app>/artisan schedule:run >> /dev/null 2>&1`, run as the site user | `ADM/app/Console/Kernel.php` |
| Node.js | Only if production sets `NODE_PATH`. Install Node LTS, run `npm ci` at the app root (dev dependencies included, or add `axios`), and set `NODE_PATH` to the node binary | `SendScheduledRideNotification.php:28-35` |
| Writable paths | `storage/` (`app/firebase`, `logs`, `framework/*`) and `bootstrap/cache` | — |
| Files to carry | The production `.env` (**same `APP_KEY`**) and `storage/app/firebase/credentials.json`, or a new rotated key | D.2 |
| Files **not** to publish | `public/padmin.php` (Adminer), `public/firebase_credentials.json`, `public/firebase_test.html`, `public/firebase_test2.html`, root `check_firebase.php` and `debug_perms.php` | H2–H4 |
| Outbound network | TCP 443 to `*.googleapis.com` and `firebasestorage.googleapis.com`; the SMTP host and port; the server's own `APP_URL` (Node script) | C.5 |
| Build | None: compiled assets and `vendor/` are committed. If you reinstall dependencies, run `composer install --no-dev` **using the PHP 8.3 binary** | B |
| Redis | Not required; the installed Redis can stay unused | D.2 |
| Do not run | `php artisan config:cache`; `php artisan migrate` against an empty database | D.2 |

---

## F. External integrations

| Integration | Used by | Depends on VPS? | Migration impact / notes |
|---|---|---|---|
| Firebase Auth (phone OTP, Google, Apple) | Apps | No | None |
| Firebase custom token (admin browser) | Admin | Yes (credential file) | Carry the credentials; H3 (security) |
| Firestore | Apps, Cloud Functions, admin (REST and browser) | Admin only | Keep `credentials.json` |
| Firebase Storage | Apps and admin browser | No | None |
| FCM | Received by apps. Sent by Cloud Functions (`sendNotification`), by Laravel (`NotificationController`), and by the Node reminder script | Admin pushes | Verify a send to a test device after cutover |
| Cloud Functions | Apps | No (they never call the VPS; both sources use `admin.initializeApp()` default credentials: `CUST/Firebase Cloud Functions/functions/index.js:7`, `ADM/Firebase Cloud Functions/functions/index.js:21`) | **Two divergent sources target the same project.** `CUST` (Node 20) has `bankilyPayment`, `bankilyCheckTransaction`, `sendNotification`, `deleteUser` and auto-cancel. `ADM` (Node 22) has `sendNotification`, the settlement and dispatch triggers, auto-cancel and `deleteUser`, but **no Bankily functions**. A `firebase deploy --only functions` from either source would offer to delete the other's functions. **Freeze function deploys during the migration** |
| eBankily B-PAY | Customer app via Cloud Functions | No | The source is configured for **TEST**: `BANKILY_ENV = 'test'` (`CUST/Firebase Cloud Functions/functions/index.js:30`) and `isProduction = false` (`CUST/lib/payment/bankily/bankily_config.dart:25`). The deployed state is [UNKNOWN] |
| Google Maps Platform | Apps (key from Firestore plus the manifest `com.google.android.geo.API_KEY` at `CUST/android/app/src/main/AndroidManifest.xml:72` and `DRV/android/app/src/main/AndroidManifest.xml:87`); admin JS API | Admin JS only | If the admin's hostname changes, add it to the key's HTTP-referrer restrictions [UNKNOWN] |
| OSRM public server, Nominatim | Customer app, admin | No | Third-party usage policies, no SLA (pre-existing) |
| Remote Config | Apps (force update) | No | None |
| Crashlytics, Analytics, App Check | Apps | No | The admin doesn't use App Check. Enforcing App Check on Firestore would break the admin (pre-existing) |
| SMTP | Laravel (password reset, ops alerts) | Yes | Host [UNKNOWN]. If mail is hosted on the old CyberPanel, keep it there and **leave MX/SPF/DKIM/DMARC untouched** |
| Legacy Stripe, Razorpay, Braintree, Paytm SDKs | Laravel routes (`ADM/routes/web.php:43,519-529`) | Yes | No current app calls them [VERIFIED]. Check provider dashboards and access logs for stray callbacks (N.11) |
| Firebase Auth authorized domains, API key restrictions | Admin web | — | Add the new hostname as a precaution if it changes (additive, safe) |

---

## G. Infrastructure compatibility

| Component | Old server | New server | Assessment / action |
|---|---|---|---|
| OS / panel | [UNKNOWN] / CyberPanel | Ubuntu 26.04.1 / CyberPanel | Verify that CyberPanel and the LiteSpeed repo support 26.04, **including installable `lsphp83`/`lsphp82` packages** [UNKNOWN] |
| Web server | LiteSpeed. Apache-style `.htaccess` with `AddHandler …php82` suggests LiteSpeed Enterprise or Apache semantics [INFERRED] | **OpenLiteSpeed** | OLS applies only rewrite rules from `.htaccess`, ignores `AddHandler`/`Header`, and needs a graceful restart after `.htaccess` edits [INFERRED from OLS docs]. Use `docRoot=public` and test rewrites |
| PHP | 8.2 [INFERRED] | CLI 8.5.4 | See below. **Use lsphp 8.3 for GoRide** |
| Database | [UNKNOWN]. CyberPanel normally ships MariaDB [INFERRED] | MariaDB 11.8.6 | Logical dump/restore; check collations and `DEFINER`s; use the `mariadb-dump`/`mariadb` client names |
| Redis | [UNKNOWN] | Present | Not used by GoRide code |
| Node | [UNKNOWN] | [UNKNOWN] | Needed only if `NODE_PATH` is used |
| TLS | [UNKNOWN] (probably Let's Encrypt via CyberPanel) | Let's Encrypt via CyberPanel | HTTP-01 for the production hostname works only after DNS moves. Pre-provision via DNS-01 or copy the current certificate (K) |

### PHP 8.2 → 8.5 risks [VERIFIED from `ADM/composer.lock`]

- **`composer install` fails on PHP 8.5.** The constraint `~8.1.0 || ~8.2.0 || ~8.3.0 || ~8.4.0` is declared by kreait/firebase-php 7.18.0, kreait/laravel-firebase 5.10.0, kreait/firebase-tokens 5.2.1, beste/json 1.5.1 and beste/in-memory-cache 1.3.1. lcobucci/jwt 5.5.0 declares `~8.2.0 || ~8.3.0 || ~8.4.0`.
- **The committed `vendor/` would load on PHP 8.5**, because `platform_check.php` only requires >= 8.2. But this combination is **untested**:
  - Laravel 10 officially supports PHP 8.1–8.3, and its security support ended in February 2025 (Laravel support policy).
  - Expect deprecation floods on 8.4 and later, and possible fatal errors in third-party code.
- **PHP 8.3 needs no dependency change** (every lock constraint allows ~8.3) and is supported until the end of 2027. **Recommended.**
- Moving to PHP 8.4 or 8.5 is a **separate upgrade project** after the migration: Laravel 11/12, kreait 7.x latest or 8.x, Sanctum and Spatie upgrades.

### MariaDB 11.8 risks

- **MySQL 8 source:** if the old server runs MySQL 8, dumps may contain `utf8mb4_0900_*` collations. Test the import, and rewrite those collations to `utf8mb4_unicode_ci` if MariaDB rejects them.
- **Default collation:** MariaDB 11.x changed its defaults (UCA 14.0). The app config pins `utf8mb4_unicode_ci`; check that imported tables keep their source collation.
- **JSON columns:** stored as `LONGTEXT` with a `JSON_VALID` check, which is fine. `enum` and `decimal` are also fine.
- **System schema:** do not import the `mysql` system schema or global users. Create a dedicated user.

---

## H. High-risk blockers (ordered)

| # | Severity | Blocker | Evidence | Required handling |
|---|---|---|---|---|
| H1 | **Critical** | **Duplicate scheduler / split-brain writes.** Two crons means double presence sweeps, double auto-cancel and stale-order release, and duplicate reminder pushes. Two writable admins means MySQL ledger divergence (the Firestore mirror is shared). `withoutOverlapping()` uses the local file cache, so it can't coordinate across servers | `ADM/app/Console/Kernel.php:36-69`; `ADM/config/cache.php:18` | Exactly **one** cron at any time. Cron stays OFF on the new server until cutover. Run `php artisan down` and disable cron on the old server before the final dump. To detect a double run, look for two `presence_sweep_audit` rows per minute |
| H2 | **Critical (security)** | **Firebase Admin private keys exposed.** There are **3 distinct private keys** for the same service account across 2 git repos: `ADM/public/firebase_credentials.json` (**in the web root**), `ADM/Firebase Cloud Functions/functions/serviceAccountKey.json`, and `ADM/` + `CUST/Firebase Import Export Collections/*adminsdk*.json` / `serviceAccountKey.json`. Also tracked: `ADM/.env` (5 commits) and a 4.7 MB Firestore export `collections.json` in both repos (may contain PII [UNKNOWN]; not opened). The local rules allow public reads of `settings` (`CUST/firestore.rules:432-434`), and `settings/notification_setting.serviceJson` is the server's credential source, so the key may be world-readable if those rules are deployed [UNKNOWN] | `git ls-files` (paths only) | If `public/firebase_credentials.json` exists on production, it is **downloadable under either docroot layout** [INFERRED]. Never place a key under `public/` on the new server. Use a **new key** on the new server. After the rollback window, **disable and delete the 3 exposed keys**, plus the key production uses if it's a fourth one. Cloud Functions use the default credentials, so revoking these keys won't affect them [VERIFIED]. **Rotation dependency:** `credentials.json` is re-fetched from `settings/notification_setting.serviceJson` on every admin page load (D.2). Either point `serviceJson` at the new key, or (better) place the key file on disk and empty `serviceJson`, so the key no longer has a Firestore-referenced source. If neither is done, the old key silently comes back. First confirm that nothing else reads `serviceJson`. Purge the keys from git history and check repo visibility (N.18) |
| H3 | **Critical (security)** | **Unauthenticated `POST /get-firebase-token`** mints a Firebase custom token for a hard-coded admin UID, and the login page calls it before anyone logs in. **`POST /store-firebase-service`** fetches a caller-supplied value with `file_get_contents()` and overwrites `credentials.json`: SSRF, local file read into the credential file, and denial of service for the push and Firestore jobs. Both are protected only by CSRF, and a token is obtainable from the login page | `ADM/routes/web.php:531-532`; `ADM/app/Providers/FirebaseAuthService.php:12-34`; `ADM/public/js/jquery.validate.js:36-60,86`; `ADM/resources/views/auth/login.blade.php:442` | The new server will expose the same endpoints, so keep the **staging site access-restricted** (IP allowlist or basic auth). A code fix is a priority follow-up (out of scope for this read-only audit). The impact depends on the deployed Firestore rules [UNKNOWN] |
| H4 | High (security) | **Adminer** (appears to be 4.8.1) committed at `ADM/public/padmin.php`, giving a public database-login surface | `git ls-files` | Exclude it from the new server. Check whether it's reachable on the old server (N.11) |
| H5 | High | **PHP runtime.** PHP 8.5 is incompatible with the lock file, and `lsphp83`/`lsphp82` availability on Ubuntu 26.04 + CyberPanel is unverified | G | Verify before anything else on the new server. Fallback: PHP-FPM 8.3 (e.g. from a maintained PPA or a container) behind an OLS proxy |
| H6 | High | **Production facts are unknown**: admin hostname, docroot, PHP handler, cron line, `NODE_PATH`, database engine/version, mail host, deployed revision | — | Collect Section N before building |
| H7 | Medium | **Root-docroot hack** (`ADM/.htaccess`, `ADM/server.php`) can expose non-public files depending on server semantics, and OLS rewrites behave differently | `ADM/.htaccess:9-18` | Set `docRoot=public` on the new server. Afterwards, `/.env`, `/composer.json`, `/storage/logs/laravel.log` and `/check_firebase.php` must return 404 |
| H8 | Medium | **External references to the admin hostname**: store-listing privacy, terms and account-deletion URLs, API-key referrer restrictions, password-reset links, the Node script's `APP_URL` | `ADM/routes/web.php:23-29`; `scheduleNotification.js:45-46` | Keep the existing hostname, pointed at the new server. If you later move to goride1.com, add 301 redirects and update the allowlists first |
| H9 | Medium | **`env()` outside config + `config:cache`** would blank the Firebase web config and `NODE_PATH` | D.2 | Don't cache config |
| H10 | Medium | **Two divergent Cloud Functions sources** for one project | F | Freeze `firebase deploy` during the migration |

**Non-blocking observations (pre-existing, unrelated to the VPS):**

- **iOS bundle ID mismatch:** `GoogleService-Info.plist` declares `BUNDLE_ID` `cloud.gorid.customer` / `cloud.gorid.driver` (`CUST/ios/Runner/GoogleService-Info.plist:18`, `DRV/ios/Runner/GoogleService-Info.plist:18`; also `iosBundleId` at `CUST/lib/firebase_options.dart:72` and `DRV/lib/firebase_options.dart:72`). The Xcode bundles are `cloud.goride.*`.
- **Corrupted `globalUrl`** in the customer app (unused).
- **Vendor-hosted images** on `siswebapp.com`.
- **Bankily in TEST mode** in source.
- **Admin timezone** set to `Asia/Kolkata`.
- **Public OSRM demo server** used as a production routing fallback.

---

## I. Prioritized migration steps

### Phase 0 — Evidence and change freeze (Day 0, read-only)

1. Collect Section N, priority items N.1–N.11 and N.17 first.
2. Freeze code and deploys for `ADM` and both Cloud Functions sources until the migration is finished.
3. **Decide the hostname strategy.** Recommended: keep the existing production admin hostname and point it at the new server. Use `goride1.com` (or `staging.goride1.com`) for staging now, and optionally as an alias or redirect later.

### Phase 1 — Verified backups (GoRide only)

1. On the old server, using `nice`/`ionice` so unrelated sites aren't affected, archive the GoRide site directory **including** `.env`, `storage/app/firebase/` and `public/`, and excluding caches and large logs.
2. Dump the database:
   ```
   mysqldump --single-transaction --quick --routines --triggers --events --hex-blob \
     --no-tablespaces --default-character-set=utf8mb4 <goride_db> | gzip > goride_<ts>.sql.gz
   ```
   Use `mariadb-dump` instead if the old server is MariaDB 11.
3. Record `sha256sum` for both artifacts, and per-table row counts.
4. Copy the backup off the server, encrypted. Remove the temporary copies from the old server afterwards.
5. **The backup counts as verified only once it has been restored on the new server** (Phase 3).

### Phase 2 — Build the replacement (no public traffic, **no cron**)

1. Confirm that `lsphp83` and its extensions are installable, then create the CyberPanel site for the staging host with `docRoot=<site>/public`. **Restrict access** with an IP allowlist or basic auth at the vhost level. Issue a Let's Encrypt certificate (HTTP-01 works for goride1.com).
2. Deploy the exact production revision, by rsync from the backup or a git checkout of the deployed commit. **Exclude:**
   - `public/padmin.php`, `public/firebase_test*.html`, `public/firebase_credentials.json`, `check_firebase.php`, `debug_perms.php`
   - `storage/framework/down`, which would otherwise be copied from a site in maintenance mode
   - Old logs and sessions
3. If the rehearsal shows that kreait's config file is still needed, keep it **outside** `public/`, for example `FIREBASE_CREDENTIALS=../storage/app/firebase/kreait.json`. `public_path()` resolves relative paths, so no code change is required.
4. Prepare the `.env`:
   - Start from the production `.env` with the **same `APP_KEY`**.
   - Use the new database credentials.
   - During the rehearsal only, set `APP_URL` to the staging URL.
   - Set `APP_DEBUG=false`.
   - Put `credentials.json` in place. If it's a **new rotated key**, apply the `serviceJson` rotation dependency in H2 first; otherwise the next admin page load replaces it with the old key.
5. Node: only if production uses `NODE_PATH`. Run `npm ci` at the app root and set `NODE_PATH`.
6. Fix permissions. Run `php artisan view:clear` and `php artisan route:clear`. **Do not** run `config:cache`.
7. **Verify there is no GoRide cron** on the new server: `crontab -l -u <siteuser>`.

### Phase 3 — Rehearsal (restore + read-only validation)

1. Create the database and user, import the dump, and compare per-table counts with the source. Check table collations.
2. Run the admin smoke tests in **read-only** mode (M): login, dashboards, lists, map pages, public pages.
   - **No writes:** no wallet changes, coupons, bans, rides or notifications.
   - Staging writes would hit **production Firestore** while landing in a throw-away MySQL copy, and that divergence corrupts the ledger.
3. Exercise Firestore REST auth with **read-only dry runs** [VERIFIED flags]:
   - `php artisan ride:auto-cancel --dry-run`
   - `php artisan drivers:release-stale-orders --dry-run`
   - `php artisan orders:sync-meter-totals` (without `--apply`, which makes it a dry run)
   - `php artisan schedule:list`
   
   Avoid `presence:expire-stale` and `ops:health-check` here: they write audit rows or send email.
4. Review `storage/logs` for PHP 8.3 errors and deprecations.
5. Time the dump, transfer and import to size the freeze window.
6. Rehearse the rollback commands (L) as a dry walkthrough.

### Phase 4 — Pre-cutover (T-48h → T-1h)

1. Lower the TTL of the admin hostname's A/AAAA records to **300 s**, at least 48 h ahead. If DNS goes through the Cloudflare proxy, switching the origin is instant.
2. Pre-provision TLS for the production hostname on the new server: use DNS-01, or copy the current certificate and key from the old server. Verify with `curl --resolve <host>:443:72.61.83.92 -I https://<host>/login`.
3. If the hostname will change, add the new one to the Maps/web API key referrers and to Firebase Auth authorized domains. These changes are additive and safe.
4. Announce the admin maintenance window, choose a low-traffic hour, and have one tester for each app ready.

### Phase 5 — Cutover (target 10–20 minutes)

1. **T0, old server:** run `php artisan down --retry=60` on the GoRide site only. This blocks web writes **and** the scheduler. Then comment out the GoRide cron line.
2. Wait for any running `artisan` scheduled processes to exit (`ps aux | grep artisan`).
3. Take the final dump, then checksum it, transfer it, import it on the new server, and verify counts against the old server.
4. **New server:**
   1. Set `APP_URL` to the production hostname.
   2. Test through `--resolve`: login, read-only pages, then **one controlled write on a designated test account**, for example a 1 MRU wallet credit followed by a 1 MRU debit. Verify both the MySQL row and the Firestore mirror.
5. **Enable cron on the new server** (one line), and watch the first runs in the logs.
6. **Switch the DNS A record** of the admin hostname to 72.61.83.92. Change no other record.
7. During propagation, the old site keeps showing its 503 maintenance page (public privacy/terms pages included, for about the TTL).
8. Run the smoke tests (M) on the admin and on **both currently published apps**.

### Phase 6 — Stabilize (24–72 h)

1. Monitor `storage/logs`, OLS access and error logs, scheduler outputs, FCM sends, the Firebase console (Firestore and Functions errors) and Crashlytics, which should stay flat.
2. Keep the old GoRide site **in maintenance with cron disabled** for 7 days. Don't delete it.
3. After 48 h, raise the TTL back to 3600 s or more.

### Phase 7 — Harden and decommission

1. Rotate the keys:
   1. Confirm the new server uses the new key, and that `settings/notification_setting.serviceJson` no longer serves the old one (H2).
   2. Disable the 3 exposed keys in GCP IAM, plus the old production key if it's a fourth one.
   3. Watch for authentication errors for 24 h.
   4. Delete the keys.
2. Fix H3, remove Adminer, purge secrets from git history, and make the repos private.
3. Decommission **only** the GoRide vhost, database, cron and files on the old server, keeping the archived backup. Leave unrelated sites untouched.
4. Plan the PHP 8.4/8.5 path: a Laravel 11/12 upgrade with dependency refresh.

---

## J. Database consistency and synchronization plan

- **Scope:** only the Laravel admin's MySQL database. Firestore, RTDB and Storage are shared and Google-hosted, so they are **not migrated**, and no ride-data sync is needed. [VERIFIED]
- **Writers:** only Laravel web actions: wallet add and deduct, coupons, bans, admin users and roles, password resets, Sanctum tokens, and `failed_jobs`. Apps and Cloud Functions have **no** MySQL access. [VERIFIED] Scheduled jobs keep their state in Firestore (`presence_sweep_audit`, `presence_recovery_state`) and in the file cache, not in MySQL. [VERIFIED]
- **Strategy: controlled write freeze plus a full logical dump.** Binlog replication is rejected because it would need a server-wide MariaDB config change and restart on the **shared** old server for very little gain on a small database.
- **Freeze:** `php artisan down` on the old server suspends web writes and the scheduler (`Event.php:361`; no `evenInMaintenanceMode`). Comment out the cron line as well.
- **Import:** create the database with `utf8mb4` / `utf8mb4_unicode_ci` and import. If the source is MySQL 8, handle `utf8mb4_0900_*` collations and `DEFINER` clauses after reviewing the dump.
- **Verification, in this order:**
  1. Per-table `COUNT(*)`.
  2. `MAX(id)` and `MAX(updated_at)` per table.
  3. An ordered export hash of the key tables (`wallet_transactions`, `coupons`, `coupon_usage_history`, `driver_bans`, `users`).
  4. Spot-check the latest `wallet_transactions` rows against the Firestore `wallet_transaction` mirror.
  
  `CHECKSUM TABLE` can differ across server versions, so treat it as advisory.
- **Duplicate-job prevention:** exactly one cron. Check `crontab -l` on **both** servers at every phase gate. Laravel's `onOneServer()` isn't usable because the servers share no cache. No queues exist (`sync`; no `ShouldQueue`).
- **Firestore mirror:** freezing the old admin also freezes its mirror writes, and the new admin resumes them. Wallet balances in Firestore are also changed by the apps and Cloud Functions; the move doesn't affect that.
- **Rollback data path:** export the delta rows created on the new server after T0 (from `wallet_transactions`, `coupons`, `coupon_usage_history`, `driver_bans`, `users`, `roles`, `permissions`) and replay them on the old server before running `php artisan up` there.

---

## K. DNS, TLS and production cutover plan

- **Production hostname X** [UNKNOWN; per the CyberPanel listing, probably goride.cloud or one of its subdomains]: point **only** X's A record (and its AAAA, if one exists) at 72.61.83.92. **Do not touch** MX, SPF, DKIM, DMARC, CAA or any record belonging to other sites. If the goride.cloud apex serves other content, move only the admin subdomain.
- **TTL:** set it to 300 s at least 48 h before cutover. The old TTL has to expire before the low value takes effect.
- **TLS:**
  - Staging (`goride1.com`): HTTP-01, available immediately.
  - X: pre-provision on the new server via DNS-01 (acme.sh with the DNS provider's API), or install a read-only copy of the current certificate and key from the old server. After cutover, issue a fresh certificate via CyberPanel and confirm auto-renewal.
  - If the old site sends **HSTS**, the new server must have a valid certificate from the very first request.
  - Make sure any CAA record allows Let's Encrypt.
- **Pre-cutover verification without DNS changes:** `curl --resolve X:443:72.61.83.92`, or a hosts-file override on the tester's machine.
- **During propagation:** the old site keeps returning 503 maintenance responses, so stragglers can't write. An optional old→new reverse proxy is only worth it if it can be confined to the GoRide vhost; it needs a LiteSpeed graceful restart, which touches the whole server.
- **Later move to goride1.com (optional):** because no mobile build uses the admin host, `admin.goride1.com` can be added safely as an alias, with **301 redirects from X** (store-listing URLs included). Update the API-key referrers and Firebase authorized domains first.
- **Monitoring during cutover:** `tail -f storage/logs/laravel.log`, the OLS access/error logs, the Firebase console (Firestore usage and Functions errors), FCM delivery, and Crashlytics.

---

## L. Rollback strategy

- **Triggers:**
  - Admin login fails, or Firebase custom-token sign-in fails.
  - Firestore REST or FCM authentication errors.
  - 3 or more consecutive failed scheduler runs.
  - A PHP fatal error.
  - Data mismatch during verification.
- **Rollback before the DNS switch** (the most likely case):
  1. Run `php artisan up` on the old server.
  2. Re-enable the old cron.
  
  Nothing else has changed, so there is no data impact.
- **Rollback after the DNS switch, within the window:**
  1. **New server:** run `php artisan down` and remove the cron.
  2. Export the MySQL delta created since T0 and replay it on the old server (J).
  3. **Old server:** restore the cron and run `php artisan up`.
  4. Point the DNS A record back to 88.222.221.162. With TTL 300, this takes effect in about 5–10 minutes.
  5. Re-run the smoke tests.
- **Keys:** keep the old service-account key valid until the rollback window closes (7 days), because the old server depends on it. Rotate afterwards.
- **Mobile apps:** no rollback is needed because they have no dependency on the VPS.
- **Rehearsal:** walk through these exact commands during Phase 3.

---

## M. Mobile and dashboard smoke-test checklist

Run these checks before, during and after cutover. **Use the currently published store builds, not new builds.**

**Customer app** (expected to be unaffected):

- [ ] Phone OTP login
- [ ] Google and Apple sign-in
- [ ] Map loads, place search and ETA work
- [ ] Request a ride; a nearby driver receives the push
- [ ] Live tracking, then driver arrived, in progress and completed
- [ ] Cash and wallet payment
- [ ] Bankily, in whatever environment is deployed
- [ ] Wallet history
- [ ] Coupon created in the admin applies in the app
- [ ] Chat
- [ ] Push notifications
- [ ] Privacy and terms screens
- [ ] Force-update check
- [ ] Intercity flow
- [ ] Order history

**Driver app:**

- [ ] Login
- [ ] Go online and stay online; the new server's presence sweep must not wrongly flip active drivers offline
- [ ] Receive a ride request (push and overlay)
- [ ] Accept, navigation and ETA
- [ ] Status transitions through completion
- [ ] Earnings and wallet
- [ ] A withdrawal request appears in the admin's payout requests
- [ ] Document upload
- [ ] "Why no rides?" diagnosis

**Admin, on the new server:**

- [ ] Valid TLS certificate for X; `/login` renders
- [ ] Login works; the browser console shows "Firebase authenticated successfully"
- [ ] Dashboard statistics load
- [ ] Users and drivers lists and detail pages load
- [ ] Zone and map pages load the Google Maps JS API (no referrer errors)
- [ ] Settings pages read Firestore
- [ ] **Designated test account only:**
  - [ ] Wallet add and deduct → MySQL row plus Firestore mirror
  - [ ] Coupon create, edit and delete → visible in the app
  - [ ] Ban and unban a test driver
- [ ] Notification sent to a **test device only**. Skip broadcast, which reaches every user
- [ ] `/privacy-policy` and `/terms-and-conditions` render
- [ ] Password-reset email arrives
- [ ] `php artisan schedule:list` is correct
- [ ] Each job logs **once** per interval
- [ ] `presence_sweep_audit` gets **one** row per minute
- [ ] `ops:health-check` email arrives, if configured
- [ ] No PHP error or deprecation flood
- [ ] **These return 404:** `/padmin.php`, `/firebase_credentials.json`, `/firebase_test.html`, `/.env`, `/composer.json`, `/storage/logs/laravel.log`, `/check_firebase.php`

---

## N. Exact evidence needed from the old CyberPanel (all read-only)

Each command is meant to be run by the operator, prints **names and metadata only**, and must touch only GoRide's site, database and logs.

1. **GoRide hostnames and docroot:**
   ```
   ls /home
   grep -il goride /usr/local/lsws/conf/vhosts/*/vhost.conf
   grep -E "docRoot|vhDomain|vhAliases|extprocessor|lsphp" <that vhost.conf>
   ```
   Also check the child domains and aliases in the CyberPanel UI.
2. **Web server edition:** `/usr/local/lsws/bin/lshttpd -v` (OpenLiteSpeed or LiteSpeed Enterprise).
3. **PHP:** the lsphp version bound to the vhost; `<lsphp>/bin/php -v`; `<lsphp>/bin/php -m`; `<lsphp>/bin/php -i | grep -E "disable_functions|allow_url_fopen|memory_limit|max_execution_time"`; and any `.user.ini` in the site directory.
4. **Deployed revision:** `cd <site> && git rev-parse HEAD && git status --short | head` if it's a git checkout. Otherwise, a file list newer than `artisan`, excluding `vendor/` and `storage/`, to diff against `ADM@16a2885a`.
5. **Production `.env`** (names, plus non-secret values only):
   ```
   grep -oE '^[A-Z_]+=' .env
   grep -E '^(APP_ENV|APP_DEBUG|APP_URL|DB_CONNECTION|DB_HOST|CACHE_DRIVER|QUEUE_CONNECTION|SESSION_DRIVER|BROADCAST_DRIVER|FILESYSTEM_DRIVER|MAIL_MAILER|MAIL_HOST|MAIL_PORT|NODE_PATH|FIREBASE_PROJECT_ID|FIREBASE_CREDENTIALS|OPS_[A-Z_]+)=' .env
   ```
6. **Cron:** `crontab -l -u <site-user>`, `crontab -l` (root), `ls /etc/cron.d`, and the CyberPanel cron UI. Record the exact PHP binary and path.
7. **Node:** `which node; node -v; ls <site>/node_modules | grep -xE 'firebase-admin|axios'`; and `grep -c "Node path is not defined" storage/logs/laravel*.log`.
8. **Database:**
   - `SELECT VERSION(), @@character_set_server, @@collation_server, @@sql_mode;`
   - `SELECT table_name, engine, table_collation, table_rows, ROUND((data_length+index_length)/1048576,2) AS mb FROM information_schema.tables WHERE table_schema='<db>';`
   - `SHOW TRIGGERS; SHOW PROCEDURE STATUS WHERE Db='<db>'; SHOW EVENTS;`
   - `SHOW GRANTS FOR '<dbuser>'@'localhost';`
9. **Runtime files** (existence and size only): `ls -la storage/app/firebase/ public/firebase_credentials.json public/padmin.php` and `du -sh storage/logs storage/framework`.
10. **TLS:** `openssl x509 -in /etc/letsencrypt/live/<host>/fullchain.pem -noout -subject -issuer -dates -ext subjectAltName`, plus the renewal mechanism.
11. **Access logs, last 30 days** (the key proof for the mobile conclusion):
    ```
    grep -cE "Dart/|okhttp|CFNetwork" <access.log>
    grep -E "Dart/|okhttp|CFNetwork" <access.log> | awk '{print $7}' | cut -d? -f1 | sort | uniq -c | sort -rn | head -50
    ```
    Also count hits and status codes for `/api/`, `/payments/`, `/payment/`, `/privacy-policy`, `/terms-and-conditions`, `/padmin.php`, `/firebase_credentials.json` and `/.env`. A `200` on a sensitive path means it is exposed.
12. **Laravel errors:** `grep -h "production.ERROR" storage/logs/laravel-*.log | cut -c1-200 | sort | uniq -c | sort -rn | head`.
13. **Mail:** CyberPanel → Email accounts for the GoRide domains; the MX records; whether `MAIL_HOST` (item 5) is the old server itself.
14. **DNS:** the authoritative name servers; the current A, AAAA, CNAME, MX, TXT and CAA records and TTLs for goride.cloud and every GoRide subdomain; registrar and DNS provider access, and whether an API exists for DNS-01.
15. **Other GoRide assets** on the old server: landing site, extra subdomains, phpMyAdmin/Adminer usage.
16. **Firebase / GCP console** (read-only):
    - Auth authorized domains.
    - Restrictions on the web API key and the Maps key.
    - IAM service-account **key list** (creation dates; compare against the 3 exposed keys).
    - The deployed Cloud Functions list (names, runtime, region, last deploy).
    - The deployed Firestore and Storage rules.
    - Whether `settings/notification_setting.serviceJson` is a URL or inline content. **Don't copy the value.**
17. **Store consoles** for both apps:
    - The privacy-policy, account-deletion (Play Data safety), support and marketing URLs.
    - The production version codes.
    - **Offline binary check:** Play Console → App bundle explorer → download the signed universal APK, then run `unzip -p app.apk 'lib/*/libapp.so' | strings | grep -oE 'https?://[A-Za-z0-9.-]+' | sort -u`. For iOS, run the same check against the archived `.app` on the release machine. This confirms that no VPS host is embedded in the published builds.
18. **GitHub:** the visibility and collaborator list of `goride606-jpg/goride`, `mohamedfahmy1318/goride-customer` and `mohamedfahmy1318/goride-driver`. This sets the blast radius of the exposed keys.

---

## O. Next five fastest safe actions

1. **Prove that the mobile apps are independent of the old server** (about 30 min): access-log analysis (N.11) plus the offline string scan of the published APK (N.17). If both are clean, the migration has no mobile risk and needs no app release.
2. **Collect the old-server evidence pack** (N.1–N.10): hostname, docroot, PHP handler, cron line, `NODE_PATH`, `.env` names, database version/size/collations, TLS.
3. **Take checksummed, off-server backups** of the GoRide files and database only (Phase 1).
4. **On the new server:** confirm that `lsphp83` and its extensions install on Ubuntu 26.04/CyberPanel. Create the **access-restricted** `goride1.com` staging site with `docRoot=public` and **no cron**, and restore the backup into it. This also verifies the backup.
5. **Lower the admin hostname's TTL to 300 s.** In parallel, have the operator send a single HEAD request each to `/firebase_credentials.json` and `/padmin.php` on the old host. If either returns 200, treat the key as compromised and rotate it immediately (new key on old and new servers, then disable the old keys), independently of the migration date.

---

### Audit method and limitations

- **What was examined:** source and config files in all three repos; git history searched with `git log --all -S` for hostnames and template endpoints; JSON credential files inspected **only for their structure** (key names, type, project id, and comparison of key IDs without printing them); `composer.lock` constraint analysis.
- **What was not examined:** no runtime behaviour, network calls, DNS queries, database contents or binaries. The Firestore export `collections.json` was not opened.
- **Unconfirmed:** everything about the deployed state (servers, Firebase console, stores) is [UNKNOWN] until Section N is completed.
