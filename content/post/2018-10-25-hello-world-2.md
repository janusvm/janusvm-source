---
title: 'Hello World! (2)'
author: 'Janus Valberg-Madsen'
date: '2018-10-25'
slug: 'hello-world-2'
categories: ['R']
tags: ['blogdown']
---

Second time's the charm, right?

Last time I made an attempt at getting my `github.io` site up and running, it ended up being half-baked and I left it unused for over a year.
In the meantime, my number of repositories with GitHub Pages steadily increased, resulting in a bunch of `janusvm.github.io/repo` links that were nowhere to be found on what ought to be the root site.

After giving three talks this month, each with slides on GH Pages, I decided it was time to organise my stuff and update this website.
Being an R enthusiast, it was a no-brainer for me to use [blogdown](https://bookdown.org/yihui/blogdown/), and following the recommendation in the book, I opted for the [Academic Hugo theme](https://sourcethemes.com/academic/) for its nice defaults, extra features, and customisation options.


## Theme customisations

The Academic theme has eight built-in colour themes and three font themes, but none of them hit the mark 100% for me.
When it comes to colour themes, I'm hopelessly in love with [Nord](https://github.com/arcticicestudio/nord) — I use it for Emacs, Bash, and whatever else that lets me configure its interface colours.
Here's how I've set it up on this site:

In a file called `/data/themes/nord.toml`:

```toml
# Theme metadata
name = "Nord"

# Is theme light or dark?
light = true

# Primary
primary = "#bf616a"
primary_light = "#d08770"
primary_dark = "#a5545b"

# Menu
menu_primary = "#3b4252"
menu_text = "#eceff4"
menu_text_active = "#88c0d0"
menu_title = "#eceff4"

# Home sections
home_section_odd = "#fff"
home_section_even = "#eff2f7"
```

and then in my `config.toml`:

```toml
color_theme = "nord"
```

This sets the colours of just about everything except for code blocks.
[Highlight.js](https://highlightjs.org/) _does_ have the Nord theme, but it turns out it isn't on the CDNJS server yet, which means that you can't just put `highlight_style = "nord"` in `config.toml`.

Instead, download `nord.css` from the [repository](https://github.com/highlightjs/highlight.js), put it in `/static/css`, and add `"nord.css"` to the `custom_css` array in the config, e.g.

```toml
custom_css = ["nord.css", "fonts/iosevka.css", "custom.css"]
```

The other two CSS files in there define the [Iosevka](https://github.com/be5invis/Iosevka) font face and some miscellaneous adjustments, respectively.
I have used many different monospace fonts for coding, but after recently discovering Iosevka, I may finally have found a font to settle down with.

The fonts used on the site are customised via the file `/data/fonts/custom.toml`:

```toml
# Font style metadata
name = "Custom"

# Optional Google font URL
google_fonts = "Abel|Roboto:400,400italic,700|Roboto+Condensed:300"

# Font families
heading_font = "Abel"
body_font = "Roboto"
nav_font = "Roboto Condensed"
mono_font = "Iosevka SS04 Web"

# Font size
font_size = "20"
font_size_small = "16"
```

and the config entry:

```toml
font = "custom"
```


## Deployment

It turns out to be relatively simple to have the source files in one repository and the published site in another, if you're comfortable using git submodules:

1. Create the repository `user.github.io` on GitHub (where `user` is your username)
2. If you don't initialise it with any files, push an empty commit (from a temporary folder somewhere else) to it first:

    ```
    git init
    git commit -m "Initial commit" --allow-empty
    git remote add origin https://github.com/user/user.github.io.git
    git push -u origin master
    ```

3. From the folder with your blog source (assuming it's already a git repository), add `user.github.io` as a submodule in `/public`:

    ```
    rm -rf public
    git submodule add -b master https://github.com/user/user.github.io.git public
    ```

4. Build the site:

    ```
    hugo
    ```

5. Commit and push the submodule:

    ```
    cd public
    git add .
    git commit -m "Build site"
    git push -u origin master
    ```

After that, the site is deployed.
In my experience, GitHub Pages sometimes don't actually build the site the first time you push after activating Pages, so if nothing shows up after a few minutes, try pushing another commit.


## Links

I found [this post](https://lmyint.github.io/post/hugo-academic-tips) by Leslie Myint in addition to [Hugo's hosting guide](https://gohugo.io/hosting-and-deployment/hosting-on-github) and the [Academic theme documentation](https://sourcethemes.com/academic/docs) to be very helpful when setting up this site.
