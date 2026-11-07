# csabapiller.com

Source of my personal page. Plain HTML and CSS, with no build step.

- `index.html`: the page
- `style.css`: styling
- `docs/server-setup.md`: how the server is set up, step by step, so it can be rebuilt

## Run locally
Open `index.html` in a browser.

## Hosting
A Hetzner Cloud VPS (Ubuntu 24.04 LTS) running nginx, with HTTPS from Let's Encrypt.
`csabapiller.com` is the main address; `www.csabapiller.com` and `csabapiller.hu` redirect to it.
