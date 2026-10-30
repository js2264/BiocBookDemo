# 4  Executing `python` code

> **WARNING:**
>
> - `BiocBook`s can execute **`python` code**, not just `R` code;
> - The code runs on **every render**, including when the *Bioconductor Build System* builds the book;
> - The `python` packages the book needs are therefore installed **at build time**, by the book itself, in the environment declared in `inst/requirements.yml`: `python` and command-line tools from `conda`, the packages that `python` chunks import from PyPI;
> - **Keep at least one `R` chunk on a `python` page**: it is what keeps `quarto` on the `knitr` engine.

## 4.1 Declaring and installing the python environment

The environment is declared, with pinned versions, in `inst/requirements.yml` – a plain `conda` environment file. This book’s is:

``` yaml
name:
    BiocBookDemo
channels:
    - conda-forge
    - bioconda
    - nodefaults
dependencies:
    - python=3.12
    - samtools=1.24
    - pip:
        - numpy==2.5.4
```

`conda` provides `python` itself and command-line tools, such as `samtools` from `bioconda`. The packages that `python` chunks import go under `pip:`, and come from PyPI. `python` chunks run inside the `R` session, and `conda` builds of these packages link libraries that `R` may already have loaded, in an older version: on the Bioconductor builders, `h5py` and `matplotlib` \>= 3.11 then fail to import. PyPI wheels carry their own (see `?BiocBook::BiocBook-python`).

Note the presence of the `nodefaults` channel: the `defaults` channel is covered by the Anaconda Terms of Service, which require a paid licence for many organisations. `conda-forge` and `bioconda` are community channels that are free to use.

Any page that needs `python` would need to have this chunk at the top:

``` r
library(reticulate)
BiocBook::setup_python()
##  ✔ Using the `python` already configured for this session: '/opt/R-cache/R/BiocBook/envs/BiocBookDemo-faf828c9fe1d/bin/python'
```

`setup_python()` finds `inst/requirements.yml` and creates the environment if it does not exist yet: its `conda` packages, then its `pip:` section, with the environment’s own `pip`. It then activates the environment, addressing it by full path rather than by name, and puts its command-line tools on the `PATH`. If `RETICULATE_PYTHON` is already set (as it is inside the book’s `Docker` image), `setup_python()` only puts that interpreter’s folder on the `PATH`.

