# artuploader — Claude Steering

## Project overview
A tool for uploading art assets. Details TBD as the project grows.

## MCP servers
Two MCP servers are configured in `.mcp.json` but **disabled by default** to avoid unnecessary startup overhead:

- **context7** — resolves up-to-date library docs on demand. Enable when working with third-party libraries where accurate API docs matter.
- **playwright** — browser automation for end-to-end testing. Enable when writing or running browser tests.

To enable a server for a session, toggle it on in Claude Code settings or update `"disabled": false` in `.mcp.json`.

## Code style
- No comments unless the WHY is non-obvious.
- No unnecessary abstractions — keep it simple and direct.
- Prefer editing existing files over creating new ones.

## Testing
- Use Playwright MCP for browser-based end-to-end tests.
- Enable the Playwright MCP server before running or writing browser tests.
