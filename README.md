# Security Alert Worker

A Cloudflare Worker that monitors security events and sends alerts with AI-powered analysis. This solution uses Cloudflare's Workers AI with Llama 4 Scout model to provide intelligent security analysis and automated alerts.

## Features

- 🔍 Real-time security event monitoring
- 🤖 AI-powered analysis using Llama 4 Scout (17B)
- 📊 Detailed attack metrics and patterns
- 🔔 Automated alerts via webhook
- 💾 Persistent state tracking with KV storage
- 🔄 Configurable alert thresholds
- 🛡️ Integration with Cloudflare Ruleset Engine

## Quick Start

1. Set up Cloudflare API token and KV namespace
2. Configure environment variables
3. Deploy with `npx wrangler deploy`

## Prerequisites

1. **Cloudflare Account and API Token**

   - Create an API token with the following permissions:
     - Account Settings: Read
     - Zone Settings: Read
     - Zone: Read
   - [Create API Token](https://dash.cloudflare.com/profile/api-tokens)

2. **Optional: Monitor Specific Rulesets**

   By default, the worker monitors all security events. However, you can optionally configure it to monitor specific types of rulesets (add filter in Graphql API):

   a) **Monitor Rate Limit Rules**

   - Using API:
     ```bash
     curl -X GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/rulesets" \
     -H "Authorization: Bearer {api_token}"
     ```
   - Or from Dashboard:
     - Go to Security > WAF > Rulesets
     - Find your rate limit ruleset and copy the ID

   b) **Monitor Custom Rules**

   - Using API:
     ```bash
     curl -X GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/rulesets" \
     -H "Authorization: Bearer {api_token}"
     ```
   - Or from Dashboard:
     - Go to Security > WAF > Rulesets
     - Find your custom ruleset and copy the ID

   c) **Monitor Managed Rules**

   - Using API:
     ```bash
     curl -X GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/rulesets" \
     -H "Authorization: Bearer {api_token}"
     ```
   - Or from Dashboard:
     - Go to Security > WAF > Rulesets
     - Find your managed ruleset and copy the ID

3. **Optional: Monitor Specific Rules**

   You can also monitor individual rules within a ruleset (e.g., specific DDoS signatures or rate limit rules):

   - Using API:
     ```bash
     curl -X GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/rulesets/{ruleset_id}" \
     -H "Authorization: Bearer {api_token}"
     ```
   - Or from Dashboard:
     - Go to Security > WAF > Rulesets
     - Find your ruleset and copy the specific rule ID

   Note: To monitor specific rules, you'll need to modify the GraphQL query in the worker code to include the rule ID filter.

4. **Create Cloudflare KV Namespace**

   ```bash
   wrangler kv:namespace create "ALERTS_KV"
   ```

   - Note the ID and add it to `wrangler.jsonc`

5. **Environment Variables**
   Create `.env.vars` file with:

   ```
   API_TOKEN="your_api_token"
   ZONE_TAG="your_zone_id"
   WEBHOOK_URL="your_webhook_url"
   ```

6. **Enable Workers AI**
   - Go to Workers & Pages > AI
   - Enable AI for your account
   - Add AI binding to `wrangler.jsonc`

## Configuration

1. **Update wrangler.jsonc**

   ```json
   {
   	"name": "ddos-alerts",
   	"main": "src/index.js",
   	"compatibility_date": "2025-06-06",
   	"kv_namespaces": [
   		{
   			"binding": "ALERTS_KV",
   			"id": "your_kv_namespace_id"
   		}
   	],
   	"ai": {
   		"binding": "AI"
   	},
   	"triggers": {
   		"crons": ["* * * * *"]
   	}
   }
   ```

   The cron trigger `"* * * * *"` will run the worker every minute. You can adjust the cron schedule as needed.

2. **Set Secrets**
   ```bash
   npx wrangler secret bulk .env.vars
   ```
   This command will upload all environment variables from your `.env.vars` file as secrets. Best practice to not to expose secrets in the code.

## Deployment

Deploy the worker using:

```bash
npx wrangler deploy
```

After deployment:

- Check Workers dashboard
- Monitor logs for successful execution
- Test webhook delivery

## How It Works

