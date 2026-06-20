# 3DGS Action Consequence Verification Project Page

This repository hosts a single-page static project website for a 3DGS robot
digital twin research project. It contains plain HTML, CSS, videos, papers, and
experiment notes. No React, Vite, Node, or other build tool is required.

## Local Preview

Run a static server from the repository root:

```bash
python3 -m http.server 8000
```

Then open:

```text
http://127.0.0.1:8000/
```

## GitHub Pages Deployment

The site is deployed with GitHub Actions using the workflow in:

```text
.github/workflows/pages.yml
```

After pushing to the `main` branch, GitHub Actions uploads the repository root
as a static Pages artifact and deploys it to GitHub Pages.

In the GitHub repository settings, set Pages to use GitHub Actions.

## Deployment URL

Replace the placeholders with your GitHub username and repository name:

```text
https://<github-username>.github.io/<repo-name>/
```
