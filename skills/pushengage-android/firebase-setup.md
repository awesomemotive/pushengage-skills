# Android Firebase Setup — Firebase Console

This step happens entirely outside the customer's repo. You cannot click for them. Your job: walk the customer through, wait at the natural pause points, collect the artifacts.

## What we're producing

- A `google-services.json` file placed at **`app/google-services.json`** in the customer's repo.
- A **Service Account JSON** (private key) file the customer keeps locally — uploaded directly to the PushEngage dashboard in the next step.
- The **Sender ID** (numeric) from the Cloud Messaging tab.
- The customer's `applicationId` (package name) — must match the Firebase Android app's package name.

These artifacts are used in the next step (`dashboard-setup.md`) and in `install.md` / `init-code.md`.

> ⚠️ **Do NOT ask the customer to paste the Service Account JSON contents in chat.** It is a private key. They upload it directly on the PushEngage dashboard. Same posture as iOS `.p12` passwords.

## Step 1 — Create or select a Firebase project

Walk the customer through (do not edit anything yourself; this is all Firebase Console-side):

1. Open <https://console.firebase.google.com>.
2. **Add Project** (or pick an existing one) → enter a project name → Continue → (Analytics on/off — customer's choice; not required for FCM) → Create.

Ask: "Confirm the Firebase project is open in the console."

## Step 2 — Register the Android app

1. From the project overview, click the **Android icon** to add an Android app.
2. **Android package name** — must match the customer's `applicationId` in `app/build.gradle` (e.g. `com.example.MyApp`).
   - Ask the customer: "What's your `applicationId` (package name)?" Capture this — you'll need it later.
3. **App nickname** — anything; for the customer's reference only.
4. **Debug signing certificate SHA-1** — leave blank. Not required for FCM-only push.
5. Click **Register app**.

Ask: "Confirm the Android app is registered in Firebase."

## Step 3 — Download `google-services.json` and place at `app/`

1. Firebase Console offers `google-services.json` for download. Click **Download `google-services.json`**.
2. Place it at **`app/google-services.json`** — the **app-module root**, NOT the project root.

> ⚠️ **Common mistake:** customers often drop this at the project root (next to top-level `build.gradle`). The Google Services plugin will fail with `File google-services.json is missing. The Google Services Plugin cannot function without it.` That is KI-A006 / ANDROID-AUDIT-04 / ANDROID-BP-07.

Once placed, you can verify by reading the file or by running `ls app/google-services.json`. Capture for later: confirm the file exists at `app/google-services.json`.

Ask: "Confirm `app/google-services.json` exists in your repo (not at the project root)."

## Step 4 — Generate the Service Account JSON

The PushEngage dashboard uses this Service Account JSON to send pushes via Firebase's HTTP v1 API on the customer's behalf.

1. In Firebase Console, click the gear icon → **Project settings**.
2. Open the **Service accounts** tab.
3. Click **Generate new private key** → Confirm → **Generate key**.
4. A `.json` file downloads. **Save it somewhere secure.** Treat as a secret — same posture as a password or API key.

> ⚠️ **Secret handling:** the Service Account JSON grants admin access to the Firebase project's FCM endpoint. Do NOT paste its contents into chat. Do NOT commit it to source control. The customer uploads it directly on the PushEngage dashboard in the next step.

Ask: "Confirm you have the Service Account JSON downloaded and saved somewhere secure."

## Step 5 — Capture the Sender ID

1. Still in Firebase Console → **Project settings**.
2. Open the **Cloud Messaging** tab.
3. Copy the **Sender ID** (a numeric value).

Ask the customer to paste the Sender ID into chat — it is NOT a secret; it's a public project identifier the Firebase SDK transmits with every push registration anyway. You'll pass it through to the next step.

## Step 6 — Capture the artifacts

Before moving on, confirm with the customer:

1. **`app/google-services.json`** is in place (not at project root).
2. **Service Account JSON** is downloaded and saved locally (path is the customer's; you don't need it).
3. **Sender ID** — pasted in chat (capture).
4. **`applicationId` / package name** — pasted in chat (capture; reused in `init-code.md` for verifying the manifest registration).

Do NOT ask for the Service Account JSON contents in chat. The customer uploads directly on the dashboard.

---

Next: hand off to `dashboard-setup.md`.
