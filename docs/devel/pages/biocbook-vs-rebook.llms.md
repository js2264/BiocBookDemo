# 5  {`BiocBook`}-based books vs. {`rebook`}-based books

{`rebook`} is another book rendering package currently used by Bioconductor to build book packages such as `OSCA.*` and `SingleRBook`.

## 5.1 Differences with {`rebook`}-based books

`OSCA` books provided by Bioconductor are rendered upon building by the *Bioconductor Build System* (BBS). They rely on {`rebook`} to orchestrate the rendering. Briefly, `OSCA` books have their book pages in `/inst/book/`, while the `vignettes/` folder contains (1) a `Makefile` and (2) a dummy `stub.Rmd` vignette (required to trigger vignette rendering and thus `make`).

When the BBS triggers `OSCA.intro` package building, upon vignette rendering, the `Makefile` triggers the following commands:

``` r
work.dir <- rebook::bookCache('OSCA.intro')
handle <- rebook::preCompileBook('../inst/book', work.dir=work.dir, desc='../DESCRIPTION')
old.dir <- setwd(work.dir)
bookdown::render_book('index.Rmd')
setwd(old.dir)
rebook::postCompileBook(work.dir=work.dir, final.dir='../inst/doc/book', handle=handle)
```

The resulting book, pre-compiled by {`rebook`} and assembled by {`bookdown`}, is eventually served by Bioconductor from the `/inst/doc/book/` folder.

{`BiocBook`}-based packages follow a strategy similar to that of `OSCA` books: they provide a `Makefile` in the `vignettes/` folder to trigger book rendering when building the package. However, the executed command does not rely on {`rebook`} and {`bookdown`}, but on a `render` command from the `quarto` software.

``` sh
quarto render ../inst/
mv ../inst/docs ../inst/doc/book
```

The resulting book, fully compiled by native `quarto`, is also located in `/inst/doc/book/` once the package is built (and in `doc/book` in the library directory once installed).

> **WARNING:**

## 5.2 `BiocBook` features missing from `rebook`

- A `BiocBook` can be readily initiated using [`BiocBook::init()`](https://rdrr.io/pkg/BiocBook/man/BiocBook.html);
- It relies on modern `.qmd` files supported by `Quarto`;
- It can work as a standalone Github-hosted package, without necessarily having to be submitted to/built by Bioconductor. The book should be rendered exactly the same way through Github or by the BBS;
- It supports versioning of the online book served by `gh-pages` **through the author Github account**;
- It distributes versioned Dockerfiles **through the author Github account**;
- `BiocBook`-based packages can actually provide fully-fledged functions in `R/`, manual pages and vignettes. They can be installed exactly the same way than other software packages.

## 5.3 `rebook` features missing from `BiocBook`

- Smart reuse of objects generated in one book in another book;

> **TIP:**
>
> This can still be achieved **within** a book by saving a data object as an `.rds` file and loading it in a subsequent chapter.
>
> ``` r
> isthisworking <- readRDS('isthisworking.rds')
> isthisworking
> ##  [1] "yes"
> ```

- Native support of cross-references across books.

## 5.4 Session info

``` r
sessioninfo::session_info()
##  ─ Session info ────────────────────────────────────────────────────────────
##   setting  value
##   version  R version 4.6.1 (2026-06-24)
##   os       Ubuntu 24.04.4 LTS
##   system   x86_64, linux-gnu
##   ui       X11
##   language (EN)
##   collate  C
##   ctype    en_US.UTF-8
##   tz       Etc/UTC
##   date     2026-10-04
##   pandoc   3.11 @ /usr/bin/ (via rmarkdown)
##   quarto   1.11.5 @ /usr/local/bin/quarto
##  
##  ─ Packages ────────────────────────────────────────────────────────────────
##   package     * version date (UTC) lib source
##   cli           3.6.6   2026-04-09 [2] RSPM (R 4.6.0)
##   digest        0.6.39  2025-11-19 [2] RSPM (R 4.6.0)
##   evaluate      1.0.5   2025-08-27 [2] RSPM (R 4.6.0)
##   fastmap       1.2.0   2024-05-15 [2] RSPM (R 4.6.0)
##   htmltools     0.5.9   2025-12-04 [2] RSPM (R 4.6.0)
##   htmlwidgets   1.6.4   2023-12-06 [2] RSPM (R 4.6.0)
##   jsonlite      2.0.0   2025-03-27 [2] RSPM (R 4.6.0)
##   knitr         1.52    2026-09-06 [2] RSPM (R 4.6.0)
##   otel          0.2.0   2025-08-29 [2] RSPM (R 4.6.0)
##   rlang         1.3.0   2026-07-05 [2] RSPM (R 4.6.0)
##   rmarkdown     2.32    2026-09-01 [2] RSPM (R 4.6.0)
##   sessioninfo   1.2.4   2026-06-04 [2] RSPM (R 4.6.0)
##   xfun          0.61    2026-09-16 [2] RSPM (R 4.6.0)
##   yaml          2.3.12  2025-12-10 [2] RSPM (R 4.6.0)
##  
##   [1] /tmp/RtmpASCh25/Rinstb7e21def1
##   [2] /usr/local/lib/R/site-library
##   [3] /usr/local/lib/R/library
##  
##  ───────────────────────────────────────────────────────────────────────────
```
