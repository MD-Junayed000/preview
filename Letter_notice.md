# Supplier Repayment Status, Reminder SMS and Legal Notices

This document covers how a supplier's repayment DPD is shown (profile, `/suppliers` list, filter and export), the reminder and legal-notice SMS, the three Bangla legal notice PDFs, the log of all of it, and how to test everything locally.

It builds on the escalation rules in [`docs/supplier_notice.md`](supplier_notice.md). DPD is **not** recalculated anywhere new: every feature here reads `SupplierOverdueEscalationService`.

## 1. Current Logic

### 1.1 DPD (days past due)

A supplier can have many invoices. Each invoice's repayment to Hishabee counts as overdue when it is `running` or `ready`, `computeDue()['remaining_amount'] > 0` and `computeDue()['days_late'] >= 1` (see `supplier_notice.md` §1.1).

| Value | Source | Meaning |
| --- | --- | --- |
| `dpd` | `status()['dpd']` | Highest `days_late` among the overdue repayments, i.e. the oldest unpaid one. `0` = nothing overdue. |
| `due_date` | `status()['due_date']` | Due date of that oldest repayment. |
| `dpd90_date` | `status()['dpd90_date']` | `due_date + 90 days`, the cheque-case date. |
| `overdue_amount` | `status()['overdue_amount']` | Sum of `remaining_amount` across all overdue repayments. |
| Invoice rows | `overdueRepayments()` | Every overdue invoice with its own due date, DPD and remaining amount. |

`SupplierOverdueEscalationService::dpdBySupplier()` returns the same `dpd` for every supplier in one pass over open repayments (`[supplier_id => dpd]`). The `/suppliers` DPD filter and the export use it, so they don't run one query per supplier.

### 1.2 Legal notice levels

| Level | Trigger | PDF template | Label | SMS event |
| --- | --- | --- | --- | --- |
| 1 | DPD ≥ 15 | `Legal_Notice_DPD_15_Bangla` | প্রথম নোটিশ | `notice_1_sms` |
| 2 | DPD ≥ 30 | `Legal_Notice_DPD_30_Bangla` | দ্বিতীয় নোটিশ | `notice_2_sms` |
| 3 (final) | DPD ≥ 60, including 90+ | `Legal_Notice_DPD_60_Bangla` | চূড়ান্ত নোটিশ (তৃতীয় ও শেষ) | `notice_3_sms` |

`SupplierLegalNoticeService::noticeLevel($dpd)` returns 3, 2, 1 or `null` (DPD below 15).

**Case a: the higher level wins.** The level comes from the supplier's highest DPD. The notice lists **every** overdue invoice, whatever its own DPD. Example: invoices at DPD 60, 20 and 5 give notice 3 with all three invoices in the table.

**Case b: cycle reset.** A "cycle" is the set of notice log rows with `resolved_at = null`. When the supplier has no overdue repayment left (DPD 0), `syncCycle()`:

1. sets `resolved_at` on every open log row of that supplier, and
2. writes a `cycle_reset` row.

The profile then shows "Not applicable yet". If the supplier becomes overdue again later, notices and notice SMS start again from level 1 with new notice numbers. `syncCycle()` runs on every profile view, every notice download and in the daily SMS command.

**Issuing a notice.** The first download of a level in a cycle creates a `notice_generated` row with:

- `notice_no` = `HTL/LN{level}/{year}/{log id padded to 5}`, e.g. `HTL/LN2/2026/00014`. HTL = Hishabee Technologies Limited, LN = Legal Notice; the prefix is configurable.
- `reference_date` = the notice date (see below);
- `payload` = a frozen snapshot, as of the notice date, of the recipient, the invoice table, the total, the DPD and the DPD 90 date.

**Earlier notices are always filled in.** Issuing level 2 or 3 first issues any lower level missing from the cycle, so notice 2 always cites notice 1 and notice 3 always cites notices 1 and 2. Notice dates:

| Notice | Date |
| --- | --- |
| The supplier's current level | Today (the day it is downloaded) |
| A lower level filled in automatically | The day its notice SMS was sent in this cycle; without a sent SMS, the oldest due date + 15 (notice 1) or + 30 (notice 2) days, i.e. the day the supplier reached that level |

Its snapshot uses each invoice's days late on that date (only invoices already overdue then). Example: invoices at DPD 60, 20 and 5 with no notice SMS → notice 1 dated 45 days ago listing one invoice at DPD 15, notice 2 dated 30 days ago, notice 3 dated today listing all three at DPD 60, 20 and 5.

Later downloads of a level in the same cycle reuse the row, so the PDF is identical (same number, date and figures). Every download also writes a `notice_downloaded` row.

**Download rules** (`GET /suppliers/{id}/legal-notice/{level}`):

| Request | Result |
| --- | --- |
| Level ≤ the supplier's current level | Issues it (and missing lower levels) if needed, returns the PDF. |
| Level above the current level, or DPD < 15 | 422. No notice exists below DPD 15. |
| Supplier not field-verified | 403. |