- Run Cloudflare workers cron
- Look at events from the last 24 hours each time it runs
- Compare the current state with the previous state stored in KV
- Only trigger alerts when there is changes in the 24-hour window
- When alerts are triggered:
  - Events are analyzed by Llama 4 Scout (17B) model
  - Each event contains:
    - `count`: Number of occurrences
    - `dimensions`: Detailed event information
      - `action`: Action taken (block, challenge, etc.)
      - `botScore`: Bot detection score (1-99)
      - `clientAsn`: Client's ASN number
      - `clientCountryName`: Client's country code
      - `clientIP`: Client's IP address
      - `clientRequestHTTPHost`: Targeted host
      - `clientRequestPath`: Requested path
      - `description`: Security rule name/description
      - `edgeResponseStatus`: HTTP response status
      - `ja4`: Client fingerprint
      - `source`: Rule source (firewallManaged, etc.)
  - AI provides a structured analysis including:
    - Attack summary
    - Severity assessment
    - Key indicators (geographic, ASNs, targets, patterns)
    - Recommended actions
  - Analysis and raw events are sent via webhook

### Event Processing Limits

1. **GraphQL Query Limit**

   - Default limit is 25 events per query
   - Can be increased by modifying the `limit` parameter in the GraphQL query
   - Consider response time when increasing the limit

2. **AI Model Token Limits**
   - Llama 4 Scout (17B) has a context window of 131,000 tokens
   - Each event consumes approximately 100 tokens
   - Maximum theoretical limit: ~1,300 events per analysis
   - Recommended to stay well below this limit for optimal performance

### Alert Conditions

- First Run (No Previous State):

  - If there are NO events:
    - No alert will be triggered
    - KV will be updated with the empty state
  - If there ARE events:
    - Alert will be triggered
    - KV will be updated with the new state

- Subsequent Runs (With Previous State):
  - If hash changes (previousHash !== currentHash):
    - Alert will be triggered
    - KV will be updated
  - If hash doesn't change:
    - No alert will be triggered
    - KV will not be updated

### You can add further Modifications for Reduced Alerts

- Tracks Total Count in KV:

  - Calculates total count from all events
  - Stores both the hash and total count in KV
  - Compares current count with previous count

- Alert Conditions:
  - Triggers alert if:
    - First run AND there are events
    - OR attack count has doubled from previous count, not just hashchange (subsequent)

## Development

For local development:

1. Create `.dev.vars` with the same content as `.env.vars`
2. Run `wrangler dev` to test locally

## Troubleshooting

1. **Check Worker Logs**

   - View logs in Cloudflare Dashboard
   - Monitor for any error messages

2. **Verify KV Storage**

   - Check KV namespace in dashboard
   - Verify state is being stored correctly

3. **Test Webhook**
   - Send test alert
   - Verify webhook URL is correct
   - Check webhook response

## API References

- [Ruleset Engine API](https://developers.cloudflare.com/ruleset-engine/basic-operations/view-rulesets/)
- [Workers KV](https://developers.cloudflare.com/workers/runtime-apis/kv/)
- [Workers AI](https://developers.cloudflare.com/workers-ai/)
- [Workers Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)

## Webhook

![Webhook](https://r2.zxc.co.in/git_readme/ai-alert.png)

## Future Enhancements

### Cloudflare Workers and Workflows Integration

The solution can be enhanced with Cloudflare Workers and Workflows to add more robust features:

```javascript
export class DDoSAlertWorkflow extends WorkflowEntrypoint<Env, Params> {
	async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
		const graphqlData = await step.do('fetch-graphql', async () => {
			// Your existing GraphQL query logic
		});

		const processed = await step.do('process-events', async () => {
			// Process events and check thresholds
		});

		const analysis = await step.do('ai-analysis', async () => {
			// AI analysis of security events using Workers AI
		});

		await step.do('send-alert', async () => {
			// Send webhook or email alerts
		});
	}
}
```

This enhancement would provide:

- Robust retry logic with exponential backoff
- Structured workflow steps for better error handling
- Integration of AI analysis as a dedicated workflow step
- Improved reliability for alert delivery
- Better separation of concerns between data fetching, processing, and alerting
