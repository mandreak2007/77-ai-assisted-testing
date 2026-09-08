# Self-Healing Reference (CLI Edition)

Full implementation of `SelfHealingLocator`, `LocatorRegistry`, and their integration with the Playwright CLI test runner. The implementation is identical to the MCP edition — the same classes work with any Playwright `Page` instance regardless of how tests are run.

---

## `src/utils/SelfHealingLocator.ts`

```typescript
import { Page, Locator } from '@playwright/test';
import { LocatorRegistry } from './LocatorRegistry';

export type LocatorStrategy =
  | { strategy: 'testid';      value: string }
  | { strategy: 'role';        value: string; options?: { name?: string | RegExp; exact?: boolean } }
  | { strategy: 'label';       value: string | RegExp }
  | { strategy: 'placeholder'; value: string | RegExp }
  | { strategy: 'text';        value: string | RegExp }
  | { strategy: 'css';         value: string }
  | { strategy: 'xpath';       value: string };

export interface SelfHealingLocatorOptions {
  /** Timeout in ms for each strategy attempt. Default: 3000 */
  strategyTimeout?: number;
  /** Whether to persist the winning strategy to disk. Default: true */
  persistWinner?: boolean;
}

/**
 * A resilient locator that tries multiple strategies in order until one resolves.
 * The winning strategy is cached in LocatorRegistry and promoted to first position
 * on subsequent runs — so healing is cumulative and gets faster over time.
 */
export class SelfHealingLocator {
  private readonly registry: LocatorRegistry;
  private readonly registryKey: string;

  static readonly NoStrategyResolvedError = class extends Error {
    constructor(elementName: string, strategies: LocatorStrategy[], pageUrl: string) {
      const tried = strategies
        .map((s, i) => `  [${i + 1}] strategy="${s.strategy}" value="${JSON.stringify((s as any).value)}"`)
        .join('\n');
      super(
        `SelfHealingLocator: No strategy resolved for element "${elementName}" on ${pageUrl}.\n` +
        `Tried ${strategies.length} strategies:\n${tried}\n` +
        `Add new fallback strategies to the Page Object or fix the element in the application.`
      );
      this.name = 'NoStrategyResolvedError';
    }
  };

  constructor(
    private readonly page: Page,
    private readonly elementName: string,
    private readonly strategies: LocatorStrategy[],
    private readonly options: SelfHealingLocatorOptions = {}
  ) {
    if (strategies.length === 0) {
      throw new Error(`SelfHealingLocator "${elementName}": at least one strategy is required.`);
    }
    this.registry = LocatorRegistry.getInstance();
    this.registryKey = `${this.getPageKey()}.${elementName.replace(/\s+/g, '_')}`;
  }

  /**
   * Resolves the element by trying each strategy in order.
   * If LocatorRegistry has a cached winning strategy, that is tried first.
   * Returns the first Playwright Locator that is attached within the timeout.
   */
  async resolve(): Promise<Locator> {
    const timeout = this.options.strategyTimeout ?? 3_000;
    const persist = this.options.persistWinner ?? true;

    const orderedStrategies = this.getOrderedStrategies();

    for (let i = 0; i < orderedStrategies.length; i++) {
      const { strategy: strat, originalIndex } = orderedStrategies[i];
      try {
        const locator = this.buildLocator(strat);
        await locator.waitFor({ state: 'attached', timeout });

        if (originalIndex > 0) {
          await this.recordHealing(strat, originalIndex, persist);
        }
        return locator;
      } catch {
        // Strategy failed — continue to next silently
      }
    }

    throw new (SelfHealingLocator.NoStrategyResolvedError)(
      this.elementName,
      this.strategies,
      this.page.url()
    );
  }

  private getOrderedStrategies(): Array<{ strategy: LocatorStrategy; originalIndex: number }> {
    const cachedIndex = this.registry.getWinningIndex(this.registryKey);
    const indexed = this.strategies.map((s, i) => ({ strategy: s, originalIndex: i }));

    if (cachedIndex !== null && cachedIndex > 0 && cachedIndex < this.strategies.length) {
      const winner = indexed.splice(cachedIndex, 1)[0];
      return [winner, ...indexed];
    }
    return indexed;
  }

  private buildLocator(strat: LocatorStrategy): Locator {
    switch (strat.strategy) {
      case 'testid':
        return this.page.getByTestId(strat.value);
      case 'role':
        return this.page.getByRole(strat.value as any, strat.options);
      case 'label':
        return this.page.getByLabel(strat.value);
      case 'placeholder':
        return this.page.getByPlaceholder(strat.value);
      case 'text':
        return this.page.getByText(strat.value, { exact: false });
      case 'css':
        return this.page.locator(strat.value);
      case 'xpath':
        return this.page.locator(strat.value);
      default:
        throw new Error(`Unknown locator strategy: ${(strat as any).strategy}`);
    }
  }

  private getPageKey(): string {
    try {
      return new URL(this.page.url()).pathname.replace(/\//g, '_').replace(/^_/, '') || 'root';
    } catch {
      return 'unknown_page';
    }
  }

  private async recordHealing(
    winningStrategy: LocatorStrategy,
    winningIndex: number,
    persist: boolean
  ): Promise<void> {
    const previousWinner = this.registry.getWinningStrategy(this.registryKey);
    this.registry.recordWin(this.registryKey, {
      page: this.page.url(),
      winningStrategyIndex: winningIndex,
      winningStrategy,
      previousWinner: previousWinner ?? this.strategies[0],
      healedAt: new Date().toISOString(),
    });
    if (persist) {
      await this.registry.save();
    }
    console.warn(
      `\n🔧 [SelfHeal] "${this.elementName}" on ${this.page.url()}\n` +
      `   Primary strategy failed. Healed via: strategy="${winningStrategy.strategy}" ` +
      `value="${JSON.stringify((winningStrategy as any).value)}"\n`
    );
  }
}
```

