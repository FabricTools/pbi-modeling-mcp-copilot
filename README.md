# Power BI Modeling MCP + GitHub Copilot Cloud Agent Example

This repository is a public example of how to use the [Power BI Modeling MCP server](https://github.com/microsoft/powerbi-modeling-mcp) with GitHub Copilot cloud agents.

The goal is simple: give a Copilot cloud agent authenticated access to Power BI semantic modeling tools so it can inspect and modify models in a Fabric or Power BI workspace as part of an agentic workflow.

This example uses the published npm package [`@microsoft/powerbi-modeling-mcp`](https://www.npmjs.com/package/@microsoft/powerbi-modeling-mcp), so the server can be started directly by the agent with `npx`.

## What this repo demonstrates

- How to configure a repository so Copilot cloud agents can start the Power BI Modeling MCP server automatically.
- How to store Power BI service principal credentials in the repository's `copilot` environment.
- How to expose those credentials to the MCP server using the required `COPILOT_MCP_` secret prefix.
- How to paste the MCP server JSON into the repository's Copilot coding agent settings.
- How to keep the coding agent firewall enabled with the recommended allowlist.

In this repository, the environment is configured against a demo Power BI tenant. To reproduce the setup in your own repository, replace the demo credentials and workspace access with your own tenant, service principal, and Power BI workspace.

## Architecture at a glance

1. A GitHub Copilot cloud agent starts work on an issue or pull request.
2. GitHub loads MCP configuration from the repository's Copilot coding agent settings.
3. The agent starts `@microsoft/powerbi-modeling-mcp` locally with `npx`.
4. GitHub injects environment secrets from the `copilot` environment into the MCP process.
5. The MCP server authenticates to Power BI using a Microsoft Entra service principal.
6. The agent can then use Power BI modeling tools against semantic models the service principal can access.

## Prerequisites

Before configuring GitHub, set up the Power BI side first.

You need:

- A GitHub repository where you can administer environments and Copilot settings.
- GitHub Copilot cloud agent enabled for the repository.
- A Microsoft Entra application and service principal.
- A client secret for that application.
- A Power BI or Fabric workspace that the service principal can access.
- Power BI tenant settings that allow service principals to call the required APIs.

## Step 1: Create and authorize a Power BI service principal

This example uses service principal authentication because Copilot cloud agent MCP configuration currently works well with environment-backed secrets.

At minimum, you need these values:

- `AZURE_TENANT_ID`
- `AZURE_CLIENT_ID`
- `AZURE_CLIENT_SECRET`

Recommended Power BI setup flow:

1. Register a Microsoft Entra app and create a client secret.
2. Optionally create a dedicated Entra security group for Power BI automation and add the service principal to that group.
3. In the Power BI Admin portal, enable the tenant settings needed for service principal access.
4. Add the service principal, or its security group, to the target workspace as `Member` or `Contributor`.

Important Power BI notes:

- `My Workspace` is not supported for service principal scenarios.
- Use least privilege. The agent can use any enabled MCP tool autonomously.
- Restrict service principal access to a dedicated security group where possible.
- If you plan to automate XMLA write operations, confirm the workspace and capacity support the required write settings.

Useful Microsoft references:

- [Embed Power BI content with service principal and an application secret](https://learn.microsoft.com/power-bi/developer/embedded/embed-service-principal)
- [Automate Premium workspace and semantic model tasks with service principals](https://learn.microsoft.com/fabric/enterprise/powerbi/service-premium-service-principal)
- [Embed Power BI content with service principal and a certificate](https://learn.microsoft.com/power-bi/developer/embedded/embed-service-principal-certificate)

Those docs cover the main setup details this repository assumes:

- creating the Entra app
- creating a client secret
- enabling `Allow service principals to use Power BI APIs` / `Service principals can call Fabric public APIs`
- adding the app or security group to a workspace

## Step 2: Create the GitHub repository environment

Create a repository environment named `copilot`.

This is required because Copilot cloud agent only exposes environment secrets and variables from the `copilot` environment to MCP server configuration.

In GitHub:

1. Open your repository.
2. Go to `Settings`.
3. Open `Environments`.
4. Create a new environment named `copilot`.

Then add these environment secrets:

| Secret name | Value |
| --- | --- |
| `COPILOT_MCP_AZURE_TENANT_ID` | Your Microsoft Entra tenant ID |
| `COPILOT_MCP_AZURE_CLIENT_ID` | Your service principal application/client ID |
| `COPILOT_MCP_AZURE_CLIENT_SECRET` | Your service principal client secret |

Important:

- Secrets that are referenced by MCP configuration must be prefixed with `COPILOT_MCP_`.
- If the prefix is missing, GitHub will not expose the value to the MCP server configuration.

Relevant GitHub documentation:

- [Extending GitHub Copilot cloud agent with MCP](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/extend-cloud-agent-with-mcp)
- [Customizing the development environment for GitHub Copilot cloud agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-environment)

## Step 3: Configure the MCP server for the coding agent

Open the repository's coding agent settings page:

- <https://github.com/FabricTools/pbi-modeling-mcp-copilot/settings/copilot/coding_agent>

Add this MCP JSON configuration:

```json
{
  "mcpServers": {
    "powerbi-modeling-mcp": {
      "type": "local",
      "command": "npx",
      "args": [
        "-y",
        "@microsoft/powerbi-modeling-mcp@latest",
        "--start",
        "--authmode=serviceprincipal",
        "--skip-confirmation"
      ],
      "env": {
        "AZURE_TENANT_ID": "$COPILOT_MCP_AZURE_TENANT_ID",
        "AZURE_CLIENT_ID": "$COPILOT_MCP_AZURE_CLIENT_ID",
        "AZURE_CLIENT_SECRET": "$COPILOT_MCP_AZURE_CLIENT_SECRET"
      },
      "tools": ["*"]
    }
  }
}
```

What this configuration does:

- Uses the published npm package instead of requiring a local clone of the MCP server.
- Starts the server in service principal mode.
- Maps GitHub environment secrets into the environment variables expected by the Power BI Modeling MCP server.
- Enables all tools exposed by the server.

If you want a stricter security posture, replace `"*"` with an explicit allowlist of tools once you know which operations your scenario requires.

## Step 4: Keep the firewall enabled

For this scenario, you do not need to reconfigure the Copilot coding agent firewall.

Keep the firewall enabled with the recommended allowlist. That is the expected setup for this example.

In other words:

- do not disable the firewall
- do not add custom firewall exceptions just for this Power BI scenario unless your own environment requires them

## Step 5: Validate the setup

After saving the MCP configuration and secrets, validate the end-to-end flow.

Suggested validation flow:

1. Create an issue in the repository.
2. Ask Copilot to connect to a semantic model in a workspace the service principal can access.
3. Assign the issue to Copilot.
4. Open the resulting Copilot session logs.
5. Confirm the `Start MCP Servers` step shows the `powerbi-modeling-mcp` server starting and listing tools.

Example prompts you can use in an issue:

- `Connect to semantic model 'Sales' in Fabric Workspace 'Demo Workspace' and summarize the model structure.`
- `Connect to semantic model 'Sales' in Fabric Workspace 'Demo Workspace', list the measures, and propose three cleanup improvements.`
- `Connect to semantic model 'Sales' in Fabric Workspace 'Demo Workspace' and add a measure named 'Gross Margin %' if it does not already exist.`

GitHub validation reference:

- [Validating your MCP configuration](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/extend-cloud-agent-with-mcp#validating-your-mcp-configuration)

## Why this example matters

This setup is useful because it turns Copilot cloud agent into a Power BI-aware automation surface.

Instead of limiting the agent to source files in the repository, you can give it controlled access to semantic model metadata and modeling operations through MCP. That makes it possible to build issue-driven and pull-request-driven workflows around semantic model analysis, refactoring, validation, and documentation.

## Security considerations

- Use a dedicated service principal for automation.
- Scope tenant settings to a specific security group where possible.
- Grant workspace access only to the workspaces you intend the agent to operate on.
- Treat the configured MCP tools as privileged capabilities.
- Prefer a certificate-based auth flow for higher-assurance production environments if it fits your operating model.

Power BI Modeling MCP package reference:

- [npm package: `@microsoft/powerbi-modeling-mcp`](https://www.npmjs.com/package/@microsoft/powerbi-modeling-mcp)

## Related references

GitHub Copilot cloud agent:

- [Configuring settings for GitHub Copilot cloud agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/configuring-agent-settings)
- [Customizing the development environment for GitHub Copilot cloud agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-environment)
- [Extending GitHub Copilot cloud agent with MCP](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/extend-cloud-agent-with-mcp)

Power BI service principal and workspace access:

- [Embed Power BI content with service principal and an application secret](https://learn.microsoft.com/power-bi/developer/embedded/embed-service-principal)
- [Automate Premium workspace and semantic model tasks with service principals](https://learn.microsoft.com/fabric/enterprise/powerbi/service-premium-service-principal)
- [Power BI Admin portal developer settings](https://learn.microsoft.com/fabric/admin/service-admin-portal-developer#service-principals-can-call-fabric-public-apis)

Power BI Modeling MCP:

- [GitHub repository: `microsoft/powerbi-modeling-mcp`](https://github.com/microsoft/powerbi-modeling-mcp)
- [npm package: `@microsoft/powerbi-modeling-mcp`](https://www.npmjs.com/package/@microsoft/powerbi-modeling-mcp)

## Summary

To replicate this example in your own repository:

1. Create a Power BI service principal and grant it workspace access.
2. Create a GitHub environment named `copilot`.
3. Add `COPILOT_MCP_AZURE_TENANT_ID`, `COPILOT_MCP_AZURE_CLIENT_ID`, and `COPILOT_MCP_AZURE_CLIENT_SECRET` as environment secrets.
4. Paste the MCP JSON into the repository's Copilot coding agent settings.
5. Leave the firewall enabled with the recommended allowlist.
6. Assign a test issue to Copilot and confirm the MCP server starts successfully.
