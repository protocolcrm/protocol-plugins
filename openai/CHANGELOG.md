# Changelog

## 0.1.0 (unreleased)

- First OpenAI plugin, in OpenAI's plugin format (root `plugin.json` on the Agent Plugins 1.0.0
  schema, `mcp.json`, `assets/`, `skills/`). One ZIP serves both ChatGPT's "Upload plugin
  archive" and the plugin directory submission.
- Connects to Protocol through the same MCP server as the Claude plugin:
  https://api.protocolcrm.com/mcp, signed in with OAuth.
- Nine skills, sharing one source with the Claude plugin. `protocol-rest-escape` is not
  included: it needs a shell and a full-access API key, and ChatGPT has neither.
- Not yet submitted to OpenAI's plugin directory.