**DPD below 15.** No notice PDF can be downloaded, the profile shows no download button, and the SMS command sends no notice SMS (only the pre-due reminders, which are not about overdue). This also holds when a supplier drops back below 15 after paying part of the overdue (no cycle reset, because DPD is still above 0).

### 1.3 PDF content

Rendered by mPDF from `resources/views/pdf/legal-notice.blade.php` (one template, `@switch($level)` for the three bodies, wording as in the three source PDFs).

| Field on the PDF | Filled with |
| --- | --- |
| Company name and ঠিকানা | `config('finance.legal_notice.company_name' / 'company_address')` |
| নোটিশ নং / তারিখ | `notice_no` / notice date |
| ডিলার আইডি / রেফারেন্স নম্বর | Supplier `id` |
| নাম | `suppliers.name` |
| ডিস্ট্রিবিউশন হাউজ | `suppliers.business_name` |
| এনআইডি নম্বর | `suppliers.nid_number` |
| মোবাইল নম্বর | `suppliers.phone` |
| এলাকা ও ঠিকানা | Zone name + `suppliers.address` |
| Invoice table | ইনভয়েস নং (`invoice_number`), ইনভয়েসের তারিখ (`created_at`), পরিশোধের নির্ধারিত তারিখ (repayment due date), বকেয়া টাকা (`remaining_amount`), বকেয়ার দিন (`days_late`); oldest first |
| মোট বকেয়া | `overdue_amount` |
| (৯০ দিন পূর্ণ হওয়ার তারিখ) | `dpd90_date` |
| Signatory নাম / পদবী / যোগাযোগ | `config('finance.legal_notice.signatory_*')`, dotted blanks when empty |

Numbers and dates print in Bangla digits (`SupplierLegalNoticeService::bn()`), dates as `dd/mm/yyyy`.

**"… দিন যাবত বকেয়া রয়েছে".** Notices 1 and 2 keep the template's ১৫ / ৩০. Notice 3 prints the exact DPD on its date (e.g. ৯৫ দিন).

**Notice 3 after day 90.** If the notice date is after the DPD 90 date, the "এটিই আপনাকে প্রদত্ত চূড়ান্ত নোটিশ। আপনাকে স্পষ্টভাবে জানানো যাচ্ছে যে, আপনি যদি … তারিখের মধ্যে …" paragraph is replaced by:

> **এটিই আপনাকে প্রদত্ত চূড়ান্ত নোটিশ।** ব্যাংক আপনার অ্যাকাউন্টটি ব্লক করেছে। আপনার ৳ [মোট বকেয়া] টাকার বকেয়া [DPD 90 date, dd-mm-yyyy] তারিখে ৯০ দিন অতিক্রম করেছে এবং এখন [DPD] দিন অতিক্রান্ত হয়েছে। ৯০ দিন অতিক্রম করায় ব্যাংক আপনার বিরুদ্ধে **চেক ডিজঅনার মামলা** দায়ের করবে। অনুগ্রহ করে অবিলম্বে আপনার বকেয়া পরিশোধ করুন।

Up to and including the DPD 90 date the original "pay by (৯০ দিন পূর্ণ হওয়ার তারিখ)" paragraph is used. The check uses the notice date, which is the download day for a newly issued notice; a notice issued before day 90 keeps its wording on re-download.

**Font.** Text uses Hind Siliguri (`resources/fonts`, SIL Open Font License, licence file alongside; covers Bangla and Latin). mPDF's bundled FreeSerif has no Bengali bold (bold Bangla prints as boxes), and mPDF can't parse the GPOS tables of the current Noto Bengali fonts. Hind Siliguri's ১ reads like ৬, so every Bangla number is wrapped in a span set in FreeSerif regular (`bndigits`), which has the standard ১. Inside bold text, numbers therefore print in regular weight.

### 1.4 Reminder and notice SMS

Command `suppliers:send-repayment-sms`, scheduled daily at 10:00 AM Bangladesh time (`app/Console/Kernel.php`; `config/app.php` timezone is `Asia/Dhaka`, and the server cron must run `php artisan schedule:run` every minute). One run sends both the pre-due reminders and the notice SMS. For every supplier (optionally `--supplier=ID`):

1. Loads the open repayments (`running`/`ready`, `remaining_amount > 0`, with a due date) and computes `status()` from them.
2. Runs `syncCycle()` (case b).
3. Skips suppliers with no phone or no open repayment.
4. **Before the due date:** open repayments are grouped by due date. For a due date of today + 3, today + 1 or today, one SMS per supplier per due date, X = sum of the remaining amounts due that day:

