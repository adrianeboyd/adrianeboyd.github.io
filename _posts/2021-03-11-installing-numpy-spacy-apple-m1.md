---
layout: post
title: Installing numpy and spaCy on an Apple M1
tags: [numpy, pip, spacy]
categories: pip
---

## Installing numpy and spaCy on an Apple M1 with Big Sur

Installing numpy or packages that compile against numpy on an Apple M1 is not
straight-forward because the most recent versions of numpy (`1.19.3`-`1.20.1`)
require a version of `wheel` in `pyproject.toml` ([PEP
518](https://www.python.org/dev/peps/pep-0518/)) that doesn't work on Big Sur.
Since numpy and spaCy are configured to use build isolation by default ([PEP
517](https://www.python.org/dev/peps/pep-0517/)), a simple `pip install spacy`
does not work: the numpy install fails due to the version of `wheel` and then
the spaCy install fails due to the failed numpy installation. (For reference,
you need `wheel>=0.36.1` for Big Sur. The good news is that it looks like this
will be fixed in numpy 1.21.)

This guide describes three options for installing numpy and spaCy that have
been tested as of March 2021:

- Option 1: Install with pip with build isolation

  - Install spaCy and its dependencies with pip with the fewest steps
  - For users who want to install spaCy once

- Option 2: Install with pip without build isolation

  - Primarily for developers who plan to recompile spaCy frequently

- Option 3: Install binary packages from conda-forge

  - No compiling required using the experimental [miniforge OS X ARM64 conda installer](https://github.com/conda-forge/miniforge)
  - For conda users who want to install spaCy once without compiling

### Install Xcode and Python

If necessary, install Xcode and Python.

Install Xcode:

```bash
xcode-select --install
```

Install Python 3.9 from:

- Python 3.9.2 from source or the universal2 installer:

  https://www.python.org/downloads/release/python-392/

- Python 3.9.2 from conda-forge:

  https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh

- Python 3.9.2 from homebrew:

  https://formulae.brew.sh/formula/python@3.9

- **NOT RECOMMENDED:** The system Python 3.8.2 in `/usr/bin/python3` does not
  seem to work correctly: I could compile and install numpy 1.20.1, but numpy
  segfaults in the middle of its core test suite. Python's official support for
  Big Sur starts at version 3.9.1.

### Option 1: Install spaCy (and numpy) with pip

The following steps describe how to install spaCy and numpy with pip with build
isolation enabled. After you've gotten through the initial setup, spaCy can be
installed or updated with a single `pip` command in a new virtual environment.

1. Install a BLAS library. As an example, install OpenBLAS through homebrew:

   ```bash
   brew install openblas
   ```

1. Create and activate a new virtual environment:

   - venv

     ```bash
     python3.9 -m venv myvenv
     source myvenv/bin/activate
     python -m pip install -U pip setuptools wheel
     ```

   - conda

     ```bash
     conda create -n myvenv python=3.9
     conda activate myvenv
     ```

1. Create a file called `constraints.txt` that specifies an older version of
   numpy without the `wheel` problem for use in isolated builds:

   ```text
   numpy==1.19.2
   ```

1. In the `pip install` command, set `PIP_CONSTRAINT` to the path to
   `constraints.txt` and `OPENBLAS` to the path for openblas (or whichever
   numpy settings you choose):

   ```bash
   PIP_CONSTRAINT=constraints.txt OPENBLAS="$(brew --prefix openblas)" pip install spacy
   ```

   It will take 7-8 minutes to compile and install all the required packages.

   Note that setting `PIP_CONSTRAINT` **IS NOT** equivalent to setting `pip
install -c constraints.txt`. The `-c constraints.txt` setting is only applied
to the packages specified at the top level with `pip install package`, not
within the isolated build environments. In contrast, the environment variable
`PIP_CONSTRAINT` is visible at the top level and within the isolated build
environments, so the numpy constraints are set correctly in the isolated
builds for spaCy and its dependencies.

### Option 2: Install numpy and spaCy without build isolation

This option is best for developers who are recompiling spaCy frequently. If you
disable build isolation, you need to install the requirements once initially by
hand, but then you can use your local numpy installation for all future builds
and recompile spaCy quickly.

To do this, you have to go step by step through the dependencies that compile
against numpy to install their requirements and disable build isolation.

If you're using python from conda-forge, you might want to go ahead and use
their numpy+openblas, too, and then you wouldn't need to install openblas
separately with homebrew.

1.  Install numpy with either conda or pip:

    - conda (BLAS libraries are included automatically):

      ```bash
      conda install numpy
      ```

    - pip (install OpenBLAS with homebrew as in option 1)

       ```bash
       pip install cython
       OPENBLAS="$(brew --prefix openblas)" pip install numpy --no-build-isolation
       ```

      With build isolation disabled, you can install the most recent version of
numpy, currently 1.20.1.

1.  Install the dependencies for `spacy` step by step, installing the
    requirements and disabling build isolation in turn for each dependency that
    compiles against numpy:

    ```bash
    pip install -r https://raw.githubusercontent.com/explosion/cython-blis/master/requirements.txt
    pip install blis --no-build-isolation
    pip install -r https://raw.githubusercontent.com/explosion/thinc/master/requirements.txt
    pip install thinc --no-build-isolation
    pip install -r https://raw.githubusercontent.com/explosion/spaCy/master/requirements.txt
    pip install spacy --no-build-isolation
    ```

    Note that the requirements from the `master` branch in the repo may be
    slightly ahead of the latest `blis`/`thinc`/`spacy` releases on PyPI. If you
    replace `master` with the exact version above (e.g., `v3.0.5`, so
    https://raw.githubusercontent.com/explosion/spaCy/v3.0.5/requirements.txt),
    you can access the exact requirements for a particular release.

1.  If you have a local copy of the spaCy repo and you're planning on
    recompiling frequently, you can create an editable install by replacing the
    last step in the previous section with:

    ```bash
    cd spaCy
    pip install --no-build-isolation --editable .
    ```

    Or using `setup.py` and parallel build jobs for spaCy to speed things up:

    ```bash
    cd spaCy
    python setup.py build_ext --inplace -j 4
    python setup.py develop
    ```

### Option 3: Binary packages from conda-forge

If you'd rather not compile anything, there are experimental OS X ARM 64
packages on [`conda-forge`](https://anaconda.org/conda-forge/spacy/) for all
recent releases of spaCy, including spaCy v2.3. You can use the [miniforge
installer](https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh)
and then install spaCy with:

```bash
conda install spacy
```

or for spaCy v2:

```bash
conda install "spacy~=2.3.5"
```

Be aware that the conda-forge packages are cross-compiled and not tested during
the conda-forge build process. However I can report that I've run all the tests
locally with no issues.

### Run the test suite

To test your local install, install the development dependencies and run the
test suite. To simplify the instructions here, these are development
dependencies for spaCy v3.0.5. Newer versions may be slightly different (check
[`requirements.txt`](https://github.com/explosion/spaCy/blob/master/requirements.txt)
in the spaCy repo):

```bash
pip install "pytest>=5.2.0" "pytest-timeout>=1.3.0,<2.0.0" "mock>=2.0.0,<3.0.0" "flake8>=3.5.0,<3.6.0" hypothesis
pytest --pyargs spacy
```
