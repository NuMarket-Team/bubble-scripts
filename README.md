# NuMarket Bubble Scripts

This repo stores all JavaScript, CSS, and HTML snippets embedded in our Bubble app.

## How it works

Every script is served live by `scripts.numarket.info`. You never need to paste code again — just paste the URL once.

**Staging URL:** `https://scripts.numarket.info/{script-name}.js?env=staging`
**Prod URL:** `https://scripts.numarket.info/{script-name}.js?env=prod`

## Managing scripts

Use Claude. Just say:
- "Show me all our Bubble scripts"
- "Create a new script that tracks form submissions"
- "Update checkout-tracker to also log the order total"
- "Push checkout-tracker to prod"
- "What changed on campaign-form-validator last week?"

## Branches

- `main` — production (only updated via PR)
- `staging` — staging (Claude commits here directly)

## Folder structure

```
scripts/
  {script-name}/
    index.js        # (or .css / .html)
    meta.json       # name, description, env_vars, where_used
```