| Event | When | Text |
| --- | --- | --- |
| `reminder_3d` | due date = today + 3 | আপনার কিস্তি ৳X পরিশোধের শেষ তারিখ ৩ দিন পর, dd/mm। - Hishabee |
| `reminder_1d` | due date = today + 1 | আপনার কিস্তি ৳X পরিশোধের শেষ তারিখ আগামীকাল, dd/mm। - Hishabee |
| `due_today` | due date = today | আজ কিস্তি ৳X পরিশোধের শেষ দিন। আজই পরিশোধ করুন। - Hishabee |

   Deduplicated by supplier + event + `reference_date` (the due date).

5. **After the due date:** the SMS for the current notice level only, once per cycle:

| Event | Text |
| --- | --- |
| `notice_1_sms` | প্রথম সতর্কবার্তা: আপনার ৳[বকেয়ার পরিমাণ] টাকা গত [দিন] দিন ধরে বকেয়া রয়েছে। অবিলম্বে পরিশোধ করুন। অন্যথায় প্রযোজ্য নিয়ম অনুযায়ী CIB রিপোর্টে বকেয়ার তথ্য প্রতিফলিত হলে ভবিষ্যতে ব্যাংক ও আর্থিক প্রতিষ্ঠান থেকে ঋণসুবিধা গ্রহণে বাধা সৃষ্টি হতে পারে। পাশাপাশি আইনানুগ ব্যবস্থা নেওয়া হইবে। |
| `notice_2_sms` | দ্বিতীয় সতর্কবার্তা: পূর্বের নোটিশ সত্ত্বেও আপনার ৳[বকেয়ার পরিমাণ] টাকা গত [দিন] দিন ধরে অপরিশোধিত রয়েছে। অবিলম্বে সম্পূর্ণ বকেয়া পরিশোধ করুন। অন্যথায় প্রযোজ্য নিয়ম অনুযায়ী CIB রিপোর্টে বকেয়ার তথ্য প্রতিফলিত হওয়া, ভবিষ্যৎ ঋণসুবিধায় বাধা এবং আইনানুগ ব্যবস্থার সম্মুখীন হইবেন। |
| `notice_3_sms` | চূড়ান্ত সতর্কবার্তা: আপনার ৳[বকেয়ার পরিমাণ] টাকা গত [দিন] দিন ধরে বকেয়া রয়েছে। [শেষ তারিখ]-এর মধ্যে সম্পূর্ণ অর্থ পরিশোধ করুন। অন্যথায় প্রযোজ্য নিয়ম অনুযায়ী CIB রিপোর্টিংসহ আইনানুগ ব্যবস্থা এবং আপনার প্রদত্ত চেকের বিপরীতে চেক ডিজঅনার মামলা দেওয়া হইবে। ভবিষ্যৎ ঋণসুবিধাও বাধাগ্রস্ত হইবেন। |

   [বকেয়ার পরিমাণ] = `overdue_amount`, [দিন] = `dpd`, [শেষ তারিখ] = `dpd90_date` (`dd/mm/yyyy`), all in Bangla digits. Because the check is "this level not yet sent in this cycle", a missed daily run catches up the next day, and a supplier who jumps from DPD 10 to DPD 35 gets only notice 2's SMS.

**Sending.** `SMSService::smsSend($message, $phone, 'supplier_repayment_<event>', 0, $supplier, true)`, which tries Banglalink first, then GP, then Robi (see §3.2). The log row's `status`:

| `status` | When |
| --- | --- |
| `sent` | One of the three gateways accepted it; `response` holds that gateway's reply. |
| `failed` | All three failed (`'error'`) or an exception. Retried on the next run. |
| `skipped` | `Setting sms_service = off` (`smsSend` returned `sms_disabled`). Retried on the next run. |
| `dry_run` | `--dry-run`: no gateway is called. A dry run never blocks a later real send. |

Real runs deduplicate on `sent` rows only. Dry runs deduplicate on `sent` and `dry_run` rows.

### 1.5 Log

Table `supplier_notice_logs` (model `SupplierNoticeLog`), one row per operation:

| Column | Notes |
| --- | --- |
| `channel` | `sms`, `pdf` or `system` |
| `event` | `reminder_3d`, `reminder_1d`, `due_today`, `notice_1_sms`, `notice_2_sms`, `notice_3_sms`, `notice_generated`, `notice_downloaded`, `cycle_reset` |
| `level`, `notice_no` | Notice rows only |
| `reference_date` | Due date (reminders), oldest due date (notice SMS), notice date (PDF rows) |
| `dpd`, `amount` | At the time of the operation |
| `message` | The exact SMS text, or a short description |
| `payload` | Notice snapshot (recipient + invoices) or the reminder's invoice ids |
| `status`, `response` | See §1.4; `response` = gateway response |
| `resolved_at` | Set by a cycle reset |
| `created_by` | Dashboard user who downloaded a notice; null for the scheduler |

Every operation also writes `Log::info('[SupplierNotice] …')` to the Laravel log.

### 1.6 Where it shows

**Supplier profile (`/suppliers/{id}`), field-verified suppliers only.** "Verified" = the latest field verification has `is_fake = 0` (`$supplier->latestVerification`, same as the header badge). The "Repayment Status" card shows:

