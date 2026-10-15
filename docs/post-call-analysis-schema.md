# Post-Call Analysis Schema

Five custom fields configured on the Retell AI agent's Post-Call Analysis, used to extract structured data from each call's transcript once the call ends. The extraction model used is GPT-5.4 Nano, chosen because structured extraction from a finished transcript doesn't require low latency or deep reasoning, so the lightest available model is sufficient for this task.

Field names and enum values below are kept in Indonesian to match the exact configuration in Retell, since the agent operates in Bahasa Indonesia and these strings are what the system actually reads and writes.

Important note on these five fields: all of them are processed based on a transcript of a conversation that actually took place. For calls that failed to connect at all, every custom field will be empty, including fields marked as required by design, because there's no transcript to analyze. The "required" status on each field below applies on the condition that the call itself succeeded in producing a conversation.

## punya_kpr_aktif (has active mortgage)

Type: Boolean
Optional: No, always asked unconditionally on every call that results in a conversation
Description: Whether the customer confirms they currently still have an active mortgage running. Set to `true` if the customer says they still have an active mortgage, `false` if the customer says they don't have one, have already paid it off, or never had a mortgage.

## lama_kpr (mortgage duration)

Type: Text
Optional: Yes, only filled if `punya_kpr_aktif` is `true`
Description: How long the customer's mortgage has been running, as stated by the customer, for example "3 tahun" (3 years), "sekitar 5 tahun" (around 5 years). Left empty if the customer doesn't mention a specific duration.

## bank_saat_ini (current bank)

Type: Text
Optional: Yes, only filled if `punya_kpr_aktif` is `true` and the customer is willing to say
Description: The name of the bank where the customer's mortgage currently runs, as stated by the customer. Left empty if the customer doesn't mention the bank name or is unwilling to share it.

## minat_lanjut (interest in proceeding)

Type: Selector, with four options: `tertarik`, `tidak_tertarik`, `minta_pikir_dulu`, `tidak_valid`
Optional: No, always required on calls that result in a conversation, including the invalid case
Description: The customer's level of interest in being contacted further by the PropertyHub team. Filled with one of four values: `tertarik` (interested) if the customer agrees to be contacted, `tidak_tertarik` (not interested) if the customer declines further contact, `minta_pikir_dulu` (wants time to think) if the customer asks for time without a firm answer, or `tidak_valid` (invalid) if the call wasn't relevant at all (for example, a wrong number, the wrong person, or the customer turned out not to have an active mortgage).

## ringkasan_singkat (short summary)

Type: Text, maximum two sentences
Optional: No, always required to have content on calls that result in a conversation
Description: A summary of this mortgage qualifying conversation's result, in at most two sentences, meant for the finance team to read. The first sentence should state the mortgage status, duration, bank, and the customer's interest level. The second sentence should note anything important about how the conversation went, for example a resolved fraud suspicion, a refusal, a wrong number, or another issue. If the conversation went smoothly with nothing notable, the second sentence can simply say so briefly.

This is the field read by the Set Result - Success node in n8n Workflow 2, and written to the Summary column in Google Sheets for successful calls.

## Adjusted Built-in Fields

The Call Summary field is a Retell built-in field that can't be deleted or renamed, so its description was edited to give it a different function from `ringkasan_singkat`. Call Summary is used as a technical summary for debugging and quality assurance purposes, focused on process-level aspects like audio quality, transcription, and technical issues, rather than repeating the business outcome already captured in `ringkasan_singkat`. This functional separation has proven to work as designed in real testing, an example of which can be seen in evidence/call-transcript-sample.md.

The built-in Call Successful and User Sentiment fields are left with their default descriptions, since both are already precise enough as technical and qualitative indicators independent from the other custom fields.

## Relationship with the Disconnection Reason Column in Sheets

The five custom fields above are different from the Disconnection Reason column added later in Google Sheets. The Disconnection Reason column isn't part of Post-Call Analysis, it's taken directly from the `disconnection_reason` field in the Retell webhook payload, and is only filled for failed calls (ones that didn't result in a conversation). Full details on this flow are in n8n-workflow-architecture.md.
