# Google Form and Google Sheets Design

## Development Order

An important decision in the development process was writing the final system prompt before building the Google Form, not the other way around. The reasoning is that the final system prompt determines the data schema needed upstream, meaning which fields are required in the Form and Sheets. Changing a prompt (plain text) is far cheaper than rebuilding the Form, Sheets, and n8n field mapping if that requirement is only discovered later.

## Google Form

Title: Cek Potensi Hemat Cicilan KPR - PropertyHub (Check Your Potential Mortgage Savings - PropertyHub)

Four fields are used:

1. Full name, Short answer type, required
2. Active phone number, Short answer type, required, with regex validation for the Indonesian phone number pattern using Matches mode, so the entire input is validated rather than just checking for a substring match
3. Active mortgage status, Multiple choice type with two options, "Ya, masih ada" (Yes, still ongoing) and "Tidak / sudah lunas / belum pernah punya KPR" (No / already paid off / never had a mortgage), using the "Go to section based on answer" feature so that a "Yes" answer proceeds to the consent section, while a "No" answer goes straight to form submission without needing to fill in consent
4. Consent, Checkbox type, required, with text that explicitly mentions automated AI-based calls, to stay consistent with the agent's honest answer when asked whether it is a robot

## Google Sheets

The sheet is named "PropertyHub - Refinancing Call Queue", automatically connected to Form responses (columns A through E), plus four manual columns added afterward (columns F through I).

Full column order:

| Column | Name | Source |
|---|---|---|
| A | Timestamp | Automatic from Form |
| B | Nama Lengkap (Full Name) | Automatic from Form |
| C | Nomor Telepon Aktif (yang bisa dihubungi) (Active Phone Number) | Automatic from Form |
| D | Apakah kamu saat ini masih punya cicilan KPR (Active mortgage status) | Automatic from Form |
| E | Saya menyetujui untuk dihubungi oleh PropertyHub... (Consent) | Automatic from Form |
| F | Status | Manual, auto-filled by n8n |
| G | Disconnection Reason | Manual, auto-filled by n8n |
| H | Summary | Manual, auto-filled by n8n |
| I | phone_e164 | Manual, formula |

### Status Column

Contains a formula that automatically fills in the value `not-called` for every new row that comes in. This formula is pasted down across many rows so it automatically applies to future submissions.

Once a call is completed and analyzed, this column is automatically overwritten by the Update Sheets node in n8n Workflow 2, with one of three possible values:
- `called`, if the call connected successfully and qualifying data was obtained
- `not-called`, if the call failed to connect at all, or connected but no qualifying conversation took place (for example, it went to voicemail, or the customer never responded)
- `error`, if there was a failure at the system level (not related to the customer not responding)

These three values reflect the current state, based on the classification guardrail design detailed further in n8n-workflow-architecture.md. More categories may be added in the future as needed.

### Disconnection Reason Column

A column added later, after identifying the need to record the technical reason for a failed call separately from Summary, so there is a clear audit trail without mixing technical notes with business outcomes. Automatically filled by n8n Workflow 2 with the raw `disconnection_reason` value from the Retell webhook payload, only for failed rows (Status `not-called` or `error`). Left empty for successful rows (Status `called`).

### Summary Column

Left empty when a new row comes in, and only filled automatically by n8n Workflow 2 if the call was successful (Status `called`), containing the qualifying result summary from the `ringkasan_singkat` field in Retell's Post-Call Analysis. Left empty for failed rows.

### phone_e164 Column

Contains a regex formula that converts three possible Indonesian phone number prefix formats (plus-six-two, six-two, and zero) into a single consistent E.164 format, since the Retell API requires this format for the `to_number` parameter on the create-phone-call endpoint.

## Important Note on the n8n Filter

The Filter in n8n Workflow 1 must check two conditions simultaneously, namely Status equal to `not-called` and mortgage status equal to "Ya, masih ada" (Yes, still ongoing), not just one condition alone. This is because rows with a "No" answer still end up in the Sheet, since the Form only skips the consent question, not the entire submission. Without this double filter, irrelevant rows would risk being called by the system anyway.

Since the Status column now has three possible values (not two), this filter specifically looks for rows where Status equals `not-called` only, rather than simply "not called", so that rows with an `error` status (which need manual investigation, not automatic retry) don't get called again automatically. Full details on this are in n8n-workflow-architecture.md.