- DPD 0: "Not applicable yet — no overdue repayment."
- Otherwise: DPD (days), oldest due date, DPD 90 date, stage, total overdue, the supplier-app block message, the overdue invoice table, a **Download Legal Notice (প্রথম / দ্বিতীয় / চূড়ান্ত নোটিশ)** button for the current level (from DPD 15; below 15 it shows the date notice 1 becomes available), and re-download buttons for earlier notices of the cycle.
- In both cases, the latest 20 log rows with a link to the full log.

For verified suppliers this card replaces the older "Repayment Overdue Escalation" card. Non-verified suppliers keep the older card and get no notices.

**`/suppliers` list.**

- "DPD" column right after "KYC Assessment Ceiling Limit". The value is a badge only on the notice days: 15 (yellow), 30 (orange), 60 (red), 90 or more (dark). Any other DPD is plain text, and 0 shows "—".
- "DPD" filter:

| Option | DPD range |
| --- | --- |
| DPD 15 | 1–15 |
| DPD 30 | 16–30 |
| DPD 60 | 31–60 |
| DPD 90 or above | 61+ |

- "Export Supplier Info" now passes the current filters (including `dpd`) and the XLSX has a "DPD" column after "KYC Assessment Ceiling Limit".
- Pagination links keep the filters (`withQueryString()`).
- "Notice & SMS Logs" button opens `/supplier-notice-logs`.

**`/supplier-notice-logs`.** Every log row, newest first, 50 per page, filterable by supplier id, channel, event and status. Resolved rows are greyed out.

The supplier app API is unchanged (`GET /api/V3/supplier/me` still returns the same `blockers` / `overdue_status` as described in `supplier_notice.md`).

## 2. Files

| File | Purpose |
| --- | --- |
| `app/Services/Supplier/SupplierLegalNoticeService.php` | `noticeLevel()`, `syncCycle()`, `cycleNotices()`, `cycleHasEvent()`, `issue()`, `renderPdf()`, `makeMpdf()`, `bn()` / `bnDate()` / `bnAmount()` |
| `app/Services/Supplier/SupplierOverdueEscalationService.php` | Existing DPD source; new `dpdBySupplier()` |
| `app/Models/SupplierNoticeLog.php` | Log model, channel/event/status constants |
| `database/migrations/2026_10_15_120000_create_supplier_notice_logs_table.php` | Log table |
| `app/Http/Controllers/SupplierLegalNoticeController.php` | PDF download |
| `app/Http/Controllers/SupplierNoticeLogController.php` | Logs page |
| `app/Http/Controllers/Api/V3/SupplierNoticeController.php`, `routes/api/supplier_v3.php` | Supplier-app APIs (§5) |
| `app/Http/Controllers/SupplierController.php` | `show()`: Repayment Status data; `index()` / `exportInfo()`: `repayment_dpd` |
| `app/Services/Supplier/SupplierService.php` | `DPD_FILTERS` and the `dpd` filter in `allPaginate()` |
| `app/Exports/SupplierInfoExport.php` | "DPD" column |
| `app/Console/Commands/SendSupplierRepaymentSms.php` | Daily SMS command and the SMS texts |
| `app/Console/Kernel.php` | Schedule (10:00 daily) |
| `resources/views/pdf/legal-notice.blade.php` | The three notice templates |
| `resources/views/content/suppliers/partials/repayment-status.blade.php` | Profile card |
| `resources/views/content/suppliers/details.blade.php` | Includes the card for verified suppliers |
| `resources/views/content/suppliers/index.blade.php` | DPD column, filter, export link, logs link |
| `resources/views/content/suppliers/notice-logs.blade.php` | Logs page |
| `resources/fonts/HindSiliguri-*.ttf`, `HindSiliguri-OFL.txt` | PDF Bangla font and licence |
| `config/finance.php` (`legal_notice`) | Letterhead and signatory |
| `app/Services/SMS/SMSService.php`, `config/sms.php` | `smsSend()` order Banglalink → GP → Robi (credentials hardcoded as before); history stores `sender_id = 0` (scheduler) as `NULL` |
| `database/seeders/SupplierNoticeDemoSeeder.php` | Local demo data |
| `composer.json` | `mpdf/mpdf ^8.3` |

## 3. Configuration

### 3.1 Letterhead (`config/finance.php` → `legal_notice`)

| Env key | Default |
| --- | --- |
| `LEGAL_NOTICE_COMPANY_NAME` | হিসাবী টেকনোলজিস লিমিটেড |
| `LEGAL_NOTICE_COMPANY_ADDRESS` | ইউনিভার্সিটি অব এশিয়া প্যাসিফিকের পাশে, লেভেল ৫, স্যুট নং–৫২৪, গ্রিন রোড, ঢাকা–১২১৫, বাংলাদেশ। |
| `LEGAL_NOTICE_SIGNATORY_NAME` | empty (dotted line) |
| `LEGAL_NOTICE_SIGNATORY_DESIGNATION` | empty (dotted line) |
| `LEGAL_NOTICE_SIGNATORY_CONTACT` | empty (dotted line) |
| `LEGAL_NOTICE_NO_PREFIX` | `HTL/LN` |

