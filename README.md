# Immotus landing page

A single, simple page for immotus.app — no build step, just one HTML file.

## How to make it live (GitHub Pages)

1. On GitHub, go to this repo's **Settings → Pages**.
2. Under "Build and deployment", set **Source** to "Deploy from a branch".
3. Pick this branch and folder `/ (root)`, then save.
4. Under "Custom domain", enter `immotus.app` and save (the `CNAME` file in this repo already has this set, so GitHub should pick it up automatically).
5. At your domain registrar (wherever you bought immotus.app), point the domain's DNS to GitHub Pages:
   - Add an **A record** for `@` pointing to each of: `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
   - Add a **CNAME record** for `www` pointing to `<your-github-username>.github.io`
6. It can take up to a few hours for DNS to update. Once it does, immotus.app will show this page.

## Editing the page

Everything is in `index.html` — the text you'll want to change is near the bottom, inside the `<div class="card">` section (the title, the "Coming soon" tag, the tagline, and the contact email). No coding tools needed — you can even edit it directly on GitHub.com by clicking the file and hitting the pencil icon.
