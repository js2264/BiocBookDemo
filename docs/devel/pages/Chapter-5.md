# 5  Serving the book to AI assistants

> **WARNING:**
>
> - Every version of a `BiocBook` serves an **`llms.txt`** file: a Markdown index of the book that AI assistants can read instead of crawling its HTML pages;
> - Every page has a **Markdown copy**, at the page’s URL with `.md` in place of `.html`, and **links to it**;
> - `llms.txt` tells an assistant **what it needs to run the book’s code**: the book package and its version, how to install it, its `Docker` image and its `python` environment;
> - There is **one `llms.txt` per Bioconductor release**, so that an assistant can point a reader to the version of the book that matches theirs.

## 5.1 `llms.txt`: an index of the book for AI assistants

Readers increasingly ask AI assistants about what they read. An assistant that crawls the HTML pages of a book finds menus, scripts and styling, but no hint of the package, version or environment that its code was run with. The [llms.txt proposal](https://llmstxt.org/) gives assistants a simpler entry point: a Markdown file at a fixed URL, with the title of the site, a short summary and links to Markdown versions of its pages.

Each version of this book serves one, e.g. [`devel/llms.txt`](https://js2264.github.io/BiocBookDemo/devel/llms.txt). It starts with the book’s title and description. Right after them, `BiocBook` adds a block of its own, between two markers. For this build of the book, it reads:

``` markdown
<!-- biocbook:start -->
- Book package: `BiocBookDemo` 1.11.9 (Bioconductor 3.24), source: <https://github.com/js2264/BiocBookDemo>
- Install: `BiocManager::install("BiocBookDemo")` (Bioconductor) or `BiocManager::install("js2264/BiocBookDemo")` (GitHub)
- Docker image with every dependency: `docker run -it ghcr.io/js2264/biocbookdemo:1.11.9 R`
- Python environment (conda): `python=3.12`, `samtools=1.24`, `pip`; from PyPI: `numpy==2.5.4`, provisioned by `BiocBook::setup_python()` from `inst/requirements.yml`
- Other versions of this book: one per Bioconductor release, at `https://js2264.github.io/BiocBookDemo/<version>/` (e.g. <https://js2264.github.io/BiocBookDemo/devel/>), indexed in <https://js2264.github.io/BiocBookDemo/llms.txt>
<!-- biocbook:end -->
```

An assistant that reads it knows which Bioconductor release the reader is looking at, how to install that version of the book package, which `Docker` image runs every chunk of the book, and which `python` environment its `python` chunks need ([sec-python](#sec-python)).

The list of pages comes next, each linked to its Markdown copy:

``` markdown
## Pages

- [Write Quarto books with Bioconductor](index.md)
- [Preamble](pages/preamble.md)
- [1 `BiocBook` versioning system](pages/Chapter-1.md)
- ...
```

## 5.2 A Markdown copy of every page

Next to `pages/Chapter-4.html`, the book serves `pages/Chapter-4.md`: the same chapter as Markdown, with its code and its results, but without the navigation, scripts and styling of the HTML page. Every page links to its copy twice:

- under its title, with a **“View as Markdown”** link, for readers who would rather paste a chapter into an assistant themselves;
- in its `<head>`, for AI assistants and crawlers, which look there for other versions of a page:

``` html
<link rel="alternate" type="text/markdown" href="Chapter-5.md">
```

`quarto` writes these copies as `<page>.llms.md`: `BiocBook` renames them after their page, and rewrites the links of `llms.txt`, and those between the copies, to match. To hide the “View as Markdown” link, add this rule to the book’s `inst/assets/book.scss`:

``` css
.biocbook-md-link { display: none; }
```

### 5.2.1 Content for readers or for assistants only

`quarto`’s conditional content can tell the HTML pages and their Markdown copies apart. Content that only makes sense in the browser, such as an interactive widget, can be left out of the copies:

``` markdown
::: {.content-hidden when-format="llms-txt"}
This paragraph is in the HTML page, but not in its Markdown copy.
:::
```

Conversely, a note for assistants can be added to the copies only, e.g. to remind them which image runs the book’s code:

``` markdown
::: {.content-visible when-format="llms-txt"}
Note for AI assistants: the code of this book runs as it is in the `Docker`
image named in the book's `llms.txt`.
:::
```

Note for AI assistants: the code of this book runs as it is in the `Docker` image named in the book’s `llms.txt`.

The first paragraph above shows on this page, the second one does not: the Markdown copy of this page has it the other way round.

## 5.3 One `llms.txt` per Bioconductor release

Each Bioconductor release has its own copy of the book, built from its own branch, and deployed in its own folder of the website (`devel/`, `3.18/`, …). Each copy has its own `llms.txt`, which names its release, its package version and its `Docker` image. At the root of the website, every deployment rebuilds an index of the copies that have one, [`llms.txt`](https://js2264.github.io/BiocBookDemo/llms.txt):

``` markdown
# BiocBookDemo

> Every version of this book, one per Bioconductor release, each with its own llms.txt.

## Versions

- [devel](devel/llms.txt)
```

Copies built before `llms.txt` existed are left out. An assistant can thus send a reader to the book, package and image that match their own Bioconductor release, rather than to `devel`.

## 5.4 How a `BiocBook` builds these files

Books created from template \>= 1.1.3 (e.g. with [`BiocBook::init()`](https://rdrr.io/pkg/BiocBook/man/BiocBook.html)) do all of this out of the box:

1.  `quarto` \>= 1.11 writes `llms.txt` and the Markdown copies when the book’s `llms-txt` option is on. The template switches it on in a `quarto` profile, `inst/_quarto-llms.yml`:

    ``` yaml
    book:
      llms-txt: true
    ```

    `vignettes/Makefile` activates this profile only when the installed `quarto` supports it, as older versions reject the option. The `biocbook` GitHub workflow pins such a `quarto` with its `QUARTO_VERSION` setting, and copies built by Bioconductor get these files once the Bioconductor builders run `quarto` \>= 1.11.

2.  After every render, `inst/assets/post-render.R` calls [`BiocBook::enrich_llms_txt()`](https://rdrr.io/pkg/BiocBook/man/BiocBook-llms.html). It adds the block above to `llms.txt`, renames the Markdown copies and links every page to its own. With an older `quarto`, it writes a minimal `llms.txt` listing the chapters instead.

3.  When it deploys the book, the `biocbook` GitHub workflow rebuilds the index of versions at the root of the website, and writes a `.nojekyll` file next to it. Without it, GitHub Pages would run `Jekyll` on the website, which turns every `.md` file into an HTML page of the same name, colliding with the book’s own.

Books created from an earlier template can add each piece by hand: see `?BiocBook::BiocBook-llms`. To build these files locally:

``` sh
QUARTO_PROFILE=llms quarto render inst/
```

## 5.5 Asking an assistant about the book

- **Start from `llms.txt`.** An assistant that can browse the web can be given the `llms.txt` of the version of the book you use, e.g. `https://js2264.github.io/BiocBookDemo/devel/llms.txt`, and follow its links to the pages it needs.
- **Or paste a page.** “View as Markdown”, under the title of each page, opens the page as plain Markdown, ready to be copied into any assistant.
- **Run the code where the book ran it.** The `Docker` image that `llms.txt` names has every dependency of the book installed, its `python` environment included:

``` sh
docker run -it ghcr.io/js2264/biocbookdemo:devel R
```

## 5.6 Session info

``` r
sessioninfo::session_info()
##  ─ Session info ────────────────────────────────────────────────────────────
##   setting  value
##   version  R version 4.6.1 (2026-06-24)
##   os       Ubuntu 24.04.5 LTS
##   system   x86_64, linux-gnu
##   ui       X11
##   language (EN)
##   collate  C
##   ctype    en_US.UTF-8
##   tz       Etc/UTC
##   date     2026-10-30
##   pandoc   3.12.1 @ /usr/bin/ (via rmarkdown)
##   quarto   1.11.5 @ /usr/local/bin/quarto
##  
##  ─ Packages ────────────────────────────────────────────────────────────────
##   package      * version date (UTC) lib source
##   askpass        1.2.1   2024-10-04 [2] RSPM (R 4.6.0)
##   BiocBook       1.11.7  2026-10-30 [2] Github (js2264/BiocBook@504d3a6)
##   BiocGenerics   0.59.12 2026-08-11 [2] Bioconductor 3.24 (R 4.6.1)
##   cli            3.6.6   2026-04-09 [2] RSPM (R 4.6.0)
##   credentials    2.0.3   2025-09-12 [2] RSPM (R 4.6.0)
##   digest         0.6.39  2025-11-19 [2] RSPM (R 4.6.0)
##   dplyr          1.2.1   2026-04-03 [2] RSPM (R 4.6.0)
##   evaluate       1.0.5   2025-08-27 [2] RSPM (R 4.6.0)
##   fastmap        1.2.0   2024-05-15 [2] RSPM (R 4.6.0)
##   fs             2.1.0   2026-04-18 [2] RSPM (R 4.6.0)
##   generics       0.1.4   2025-05-09 [2] RSPM (R 4.6.0)
##   gert           2.5.0   2026-10-14 [2] RSPM (R 4.6.0)
##   gh             1.6.1   2026-07-20 [2] RSPM (R 4.6.0)
##   gitcreds       0.1.2   2022-09-08 [2] RSPM (R 4.6.0)
##   glue           1.8.1   2026-04-17 [2] RSPM (R 4.6.0)
##   htmltools      0.5.9   2025-12-04 [2] RSPM (R 4.6.0)
##   htmlwidgets    1.6.4   2023-12-06 [2] RSPM (R 4.6.0)
##   httr           1.4.9   2026-09-01 [2] RSPM (R 4.6.0)
##   jsonlite       2.0.0   2025-03-27 [2] RSPM (R 4.6.0)
##   knitr          1.52    2026-09-06 [2] RSPM (R 4.6.0)
##   lifecycle      1.0.5   2026-01-08 [2] RSPM (R 4.6.0)
##   magrittr       2.0.5   2026-04-04 [2] RSPM (R 4.6.0)
##   openssl        2.4.2   2026-06-09 [2] RSPM (R 4.6.0)
##   otel           0.2.0   2025-08-29 [2] RSPM (R 4.6.0)
##   pak            0.11.1  2026-07-22 [2] RSPM (R 4.6.0)
##   pillar         1.11.1  2025-09-17 [2] RSPM (R 4.6.0)
##   pkgconfig      2.0.3   2019-09-22 [2] RSPM (R 4.6.0)
##   purrr          1.2.2   2026-04-10 [2] RSPM (R 4.6.0)
##   R6             2.6.1   2025-02-15 [2] RSPM (R 4.6.0)
##   renv           1.3.1   2026-10-06 [2] RSPM (R 4.6.0)
##   rlang          1.3.0   2026-07-05 [2] RSPM (R 4.6.0)
##   rmarkdown      2.32    2026-09-01 [2] RSPM (R 4.6.0)
##   rprojroot      2.1.1   2025-08-26 [2] RSPM (R 4.6.0)
##   sessioninfo    1.2.4   2026-06-04 [2] RSPM (R 4.6.0)
##   stringi        1.8.9   2026-08-04 [2] RSPM (R 4.6.0)
##   stringr        1.6.0   2025-11-04 [2] RSPM (R 4.6.0)
##   sys            3.4.3   2024-10-04 [2] RSPM (R 4.6.0)
##   tibble         3.3.1   2026-01-11 [2] RSPM (R 4.6.0)
##   tidyselect     1.2.1   2024-03-11 [2] RSPM (R 4.6.0)
##   usethis        3.2.2   2026-09-10 [2] RSPM (R 4.6.0)
##   vctrs          0.7.3   2026-04-11 [2] RSPM (R 4.6.0)
##   xfun           0.62    2026-10-16 [2] RSPM (R 4.6.0)
##   yaml           2.3.12  2025-12-10 [2] RSPM (R 4.6.0)
##  
##   [1] /tmp/Rtmp2UlXxq/Rinstb7a8b5f19
##   [2] /usr/local/lib/R/site-library
##   [3] /usr/local/lib/R/library
##  
##  ───────────────────────────────────────────────────────────────────────────
```