Run `php artisan config:clear` (or `config:cache`) after changing them. Already-issued notices keep their notice number, but the letterhead and signatory are read at render time.

### 3.2 SMS sending

`SMSService::smsSend()` (used by this feature and most other SMS in the app) delivers in this order, stopping at the first success:

| Order | Gateway | Counts as success | On failure |
| --- | --- | --- | --- |
| 1 | Banglalink (`banglalinkSMS()`, hardcoded OTP / notification accounts) | `statusInfo.statusCode = 1000` | Falls back to GP |
| 2 | GP (`gpSMS()` via `https://dashboard.hishabee.io/send/gp/sms`, no internal fallback here) | `statusInfo.errordescription = "Success"` | Falls back to Robi |
| 3 | Robi (`robiSMS()`, `config('sms.robi_link_sms_url')`) | `ServiceClass.ErrorCode = 0` | Returns `'error'` |

Every gateway that succeeds writes a `sms_send_histories` row (operator `blink` / `gp` / `robi`). No new env keys are needed. `ROBI_SMS_URL` falls back to the built-in Robi URL when it's blank in `.env`. `Setting sms_service = off` stops all non-OTP SMS, and the command logs those as `skipped`.

Code that calls a gateway directly keeps its own order: V3 OTP (`V3AuthController`, GP then Banglalink), `SendV3OtpGpFallbackJob` and `OrderCreatedConfirmationSmsService` (GP).

**Local Windows PHP and SSL.** If every gateway fails locally with `cURL error 60: SSL certificate problem`, PHP has no CA bundle (`curl.cainfo` is empty in `php.ini`). Download `https://curl.se/ca/cacert.pem` and either set `curl.cainfo` / `openssl.cafile` in `php.ini`, or pass them for one command: `php -d curl.cainfo=C:\path\cacert.pem -d openssl.cafile=C:\path\cacert.pem artisan …`. Servers with a system CA bundle don't need this.

## 4. Routes and commands to check

| What | Route / command |
| --- | --- |
| List with DPD column | `GET /suppliers` |
| DPD filter | `GET /suppliers?dpd=15` (also `30`, `60`, `90`) |
| Export with the filter | `GET /suppliers-export?dpd=60` (the "Export Supplier Info" button adds the current filters) |
| Profile Repayment Status | `GET /suppliers/{id}` |
| Legal notice PDF | `GET /suppliers/{id}/legal-notice/{level}` (`level` 1, 2 or 3; route name `suppliers.legal-notice.download`) |
| Log | `GET /supplier-notice-logs?supplier_id=&channel=&event=&status=` |
| SMS (no gateway) | `php artisan suppliers:send-repayment-sms --dry-run` |
| SMS for one supplier | `php artisan suppliers:send-repayment-sms --supplier=ID [--dry-run]` |
| Schedule | `php artisan schedule:list` (10:00 daily) |
| Supplier app | `GET /api/V3/supplier/me`, `GET /api/V3/supplier/repayment-status`, `GET /api/V3/supplier/legal-notices/{level}`, `GET /api/V3/supplier/repayment-notifications` (§5) |

All web routes are in the `auth` (web) group, like `/suppliers`.

## 5. APIs for the frontend

### 5.1 Who calls what

| Frontend | Auth | Endpoints |
| --- | --- | --- |
| Supplier (distributor) app | `Authorization: Bearer <supplier token>` (`auth:supplier_api`), `Accept: application/json` | `/api/V3/supplier/me`, `/api/V3/supplier/repayment-status`, `/api/V3/supplier/legal-notices/{level}`, `/api/V3/supplier/repayment-notifications` |
| Dashboard (HQ) | Logged-in web session | The Blade pages in §4. They render HTML, plus a PDF (notice download) and an XLSX (export); there is no JSON dashboard API. |

All supplier-app JSON uses the app's standard envelope (`App\Traits\ApiResponse`):

```json
{ "success": true, "status_code": 200, "message": "...", "data": { }, "metadata": null }
```

Errors: `{ "success": false, "status_code": 4xx, "message": "...", "errors": { }, "metadata": null }`. A missing or expired token gives the usual `401 {"message": "Unauthenticated."}`.

The supplier app is **read-only** here: it shows status, notices HQ has already issued, and the SMS the supplier received. Notices are only issued from the dashboard, and SMS only by the scheduled command.

Recommended app flow:

1. On login / resume, call `GET /supplier/me`. If `data.blockers` is not empty, show the block pop-up (`title`, `message`; `title_en` / `message_en` for English).
2. On the Repayment / Notices screen, call `GET /supplier/repayment-status`. Show the DPD, total overdue, DPD 90 date and invoices, plus one card per entry in `notices` with a download button that opens `download_url` (same Bearer token).
3. In a Notifications / History list, call `GET /supplier/repayment-notifications?page=N`.

