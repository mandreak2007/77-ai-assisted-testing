# Playwright MCP Agent Patterns

Reference for using Playwright MCP tools and orchestrating AI agents in test automation.

---

## What is Playwright MCP?

Playwright MCP exposes a Playwright browser as a set of **MCP tools** that an AI agent can call. This enables AI-driven test execution where the agent navigates, interacts, and asserts on web pages using natural language + structured tool calls.

---

## Available MCP Tools

| Tool | Description | Key Parameters |
|---|---|---|
| `browser_navigate` | Navigate to a URL | `url` |
| `browser_screenshot` | Capture current page state | `name?` |
| `browser_click` | Click an element | `element` (description or selector) |
| `browser_type` | Type text into a field | `element`, `text` |
| `browser_hover` | Hover over element | `element` |
| `browser_select_option` | Select dropdown option | `element`, `values` |
| `browser_check` | Check/uncheck checkbox | `element` |
| `browser_wait_for` | Wait for element/condition | `text?`, `selector?` |
| `browser_evaluate` | Execute JavaScript | `function` |
| `browser_get_text` | Get element text | `element` |
| `browser_fill` | Clear and fill input | `element`, `value` |
| `browser_press_key` | Press keyboard key | `key` |
| `browser_scroll` | Scroll the page | `direction`, `amount` |
| `browser_close` | Close browser | - |

---

## MCP Configuration

### `playwright.config.ts` with MCP

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  // Standard config...
  use: {
    baseURL: process.env.BASE_URL,
  },
  // MCP server is started separately:
  // npx @playwright/mcp@latest --port 3001
});
```

### Starting MCP Server

```bash
# Start MCP server (separate terminal or in CI pre-step)
npx @playwright/mcp@latest --port 3001 --headless

# With specific browser
npx @playwright/mcp@latest --browser chromium --port 3001

# With custom viewport
npx @playwright/mcp@latest --viewport-size 1280,720 --port 3001
```

---

## Agent-Driven Test Pattern

### Using MCP in a Cucumber Step

```typescript
// step-definitions/agent/agentSteps.ts
import { Given, When, Then } from '@cucumber/cucumber';
import { CustomWorld } from '@types/world';
import { PlaywrightMcpAgent } from '@utils/PlaywrightMcpAgent';

let agent: PlaywrightMcpAgent;

Before(async function (this: CustomWorld) {
  agent = new PlaywrightMcpAgent({
    serverUrl: process.env.MCP_SERVER_URL || 'http://localhost:3001/mcp',
  });
});

When('the agent navigates to the login page', async function (this: CustomWorld) {
  await agent.navigate(`${process.env.BASE_URL}/login`);
  await agent.screenshot('login-page');
});

When('the agent logs in as {string}', async function (
  this: CustomWorld,
  userType: string
) {
  const creds = this.getCredentials(userType as any);
  
  await agent.fill('email input field', creds.email);
  await agent.fill('password input field', creds.password);
  await agent.click('sign in button');
});

Then('the agent should see the dashboard', async function (this: CustomWorld) {
  await agent.waitFor('dashboard heading');
  await agent.screenshot('dashboard');
});
```

### PlaywrightMcpAgent Utility Class

```typescript
// src/utils/PlaywrightMcpAgent.ts
import { expect } from '@playwright/test';

interface McpConfig {
  serverUrl: string;
}

interface McpToolCall {
  tool: string;
  arguments: Record<string, unknown>;
}

/**
 * Wrapper around Playwright MCP server for structured agent interactions
 */
export class PlaywrightMcpAgent {
  private readonly serverUrl: string;

  constructor(config: McpConfig) {
    this.serverUrl = config.serverUrl;
  }