---

## `src/utils/LocatorRegistry.ts`

```typescript
import * as fs from 'fs';
import * as path from 'path';
import { LocatorStrategy } from './SelfHealingLocator';

interface RegistryEntry {
  page: string;
  winningStrategyIndex: number;
  winningStrategy: LocatorStrategy;
  previousWinner: LocatorStrategy;
  healedAt: string;
  healCount?: number;
}

type RegistryData = Record<string, RegistryEntry>;

/**
 * Singleton that persists the winning locator strategy for each element to disk.
 * On next run, the engine tries the cached winner first — avoiding re-healing.
 */
export class LocatorRegistry {
  private static instance: LocatorRegistry | null = null;
  private data: RegistryData = {};
  private readonly registryPath: string;

  private constructor() {
    this.registryPath = path.resolve(
      process.env.LOCATOR_REGISTRY_PATH ?? 'reports/locator-registry.json'
    );
    this.load();
  }

  static getInstance(): LocatorRegistry {
    if (!LocatorRegistry.instance) {
      LocatorRegistry.instance = new LocatorRegistry();
    }
    return LocatorRegistry.instance;
  }

  getWinningIndex(key: string): number | null {
    return this.data[key]?.winningStrategyIndex ?? null;
  }

  getWinningStrategy(key: string): LocatorStrategy | null {
    return this.data[key]?.winningStrategy ?? null;
  }

  recordWin(key: string, entry: Omit<RegistryEntry, 'healCount'>): void {
    const existing = this.data[key];
    this.data[key] = {
      ...entry,
      healCount: (existing?.healCount ?? 0) + 1,
    };
  }

  async save(): Promise<void> {
    const dir = path.dirname(this.registryPath);
    if (!fs.existsSync(dir)) {
      fs.mkdirSync(dir, { recursive: true });
    }
    fs.writeFileSync(this.registryPath, JSON.stringify(this.data, null, 2), 'utf-8');
  }

  getHealingReport(): Array<{ key: string; entry: RegistryEntry }> {
    return Object.entries(this.data).map(([key, entry]) => ({ key, entry }));
  }

  private load(): void {
    try {
      if (fs.existsSync(this.registryPath)) {
        const raw = fs.readFileSync(this.registryPath, 'utf-8');
        this.data = JSON.parse(raw) as RegistryData;
      }
    } catch {
      this.data = {};
    }
  }
}
```

---

## Integration in the Playwright CLI Test Runner

### Flushing the Registry After Each Test (via Fixtures)

The `src/fixtures/test-fixtures.ts` fixture overrides the `page` fixture to flush the registry after every test:

```typescript
// src/fixtures/test-fixtures.ts (relevant excerpt)
import { test as base } from '@playwright/test';
import { LocatorRegistry } from '@utils/LocatorRegistry';

export const test = base.extend({
  page: async ({ page }, use) => {
    await use(page);
    // Flush the registry to disk after every test — cumulative healing
    await LocatorRegistry.getInstance().save();
  },
  // ... other fixtures
});
```

This means every test's healing events are persisted to `reports/locator-registry.json` automatically, with no manual hook setup required (unlike the Cucumber MCP approach which uses `src/hooks/hooks.ts`).

---

### Printing a Healing Summary After a Test Suite

Add a global teardown to print the full healing report after all tests complete:

```typescript
// global-teardown.ts
import { LocatorRegistry } from './src/utils/LocatorRegistry';

async function globalTeardown(): Promise<void> {
  const registry = LocatorRegistry.getInstance();
  const report = registry.getHealingReport();

  if (report.length === 0) {
    console.log('\n✅ No self-healing events recorded this run.\n');
    return;
  }

  console.log(`\n🔧 Self-Healing Summary (${report.length} elements healed this session):\n`);
  for (const { key, entry } of report) {
    console.log(
      `   ${key}\n` +
      `     Page:       ${entry.page}\n` +
      `     Healed via: strategy="${entry.winningStrategy.strategy}" ` +
      `value="${JSON.stringify((entry.winningStrategy as any).value)}"\n` +
      `     Heal count: ${entry.healCount ?? 1}\n`
    );
  }
  console.log(
    '   Recommendation: add data-testid attributes to healed elements\n' +
    '   in your application for long-term stability.\n'
  );
}

export default globalTeardown;
```

Register in `playwright.config.ts`:
```typescript
export default defineConfig({
  globalTeardown: './global-teardown.ts',
  // ...
});
```

---

## Locator Registry File Example

`reports/locator-registry.json` after a healing event:

```json
{
  "login_emailInput": {
    "page": "http://localhost:3000/login",
    "winningStrategyIndex": 2,
    "winningStrategy": {
      "strategy": "placeholder",
      "value": "/email/i"
    },
    "previousWinner": {
      "strategy": "testid",
      "value": "email-input"
    },
    "healedAt": "2025-01-15T10:23:44.123Z",
    "healCount": 1
  },
  "checkout_submitButton": {
    "page": "http://localhost:3000/checkout",
    "winningStrategyIndex": 1,
    "winningStrategy": {
      "strategy": "role",
      "value": "button",
      "options": { "name": "/place order/i" }
    },
    "previousWinner": {
      "strategy": "testid",
      "value": "checkout-submit"
    },
    "healedAt": "2025-01-15T10:24:01.456Z",
    "healCount": 2
  }
}
```

---

## Locator Fallback Priority (Always Generate in This Order)

| Priority | Strategy | Example |
|---|---|---|
| 1 | `data-testid` | `{ strategy: 'testid', value: 'email-input' }` |
| 2 | ARIA role + name | `{ strategy: 'role', value: 'button', options: { name: /sign in/i } }` |
| 3 | Label | `{ strategy: 'label', value: /email/i }` |
| 4 | Placeholder | `{ strategy: 'placeholder', value: /email/i }` |
| 5 | Visible text | `{ strategy: 'text', value: /sign in/i }` |
| 6 | CSS selector | `{ strategy: 'css', value: 'input[type="email"]' }` |
| 7 | XPath (last resort) | `{ strategy: 'xpath', value: '//input[@name="email"]' }` |

**Never skip priorities.** Always provide the full chain so the healing engine has maximum fallback depth. A locator with only XPath is as brittle as a plain Playwright locator — the value comes from the full ranked chain.

---

## Self-Healing vs CLI Trace Viewer

Self-healing and Playwright's built-in trace viewer are complementary:

- **Self-healing** → fixes locator failures automatically at runtime, persists the winning strategy
- **Trace viewer** → provides visual step-through of a test run for manual debugging (`npx playwright show-trace`)

Use both together:
1. Let self-healing handle transient locator issues automatically
2. When self-healing fails (all strategies exhausted), open the trace file to visually inspect the DOM state at failure:
   ```bash
   npx playwright show-trace test-results/<test-name>/trace.zip
   ```
3. Identify the correct new locator from the trace viewer's DOM snapshot
4. Add it to the Page Object's `SelfHealingLocator` strategy chain