### 5.2 `GET /api/V3/supplier/me` (existing, block pop-up)

Only the keys relevant here are shown; the rest of the profile payload is unchanged.

```json
{
  "success": true,
  "status_code": 200,
  "message": "...",
  "data": {
    "block_reason": null,
    "blockers": [
      {
        "type": "supplier_blocked",
        "stage": "legal_action",
        "title": "অ্যাকাউন্ট ব্লক করা হয়েছে",
        "message": "ব্যাংক আপনার অ্যাকাউন্টটি ব্লক করেছে। আপনার 26,312.00 টাকার বকেয়া 13-10-2026 তারিখে ৯০ দিন অতিক্রম করেছে এবং এখন 95 দিন অতিক্রান্ত হয়েছে। ৯০ দিন অতিক্রম করায় ব্যাংক আপনার বিরুদ্ধে চেক ডিজঅনার মামলা দায়ের করবে। অনুগ্রহ করে অবিলম্বে আপনার বকেয়া পরিশোধ করুন।",
        "title_en": "Account Blocked",
        "message_en": "The bank has blocked your account. Your outstanding amount of 26,312.00 Taka crossed 90 days on 13-10-2026 and is now 95 days overdue. ...",
        "dpd": 95,
        "overdue_amount": 26312,
        "due_date": "2026-07-15",
        "dpd90_date": "2026-10-13"
      }
    ],
    "overdue_status": {
      "is_blocked": true,
      "block_source": "repayment_overdue",
      "is_manually_blocked": false,
      "dpd": 95,
      "stage": "legal_action",
      "stage_label": "Eligible for Cheque Case",
      "overdue_amount": 26312,
      "overdue_invoice_count": 1,
      "due_date": "2026-07-15",
      "dpd90_date": "2026-10-13",
      "cheque_case_eligible": true
    }
  }
}
```

`blockers` is `[]` when the supplier isn't blocked. `stage` is `block` (DPD 1–14), `notice_1`, `notice_2`, `notice_3`, `legal_action` (DPD 90+) or `manual`. Full stage and message rules: `docs/supplier_notice.md`.

### 5.3 `GET /api/V3/supplier/repayment-status`

The supplier's repayment status, current notice level and the notices issued in the open cycle. It also runs the cycle reset (case b) when nothing is overdue.

```json
{
  "success": true,
  "status_code": 200,
  "message": "Repayment status.",
  "data": {
    "is_field_verified": true,
    "dpd": 95,
    "stage": "legal_action",
    "stage_label": "Eligible for Cheque Case",
    "is_blocked": true,
    "overdue_amount": 26312,
    "due_date": "2026-07-15",
    "dpd90_date": "2026-10-13",
    "cheque_case_eligible": true,
    "notice_level": 3,
    "notice_label": "চূড়ান্ত নোটিশ",
    "next_notice": null,
    "overdue_invoices": [
      { "invoice_id": 199, "invoice_number": 88001060, "invoice_date": "2026-06-15", "due_date": "2026-07-15", "days_late": 95, "remaining_amount": 26312 }
    ],
    "notices": [
      { "level": 1, "label": "প্রথম নোটিশ", "notice_no": "HTL/LN1/2026/00056", "notice_date": "2026-07-30", "dpd": 15, "amount": 26312, "download_url": "https://<host>/api/V3/supplier/legal-notices/1" },
      { "level": 2, "label": "দ্বিতীয় নোটিশ", "notice_no": "HTL/LN2/2026/00057", "notice_date": "2026-08-14", "dpd": 30, "amount": 26312, "download_url": "https://<host>/api/V3/supplier/legal-notices/2" },
      { "level": 3, "label": "চূড়ান্ত নোটিশ", "notice_no": "HTL/LN3/2026/00058", "notice_date": "2026-10-18", "dpd": 95, "amount": 26312, "download_url": "https://<host>/api/V3/supplier/legal-notices/3" }
    ]
  },
  "metadata": null
}
```

| Field | Meaning |
| --- | --- |
| `is_field_verified` | Latest field verification `is_fake = 0`. Legal notices are only issued for verified suppliers. |
| `dpd`, `stage`, `stage_label`, `is_blocked`, `overdue_amount`, `due_date`, `dpd90_date`, `cheque_case_eligible` | Same values as `/supplier/me` → `overdue_status` (§1.1). |
| `notice_level`, `notice_label` | 1 / 2 / 3 from the DPD (§1.2); `null` below DPD 15. |
| `next_notice` | `{level, dpd, date}`: the next level and the date it is reached. `null` at level 3 or when nothing is overdue. |
| `overdue_invoices` | Every overdue invoice, oldest first (the table printed on the notice). |
| `notices` | Notices HQ has issued in the open cycle (empty until HQ downloads one). Each has its own `notice_no`, `notice_date` and `download_url`. |

Nothing overdue: `dpd: 0`, every date / level `null`, `overdue_invoices: []`, `notices: []`. Show "Not applicable yet".

