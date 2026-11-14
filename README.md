# csabapiller.com

Source of my personal page. Plain HTML and CSS, with no build step.

- `site/`: the website itself (`index.html`, `style.css`)
- `.github/workflows/deploy.yml`: deploys `site/` to the server on every push to `main`
- `docs/setup-guide.md`: how to build the server from scratch

## Run locally
Open `site/index.html` in a browser.

## Hosting
A Hetzner Cloud VPS (Ubuntu 26.04 LTS) running nginx, with HTTPS from Let's Encrypt.
`csabapiller.com` is the main address; `www.csabapiller.com` and `csabapiller.hu` redirect to it.

## Deployment
GitHub Actions copies `site/` to the server with rsync, as a `deploy` user whose SSH key
can only run rsync into the web root. Files removed from `site/` are removed from the server too.