Environments are cached, and keyed on the contents of `requirements.yml`: an environment is re-used rather than resolved again **as long as `requirements.yml` is unchanged**. Editing the file (adding, removing or re-pinning a package) gives a new environment on the next render, including on machines that build the book every day, and two books only share an environment when they declare exactly the same thing. [`BiocBook::python_envs()`](https://rdrr.io/pkg/BiocBook/man/BiocBook-python.html) lists the environments cached on a machine, and `python_envs(remove = ...)` deletes the ones that are no longer needed.

## 4.2 Where `conda` itself comes from

Building a `conda` environment needs a `conda` implementation, and the machines that build this book do not necessarily have one. Bioconductor’s current builders may, and the `r-universe` build image ships `quarto`, `python3` and `pip`, but no `conda` at all. So `BiocBook` brings its own.

`micromamba` is a single self-contained binary: no `conda` installation, no base environment, no `python`.

[`BiocBook::micromamba()`](https://rdrr.io/pkg/BiocBook/man/BiocBook-python.html) resolves it in order:

1.  `RETICULATE_CONDA`, if you have set it;
2.  a `micromamba` already on the `PATH`;
3.  a copy previously downloaded into `BiocBook`’s cache (this book’s `Docker` image keeps the one it downloads while it is built);
4.  failing all of those, it downloads a **pinned** release and checks it against a recorded `SHA256` before using it.

> **NOTE:**
>
> `quarto` chooses its execution engine **per file**. A page with at least one `R` chunk uses `knitr`, and runs its `python` chunks through `reticulate`. A page whose only code is `python` uses `jupyter` instead. However, the Bioconductor builders do not provide `jupyter`, so a `jupyter` page cannot be rendered there at all.
>
> To force `quarto` to use `knitr`, keep at least one `R` chunk on the page, even if it is empty.

## 4.3 A worked example

`reticulate` hosts a single, persistent `python` session that is shared with `R`, so objects can be passed in both directions.

Start from an `R` object:

``` r
counts <- matrix(
    c(12, 0, 45, 3, 7, 91, 0, 5, 33),
    nrow = 3,
    dimnames = list(c("geneA", "geneB", "geneC"), c("s1", "s2", "s3"))
)
counts
##        s1 s2 s3
##  geneA 12  3  0
##  geneB  0  7  5
##  geneC 45 91 33
```

The `R` object is reachable from `python` through the `r` object:

``` python
import numpy as np
counts = np.array(r.counts)
print("shape:", counts.shape)
##  shape: (3, 3)
print("library sizes:", counts.sum(axis = 0))
##  library sizes: [ 57. 101.  38.]
```

State persists from one `python` chunk to the next, exactly as it would in a notebook:

``` python
cpm = counts / counts.sum(axis = 0) * 1e6
print("CPM values:")
##  CPM values:
print(np.round(cpm, 1))
##  [[210526.3  29703.       0. ]
##   [     0.   69306.9 131578.9]
##   [789473.7 900990.1 868421.1]]
```

Finally, the `python` object can be passed back to `R`:

``` r
cpm <- reticulate::py$cpm
cpm
##           [,1]      [,2]     [,3]
##  [1,] 210526.3  29702.97      0.0
##  [2,]      0.0  69306.93 131578.9
##  [3,] 789473.7 900990.10 868421.1
```

## 4.4 Command-line tools from `bioconda`

The `conda` part of the environment can hold any command-line tool from the `bioconda` channel, e.g. `samtools`. A tool runs in its own process, with its own libraries, so it cannot clash with the `R` session. `setup_python()` puts it on the `PATH`, for `bash` chunks…

``` sh
samtools --version | head -n 1
##  samtools 1.24
```

… as well as for [`system2()`](https://rdrr.io/r/base/system2.html) in `R`, and for `subprocess` in `python`:

``` python
import subprocess
v = subprocess.run(["samtools", "--version"], capture_output = True, text = True).stdout.splitlines()[0]
```

Then read that in `R`:

``` r
reticulate::py$v
##  [1] "samtools 1.24"
```

## 4.5 Session info

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
##   BiocBook       1.11.6  2026-10-30 [2] Github (js2264/BiocBook@3c3747d)
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
##   lattice        0.23-1  2026-08-12 [2] RSPM (R 4.6.1)
##   lifecycle      1.0.5   2026-01-08 [2] RSPM (R 4.6.0)
##   magrittr       2.0.5   2026-04-04 [2] RSPM (R 4.6.0)
##   Matrix         1.7-6   2026-07-25 [2] RSPM (R 4.6.1)
##   openssl        2.4.2   2026-06-09 [2] RSPM (R 4.6.0)
##   otel           0.2.0   2025-08-29 [2] RSPM (R 4.6.0)
##   pak            0.11.1  2026-07-22 [2] RSPM (R 4.6.0)
##   pillar         1.11.1  2025-09-17 [2] RSPM (R 4.6.0)
##   pkgconfig      2.0.3   2019-09-22 [2] RSPM (R 4.6.0)
##   png            0.1-9   2026-03-15 [2] RSPM (R 4.6.0)
##   purrr          1.2.2   2026-04-10 [2] RSPM (R 4.6.0)
##   R6             2.6.1   2025-02-15 [2] RSPM (R 4.6.0)
##   Rcpp           1.1.2   2026-07-05 [2] RSPM (R 4.6.0)
##   renv           1.3.1   2026-10-06 [2] RSPM (R 4.6.0)
##   reticulate   * 1.47.0  2026-09-03 [2] RSPM (R 4.6.0)
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
##   withr          3.0.3   2026-06-19 [2] RSPM (R 4.6.0)
##   xfun           0.62    2026-10-16 [2] RSPM (R 4.6.0)
##   yaml           2.3.12  2025-12-10 [2] RSPM (R 4.6.0)
##  
##   [1] /tmp/RtmpAfdFZk/Rinsta6519127b
##   [2] /usr/local/lib/R/site-library
##   [3] /usr/local/lib/R/library
##   * ── Packages attached to the search path.
##  
##  ─ Python configuration ────────────────────────────────────────────────────
##   python:         /opt/R-cache/R/BiocBook/envs/BiocBookDemo-faf828c9fe1d/bin/python
##   libpython:      /opt/R-cache/R/BiocBook/envs/BiocBookDemo-faf828c9fe1d/lib/libpython3.12.so
##   pythonhome:     /opt/R-cache/R/BiocBook/envs/BiocBookDemo-faf828c9fe1d:/opt/R-cache/R/BiocBook/envs/BiocBookDemo-faf828c9fe1d
##   version:        3.12.16 (main, Oct 26 2026, 11:37:43) [GCC 15.3.0]
##   numpy:          /opt/R-cache/R/BiocBook/envs/BiocBookDemo-faf828c9fe1d/lib/python3.12/site-packages/numpy
##   numpy_version:  2.5.4
##   
##   NOTE: Python version was forced by RETICULATE_PYTHON
##  
##  ───────────────────────────────────────────────────────────────────────────
```
