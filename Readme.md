# n8n LLM Automation

An n8n workflow that receives support requests through a webhook, classifies and summarizes them with the Groq API, and routes urgent requests to an external REST endpoint.

This project demonstrates a practical no-code/low-code LLM automation pattern using:

- n8n
- Docker Compose
- Webhooks
- Groq's OpenAI-compatible chat completions API
- Conditional routing
- REST API integration

## Workflow Overview

```text
Support Webhook
			 |
			 v
Classify With Groq
			 |
			 v
Normalize Result
			 |
			 v
Urgent?
	 /       \
 Yes        No
	|          |
	v          v
Send to      Return normal
destination  result
	|
	v
Return urgent result
```

## What the Workflow Does

1. The `Support Webhook` node accepts a JSON `POST` request.
2. The `Classify With Groq` node sends the support request to Groq.
3. Groq returns structured JSON containing:
	 - `category`
	 - `priority`
	 - `summary`
	 - `sentiment`
	 - `recommended_action`
4. The `Normalize Result` node combines the classification with request metadata.
5. The `Urgent?` node checks whether `priority` equals `urgent`.
6. Urgent requests are sent to the configured destination REST endpoint.
7. Normal requests are returned directly to the webhook caller.

## Project Structure

```text
n8n-llm-automation/
├── workflow/
│   └── groq-support-classifier.json
├── docs/
│   └── setup.md
├── .env.example
├── .gitignore
├── docker-compose.yml
└── README.md
```

## Prerequisites

Install or create accounts for:

- Docker Desktop
- A Groq account and API key
- n8n, provided by the Docker container in this project
- An HTTP destination for urgent requests, such as Webhook.site, Slack, an internal API, or a ticketing system

Docker Desktop must be running before starting n8n.

## Configuration

Create a file named `.env` in the project root. Do not commit this file.

```env
GROQ_API_KEY=gsk_your_real_groq_api_key
DESTINATION_URL=https://your-destination-endpoint.example.com
```

`GROQ_API_KEY` is used by the Groq HTTP Request node through the n8n Header Auth credential.

`DESTINATION_URL` is the REST endpoint that receives urgent classifications.

