# Monetization & Licensing Guide — File Organizer

Goal
- Enable a privacy-first, offline-friendly purchase flow with a simple one-time purchase license (Pro). Provide optional maintenance/upgrade offers without forcing online activation.

Recommended Model
- One-time purchase for Pro edition (recommended for a solo developer and offline users).
- Optional paid maintenance plan (e.g., 1 year of updates) or paid major upgrade for future versions.
- Offer a free trial (time-limited, e.g., 14–30 days) so users can evaluate before purchasing.

Pricing guidance (example)
- Single Pro product: $29.99–$49.99 one-time.
- Consider a lower “home” tier if you add many features later.

License design (offline-friendly)
- Signed license key approach:
  - You generate license keys offline or on your server. Keys are signed using an HSM or private key (recommended) and verified in-app using a public key.
  - Key payload contains: product id, license type, expiration (optional for trial or maintenance), issuance date, and a nonce/ID.
  - App verifies signature locally — no network required.
- Offline activation file:
  - Provide an option for users to paste a license key or import a signed license file.
  - For convenience, offer optional online activation (phone-home) — but keep it optional and clearly documented.
- Trial flow:
  - Begin trial on first-run; store trial start hashed/obfuscated in local app data.
  - Allow graceful expiry and an upgrade prompt.

License enforcement & UX
- Keep enforcement light: the app should function offline and not be crippled for small verification errors.
- Provide a clear “Enter license” UI and a way to view license details (issued to, date, expiry).
- Provide an obvious path for refunds in your purchase flow (e.g., 14-day money-back).

Distribution & payments
- Payment processors that handle desktop and VAT complexities:
  - Paddle (recommended for desktop apps: license delivery, VAT handling, chargeback handling).
  - Gumroad (simpler, fewer enterprise features).
  - Stripe + custom license server (more work).
- Delivery:
  - Serve signed license emails automatically via your payment provider.
  - Offer manual license file/email option for privacy-focused customers.

Code signing & packaging
- Obtain code signing certificate (EV if available to reduce SmartScreen friction).
- Build MSIX packages (recommended) and sign them. MSIX allows clean install/uninstall and store later.
- For users who refuse online check, provide an offline installer plus manual license activation.

Refunds, EULA, and privacy
- EULA: Include usage rights, disclaimers, support policy.
- Privacy statement: Explicitly state “No telemetry by default; all processing is local.” If you add optional crash reporting, make it opt-in.
- Refund policy: For one-time purchases, a 14-day refund window is common.

Implementation checklist (practical)
- [ ] Decide final price and trial length.
- [ ] Choose payment provider (Paddle/Gumroad/Stripe).
- [ ] Design license key format (signed payload + public key verification).
- [ ] Implement license UI (enter/paste key, import file, view license).
- [ ] Implement optional online activation (opt-in).
- [ ] Implement trial logic and local trial state storage.
- [ ] Draft EULA, privacy statement, and refund policy pages for your site.
- [ ] Acquire code signing cert and test signed builds.
- [ ] Implement delivery flow: after purchase, user receives license key/email with instructions.
- [ ] Test offline activation & edge cases (reinstalls, system clock tampering).
- [ ] Prepare support flow for refunds & license re-issuance.

Security best practices
- Do not store private key in the app. Use private key only to sign licenses on your side.
- Store only public key in the app to verify signatures.
- Make license verification tolerant; do not brick users for minor issues — support flow for re-issuance.

Support & anti-fraud
- Keep a lightweight back-office list of issued license IDs so you can revoke or re-issue if needed.
- For low-volume sales, manual checks are feasible; for higher volume, Paddle/Gumroad reduce overhead.

Marketing & Launch tips
- Create a clear landing page describing privacy, offline use, and safety features (dry-run, undo).
- Offer screenshots/video of rule wizard and preview.
- Provide a short, clear installation & activation FAQ to reduce support requests.

Optional: Microsoft Store
- Listing on Microsoft Store increases discoverability but changes distribution model and may require Store-specific packaging and policies. Consider this as a secondary channel after an initial direct sales launch.

Support templates (examples to prepare)
- Purchase email template with license key and instructions.
- Troubleshooting guide for activation problems (system clock; firewall blocking activation; reinstall process).
- Refund/returns policy page.

Notes about pricing & upgrades
- If you later add cloud features or continuous-value services, you can introduce subscriptions for those features while continuing one-time purchases for the core offline app.

---

If you want, I can:
- Expand any roadmap phase into a detailed sprint backlog (tasks, sub-tasks, estimates).
- Produce wireframes for the Rule Wizard and Preview pages next.
- Draft Product Spec and a minimal Test Plan for Phase 0.

Which do you want me to do next?