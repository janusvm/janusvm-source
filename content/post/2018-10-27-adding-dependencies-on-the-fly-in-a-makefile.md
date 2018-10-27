---
title: 'Adding dependencies on the fly in a Makefile'
author: 'Janus Valberg-Madsen'
date: '2018-10-27'
slug: 'adding-dependencies-on-the-fly-in-a-makefile'
categories: []
tags: ['makefile', 'metaprogramming']
image:
  caption: ''
  focal_point: ''
---

Imagine the following situation:

You have a bunch of files that are being built by the same rule in a Makefile, but some of them further depend on some other input files.
It's far from the most of the targets that have these dependences, and those that do depend on different subsets of the extra dependences.
How should the Makefile for this look?

I found myself in this very situation not long ago, when I was working on the slides for my [Git Workshop](/talk/aau-git-workshop/), which included a lot of SVG figures depicting commit graphs.
Each of these figures were generated from a `.tex` file using Latexmk, so the associated build targets and rule looked something like this:

```makefile
SVG := $(patsubst %.tex,%.svg,$(wildcard figs/*.tex))

$(SVG): %.svg: %.tex
	@cd figs && \
	latexmk -pdf -quiet $(<F) && \
	pdf2svg $(<F:.tex=.pdf) $(@F) && \
	latexmk -C $(<F)
```

The first line defines a list of the eventual SVG files by looking up all the `.tex` files in subdirectory `figs` and changing their file extensions to `.svg`.
In the next line, a [static pattern rule](https://www.gnu.org/software/make/manual/html_node/Static-Usage.html) is telling Make to use the pattern `%.svg: %.tex` only for the targets defined in that list.
The files are built by compiling a PDF with `pdflatex` (via `latexmk`), converting that into an SVG with `pdf2svg`, and finally deleting all the auxiliary build files again with `latexmk`.

So far so good, but there was one problem:
in a few of those SVGs, I included some icons in the form of PDFs from another subdirectory `figs/icons`.
To make a long story short, I wanted to use some FontAwesome icons for certain things, which required me to compile with `xelatex`, but the graphs in the SVGs got messed up, if I didn't use `pdflatex` for those.

So, as a compromise, I compiled PDF files for each of the FA symbols I needed and input them with `\includegraphics` in the graphs, adding the following lines to my Makefile:

```makefile
PDF := $(patsubst %.tex,%.pdf,$(wildcard figs/icons/*.tex))

$(PDF): %.pdf: %.tex
	@cd figs/icons && \
	latexmk -xelatex -quiet $(<F) && \
	latexmk -c $(<F)
```

Same as before, except it uses `xelatex` (again via `latexmk`) to compile, and there's no conversion to SVG.

The question was now: how do I express the relationship between the PDF icons and the SVG figures in my Makefile?


### Attempt 1: just slapping it in there

My first impulse was to simply add the `PDF` list as a dependency to `SVG`.
Simple.

```makefile
$(SVG): %.svg: %.tex $(PDF)
```

And sure, it worked, but it caused **every single SVG figure** to be rebuilt if **any** PDF icon had been changed.
No good.


### Attempt 2: adding an additional rule to a subset of the SVGs

Then, I had the idea to peek a bit in the files to determine which of the `SVG` targets should _actually_ depend on `PDF`, taking advantage of the fact that you can define [multiple rules for the same target](https://www.gnu.org/software/make/manual/html_node/Multiple-Rules.html) (as long as you only have one build recipe):

```makefile
SVG_WITH_ICONS := $(shell grep -l '\\includegraphics' figs/*.tex)

$(SVG_WITH_ICONS): $(PDF)
```

Adding these two lines improved the situation slightly;
only the targets that included at least one PDF icon were rebuilt when the PDF changed, but still all of them for any PDF.
Hmm.


### Attempt 3: enter metaprogramming

Long after I gave the talk those figures were for, I finally found an acceptable solution for the problem: Make's [eval function](https://www.gnu.org/software/make/manual/html_node/Eval-Function.html).

Put shortly, you can define a rule which is deferred and can be expanded (`eval`'ed) in a different context.
This was exactly what I needed, as it allowed me to write automatic, extra rules for each individual `SVG` target!
Here's how:

```makefile
define PDF_RULE
$T: $(shell grep -hoE '\bicons/.+\b' $(T:.svg=.tex) \
      | sort -u | sed -e 's/$$/.pdf/g' -e 's/^/figs\//g' \
      | paste -s -d ' ')
endef

$(foreach T,$(SVG),$(eval $(PDF_RULE)))
```

There's a lot going on in `PDF_RULE`, so here's a breakdown:

- `$T` is a variable to be expanded later into the individual files in `SVG`
- `grep -hoE '\bicons/.+\b' $(T:.svg=.tex)` prints all the occurences of a string starting with 'icons/' in the corresponding `.tex` file as individual lines, e.g.

    ```
    icons/user
    icons/laptop
    icons/server
    ```

- `sort -u` sorts the lines and (more importantly) deletes duplicates
- `sed -e 's/$$/.pdf/g' -e 's/^/figs\//g'` first appends `.pdf` to the end (`$`) of the lines, then prepends `figs/` to the beginning (`^`), e.g.

    ```
    figs/icons/user.pdf
    figs/icons/laptop.pdf
    figs/icons/server.pdf
    ```

    (since `$` is a special character used for expansion in Make, you have to put `$$` to treat it as a literal `$` symbol)

- lastly, `paste -s -d ' '` collapses the lines into one line, separated by spaces

When the `foreach` loop is run, the contents of `PDF_RULE` is expanded for each of the files in `SVG`, resulting in _new rules being added at runtime_.
So for example, when expanded for a file which _doesn't_ include any PDFs, e.g. `figs/git-add.svg`, it produces the empty rule

```makefile
figs/git-add.svg:
```

which does nothing.
However, when expanded for a file which _does_ include PDFs, e.g. `figs/git-push.svg`, the `shell` command looks through `figs/git-push.tex`, finds the included graphics `icons/laptop.pdf` and `icons/server.pdf` and creates the rule

```makefile
figs/git-push.svg: figs/icons/laptop.pdf figs/icons/server.pdf
```

which causes `figs/git-push.svg` to be rebuilt whenever those specific PDFs are changed.
So basically, this corresponds to manually writing out rules for each individual SVG file, but it's all done automatically!

Here's the full Makefile:

```makefile
SVG := $(patsubst %.tex,%.svg,$(wildcard figs/*.tex))
PDF := $(patsubst %.tex,%.pdf,$(wildcard figs/icons/*.tex))

all: $(SVG)

define PDF_RULE
$T: $(shell grep -hoE '\bicons/.+\b' $(T:.svg=.tex) \
      | sort -u | sed -e 's/$$/.pdf/g' -e 's/^/figs\//g' \
      | paste -s -d ' ')
endef

$(foreach T,$(SVG),$(eval $(PDF_RULE)))

$(SVG): %.svg: %.tex
	@cd figs && \
	latexmk -pdf -quiet $(<F) && \
	pdf2svg $(<F:.tex=.pdf) $(@F) && \
	latexmk -C $(<F)

$(PDF): %.pdf: %.tex
	@cd figs/icons && \
	latexmk -xelatex -quiet $(<F) && \
	latexmk -c $(<F)

clean:
	@rm -f $(SVG)
	@rm -f $(PDF)
	@cd figs && latexmk -C
	@cd figs/icons && latexmk -C

.PHONY: all clean
```

As a final comment, I should probably point out that I use GNU Make — I'm not sure if any of the above features are GNU specific.
