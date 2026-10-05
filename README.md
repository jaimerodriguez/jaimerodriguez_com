# Blog setup: GitHub Pages with Minima v3

How this blog was set up, in case it needs to be repeated.

Minima v3 has never been released as a gem, and GitHub Pages still bundles v2.5.1. The way to get v3 is to load the theme straight from its GitHub repository with `remote_theme`, pinned to a specific commit.

**Tested:** October 4, 2026, with Jekyll 3.10.0 and github-pages 232 (the versions GitHub Pages uses) and Minima commit `4de3223`. The configuration was built locally against those versions, not deployed to GitHub itself.

## 1. Create the repository

1. On GitHub, create a public repository named `USERNAME.github.io` (your GitHub username). The site is served at `https://USERNAME.github.io`.
2. Clone it locally, or add the files below through the GitHub web editor.

If you use any other repository name, the site is served at `https://USERNAME.github.io/REPO`, and you must set `baseurl: "/REPO"` in step 2.

## 2. Add `_config.yml`

```yaml
title: Your Blog Title
description: >-
  One or two sentences about the blog.
url: "https://USERNAME.github.io"
baseurl: ""

author:
  name: Your Name
  email: you@example.com

# Minima v3, pinned to a tested commit
remote_theme: jekyll/minima@4de3223
plugins:
  - jekyll-remote-theme
  - jekyll-feed
  - jekyll-seo-tag

minima:
  skin: auto            # classic | dark | auto | solarized | solarized-light | solarized-dark
  nav_pages:
    - about.md
  show_excerpts: true
  date_format: "%b %-d, %Y"
  social_links:
    - title: GitHub
      icon: github
      url: "https://github.com/USERNAME"
    - title: LinkedIn
      icon: linkedin
      url: "https://www.linkedin.com/in/USERNAME"

# Keep non-site files out of the published site
exclude:
  - README.md
  - Gemfile
  - Gemfile.lock
```

Things to know about this file:

- **Do not add `theme: minima`.** That line loads the bundled v2.5.1, and the build fails with "File to import not found: minima/skins/auto".
- **Keep the commit pin.** The theme's `master` branch is under active development and can introduce breaking changes. Update the commit deliberately when you want newer changes (see "Updating the theme" below).
- **Leave out `email` if you prefer.** It is displayed in the site footer.
- **The `exclude` list** keeps this README out of the published site. Setting `exclude` replaces Jekyll's default list, so `Gemfile` and `Gemfile.lock` are listed explicitly too.

## 3. Add the home page: `index.md`

```markdown
---
layout: home
---
```

## 4. Add an About page: `about.md`

```markdown
---
layout: page
title: About
permalink: /about/
---

Your bio here.
```

This site's About page uses a custom layout instead of `page`; see section 10.

## 5. Add a post

Posts go in a `_posts` folder, and the filename must follow `YYYY-MM-DD-title.md`, for example `_posts/2026-10-04-hello-world.md`:

```markdown
---
layout: post
title: "Hello, world"
date: 2026-10-04 09:00:00 -0700
---

Post content in Markdown.
```

A post dated in the future does not appear until that time passes.

## 6. Turn on GitHub Pages

1. Commit and push the files to the `main` branch.
2. In the repository, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to "Deploy from a branch", choose `main` and `/ (root)`, then save.
4. Watch the **Actions** tab. When the "pages build and deployment" run finishes, the site is live, usually within a minute or two.

## 7. Optional: customize

These files in the repository override the theme without copying it:

| File | Purpose |
|---|---|
| `_sass/minima/custom-styles.scss` | Your own CSS rules |
| `_sass/minima/custom-variables.scss` | Theme variables, such as `$content-width: 860px;` |
| `_includes/custom-head.html` | Extra tags in `<head>`, such as favicons |

All three were confirmed to take effect with the remote theme.

## 8. Optional: preview locally

Install Ruby, then add a `Gemfile`:

```ruby
source "https://rubygems.org"
gem "github-pages", "~> 232", group: :jekyll_plugins
gem "webrick"
```

Then run:

```sh
bundle install
bundle exec jekyll serve
```

Open `http://localhost:4000`.

If the build fails with "Invalid US-ASCII character", the shell is not using a UTF-8 locale. On macOS or Linux, run `export LANG=en_US.UTF-8` first. The equivalent on Windows has not been tested.

## 9. Optional: skin switcher and RSS link in the header

The site header has two extras before the page links: a "Theme:" dropdown that lets a visitor switch between Minima's skins, and an RSS icon that links to the feed. The order is Theme, RSS, then the page links. The sub-footer bar has a second skin dropdown; the two stay in sync. The switcher is adapted from Minima's own demo site, with the demo's plugin replaced by static files.

### Why not the plugin

Minima's demo site uses a plugin, `_plugins/skin_manager.rb`, to build the list of skins and generate one stylesheet per skin. That plugin does not run here:

- The `github-pages` gem forces Jekyll into safe mode and ignores the `_plugins` folder. This applies to local builds with that gem and to GitHub Pages itself.
- The symptom is an empty dropdown (`page.available_skins` is never set) and only `style.css` in `_site/assets/css`.
- The plugin's module name is not the cause. Jekyll loads every `.rb` file in `_plugins` when plugins are allowed.

The plugin itself works under plain Jekyll 4. Keeping it would mean replacing the `github-pages` gem with plain Jekyll and deploying through a GitHub Actions workflow instead of "Deploy from a branch".

`_plugins/skin_manager.rb` is not used by this setup and can be deleted.

### Files

| File | Purpose |
|---|---|
| `_data/skins.yml` | The list of skin names shown in the dropdowns |
| `assets/css/<skin>.scss` | One per skin; each compiles to `assets/css/<skin>.css` |
| `_includes/skin-options.html` | The `<option>` list, shared by both dropdowns |
| `_includes/nav-items.html` | Override of Minima's navigation include: the "Theme:" dropdown and the RSS icon, followed by the page links |
| `_includes/sub-footer.html` | The sub-footer bar with its dropdown, and the script that drives both dropdowns. Minima's base layout includes this file just before `</body>`. |
| `_includes/custom-head.html` | A small script that applies the visitor's saved skin before the page paints |
| `_sass/minima/custom-styles.scss` | Styles for the sub-footer, the header RSS icon, and the header dropdown |

`_data/skins.yml`:

```yaml
- auto
- classic
- dark
- solarized
- solarized-dark
- solarized-light
```

Each stylesheet, for example `assets/css/dark.scss` (the two `---` lines are required):

```scss
---
---

@import
  "minima/skins/dark",
  "minima/initialize";
```

`_includes/skin-options.html`:

```liquid
{%- assign default_skin = site.minima.skin | default: 'classic' -%}
{%- for skin in site.data.skins %}
  {%- if skin == default_skin %}
  <option value="style.css">{{ skin }}</option>
  {%- else %}
  <option value="{{ skin }}.css">{{ skin }}</option>
  {%- endif %}
{%- endfor %}
```

A dropdown is any `<select>` with the class `skin-select` that includes those options:

```liquid
<select id="theme-switch" class="skin-select">
  {%- include skin-options.html %}
</select>
```

### How it works

