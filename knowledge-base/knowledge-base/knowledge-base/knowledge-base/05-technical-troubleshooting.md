NOAVIA — Technical Troubleshooting Guide
My AI Agent Is Not Responding
Check if the agent service is running by visiting your agent dashboard at https://portal.noavia.de/agents
If the status shows "Offline," try restarting the agent from the dashboard (click Restart)
If using an on-premise deployment, verify that Docker containers are running: `docker ps` should list the NOAVIA agent containers
Check your internet connection and firewall rules — the agent requires outbound HTTPS access on port 443
If the issue persists, contact support@noavia.de with your Kundennummer and agent ID
Webhook Is Not Receiving Data
Verify the webhook URL is correctly configured in your source system
Ensure the webhook endpoint is accessible (test with a simple POST request using curl or Postman)
Check that the payload format matches the expected schema (JSON with required fields)
Review the n8n execution log at https://portal.noavia.de/workflows for error details
If your company uses a corporate firewall or proxy, ensure the NOAVIA webhook domain is whitelisted
AI Responses Are Inaccurate or Irrelevant
Check if the knowledge base is up to date — outdated documents lead to outdated answers
Review the RAG retrieval logs to see which documents are being retrieved
The AI may need retraining or prompt adjustment if your business processes have changed
Contact your NOAVIA account manager to schedule a knowledge base refresh
For critical accuracy issues, submit a ticket with category "AI Quality" and include example queries with expected vs. actual responses
Integration Issues with ERP / External Systems
Verify API credentials have not expired — most API keys rotate every 90 days
Check the API rate limits for your plan (Standard: 1,000 req/hour, Enterprise: unlimited)
Review the integration log at https://portal.noavia.de/integrations
For SAP or DATEV integrations, ensure the middleware connector version is current
Contact support with the integration name, error message, and timestamp
Performance Is Slow
Check the NOAVIA status page at https://status.noavia.de for any ongoing incidents
For on-premise deployments, verify that the server meets minimum requirements: 8 GB RAM, 4 CPU cores, 50 GB SSD
Large document processing jobs (>100 documents) may queue during peak hours — check the job queue in your dashboard
If using vector search, the Qdrant instance may need more memory — contact support to review your configuration
How to Access Logs
All execution logs are available in the client portal:
Go to https://portal.noavia.de
Navigate to Workflows → Execution History
Filter by date, status (success/error), or workflow name
Click on any execution to see detailed step-by-step logs
Logs are retained for 30 days. For longer retention, enable log export to your own storage.
