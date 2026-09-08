# Self-Healing Reference

Full implementation of `SelfHealingLocator`, `LocatorRegistry`, and their integration into `BasePage`.

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
    // Registry key combines the page path and element name for scoped caching
    this.registryKey = `${this.getPageKey()}.${elementName.replace(/\s+/g, '_')}`;
  }

  /**
   * Resolves the element by trying each strategy in order.
   * If LocatorRegistry has a cached winning strategy, that is tried first.
   * Returns the first Playwright Locator that is visible within the timeout.
   */
  async resolve(): Promise<Locator> {
    const timeout = this.options.strategyTimeout ?? 3_000;
    const persist = this.options.persistWinner ?? true;

    // Build the ordered list: cached winner first (if any), then original order
    const orderedStrategies = this.getOrderedStrategies();

    for (let i = 0; i < orderedStrategies.length; i++) {
      const { strategy: strat, originalIndex } = orderedStrategies[i];
      try {
        const locator = this.buildLocator(strat);
        await locator.waitFor({ state: 'attached', timeout });

        // Success — log if this was a healing event (not the primary strategy)
        if (originalIndex > 0) {
          await this.recordHealing(strat, originalIndex, persist);
        }
        return locator;
      } catch {
        // Strategy failed — continue to the next one silently
      }
    }

    // All strategies exhausted
    throw new (SelfHealingLocator.NoStrategyResolvedError)(
      this.elementName,
      this.strategies,
      this.page.url()
    );
  }

  /**
   * Returns strategies sorted so the cached winner (if any) is tried first.
   */
  private getOrderedStrategies(): Array<{ strategy: LocatorStrategy; originalIndex: number }> {
    const cachedIndex = this.registry.getWinningIndex(this.registryKey);
    const indexed = this.strategies.map((s, i) => ({ strategy: s, originalIndex: i }));

    if (cachedIndex !== null && cachedIndex > 0 && cachedIndex < this.strategies.length) {
      // Move the cached winner to front, keep the rest in original order
      const winner = indexed.splice(cachedIndex, 1)[0];
      return [winner, ...indexed];
    }
    return indexed;
  }

  /**
   * Builds a Playwright Locator from a strategy descriptor.
   */
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
    // Emit a console warning so it's visible in CI logs
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

  /** Returns the index of the previously winning strategy, or null if no record. */
  getWinningIndex(key: string): number | null {
    return this.data[key]?.winningStrategyIndex ?? null;
  }

  /** Returns the previously winning strategy descriptor, or null. */
  getWinningStrategy(key: string): LocatorStrategy | null {
    return this.data[key]?.winningStrategy ?? null;
  }

  /** Records a healing event for a given key. */
  recordWin(key: string, entry: Omit<RegistryEntry, 'healCount'>): void {
    const existing = this.data[key];
    this.data[key] = {
      ...entry,
      healCount: (existing?.healCount ?? 0) + 1,
    };
  }

  /** Persists the current registry data to disk (async). */
  async save(): Promise<void> {
    const dir = path.dirname(this.registryPath);
    if (!fs.existsSync(dir)) {
      fs.mkdirSync(dir, { recursive: true });
    }
    fs.writeFileSync(this.registryPath, JSON.stringify(this.data, null, 2), 'utf-8');
  }

  /** Returns a summary of all healed elements (for reporting). */
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
      // Registry doesn't exist yet or is corrupt — start fresh
      this.data = {};
    }
  }
}
```

---

## Updated `src/pages/BasePage.ts` (with self-healing integration)

```typescript
import { Page, Locator, expect } from '@playwright/test';
import { LocatorRegistry } from '@utils/LocatorRegistry';

export abstract class BasePage {
  protected readonly page: Page;
  protected readonly registry: LocatorRegistry;

  constructor(page: Page) {
    this.page = page;
    this.registry = LocatorRegistry.getInstance();
  }

  async navigate(path: string = ''): Promise<void> {
    const baseUrl = process.env.BASE_URL || 'http://localhost:3000';
    await this.page.goto(`${baseUrl}${path}`);
    await this.waitForPageLoad();
  }

  async waitForPageLoad(): Promise<void> {
    await this.page.waitForLoadState('domcontentloaded');
    await this.page.waitForLoadState('networkidle');
  }

  async waitForVisible(locator: Locator, timeout = 10_000): Promise<void> {
    await locator.waitFor({ state: 'visible', timeout });
  }

  async waitForHidden(locator: Locator, timeout = 10_000): Promise<void> {
    await locator.waitFor({ state: 'hidden', timeout });
  }

  async getCurrentUrl(): Promise<string> {
    return this.page.url();
  }

  async expectUrlContains(path: string): Promise<void> {
    await expect(this.page).toHaveURL(new RegExp(path));
  }

  async takeScreenshot(name: string): Promise<Buffer> {
    return this.page.screenshot({
      path: `reports/screenshots/${name}-${Date.now()}.png`,
      fullPage: true,
    });
  }

  /**
   * Prints a healing summary to the console after a test completes.
   * Called from the Cucumber After hook.
   */
  printHealingSummary(): void {
    const report = this.registry.getHealingReport();
    if (report.length === 0) return;

    console.log('\n🔧 Self-Healing Summary:');
    for (const { key, entry } of report) {
      console.log(
        `   ${key}\n` +
        `     Healed at:  ${entry.healedAt}\n` +
        `     Page:       ${entry.page}\n` +
        `     Healed via: strategy="${entry.winningStrategy.strategy}"\n` +
        `     Heal count: ${entry.healCount ?? 1}`
      );
    }
    console.log(
      '\n   Recommendation: update your application\'s test IDs to match\n' +
      '   the primary strategies (data-testid) for long-term stability.\n'
    );
  }
}
```

---

## Updated `src/hooks/hooks.ts` (healing report in After hook)

```typescript
import { Before, After, BeforeAll, AfterAll, Status } from '@cucumber/cucumber';
import { chromium, Browser, BrowserContext, Page } from '@playwright/test';
import { CustomWorld } from '../types/world';
import { LocatorRegistry } from '../utils/LocatorRegistry';

let browser: Browser;

BeforeAll(async function () {
  browser = await chromium.launch({
    headless: process.env.HEADLESS !== 'false',
    slowMo: process.env.SLOW_MO ? parseInt(process.env.SLOW_MO) : 0,
  });
});

AfterAll(async function () {
  await browser?.close();
});

Before(async function (this: CustomWorld, scenario) {
  this.context = await browser.newContext({
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    viewport: { width: 1280, height: 720 },
  });
  this.page = await this.context.newPage();
  this.scenarioName = scenario.pickle.name;
  console.log(`\n▶ Starting: ${this.scenarioName}`);
});

After(async function (this: CustomWorld, scenario) {
  if (scenario.result?.status === Status.FAILED) {
    const screenshot = await this.page?.screenshot({ fullPage: true });
    if (screenshot) this.attach(screenshot, 'image/png');
  }

  // Flush the locator registry to disk after every scenario
  await LocatorRegistry.getInstance().save();

  await this.page?.close();
  await this.context?.close();

  const icon = scenario.result?.status === Status.PASSED ? '✅' : '❌';
  console.log(`${icon} Finished: ${this.scenarioName}`);
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

## package.json additions for self-healing

```json
{
  "scripts": {
    "healing:report": "node scripts/print-healing-report.js",
    "healing:clear": "rm -f reports/locator-registry.json"
  }
}
```
