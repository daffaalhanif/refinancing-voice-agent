# Retell AI Agent Configuration Summary

This document summarizes all the technical settings configured on the Retell AI dashboard for the PropertyHub qualifying call agent.

## Model

Main model for real-time conversation: GPT-4.1. Chosen because it's a non-reasoning model with low latency, directly recommended by Retell's official documentation for most production voice agents. Reasoning models like the GPT-5 series add internal thinking time before responding, a trade off that isn't proportional for a qualifying call, which doesn't need complex reasoning but does need natural responsiveness in voice conversation.

Post-Call Analysis model: GPT-5.4 Nano. Chosen because structured extraction from a finished transcript doesn't require low latency or deep reasoning, so the lightest and most cost-efficient model is sufficient for this need.

## Voice

Custom Voice ID from ElevenLabs, sourced from the ElevenCreative platform, set up through Retell's Add custom voice feature. The combination of Retell as the orchestration layer and ElevenLabs as the voice provider is an officially supported and documented integration path for both platforms, not a workaround.

Voice Model: Auto, using ElevenLabs Flash V2.5
Voice Speed: 1.00, normal speaking pace.
Voice Temperature: 1.00, default voice expression variation.
Voice Volume: 1.00, normal volume.

## Speech Settings

Response Eagerness: 0.95. This value climbed gradually from 0.7 at the start, to 0.8 in the first iteration, and now sits at 0.95, as repeated testing kept showing the agent still felt too slow to respond at lower values.

Interruption Sensitivity: 1, up from 0.9 previously, so the agent stops speaking even faster when interrupted by the customer.

Transcription Mode: Optimize for accuracy, changed from Optimize for speed after real testing showed transcription errors on bank names spoken by customers. This setting has been kept as-is since then, with no further changes.

Boosted Keywords: filled with a list of terms prone to transcription errors because they're acronyms or names that rarely appear in everyday conversation, consisting of PropertyHub, KPR, Take Over, refinancing, along with the names of major Indonesian banks: BRI, BCA, Bank Mandiri, BNI, BTN, CIMB Niaga, Panin, Permata, OCBC NISP, BSI, Danamon, and SeaBank.

## Call Settings

Welcome message: AI speaks first, required for outbound calls since the customer doesn't know they're about to be called.
Voicemail Detection: enabled, so the agent doesn't deliver the survey content to an answering machine.
Max Call Duration: lowered from the one-hour default to around five minutes, matching the design of a short qualifying call.
End Call on Silence: lowered from the ten-minute default to around thirty seconds.

## Post-Call Analysis

Five custom fields are configured, fully explained in post-call-analysis-schema.md. The extraction model used is GPT-5.4 Nano as mentioned above.

## Versioning

The agent is currently running on version V3, with the following history. V0 was the first publish. V1 added handling for eleven edge cases. V2 added handling for conversations that drift to other topics before a question gets answered, a finding from real testing. V3 didn't change the system prompt content at all, only raising Response Eagerness to 0.95 and Interruption Sensitivity to 1, the result of a further Speech Settings iteration after additional testing.
