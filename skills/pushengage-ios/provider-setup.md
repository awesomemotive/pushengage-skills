# iOS Provider Setup — Apple Developer portal + Keychain

This step happens entirely outside the customer's repo. You cannot click for them. Your job: walk the customer through, wait at the natural pause points, collect the artifacts.

## What we're producing

- A `.p12` certificate file (cert + private key).
- The password the customer set when exporting the `.p12`.
- The customer's iOS Bundle Identifier (e.g. `com.example.MyApp`).

These three artifacts are uploaded in the next step (`dashboard-setup.md`).

> ⚠️ The PushEngage dashboard accepts `.p12` only — not `.p8`.

## Step 1 — Enable Push Notifications capability on the App ID

Walk the customer through (do not edit anything yourself; this is all Apple-side):

1. Open Xcode → project → main app target → **Signing & Capabilities**.
2. Click **+ Capability** → choose **Push Notifications**.
3. If the capability isn't in the list:
   - Apple Developer → **Certificates, Identifiers & Profiles** → **Identifiers** → select the App ID → **Edit** → check **Push Notifications** → **Save**.
   - Return to Xcode and add the capability.

Ask: "Confirm the capability is now showing in Signing & Capabilities."

## Step 2 — Generate the CSR from Keychain

1. Customer opens **Keychain Access** on their Mac.
2. Menu → **Keychain Access → Certificate Assistant → Request a Certificate from a Certificate Authority…**
3. **Email Address:** leave blank.
4. **Common Name:** whatever they want (their name is fine).
5. Select **"Request is saved to disk"**.
6. Click **Continue**, save to disk.

Ask: "Have you saved the `.certSigningRequest` file?"

## Step 3 — Create the APNs SSL Certificate in Apple Developer portal

1. Apple Developer → **Certificates, Identifiers & Profiles → Certificates → + (add)**.
2. Choose **Apple Push Notification service SSL (Sandbox & Production)**. (Single cert covers both environments.)
3. Continue → select the App ID from Step 1.
4. Continue → upload the `.certSigningRequest` from Step 2.
5. Continue → download the `.cer` file.

Ask: "Confirm you've downloaded the `.cer`."

## Step 4 — Import .cer, export .p12

1. Customer double-clicks the `.cer` → Keychain Access imports it (login keychain).
2. Open Keychain Access → **My Certificates** category → locate **"Apple Push Services: <bundle-id>"**.
3. Right-click → **Export "Apple Push Services: …"**.
4. Format: **`.p12` (Personal Information Exchange)**.
5. Set a **password** when prompted. **Customer must save this password** — they need it in the dashboard.

Ask: "Confirm you've exported the `.p12` and saved the password somewhere secure."

## Step 5 — Capture the artifacts

Before moving on, ask the customer to confirm:

1. **Path to the `.p12` file** (we don't need it; just confirm they have it).
2. **Bundle Identifier** of their main app target — paste back as a string (e.g. `com.example.MyApp`). You'll need this for `dashboard-setup.md` (it's the "App Push Id" field).

Do NOT ask for the `.p12` password in chat. Tell the customer they'll enter it directly in the dashboard.

---

Next: hand off to `dashboard-setup.md`.