### 5.4 `GET /api/V3/supplier/legal-notices/{level}`

Downloads the PDF of a notice already issued in the open cycle. `level` is 1, 2 or 3. Send the same Bearer token as the other calls.

| Case | Response |
| --- | --- |
| Issued | `200`, `Content-Type: application/pdf`, `Content-Disposition: attachment; filename="Legal_Notice_3_HTL-LN3-2026-00058.pdf"`. The same document HQ downloads. |
| Not issued (yet, or the cycle was reset) | `404` `{"success": false, "status_code": 404, "message": "Notice not issued.", "errors": {"level": "এই নোটিশটি এখনো ইস্যু করা হয়নি।"}, "metadata": null}` |

Each download is logged as `notice_downloaded` (with `created_by = null`, i.e. not a dashboard user).

### 5.5 `GET /api/V3/supplier/repayment-notifications?page=1&per_page=20`

The supplier's repayment SMS that were actually sent (`status = sent`), plus the notices issued, newest first. `per_page` defaults to 20 (max 50). Dry-run, failed and skipped SMS and the internal download / reset rows are not returned.

```json
{
  "success": true,
  "status_code": 200,
  "message": "Repayment notifications.",
  "data": [
    {
      "id": 59,
      "type": "sms",
      "event": "notice_3_sms",
      "level": 3,
      "notice_no": null,
      "message": "চূড়ান্ত সতর্কবার্তা: আপনার ৳২৬,৩১২ টাকা গত ৯৫ দিন ধরে বকেয়া রয়েছে। ১৩/১০/২০২৬-এর মধ্যে সম্পূর্ণ অর্থ পরিশোধ করুন। ...",
      "dpd": 95,
      "amount": 26312,
      "reference_date": "2026-07-15",
      "is_resolved": false,
      "created_at": "2026-10-18 11:16:00"
    },
    {
      "id": 58,
      "type": "legal_notice",
      "event": "notice_generated",
      "level": 3,
      "notice_no": "HTL/LN3/2026/00058",
      "message": "চূড়ান্ত নোটিশ — HTL/LN3/2026/00058",
      "dpd": 95,
      "amount": 26312,
      "reference_date": "2026-10-18",
      "is_resolved": false,
      "created_at": "2026-10-18 11:16:00"
    }
  ],
  "metadata": { "current_page": 1, "last_page": 1, "per_page": 20, "total": 4 }
}
```

| Field | Meaning |
| --- | --- |
| `type` | `sms` or `legal_notice` |
| `event` | `reminder_3d`, `reminder_1d`, `due_today`, `notice_1_sms`, `notice_2_sms`, `notice_3_sms` or `notice_generated` |
| `message` | The exact SMS text the supplier received, or the notice label and number |
| `reference_date` | Due date (reminders), oldest due date (notice SMS) or notice date |
| `is_resolved` | `true` once the overdue was cleared and the cycle reset; show as history |

## 6. Local testing

### 6.1 Setup

```bash
composer install            # mpdf/mpdf; on Windows add --ignore-platform-req=ext-pcntl --ignore-platform-req=ext-posix
php artisan migrate         # creates supplier_notice_logs
php artisan db:seed --class=SupplierNoticeDemoSeeder
```

The seeder needs at least one zone, supplier category, brand and company. It prints the ids it created:

| Phone | Supplier | Verified | Repayments (relative to today) | Expected |
| --- | --- | --- | --- | --- |
| 01700000991 | `[DEMO] Verified - No Repayment` | yes | none | Profile: "Not applicable yet" |
| 01700000992 | `[DEMO] Verified - Pre-due Reminders` | yes | due in 3 days, tomorrow, today | Profile: "Not applicable yet"; SMS `reminder_3d`, `reminder_1d`, `due_today` |
| 01700000993 | `[DEMO] Verified - DPD 15` | yes | DPD 15 | Notice 1, SMS `notice_1_sms`; list badge 15 (yellow) |
| 01700000994 | `[DEMO] Verified - DPD 30 + 10` | yes | DPD 30, DPD 10 | Notice 2 listing both invoices (case a); badge 30 (orange) |
| 01700000995 | `[DEMO] Verified - DPD 60 + 20 + 5` | yes | DPD 60, 20, 5 | Notice 3 listing all three; badge 60 (red) |
| 01700000996 | `[DEMO] Verified - DPD 95` | yes | DPD 95 | Notice 3, cheque-case line on the profile; badge 95 (dark) |
| 01700000997 | `[DEMO] Not Verified - DPD 20` | no | DPD 20 | In the list, filter and export (plain 20); old escalation card on the profile, no notices (download → 403) |

All demo repayments use a 30-day tenure and a `[DEMO] Notice Retailer` (01700000990). Re-running the seeder moves the dates so the DPDs are exact again, reopens the repayments and **deletes the demo suppliers' notice logs**, so every run starts from a clean cycle.

### 6.2 SMS

