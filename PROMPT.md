# Prompts

## 2026-03-05: Initial Implementation

Implement the GitHub Pages site for Newsletter Archive. Python build script + client-side JS approach. Build script reads from ../newsletters/, parses YAML front matter, copies HTML email files, generates JSON manifest. Three pages: homepage (card grid), newsletter listing (date-sorted emails), email viewer (iframe). Modern minimal UI with dark mode. GitHub Actions deployment.

## 2026-03-05: Fix GitHub Pages Deployment Error

Fix GitHub Pages deployment error by adding `enablement: true` to `actions/configure-pages@v5` in the workflow file, so Pages is auto-enabled on the repository.
