# Running locally

This site uses Jekyll with the Chirpy theme. Run these commands from the repository
root with [mise](https://mise.jdx.dev/) installed.

## First-time setup

Install Ruby and the site's dependencies:

```bash
MISE_RUBY_COMPILE=false mise install ruby@3.3
mise exec ruby@3.3 -- bundle config set --local path vendor/bundle
mise exec ruby@3.3 -- bundle install
```

This uses a precompiled Ruby runtime and stores gems in the ignored `vendor/bundle`
directory. No global Ruby activation is needed.

## Start the preview

```bash
mise exec ruby@3.3 -- bundle exec jekyll serve --host 127.0.0.1 --port 4000 --livereload --drafts --future
```

Open <http://localhost:4000/> in your browser.

The preview includes drafts and future-dated posts. Saving a post automatically
rebuilds the site and refreshes the browser.

Keep the command running while previewing. Press `Ctrl+C` to stop it. If you change
`_config.yml`, stop and restart the preview to apply the configuration changes.
