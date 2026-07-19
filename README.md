# cvitals

`cvitals` is a Hugo CV site that uses the private `cvnewtheme` repository as a Git submodule at `themes/cvnewtheme`. Both repositories use the `dev` branch and remain private on GitHub.

> Private GitHub repositories do not make the deployed Netlify site private. Configure Netlify access control separately if the website itself must be restricted.

## Requirements

- Git and the [GitHub CLI](https://cli.github.com/)
- A GitHub account with SSH access configured
- Hugo Extended `0.164.0`

## Create the private repositories

These commands assume both local repositories are initialized, committed, and use the `dev` branch.

Authenticate with GitHub and set your username:

```bash
gh auth login --web --git-protocol ssh
gh auth status
github_username="<github_username>"
```

Publish `cvnewtheme` first:

```bash
cd /path/to/cvnewtheme
git switch dev
gh repo create "$github_username/cvnewtheme" --private --source=. --remote=origin --push
gh repo edit "$github_username/cvnewtheme" --default-branch dev
```

Create the private `cvitals` repository with the GitHub CLI:

```bash
cd /path/to/cvitals
git switch dev
gh repo create "$github_username/cvitals" --private --source=. --remote=origin --push
gh repo edit "$github_username/cvitals" --default-branch dev
```

## Add the private theme submodule

From the `cvitals` repository, add the theme using its SSH repository URL:

```bash
git submodule add <repository_url> themes/cvnewtheme
```

Use `git@github.com:<github_username>/cvnewtheme` as `<repository_url>`. The destination must be `themes/cvnewtheme` so it matches Hugo's theme name and the submodule path. Update `.gitmodules` to:

```ini
[submodule "themes/cvnewtheme"]
	path = themes/cvnewtheme
	url = git@github.com:<github_username>/cvnewtheme
	branch = dev
```

Replace `<github_username>`, then synchronize, commit, and push the submodule configuration and pinned theme commit:

```bash
git submodule sync -- themes/cvnewtheme
git add .gitmodules themes/cvnewtheme
git commit -m "Add cvnewtheme submodule"
git push origin dev
```

Clone the complete private site with:

```bash
git clone --branch dev --recurse-submodules \
  git@github.com:<github_username>/cvitals
```

For an existing clone, initialize the theme with:

```bash
git submodule sync -- themes/cvnewtheme
git submodule update --init -- themes/cvnewtheme
```

## Run locally

Replace the placeholder values in `config.toml` and `static/`, then run:

```bash
hugo server --panicOnWarning
```

Create a production build with:

```bash
hugo --gc --minify --panicOnWarning
```

Keep `params.seo.robots = "noindex, nofollow"` until the content and production URL are ready.

## Deploy the private repositories with Netlify

Netlify accesses the private `cvitals` repository through its GitHub App. It accesses the private `cvnewtheme` submodule through a read-only deploy key.

1. In Netlify, select **Add new project** → **Import an existing project** → **GitHub**.
2. Authorize the Netlify GitHub App to access the private `cvitals` repository, then select that repository.
3. Set the production branch to `dev`. The committed `netlify.toml` provides:

   - Build command: `hugo --gc --minify --panicOnWarning --baseURL "$URL"`
   - Publish directory: `public`
   - Hugo version: `0.164.0`

4. Create the site. The first build may fail until Netlify can clone the private theme.
5. Open **Project configuration** → **Build & deploy** → **Continuous deployment** → **Deploy key**, then select **Generate public key**.
6. Copy the generated public key. In the private GitHub `cvnewtheme` repository, open **Settings** → **Deploy keys** → **Add deploy key**.
7. Paste the key and leave **Allow write access** unchecked. Add this key only to `cvnewtheme`, not to `cvitals`.
8. Retry the Netlify deploy and confirm that both the site repository and `themes/cvnewtheme` are cloned successfully.

Never commit an SSH private key, access token, or Netlify deploy key to either repository.

## Update the theme

Fully verify, commit, and push theme changes to `cvnewtheme` first. Record the
exact 40-character commit SHA that passed the theme and adjacent-consumer
checks. Pin `cvitals` to that verified commit rather than resolving a moving
branch tip:

```bash
cd /path/to/cvitals
theme_sha="<verified-40-character-cvnewtheme-commit-sha>"
git submodule update --init -- themes/cvnewtheme
git -C themes/cvnewtheme fetch origin "$theme_sha"
git -C themes/cvnewtheme checkout --detach "$theme_sha"
test "$(git -C themes/cvnewtheme rev-parse HEAD)" = "$theme_sha"

hugo --renderToMemory \
  --cacheDir /tmp/cvitals-submodule-hugo-cache \
  --gc --ignoreCache --minify --panicOnWarning --noBuildLock

git add themes/cvnewtheme
git diff --cached --submodule=log -- themes/cvnewtheme
git commit -m "Update cvnewtheme"
git push origin dev
```

The detached checkout is intentional: the superproject gitlink records the
verified commit exactly. Netlify redeploys only after `cvitals` records and
pushes that gitlink.

## Preview

[Desktop preview](docs/screenshots/cvitals-desktop-1440x1200.png) · [Mobile preview](docs/screenshots/cvitals-mobile-390x844.png)
