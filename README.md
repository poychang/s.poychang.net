# Poy Chang Short URL App

A lightweight short URL service hosted on GitHub Pages. Add memorable paths that redirect visitors to destination URLs—no server or build step required.

## How it works

Short URL mappings are stored in [`routes.json`](routes.json) as key-value pairs. When a visitor opens `https://s.poychang.net/<key>`, GitHub Pages serves `404.html`, which looks up the key and redirects the browser to its destination. If no matching key exists, the visitor is sent to the home page.

For example, this mapping:

```json
{
  "github": "https://github.com/poychang"
}
```

creates the short URL `https://s.poychang.net/github`. Keys are case-sensitive.

## Add or update a short URL

1. Add a new key-value pair to `routes.json`, or edit an existing pair. Use an absolute destination URL, preferably with `https://`.
2. Commit and push the change to the branch configured for GitHub Pages.
3. GitHub Pages publishes the updated links.

## Project structure

- `index.html` — home page with a searchable list of short URLs.
- `404.html` — looks up requested paths in `routes.json` and redirects to their destinations.
- `routes.json` — short URL keys and destination URLs.
- `meta.json` — optional Open Graph descriptions for short URLs.
- `CNAME` — custom domain configuration.
