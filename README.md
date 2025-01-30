# Open-Source AI Model Reference and Usage Calculator

This document describes a static site that consolidates data from multiple AI models (for example, Amazon’s Nova, DeepSeek’s V3 and R1, Alibaba’s Qwen 2.5, Anthropic’s Claude 3.5 Sonnet, and Google’s Gemini 1.5 Pro). Each model’s data (pricing, version details, usage guidance, and documentation links) is stored in JSON or Markdown files. Astro generates the site from these files, while GitHub Actions automates both data-fetching and site deployment. Bun serves as the JavaScript runtime locally and in GitHub Actions for scraping or API calls, as well as for the Astro build process.

## Overview

- **Data Files:** Each AI model has its own file under a `data` directory, containing fields such as pricing, version names, context window sizes, and URLs to official docs.
- **Static Site Generation:** Astro compiles these data files into static pages, ensuring high performance and minimal upkeep.
- **Usage Calculator:** A front-end component can compute approximate costs based on the pricing data.
- **Automation:** GitHub Actions fetches updated data on a set schedule (or manually) using JavaScript scripts run under Bun, then commits changes if any data has been modified. It then rebuilds and deploys the updated site.
- **Open Source:** The repository can be published under a permissive licence and accept community contributions, so the data and features remain current.

## Repository Structure

```
ai-model-pricing-calculator/
├── .github/
│   └── workflows/
│       ├── update-model-data.yml       # Fetches/scrapes data using Bun scripts
│       └── deploy-site.yml             # Builds and deploys the site
├── data/
│   ├── amazon-nova.json
│   ├── deepseek-v3.json
│   ├── deepseek-r1.json
│   ├── alibaba-qwen2.5.json
│   ├── anthropic-claude-3.5-sonnet.json
│   ├── google-gemini-1.5-pro.json
│   └── ... (more models if needed)
├── scripts/
│   └── fetchData.js                    # Example JavaScript file for data scraping
├── src/
│   ├── pages/
│   │   └── index.astro                 # Main landing or overview page
│   ├── components/
│   │   ├── ModelTable.astro
│   │   └── UsageCalculator.astro
│   └── ... (additional components)
├── package.json
├── astro.config.mjs
├── bun.lockb                           # Lockfile for Bun (commit to ensure consistency)
├── README.md
├── CONTRIBUTING.md
└── LICENSE
```

- **`data/`:** Houses the individual JSON or Markdown files for each AI model.
- **`scripts/`:** Contains JavaScript files that handle fetching or scraping from provider APIs or web pages, executed with Bun.
- **`src/`:** The Astro code, including pages (`.astro` files) and reusable components.
- **`bun.lockb`:** Lockfile to keep dependencies consistent across environments.

## GitHub Actions

### 1. Update Model Data Workflow

A file named `update-model-data.yml` manages data-fetching tasks. It installs Bun, runs your JavaScript scripts (for example, `scripts/fetchData.js`), and updates any changed JSON files in the `data` folder.

```yaml
name: Update Model Data

on:
  schedule:
    - cron: "0 2 * * *" # Runs daily at 2 AM UTC
  workflow_dispatch:

jobs:
  data-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: "latest"

      - name: Fetch Model Data
        run: bun run scripts/fetchData.js

      - name: Commit Changes
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add data/*.json
          if git diff --cached --quiet; then
            echo "No changes detected."
          else
            git commit -m "Update model data"
            git push
```

The `fetchData.js` script might scrape various provider endpoints, parse the results, and overwrite the JSON files in `data`.

### 2. Build and Deploy Workflow

A second file, `deploy-site.yml`, listens for pushes to the main branch (including data updates). It checks out the code, installs dependencies, builds the Astro site with Bun, and deploys the resulting static files to GitHub Pages.

```yaml
name: Deploy Site

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: "latest"

      - name: Install Dependencies
        run: bun install

      - name: Build with Astro
        run: bun run astro build

      - name: Deploy to GitHub Pages
        uses: withastro/action@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          dist: dist
```

If you prefer Node for CI, remove the Bun steps and replace the install/build commands with `npm install` and `npx astro build`.

## Astro Configuration

A minimal `astro.config.mjs` might look like this:

```js
import { defineConfig } from "astro/config";

export default defineConfig({
  output: "static",
  // base: '/my-subpath/' if you need a subdirectory deployment
});
```

Always confirm you are using the latest Astro version by running `bun install astro@latest` (or `npm install astro@latest` if you switch to Node).

## Usage Calculator Example

A simple calculator for token-based pricing can be an Astro component. Below is a minimal example that reads a model’s `pricingPer1KTokens` field:

```astro
---
const { model } = Astro.props;
let tokenCount = 0;
function calculateCost() {
  return (tokenCount / 1000) * (model.pricingPer1KTokens || 0);
}
---

<div>
  <h3>Cost Calculator for {model.name}</h3>
  <label>
    Enter tokens:
    <input
      type="number"
      onInput={(e) => tokenCount = parseFloat(e.target.value)}
    />
  </label>
  <button onClick={() => alert(`Estimated Cost: $${calculateCost().toFixed(4)}`)}>
    Calculate
  </button>
</div>
```

You can extend this to account for multiple usage tiers, input vs. output tokens, or region-based pricing.

## Open-Source Collaboration

- **License:** Place a permissive license (MIT or Apache 2.0) in the repository root.
- **Contributing Guide:** A file called `CONTRIBUTING.md` should explain how to add or update model data, run the project locally with Bun, and make pull requests.
- **Community Scripts:** Contributors can share or improve JavaScript scripts for data fetching, ensuring you always have the latest pricing and model versions.

## Future Enhancements

- **Comparison Charts:** Display side-by-side cost comparisons or usage visualizations.
- **Feature Matrices:** Show supported context windows, fine-tuning options, or region availability.
- **Model Release Tracking:** Automate detection of new model versions, prompting updates.
- **JSON Endpoints:** Generate a JSON feed for external services that need real-time pricing data.

By centralizing each AI model’s information in a dedicated file and using Bun-based scripts to update data, this design provides a clean, reliable workflow. Astro’s static output requires no special infrastructure, and GitHub Actions ensures your site’s model data remains current with minimal manual intervention.
