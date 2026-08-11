# Android Dashboard Setup — PushEngage

Out-of-code; walk the customer through.

**Pre-req from `firebase-setup.md`:** customer has the Service Account JSON file saved locally, the Sender ID copied, and their `applicationId` (package name).

## Step 1 — Open the Android SDK tab

1. Customer logs in to <https://www.pushengage.com/>.
2. Navigate: **Site Settings → Installation → Android SDK**.

If the customer sees an older/different breadcrumb in their dashboard, the destination is the same.

## Step 2 — Fill the Android SDK form

Fields on the Android SDK page:

| Field | Value |
|---|---|
| **Sender ID** | From Firebase Console → Project settings → Cloud Messaging tab. Customer pastes the numeric Sender ID captured in `firebase-setup.md` Step 5. |
| **Service Account JSON** | Upload field. Customer **uploads the `.json` file** generated in `firebase-setup.md` Step 4. |
| **Save / Update** (button) | Save the configuration. |

Walk the customer through. **Do NOT ask the customer to paste the Service Account JSON contents into chat.** They upload it directly on the dashboard — same posture as iOS `.p12` files.

## Step 3 — Copy the App ID UUID

After saving, the page displays:

- A read-only **App ID** field showing a UUID (shaped like `00000000-1111-2222-3333-444444444444`).
- A **Copy** button next to it.

Ask the customer to **click Copy and paste the UUID into chat**. The App ID UUID is **not** a secret — it's a public identifier the SDK transmits in every API call. You'll inject this into the code in `init-code.md`'s `.setAppId(...)` call.

> ⚠️ **Disambiguation:** "App ID" in the dashboard = a UUID. It is NOT the same as your Android `applicationId` (package name like `com.example.MyApp`). If the customer says "I have the App ID," confirm it's a UUID, not a package name.

## Step 4 — Confirm dashboard is configured

Confirm with the customer:

1. Sender ID is pasted into the form.
2. Service Account JSON is uploaded.
3. The customer clicked **Save / Update**.
4. They have copied the **App ID UUID** (pasted into chat).

If any of these are missing, do not proceed.

---

Next: install the SDK (`install.md`), then write init code (`init-code.md`).
