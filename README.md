# HanseStack — Developer Context & AI Agents

This repository provides machine-readable integration guidelines and context files for the HanseStack API. 

These files are specifically optimized to be used as context for AI coding assistants (like Cursor, Windsurf, or GitHub Copilot). They ensure your AI generates integration code that strictly adheres to our architectural requirements, such as k-anonymity and the fail-open principle.

## Quickstart

Fetch the latest AI context file directly into your local project directory:

```bash
curl -s https://raw.githubusercontent.com/hansestack/developer-context/main/agents/leakcheck.md -o docs/hansestack-agent.md
```


## Usage
Simply reference the downloaded file in your AI prompt.

Example prompt:

> Read @docs/hansestack-agent.md and implement a Go/Node.js/Python HTTP client for our login flow. Ensure it uses the k-anonymity mechanism and acts strictly 'fail-open' on timeouts.

## License
[MIT License](LICENSE) - Feel free to use the provided prompts and code examples in your commercial projects.