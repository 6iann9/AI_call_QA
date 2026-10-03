# AI Sales Call Quality Analysis

An n8n pipeline that retrieves Ringostat call recordings, transcribes each recording in Russian and Ukrainian with AssemblyAI, and uses Gemini to evaluate sales-call quality. Google Sheets stores transcripts, call metadata, and structured feedback; Google Drive organizes the generated spreadsheets.

**Security notice:** The uploaded source contained an exposed plaintext Ringostat `Auth-key`. Revoke or rotate it **before publishing**, even though this project removes it. Never upload the original export. If it has already been committed, remove it from repository history as well; deleting it from the latest file does not revoke it.

## Pipeline

Manual trigger → date window → create/move spreadsheets → retrieve/filter calls → parallel RU/UK transcription and polling → store transcripts → evaluate batches of 20 → write feedback.

The Russian-language prompt evaluates ten criteria: Ukrainian language, upselling, product presentation, weight clarification, final order confirmation, order details, politeness, alternatives, responsiveness, and complaint handling. Ratings are `да`, `частично`, `нет`, or `неприменимо`, plus buyer context, comments, and a transcript. It compares two ASR versions of the same recording and requires one result per call in input order. Retail/grill examples remain customizable.

## Requirements and setup

1. Use an n8n installation supporting the node versions in `workflows/sales-call-qa.json`. The export does not specify its original n8n or community-package release, so no minimum runtime version is claimed. Install the community package `n8n-nodes-assemblyai` providing `n8n-nodes-assemblyai.assemblyAi` version 1; your hosting environment must support that package. See [community-node installation](https://docs.n8n.io/integrations/community-nodes/installation/gui-install/).
2. Import the workflow JSON. It is inactive and uses a manual trigger. Check for unknown nodes or unsupported versions before execution.
3. Create credentials inside n8n and select them on every relevant node:

   | Service | Credential and nodes |
   | --- | --- |
   | Ringostat | **Header Auth**: header name `Auth-key`, value your newly rotated key. Select it in `HTTP Request`, configured for Generic Credential Type → Header Auth. |
   | AssemblyAI | AssemblyAI API credential on both Create and both Get transcription nodes. |
   | Gemini | Google Gemini API credential on both Google Gemini Chat Model nodes. |
   | Google Sheets | Google Sheets OAuth2 credential on every Sheets node. |
   | Google Drive | Google Drive OAuth2 credential on both Move spreadsheet nodes. |

   [Header Auth documentation](https://docs.n8n.io/integrations/builtin/credentials/httprequest/) · [Google OAuth setup](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/). For your own Google OAuth client, enable Sheets/Drive APIs and configure the redirect URI shown by n8n. The connected account needs spreadsheet creation/editing and access to both destination folders.
4. Replace `REPLACE_WITH_CALLDATA_FOLDER_ID` and `REPLACE_WITH_CALLFEEDBACK_FOLDER_ID` in the corresponding Move nodes. Replace `REPLACE_WITH_EXCLUDED_EMPLOYEE_LABEL` in `Filter`, or remove that condition. Spreadsheet IDs are generated during execution; their dynamic expressions are preserved.
5. Customize the AI Agent brand/store examples and business rules. Confirm the recording channels: the prompt assumes channel 1 is the seller and channel 2 the buyer. Both transcription branches request multichannel audio with fixed language codes `ru` and `uk`.
6. Review the workflow timezone, date subtraction, and Ringostat query window (currently `00:00:00`–`08:30:00` for the derived date). Confirm the two retained Gemini model names are available for your account; choose supported models if necessary. Budget for two transcriptions per call and Gemini evaluation/parser retries.

No `.env.example` is needed: this workflow uses n8n-managed credentials and UI configuration, with no environment-variable references. Importing a `.env` file would not bind credentials or configure the folders.

## Verify before operational use

This is a sanitized source export, not a verified production deployment. JSON parsing, node count, connection endpoints, unchanged connections, the QA schema, and removal of original sensitive values were checked. A live n8n import and API execution were not performed.

Inherited behavior needs a small controlled test:

- Polling checks for empty transcript text, with no explicit failed-status handling or retry limit. Add bounded polling and error handling before processing many calls.
- The QA output omits `recording`, but `Append or update row in sheet` matches on `recording`. Restore the matching key from the corresponding input call before that node, and verify QA columns are mapped and saved.
- Transcripts are arrays upstream, while the output schema requires `transcript` as a string. Define serialization explicitly and verify ordering/result counts; the schema alone cannot enforce one result per input call.
- Check header creation and automatic column mapping in the newly created `dialogues` and `feedback` sheets. Reruns create new spreadsheets.

Use authorized test recordings and restricted output folders. Execution data contains employee names, phone-related fields, recording links, and transcripts; keep it out of GitHub and configure suitable retention/access controls.

## Publication

Upload only this project directory. Credential references, private folder IDs/URLs, cached resource names, instance metadata, Wait webhook IDs, and business/location identifiers were removed or replaced. Internal node/condition IDs were regenerated; all 39 nodes, connections, type versions, and the structured QA schema were retained. No static private spreadsheet ID was present; runtime spreadsheet references remain intact. Pinned data is empty.

Review future n8n exports again: saving configured credentials, resource selections, or pinned executions can reintroduce private identifiers. `.gitignore` cannot sanitize tracked files or their history.