For local testing, create a temporary endpoint at [Webhook.site](https://webhook.site), then use its unique URL as `DESTINATION_URL`.

## Start n8n with Docker

Open a terminal in the project directory:

```powershell
cd E:\GitProjects\n8n-llm-automation
```

Start the container:

```powershell
docker compose up -d
```

Open n8n in your browser:

```text
http://localhost:5678
```

Useful Docker commands:

```powershell
# Show running containers
docker compose ps

# View n8n logs
docker compose logs -f n8n

# Stop n8n
docker compose down

# Restart n8n after changing .env or docker-compose.yml
docker compose down
docker compose up -d
```

If Docker reports that it cannot connect to `dockerDesktopLinuxEngine`, start Docker Desktop and wait until its Linux engine is running.

## Import the Workflow

1. Open `http://localhost:5678`.
2. Create or sign in to your local n8n account.
3. Select **Import from File** from the workflow menu.
4. Choose `workflow/groq-support-classifier.json`.
5. Open the imported workflow.
6. Save the workflow.

The exported JSON is the portable workflow definition. Credentials are stored separately by n8n and are not included in the export.

## Configure the Groq Credential

The `Classify With Groq` node uses a Header Auth credential.

1. Open **Credentials** in n8n.
2. Select **Create New**.
3. Choose **Header Auth**.
4. Create a credential with:

```text
Name: Authorization
Value: Bearer gsk_your_real_groq_api_key
```

5. Save the credential.
6. Open the `Classify With Groq` node.
7. Set:

```text
Authentication: Generic Credential Type
Generic Auth Type: Header Auth
Credential: your Header Auth credential
```

Under **Send Headers**, keep only this manually configured header:

```text
Name: Content-Type
Value: application/json
```

Do not configure `Authorization` both as a credential and as a manual header. Duplicate authorization headers can cause authentication failures.

## Groq Request Configuration

The `Classify With Groq` node must use:

```text
Method: POST
URL: https://api.groq.com/openai/v1/chat/completions
Body Content Type: JSON
Specify Body: Using JSON
```

The currently working model ID is:

```text
openai/gpt-oss-120b
```

Model availability can change. If Groq reports that the model does not exist, list the models available to your key:

```bash
curl https://api.groq.com/openai/v1/models \\
	-H "Authorization: Bearer YOUR_REAL_GROQ_KEY"
```

Use an available chat or text-generation model, not a model intended only for moderation or classification if the request contains multiple chat messages.

The JSON body should produce a string for the user message. In n8n, use the JSON field in **Expression** mode and use this expression:

```javascript
={{ JSON.stringify({
	model: "openai/gpt-oss-120b",
	temperature: 0.1,
	response_format: {
		type: "json_object"
	},
	messages: [
		{
			role: "system",
			content: "You classify support messages. Return only valid JSON with these fields: category, priority, summary, sentiment, recommended_action. category must be one of billing, technical, account, shipping, other. priority must be one of low, medium, high, urgent. sentiment must be one of positive, neutral, negative."
		},
		{
			role: "user",
			content: JSON.stringify($json.body || $json)
		}
	]
}) }}
```

Do not include Markdown code fences when pasting the expression into n8n.

## Webhook Configuration

The `Support Webhook` node must use:

```text
HTTP Method: POST
Path: support-intake
Respond: Using Respond to Webhook Node
```

The test URL is:

```text
http://localhost:5678/webhook-test/support-intake
```

The production URL is:

```text
http://localhost:5678/webhook/support-intake
```

The test URL works only while the workflow is listening for a test event. The production URL requires the workflow to be active.

Opening the URL in a browser sends a `GET` request and produces a webhook method error. Use `POST` with JSON instead.

## Test with Bash or Git Bash

Start a test execution in n8n, then run:

```bash
curl -X POST "http://localhost:5678/webhook-test/support-intake" \\
	-H "Content-Type: application/json" \\
	-d '{
		"request_id": "ticket-1001",
		"customer_email": "customer@example.com",
		"message": "I have a question regarding an order."
	}'
```

Expected behavior for a normal request:

- Groq classifies the message.
- The `Urgent?` node follows the false branch.
- The response contains the normalized classification.

## Test an Urgent Request

```bash
curl -X POST "http://localhost:5678/webhook-test/support-intake" \\
	-H "Content-Type: application/json" \\
	-d '{
		"request_id": "ticket-1002",
		"customer_email": "customer@example.com",
		"message": "My payment failed and I need urgent help immediately."
	}'
```

Expected behavior for an urgent request:

- Groq returns `priority` as `urgent`.
- The `Urgent?` node follows the true branch.
- `Send Urgent Item` sends the normalized JSON to `DESTINATION_URL`.
- The urgent response node returns the incoming destination response.

The destination may return a message such as:

```json
{
	"data": "This URL has no default content configured."
}
```

That means the request reached Webhook.site successfully. Open Webhook.site to inspect the received request.

## Test with PowerShell

Use PowerShell syntax only in a PowerShell terminal, not Git Bash:

```powershell
$body = @{
	request_id = "ticket-1001"
	customer_email = "customer@example.com"
	message = "I have a question regarding an order."
} | ConvertTo-Json

Invoke-RestMethod `
	-Method Post `
	-Uri "http://localhost:5678/webhook-test/support-intake" `
	-ContentType "application/json" `
	-Body $body
```

## Activate the Workflow

After the test workflow succeeds:

1. Save the workflow.
2. Activate it using the **Active** switch.
3. Send requests to:

```text
http://localhost:5678/webhook/support-intake
```

Do not use the `webhook-test` URL for production integrations.

## Troubleshooting

### Docker engine is unavailable

Start Docker Desktop, then run:

```powershell
docker version
docker compose up -d
```

Both the Docker Client and Server sections should be available.

### Credentials not found

Open `Classify With Groq` and confirm:

```text
Authentication: Generic Credential Type
Generic Auth Type: Header Auth
```

Also confirm that the selected Header Auth credential contains:

```text
Name: Authorization
Value: Bearer gsk_your_real_groq_api_key
```

### Authorization failed

The key may be invalid, revoked, copied with extra spaces, or duplicated in the node configuration. Test it without exposing it in project files:

```bash
curl https://api.groq.com/openai/v1/models \\
	-H "Authorization: Bearer YOUR_REAL_GROQ_KEY"
```

If the key was exposed, revoke it and create a replacement.

### Model not found

Run the models request above and replace the model with an available model ID. Include the complete provider prefix, such as `openai/`, when Groq returns one.

### `messages.1.content` must be a string

The user message must use:

```javascript
content: JSON.stringify($json.body || $json)
```

Do not pass `$json.body` directly if it is an object.

### Invalid JSON in the Groq request body

Ensure the complete JSON body is wrapped with:

```javascript
={{ JSON.stringify({ ... }) }}
```

The n8n JSON field must be in **Expression** mode, and Markdown code fences must not be pasted into the field.

### Invalid JSON in Respond to Webhook

The working original configuration for the urgent response is:

```text
Respond With: First Incoming Item
```

Remove the Response Body expression when using this option. If using `Respond With: JSON`, n8n may convert an object expression into `[object Object]` and reject it as invalid JSON.

### Webhook is not registered for GET requests

The webhook accepts `POST`, not `GET`. Use `curl`, PowerShell, Postman, or another HTTP client.

## Security Notes

- Never commit `.env`.
- Never commit a Groq API key in the workflow JSON.
- Do not share screenshots containing credentials.
- Revoke any key that was exposed in terminal output, screenshots, or public repositories.
- Use webhook authentication before exposing n8n outside your local machine.
- Add input validation and rate limiting for production use.
- Use HTTPS and restrict the destination endpoint.
- Avoid sending sensitive customer data to an external LLM unless your data-handling requirements permit it.


## Workflow Screenshot

Capture the n8n canvas after arranging the nodes from left to right and save the image as:

```text
docs/workflow-screenshot.png
```

Then add this section to the README:

```markdown
## Workflow Screenshot

![n8n workflow](docs/workflow-screenshot.png)
```

## Future Improvements

- Replace Webhook.site with email, Google Sheets, Slack, a database, or a ticketing system.
- Add webhook authentication.
- Validate the incoming schema before calling Groq.
- Add retries and error handling for Groq and the destination API.
- Store results in a database for reporting.
- Add a scheduled trigger for batch processing.
- Add monitoring and execution alerts.
- Add automated tests for normal, urgent, malformed, and missing-message requests.