- The theme's `style.css` is built from the skin set in `minima.skin` in `_config.yml` (`classic` when unset). The dropdown option for that skin points at `style.css`; every other option points at `<skin>.css`.
- The script in `sub-footer.html` finds every `<select class="skin-select">` on the page. Choosing a skin in one swaps the `href` of the `<link id="main-stylesheet">` tag, which the theme's head provides, and updates the other dropdown to match.
- The choice is saved in the browser's `localStorage` under `minima-ss`, so it applies to that visitor on that browser only.
- On the next page load, the script in `custom-head.html` reads the saved choice and applies it before the page paints. Without it, a returning visitor would see the default skin flash first.
- A saved value that no longer matches a skin falls back to `style.css`.
- The RSS icon and the "Theme:" dropdown sit inside Minima's `.nav-items` container. On screens narrower than 600px the page links and the RSS icon collapse into the menu button.
- The "Theme:" dropdown is hidden at 720px and narrower, by a media query in `custom-styles.scss`. Below that width the site title and the full navigation do not fit on one line. The sub-footer dropdown stays available at every width, so phone visitors can still switch skins. If the site title gets longer or shorter, adjust the 720px value to match.
- The RSS icon links to `feed.xml`, which the `jekyll-feed` plugin generates. `hide_site_feed_link: true` under `minima` in `_config.yml` hides the theme's own feed link in the footer, so the header is the only place it appears.

### Maintenance

- **A new Minima skin:** add its name to `_data/skins.yml` and create a matching `assets/css/<skin>.scss`.
- **Theme updates:** `_includes/nav-items.html` is a copy of Minima's file with additions. If a later Minima version changes its `nav-items.html`, compare the two.
- **Removing the sub-footer bar:** delete the `<div class="sub-footer">` block and the `toggleSubFooter` function from `sub-footer.html`, and keep the rest of the script. The header dropdown and the comment theme (section 11) depend on it.

**Tested:** October 5, 2026, with Jekyll 3.10.0 and github-pages 232. Choosing a skin in either dropdown changed the stylesheet and the other dropdown, the choice carried over to other pages, the RSS icon opened the feed, and the menu worked at phone width, all with no script errors. The header stayed on one line at every width from 601px up.

## 10. Optional: About page without the title heading

Minima's `page` layout prints the page title as a heading at the top ("About"). The About page on this site uses its own layout to leave that heading out. Layouts are plain files with no plugin involved, so this works on GitHub Pages.

`_layouts/about.html` is Minima's `page.html` with the `<header class="post-header">` block removed:

```html
---
layout: base
---
<article class="post">

  <div class="post-content">
    {{ content }}
  </div>

</article>
```

`about.md` points at it in its front matter:

```yaml
---
layout: about
title: About
permalink: /about/
---
```

Things to know:

- **Keep `title: About`.** The navigation link and the browser tab title both read it. Removing the title to hide the heading would break the navigation link.
- **`layout: base` keeps the rest of the site.** The site header, footer, and skin switcher still render.
- **Other pages are unaffected.** Anything using `layout: page` still gets its heading.
- **The layout is a copy.** If a later Minima version changes `page.html`, `_layouts/about.html` does not pick that up. Compare the two when updating the theme.

**Tested:** October 5, 2026, with Jekyll 3.10.0 and github-pages 232. The heading was gone, and the "About" navigation link and browser tab title were unchanged.

## 11. Optional: comments on posts with giscus