```bash
php artisan suppliers:send-repayment-sms --dry-run
```

Expected output (supplier ids vary):

```
Supplier #6 reminder_3d: dry_run
Supplier #6 reminder_1d: dry_run
Supplier #6 due_today: dry_run
Supplier #7 notice_1_sms: dry_run
Supplier #8 notice_2_sms: dry_run
Supplier #9 notice_3_sms: dry_run
Supplier #10 notice_3_sms: dry_run
Supplier #11 notice_1_sms: dry_run
[dry-run] Processed 6 supplier(s) with open repayments: 8 SMS logged, 0 cycle reset(s).
```

Run it again: `0 SMS logged` (deduplicated). Open `/supplier-notice-logs?channel=sms` to read the exact Bangla texts. Only run without `--dry-run` locally if you really want SMS sent to the demo numbers.

### 6.3 Profile and PDFs

1. Log in to the dashboard, open `/suppliers?dpd=15` and open `[DEMO] Verified - DPD 15`.
2. The "Repayment Status" card shows DPD 15, the due date, the DPD 90 date and one invoice. Click **Download Legal Notice (প্রথম নোটিশ)**. The PDF has the letterhead, a notice number like `HTL/LN1/2026/000NN`, today's date, the recipient block, the invoice table, the total and the DPD 90 date.
3. Download again: same notice number. The log table on the profile shows one `notice_generated` row and one `notice_downloaded` row per download.
4. DPD 30 + 10 supplier → **notice 2**: both invoices, "৩০ দিন", and "ইতোপূর্বে আমাদের নোটিশ নং …, তারিখ …" filled with notice 1, which is issued automatically and dated oldest due date + 15 days (or the day its notice-1 SMS was sent). The 90-day date = oldest due date + 90.
5. DPD 60 + 20 + 5 supplier → **notice 3**: three invoices, "৬০ দিন", references to notices 1 and 2 (dated due + 15 and due + 30), and the "pay by (৯০ দিন পূর্ণ হওয়ার তারিখ)" paragraph.
6. DPD 95 supplier → **notice 3**: "৯৫ দিন" and the after-day-90 paragraph ("… তারিখে ৯০ দিন অতিক্রম করেছে এবং এখন ৯৫ দিন অতিক্রান্ত হয়েছে …"). The profile offers re-download buttons for notices 1 and 2.
7. DPD below 15: move the DPD 15 supplier back 5 days (tinker) and check there's no download button, `/suppliers/{id}/legal-notice/1` → 422, and `suppliers:send-repayment-sms --dry-run --supplier=ID` logs no notice SMS:

   ```php
   $ids = DB::table('retailer_orders')->where('supplier_id', ID)->pluck('invoice_id');
   DB::table('retailer_invoices')->whereIn('id', $ids)->update(['delivery_date' => DB::raw('DATE_ADD(delivery_date, INTERVAL 5 DAY)')]); // now DPD 10
   ```

8. Error cases: `/suppliers/{DPD 15 id}/legal-notice/2` → 422; `/suppliers/{not-verified id}/legal-notice/1` → 403.

### 6.4 List, filter and export

| Filter | Demo suppliers shown |
| --- | --- |
| `dpd=15` | DPD 15 |
| `dpd=30` | DPD 30 + 10, Not Verified DPD 20 |
| `dpd=60` | DPD 60 + 20 + 5 |
| `dpd=90` | DPD 95 |

With a filter selected, click **Export Supplier Info**: the URL carries `?dpd=…` and the XLSX ends with the "DPD" column.

### 6.5 Cycle reset (case b)

```php
// tinker: mark every repayment of the DPD 60 + 20 + 5 supplier as repaid
$ids = DB::table('retailer_orders')->where('supplier_id', ID)->pluck('invoice_id');
DB::table('retailer_invoices')->whereIn('id', $ids)->update(['repayment_status' => 'closed']);
```

Open the profile (or run the SMS command): the card shows "Not applicable yet", the log has a `cycle_reset` row and all earlier rows are marked "(resolved)". Set them back to `running` (or re-run the seeder): the supplier starts a new cycle, and the next download issues a new notice 3 number.

### 6.6 Removing the demo data

```php
// php artisan tinker
$supplierIds = DB::table('suppliers')->where('phone', 'like', '0170000099%')->pluck('id');
$invoiceIds = DB::table('retailer_invoices')->whereBetween('invoice_number', [88001000, 88001099])->pluck('id');
DB::table('supplier_notice_logs')->whereIn('supplier_id', $supplierIds)->delete();
DB::table('retailer_orders')->whereIn('invoice_id', $invoiceIds)->delete();
DB::table('retailer_invoices')->whereIn('id', $invoiceIds)->delete();
DB::table('supplier_verifications')->whereIn('supplier_id', $supplierIds)->delete();
DB::table('suppliers')->whereIn('id', $supplierIds)->delete();
DB::table('retailers')->where('mobile_number', '01700000990')->delete();
```
