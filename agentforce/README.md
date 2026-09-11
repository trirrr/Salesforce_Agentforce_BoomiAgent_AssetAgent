# Agentforce Account Lookup Tool

This directory contains the tool definition and configuration for integrating the Aprimo SF Account Lookup integration with Salesforce Agentforce.

## Files

- `account-lookup-tool-definition.json` — Agentforce tool definition schema
- `implementation-guide.md` — Step-by-step integration guide

## Quick Start

1. Copy the tool definition JSON
2. Register in Agentforce as a new agent tool
3. Configure the endpoint URL and authentication
4. Test with sample payloads
5. Deploy to your Agentforce agents

## Endpoint

```
POST https://aprimo-prod.boomi.cloud/ws/simple/executeAprimo_SF_AccountLookup
```

## Authentication

Basic Auth with Salesforce-dedicated credential:
- Username: `Salesforce-aprimo-master@aprimo-UXVOIK.B9T0OD`
- Token: (Available in Boomi credential store)

## Input Schema

```json
{
  "accountName": "string (required)",
  "industry": "string (optional)",
  "type": "string (optional)",
  "billingCountry": "string (optional)"
}
```

## Response Schema

Returns Salesforce Account objects with 30+ fields including:
- ID, Name, Type, Industry, Website
- AnnualRevenue, NumberOfEmployees
- Rating, BillingCountry, BillingState
- Custom fields (SDO_Sales_Region, Tier, etc.)

## Latency

~3-4 seconds (Salesforce query + response transformation)

## Support

See `/README.md` for complete documentation and flow diagrams.
