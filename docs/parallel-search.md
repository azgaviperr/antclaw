# Parallel Search MCP

Add free web search and page fetching to one NanoClaw agent group with
[Parallel Search MCP](https://docs.parallel.ai/integrations/mcp/search-mcp).
The anonymous endpoint needs no account or API key; searches use Fast mode.
Free access has rate limits.

This guide uses NanoClaw v2's existing per-group MCP configuration with the
Claude provider. NanoClaw registers stdio servers, so
[`mcp-remote`](https://github.com/geelen/mcp-remote) bridges stdio to Parallel's
Streamable HTTP endpoint. No source edits or `.env` credentials are needed.

## Before you start

Run these commands on the host, from a NanoClaw checkout that is already set up
and running. The agent group must use the Claude provider, and Docker must be
available to build its image.

```bash
./bin/ncl groups list --json
```

Choose the agent group's UUID, then inspect its configuration:

```bash
GROUP_ID='replace-with-your-agent-group-uuid'
./bin/ncl groups config get --id "$GROUP_ID" --json
```

Check `provider` and `mcp_servers`. If `parallel-search` already exists, keep it
unless you intend to replace it. Use another unused name throughout this guide
if you want a separate connection. Existing authenticated search or Task MCP
servers can stay in place; this guide only configures anonymous Search MCP.

## Install the bridge and register the server

Install a pinned bridge version in this group's image:

```bash
./bin/ncl groups config add-package --id "$GROUP_ID" --npm mcp-remote@0.14.3
```

This records the package for the next image build. Register its stdio command:

```bash
./bin/ncl groups config add-mcp-server --id "$GROUP_ID" \
  --name parallel-search \
  --command mcp-remote \
  --args '["https://search.parallel.ai/mcp","--transport","http-only","--header","User-Agent:NanoClaw"]'
./bin/ncl groups config get --id "$GROUP_ID" --json
```

The add command preserves servers with other names but replaces an entry with
the same name. Confirm the URL, command, and arguments before restarting.
The header identifies NanoClaw to the remote service. No Authorization header
is required.

Build the group's image and restart its containers:

```bash
./bin/ncl groups restart --id "$GROUP_ID" --rebuild
```

This interrupts running sessions in that group. Send the next message to start
a container with the new configuration. Other groups' configurations are unchanged.
The Claude provider automatically allows tools from registered MCP servers.

## Try search and fetch

Message the selected agent through its usual channel, for example:

> Use parallel-search to find the official Model Context Protocol documentation.
> Fetch the relevant page, summarize what Streamable HTTP does, and cite the URL.

The server exposes `web_search` and `web_fetch` (Claude tool names
`mcp__parallel-search__web_search` and `mcp__parallel-search__web_fetch`).
Ask the agent to reuse one stable `session_id` across related search and fetch
calls, as described in the [tool parameters](https://docs.parallel.ai/integrations/mcp/search-mcp).
Verify that it used those tools and returned source URLs. Built-in web tools
remain available; this setup does not force the agent to choose Parallel.

If the tools are missing, check the group's configuration, confirm the image
rebuild succeeded, and look for `Additional MCP server: parallel-search` in
container output. The bridge needs outbound HTTPS access to
`https://search.parallel.ai/mcp`; its `--enable-proxy` option supports deployments
that require an HTTP proxy. If a request hits the anonymous quota, respect the
service's retry instructions. This endpoint does not provide Task MCP deep research.

## Remove the connection

```bash
./bin/ncl groups config remove-mcp-server --id "$GROUP_ID" --name parallel-search
./bin/ncl groups restart --id "$GROUP_ID"
```

The next container will no longer register this server. The bridge package stays
installed and can be reused by other MCP connections in that group.