Posts get a comment section from [giscus](https://giscus.app), a free, open-source widget that stores each post's comments as a GitHub Discussion in this repository. It has no ads and no tracking. Visitors need a GitHub account to comment.

Minima ships with Disqus support instead. Its post layout includes `_includes/comments.html`, so a file of that name in this repository replaces the Disqus include with giscus. No plugin is involved, so this works on GitHub Pages.

### One-time setup on GitHub

1. Make sure the repository is public.
2. Turn on Discussions: repository **Settings → General → Features → Discussions**.
3. Install the [giscus app](https://github.com/apps/giscus) and give it access to this repository.
4. Go to [giscus.app](https://giscus.app), enter the repository name, and choose the **Announcements** category. That category type lets only maintainers and giscus start new discussions.
5. The page generates a snippet. Copy the values of `data-repo-id` and `data-category-id` from it.

### Site configuration

Paste the two IDs into `_config.yml`:

```yaml
giscus:
  repo: jaimerodriguez/jaimerodriguez_com
  repo_id: R_...          # data-repo-id from giscus.app
  category: Announcements
  category_id: DIC_...    # data-category-id from giscus.app
```

Comments stay off until both IDs are filled in. Restart `jekyll serve` after editing `_config.yml`; Jekyll does not reload that file.

### Files

| File | Purpose |
|---|---|
| `_config.yml` | The `giscus:` block with the repository, category, and their IDs |
| `_includes/comments.html` | Loads giscus with those values and keeps its theme in step with the site skin |
| `_includes/sub-footer.html` | The skin switcher fires a `skin-change` event that `comments.html` listens for |
| `_sass/minima/custom-styles.scss` | Spacing and a divider above the comment section |

### How it works

- **Where comments appear.** Under every post. Pages such as About and the home page have none, because only Minima's post layout includes `comments.html`.
- **Production only.** Minima includes comments only when `JEKYLL_ENV` is `production`. GitHub Pages builds that way. A plain `bundle exec jekyll serve` does not, so comments are missing from the local preview. To see them locally, run `JEKYLL_ENV=production bundle exec jekyll serve`.
- **Matching posts to discussions.** Each post maps to a discussion by its URL path (`data-mapping="pathname"`). giscus creates the discussion the first time someone comments or reacts. Changing a post's URL later disconnects it from its existing comments.
- **Turning comments off for one post.** Add `comments: false` to that post's front matter. Minima then shows "Comments have been disabled for this post."
- **Theme.** `comments.html` maps each Minima skin to the closest giscus theme, and updates giscus when a visitor changes the skin:

| Minima skin | giscus theme |
|---|---|
| classic | `light` |
| dark | `dark` |
| auto | `preferred_color_scheme` |
| solarized | `preferred_color_scheme` |
| solarized-light | `light` |
| solarized-dark | `dark_dimmed` |

### Maintenance

- **Moderation** happens in the repository's Discussions tab on GitHub.
- **A new Minima skin** needs a line in the `THEMES` map in `comments.html`. Without one it falls back to `preferred_color_scheme`.
- **Other giscus options** (reactions, comment box position, language) are the `data-` values in `comments.html`. The giscus.app page explains each one.

**Tested:** October 5, 2026, with Jekyll 3.10.0 and github-pages 232, using placeholder IDs and a stand-in for the giscus widget. The include rendered only on posts and only in production mode, rendered nothing with the IDs empty, and passed the right theme on load and on every skin change. The real widget has not been tested, because that needs the IDs from the setup steps.

## Updating the theme

1. Check the [Minima commit log](https://github.com/jekyll/minima/commits/master/) and README for breaking changes.
2. Replace `4de3223` in `remote_theme` with the new commit's short SHA.
3. Preview locally (step 8) or push and check the build in the **Actions** tab.

Once Minima 3.0 is officially released and GitHub Pages bundles it, `remote_theme` can be replaced with `theme: minima` and the `jekyll-remote-theme` plugin removed.

## What differs from older Minima guides

Most tutorials describe v2. These settings changed in v3:

| Minima 2.x | Minima 3 |
|---|---|
| `author: Name` and `email:` | `author:` with nested `name:` and `email:` |
| `header_pages:` | `minima: nav_pages:` |
| `github_username:`, `twitter_username:` | `minima: social_links:` list |
| `_layouts/default.html` | `_layouts/base.html` |

Social icons are loaded from the Font Awesome CDN, so `icon:` takes any [Font Awesome brand name](https://fontawesome.com/search?ic=brands).

## Sources

- [jekyll/minima README](https://github.com/jekyll/minima)
- [jekyll/minima `_config.yml` on master](https://raw.githubusercontent.com/jekyll/minima/master/_config.yml)
- [GitHub Pages dependency versions](https://pages.github.com/versions/)
- [jekyll/minima issue #411: using v3 on GitHub Pages](https://github.com/jekyll/minima/issues/411)
