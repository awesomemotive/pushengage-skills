# iOS Dashboard Setup — PushEngage

Out-of-code; walk the customer through.

**Pre-req from `provider-setup.md`:** customer has the `.p12` file, the password they set when exporting it, and their iOS Bundle Identifier.

## Step 1 — Open the iOS SDK tab

1. Customer logs in to <https://www.pushengage.com/>.
2. Navigate: **Site Settings → Installation → iOS SDK**.

The page has two clearly numbered sections.

## Step 2 — Section 1: "Configure your Apple iOS APNs settings"

Fields (in order on the page):

| Field | Value |
|---|---|
| **App Push Id** | The customer's iOS Bundle Identifier (e.g. `com.example.MyApp`). |
| **Certificate (.p12)** | Upload the `.p12` from `provider-setup.md`. |
| **Private Key Password** | The password set during `.p12` export. |
| **Update** (button) | Save. |

Walk the customer through. Don't ask them to paste the password into chat — they enter it directly on the dashboard.

> ⚠️ **Disambiguation:** "App Push Id" on this page is the iOS Bundle Identifier (customer enters). It is NOT the same as the "App ID" UUID below. If the customer says "I pasted the App ID," ask which one.

## Step 3 — Section 2: "Install the SDK"

This section displays:

- A read-only **App ID** field showing a UUID (shaped like `00000000-1111-2222-3333-444444444444`).
- A **Copy** button next to it.

Ask the customer to **click Copy and paste the UUID into chat**. You'll inject this into the code in the next step (`init-code.md`'s `setAppID(id:)` call).

## Step 4 — Confirm dashboard is configured

Confirm with the customer:

1. Section 1 is filled and they clicked Update.
2. They have copied the App ID UUID.

If either is missing, do not proceed.

---

Next: install the SDK (`install.md`), then write init code (`init-code.md`).