  private async callTool(toolCall: McpToolCall): Promise<unknown> {
    const response = await fetch(this.serverUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        jsonrpc: '2.0',
        id: Date.now(),
        method: 'tools/call',
        params: {
          name: toolCall.tool,
          arguments: toolCall.arguments,
        },
      }),
    });

    if (!response.ok) {
      throw new Error(`MCP tool call failed: ${response.statusText}`);
    }

    const result = await response.json();
    if (result.error) {
      throw new Error(`MCP error: ${result.error.message}`);
    }
    return result.result;
  }

  async navigate(url: string): Promise<void> {
    await this.callTool({ tool: 'browser_navigate', arguments: { url } });
  }

  async click(elementDescription: string): Promise<void> {
    await this.callTool({ 
      tool: 'browser_click', 
      arguments: { element: elementDescription } 
    });
  }

  async fill(elementDescription: string, value: string): Promise<void> {
    await this.callTool({ 
      tool: 'browser_fill', 
      arguments: { element: elementDescription, value } 
    });
  }

  async type(elementDescription: string, text: string): Promise<void> {
    await this.callTool({ 
      tool: 'browser_type', 
      arguments: { element: elementDescription, text } 
    });
  }

  async screenshot(name?: string): Promise<void> {
    await this.callTool({ 
      tool: 'browser_screenshot', 
      arguments: name ? { name } : {} 
    });
  }

  async waitFor(condition: string): Promise<void> {
    await this.callTool({ 
      tool: 'browser_wait_for', 
      arguments: { text: condition } 
    });
  }

  async getText(elementDescription: string): Promise<string> {
    const result = await this.callTool({ 
      tool: 'browser_get_text', 
      arguments: { element: elementDescription } 
    });
    return (result as { text: string }).text;
  }

  async evaluate(jsFunction: string): Promise<unknown> {
    return this.callTool({ 
      tool: 'browser_evaluate', 
      arguments: { function: jsFunction } 
    });
  }

  async pressKey(key: string): Promise<void> {
    await this.callTool({ tool: 'browser_press_key', arguments: { key } });
  }

  async scroll(direction: 'up' | 'down' | 'left' | 'right', amount = 3): Promise<void> {
    await this.callTool({ 
      tool: 'browser_scroll', 
      arguments: { direction, amount } 
    });
  }
}
```

---

## Self-Healing Locator Strategy

When using MCP agents, describe elements in natural language for resilience:

```typescript
// FRAGILE — breaks on DOM changes
await agent.click('#login-form > div:nth-child(2) > button');

// RESILIENT — describes intent, not structure
await agent.click('the primary "Sign In" submit button');
await agent.fill('the email address input field', 'user@example.com');
await agent.waitFor('the success notification toast');
```

---

## Hybrid Testing Pattern

Combine MCP agents with direct Playwright for best results:

```typescript
// Use direct Playwright for setup (fast, reliable)
// Use MCP agent for the main test flow (natural, resilient)

Before(async function (this: CustomWorld) {
  // Fast API-level setup
  const api = new ApiHelper(this.page.request);
  this.testData.userId = await api.createTestUser();

  // Direct Playwright for authentication shortcut (skip login UI)
  await this.context.addCookies([{
    name: 'auth_token',
    value: this.testData.authToken,
    domain: 'localhost',
    path: '/',
  }]);
});

// Then use MCP agent for the actual test scenarios
When('the agent creates a new product listing', async function (this: CustomWorld) {
  await agent.navigate('/products/new');
  await agent.fill('product name field', 'Test Widget');
  await agent.fill('price field', '29.99');
  await agent.click('the "Publish" button');
  await agent.waitFor('success confirmation message');
});
```

---

## CI/CD MCP Integration

```yaml
# .github/workflows/playwright.yml addition for MCP
- name: Start Playwright MCP Server
  run: |
    npx @playwright/mcp@latest --port 3001 --headless &
    echo "MCP_PID=$!" >> $GITHUB_ENV
    sleep 3  # Wait for server to start

- name: Run agent tests
  run: npm run test:agent
  env:
    MCP_SERVER_URL: http://localhost:3001/mcp

- name: Stop MCP Server  
  if: always()
  run: kill ${{ env.MCP_PID }} || true
```
