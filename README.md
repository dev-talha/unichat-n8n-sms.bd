# Unichat to sms.bd SMS Automation for n8n

This repository contains an n8n workflow that connects Unichat with the sms.bd SMS API. When an agent sends an outgoing message from a Unichat inbox, the workflow extracts the contact phone number and message content, sends the message through sms.bd, and then writes the result back to the same Unichat conversation as a private note.

## Features

- Receives Unichat `message_created` webhook events.
- Sends SMS only when `event` equals `message_created`.
- Extracts the recipient number from the Unichat contact data.
- Extracts the outgoing message content from the Unichat payload.
- Sends the message using the sms.bd Send SMS API.
- Adds an optional SMS footer/signature from the workflow configuration.
- Posts a clean private note back to Unichat after sending.
- Shows a success private note when sms.bd returns `error: 0`.
- Shows the actual sms.bd error code and message when sending fails.
- Keeps all configuration inside the n8n workflow. No `.env` file is required.

## Workflow Overview

```text
Unichat Webhook
  -> Workflow Config - Edit Here
  -> Normalize + Add Footer
  -> Send SMS via sms.bd
  -> Prepare Unichat Private Note
  -> Add Private Note to Unichat
```

## Requirements

- n8n instance with public webhook access
- Unichat account
- Unichat user API access token
- sms.bd API key
- A Unichat inbox that sends webhook events to n8n

## Importing the Workflow

1. Open your n8n dashboard.
2. Go to **Workflows**.
3. Click **Import from File**.
4. Select the workflow JSON file.
5. Open the imported workflow.
6. Edit the node named **Workflow Config - Edit Here**.
7. Add your sms.bd API key and Unichat API token.
8. Save and activate the workflow.

## Configuration

All configuration is stored inside the n8n node named:

```text
Workflow Config - Edit Here
```

Update these values:

```js
smsbdApiKey: 'YOUR_SMSBD_API_KEY',
UnichatBaseUrl: 'https://app.unichat.com.bd',
UnichatApiToken: 'YOUR_Unichat_USER_API_TOKEN',

smsFooterEnabled: true,
smsFooterText: 'Company Name\nSign: Your Team',

senderId: '',
contentId: '',
schedule: ''
```

### Configuration Fields

| Field | Description |
|---|---|
| `smsbdApiKey` | Your sms.bd API key. |
| `UnichatBaseUrl` | Your Unichat base URL, for example `https://app.unichat.com.bd`. |
| `UnichatApiToken` | Your Unichat user API access token. |
| `smsFooterEnabled` | Set to `true` to append a footer to every SMS. Set to `false` to disable it. |
| `smsFooterText` | Footer text added under the SMS message. You can use company name, signature, or support text. |
| `senderId` | Optional approved sms.bd sender ID. Leave blank if not used. |
| `contentId` | Optional sms.bd campaign content ID. Leave blank if not used. |
| `schedule` | Optional schedule time in `YYYY-MM-DD HH:mm:ss` format. Leave blank to send immediately. |

## SMS Footer Example

To add a company name or signature:

```js
smsFooterEnabled: true,
smsFooterText: 'ABC Company\nSupport Team'
```

The SMS will be sent like this:

```text
Original Unichat message

ABC Company
Support Team
```

To disable the footer:

```js
smsFooterEnabled: false
```

## Unichat Webhook Setup

In Unichat, add the n8n webhook URL as a webhook endpoint.

The webhook URL will look like this:

```text
https://YOUR_N8N_DOMAIN/webhook/Unichat-to-smsbd
```

Use the production webhook URL after activating the workflow.

The workflow expects Unichat payloads with this event:

```json
{
  "event": "message_created"
}
```

Only this event is allowed to continue to the sms.bd API call. Any other event is ignored.

## sms.bd API Used

The workflow uses the sms.bd Send SMS endpoint:

```text
https://api.sms.net.bd/sendsms
```

Required parameters:

| Parameter | Description |
|---|---|
| `api_key` | sms.bd API key |
| `msg` | SMS message body |
| `to` | Recipient phone number |

Optional parameters:

| Parameter | Description |
|---|---|
| `sender_id` | Approved sender ID |
| `content_id` | Approved campaign content ID |
| `schedule` | Scheduled sending time |

## Unichat Private Note

After sms.bd responds, the workflow posts a private note back to the same Unichat conversation.

Unichat API format:

```http
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/messages
```

Request body:

```json
{
  "content": "Message sent successfully.\nRequest ID: 0000",
  "message_type": "outgoing",
  "private": true,
  "content_type": "text"
}
```

## Success Response

When sms.bd returns:

```json
{
  "error": 0,
  "msg": "Request successfully submitted",
  "data": {
    "request_id": 1234
  }
}
```

Unichat private note will show:

```text
Message sent successfully.
Request ID: 1234
```

## Error Response

When sms.bd returns an error, for example:

```json
{
  "error": 417,
  "msg": "Insufficient balance"
}
```

Unichat private note will show:

```text
Message send failed.
Error 417: Insufficient balance.
```

## Common sms.bd Error Codes

| Error Code | Meaning |
|---|---|
| `0` | Success |
| `400` | Missing or invalid parameter |
| `403` | Permission denied |
| `404` | Resource not found |
| `405` | Authorization required |
| `409` | Unknown server error |
| `410` | Account expired |
| `411` | Reseller account expired or suspended |
| `412` | Invalid schedule |
| `413` | Invalid sender ID |
| `414` | Message is empty |
| `415` | Message is too long |
| `416` | No valid number found |
| `417` | Insufficient balance |
| `420` | Content blocked |

## Expected Unichat Payload Fields

The workflow reads these fields from the Unichat webhook payload:

```js
event
content
account.id
conversation.id
conversation.meta.sender.phone_number
conversation.meta.sender.identifier
```

Phone number priority:

1. `conversation.meta.sender.phone_number`
2. `conversation.meta.sender.identifier`

For WhatsApp identifiers like:

```text
8801765447530@s.whatsapp.net
```

the workflow normalizes the number to:

```text
8801765447530
```

## Important Notes

- The workflow sends SMS only for `message_created` events.
- Private notes are created with `private: true`, so customers will not see them.
- The workflow does not require a `.env` file.
- Keep your sms.bd API key and Unichat API token private.
- Do not commit a production workflow JSON with real API keys to a public repository.

## Troubleshooting

### SMS sends but private note does not appear

Check the n8n execution history and open the node:

```text
Add Private Note to Unichat
```

Common causes:

- Wrong Unichat API token
- Wrong Unichat base URL
- Token user does not have permission for the account/conversation
- Incorrect `account_id` or `conversation_id`
- Unichat API request body was changed manually

### Private note appears but SMS does not send

Check the node:

```text
Send SMS via sms.bd
```

Common causes:

- Invalid sms.bd API key
- Invalid phone number
- Empty message
- Insufficient sms.bd balance
- Message content blocked by sms.bd

### Workflow triggers for other Unichat events

The workflow has a guard condition that only allows:

```js
event === 'message_created'
```

Other events are ignored automatically.

## Security Recommendation

For production use, store API keys securely. This workflow keeps credentials inside the workflow because it is designed for installations where `.env` files cannot be edited. Avoid publishing real credentials in GitHub.

## License

You can use and modify this workflow for your own Unichat and sms.bd automation.
