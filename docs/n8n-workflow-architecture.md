# n8n Workflow Architecture

This system uses two separate n8n workflows rather than combining everything into a single canvas. The reasoning is that the two triggers have fundamentally different natures. Workflow 1 is triggered on a schedule from within the system itself, while Workflow 2 is triggered reactively by an external webhook. Both also have independent lifecycles, and this separation makes each workflow's responsibility clearer to understand and maintain.

## Workflow 1: Outbound Call Trigger

```
Trigger - Poll Every 5 Minutes
    -> Get All Rows from Sheets
        -> Filter - Not Called & Active Mortgage (Status == not-called AND active mortgage == "Ya, masih ada")
            -> Limit (Max Items = 1)
                -> Trigger Call - Retell API (POST to Retell create-phone-call)
```

### Trigger - Poll Every 5 Minutes

Runs every 5 minutes, triggering the entire Workflow 1 flow from the start each time.

### Get All Rows from Sheets

Fetches every row from the "Form responses 1" sheet in the "PropertyHub - Refinancing Call Queue" spreadsheet, with no filtering applied at this level. Filtering happens in the next node.

### Filter - Not Called & Active Mortgage

Checks two conditions at once with an AND combinator, not just one:
1. The Status column equals `not-called`
2. The "Apakah kamu saat ini masih punya cicilan KPR yang berjalan?" (Do you currently have an active mortgage?) column equals "Ya, masih ada" (Yes, still ongoing)

This double filter is required because rows answered "No" still end up in the Sheet, since the Form only skips the consent question rather than skipping submission entirely. Without the second condition, irrelevant rows would risk being called by the system.

Since the Status column now has three possible values (`called`, `not-called`, `error`), this filter specifically looks for the `not-called` value only, rather than simply "not called". This matters so that rows with an `error` status, which need manual investigation rather than automatic retry, don't get called again on their own.

### Limit

Set with Max Items equal to 1, First Items mode. This node restricts things so that out of however many rows pass the Filter, only one row is forwarded to the next node within a single Schedule Trigger execution.

The reasoning behind this restriction is worth understanding. If a single polling cycle finds two or more rows passing the Filter, without the Limit node, the HTTP Request node would process all of them at once, triggering two outbound calls almost simultaneously from the same Twilio number. This carries two risks. First, a technical risk of concurrency errors on the SIP trunk, since a single Twilio number isn't necessarily able to handle two simultaneous outbound calls without special configuration. Second, a reputational risk for the number, since a pattern of simultaneous calls from the same number resembles a robocall or spam campaign pattern, which could get the number flagged as suspicious by telco carriers and lower its call answer rate going forward.

With the Limit set to 1, any row not processed in a given cycle automatically waits for the next Schedule Trigger cycle, five minutes later, because the status of the row that was processed will have changed (no longer `not-called`), so it won't pass the Filter again next time, letting the next row in the queue pass through instead.

### Trigger Call - Retell API

An HTTP Request node that sends a POST request to Retell's create-phone-call endpoint, with a JSON body containing the origin number (`from_number`, a US Twilio number), the destination number (`to_number`, taken from the `phone_e164` column), the Retell agent ID being used (`override_agent_id`), and a dynamic customer name variable that gets injected into the agent's system prompt (`retell_llm_dynamic_variables.customer_name`).

## Workflow 2: Call Result Handler

```
Receive Webhook - All Events from Retell (path /retell-webhook)
    -> Filter - Event Call Analyzed
        -> Find Row - Match Phone Number
            -> Switch (classifies into 4 categories)
                -> Set node (one per category)
                    -> Update Sheets - Status and Summary
```

### Receive Webhook - All Events from Retell

Receives every type of event Retell sends throughout a single call's lifecycle (`call_started`, `call_ended`, `call_analyzed`, and others), not just one specific event. Event filtering happens in the next node.

### Filter - Event Call Analyzed

Checks whether `body.event` equals `call_analyzed`. This event was chosen specifically because it's the first point at which Retell has finished processing the call's analysis results (`call_analysis`, including the five custom Post-Call Analysis fields). Other events like `call_started` or `call_ended` occur earlier, before that analysis process is complete, so processing at those points would mean the needed data isn't available yet.

### Find Row - Match Phone Number

Searches for the row in the Sheet whose `phone_e164` column matches the `body.call.to_number` value from the webhook payload, which is the number that was just called.

### Switch - Classify Call Result

This node is the most substantial part of Workflow 2. Every completed call is classified into one of four categories, based on `call_status`, `disconnection_reason`, and `transcript` from the Retell webhook payload.

| Category | Condition | Status written |
|---|---|---|
| Ended - Success | call_status equals `ended`, AND disconnection_reason is not `voicemail_reached`, AND not `ivr_reached`, AND transcript is not empty | `called` |
| Ended - No Conversation | call_status equals `ended`, BUT disconnection_reason is `voicemail_reached` or `ivr_reached`, OR transcript is empty/missing | `not-called` |
| Not Connected | call_status equals `not_connected` | `not-called` |
| Error | call_status equals `error` | `error` |

#### Why four categories, not just two (called/not-called)

The original design only distinguished `called` from `not-called` based on whether the API call to Retell was successfully sent, without accounting for what actually happened during the call. This meant that calls which failed to connect entirely, or connected but had no real conversation, were still recorded as `called`, so the row would never be retried by Workflow 1, even though the customer had never actually been asked anything.

The `error` category was separated from ordinary `not-called` because the two need different handling. A failure because the customer didn't answer is normal and worth retrying automatically. A failure due to a system error, such as a billing issue or a Retell concurrency limit, won't be resolved just by retrying, and needs manual investigation.

#### Why check the transcript, not just disconnection_reason

Checking disconnection_reason alone (excluding `voicemail_reached` and `ivr_reached`) was initially thought to be enough to filter out the "connected but no real conversation" case. However, testing revealed another case: calls that technically connected (call_status `ended`) but where the customer never responded to anything throughout the call, with a disconnection_reason that wasn't voicemail or ivr. Cases like this still need to be treated as a failed qualifying attempt, not a success, even though the call technically "ended normally". The transcript check, confirming there's transcript content that isn't empty, was added to catch this case more generally than listing out every possible disconnection_reason one by one.

### Set Node (four variants: Success, No Conversation, Not Connected, Error)

Each of the four Switch routes connects to a separate Set node, whose job is to write explicit values for three fields, namely Status, Summary, and Disconnection Reason, according to that category.

Set Result - Success writes Status `called`, Summary filled from the `ringkasan_singkat` field in Retell's Post-Call Analysis, and Disconnection Reason left empty.

Set Result - No Conversation, Set Result - Not Connected, and Set Result - Error all write an empty Summary, and Disconnection Reason filled from the raw disconnection_reason value in the webhook payload. The only difference is the Status value written, `not-called` for the first two categories and `error` for the last.

All four Set nodes converge into the same Update Sheets node.

### Update Sheets - Status and Summary

Performs an update operation on the matching row, matched by the phone_e164 column. The Status, Summary, and Disconnection Reason fields being written all read from the data the Set node just produced ($json), not directly from the raw webhook payload. This matters, because if these fields were left reading directly from the webhook, this node wouldn't know which Route actually fired for that item, and would write an incorrect or inconsistent value.
