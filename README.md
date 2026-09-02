![LangChain Academy](https://cdn.prod.website-files.com/65b8cd72835ceeacd4449a53/66e9eba1020525e5a7873f96_LCA-big-green%20(2).svg)

# LangChain Academy — Emacs and uv edition

This fork adapts [LangChain Academy](https://github.com/langchain-ai/langchain-academy) to a personal Arch Linux workflow built around:

- Python 3.13 managed by [uv](https://docs.astral.sh/uv/)
- project-local environment variables managed by [direnv](https://direnv.net/)
- Doom Emacs with Eglot, Org Babel, and `emacs-jupyter`
- executable, stateful Org tutorials alongside the original Jupyter notebooks
- LangGraph Studio launched from the same project environment

The repository is an environment for coursework and executable writing; it is deliberately not an installable Python package.

## Academy overview

The academy introduces LangGraph and foundational concepts in the LangChain ecosystem. Module 0 covers setup, Modules 1–5 progressively develop LangGraph applications, and Module 6 covers deployment. Each module contains lesson notebooks; Modules 1–5 also contain a `studio/` directory with graphs for the LangGraph API and Studio.

## Machine assumptions

These notes target the author's Arch Linux installation:

- GNU Emacs 30.2
- Doom Emacs 3.0.0-pre
- Bash
- Python 3.13
- uv 0.12 or newer
- direnv 2.37 or newer

The pinned Python version lives in `.python-version`; exact Python dependencies live in `uv.lock`.

## 1. Install system prerequisites

```bash
sudo pacman -S --needed git uv direnv
```

Enable direnv in Bash by adding this line to `~/.bashrc`:

```bash
eval "$(direnv hook bash)"
```

Open a new shell, or reload the current one:

```bash
source ~/.bashrc
```

## 2. Clone this fork

```bash
git clone git@github.com:0xstubbs/langchain-academy.git
cd langchain-academy
```

The remotes in an existing development clone should be:

```text
origin    git@github.com:0xstubbs/langchain-academy.git
upstream  git@github.com:langchain-ai/langchain-academy.git
```

To incorporate upstream changes later:

```bash
git fetch upstream
git merge upstream/main
```

## 3. Create the uv environment

```bash
uv sync
```

This creates `.venv/` and installs the locked dependencies. There is no separate `python -m venv`, `pip install`, or package activation step. Commands can always be run explicitly through uv:

```bash
uv run python --version
uv run jupyter notebook
uv run langgraph dev
```

The committed `pyproject.toml` only describes the shared environment. It has no build system, console script, or `src/` package because the academy itself is not installed into `.venv`.

## 4. Configure project secrets with direnv

Create one root secrets file:

```bash
cp .env.example .env
$EDITOR .env
```

Populate the credentials required by the lessons:

```dotenv
OPENAI_API_KEY=...
LANGSMITH_API_KEY=...
LANGSMITH_TRACING_V2=true
LANGSMITH_PROJECT=langchain-academy
TAVILY_API_KEY=...
```

Both `.env` and `.venv/` are ignored by Git. Never commit real credentials.

The committed `.envrc` performs two operations:

```bash
export VIRTUAL_ENV="$PWD/.venv"
PATH_add "$VIRTUAL_ENV/bin"
dotenv_if_exists .env
```

Review it and authorize it once:

```bash
direnv allow
```

Afterward, entering the repository or any child directory exposes the project interpreter and root environment variables. The module-specific `langgraph.json` files inherit this environment, so duplicated `module-X/studio/.env` files are unnecessary.

Check the active environment with:

```bash
direnv status
command -v python
python --version
```

The interpreter should resolve to `langchain-academy/.venv/bin/python` and report Python 3.13.

## 5. Configure Doom Emacs

The private Doom configuration lives at `~/.config/doom/`. Enable these modules in `~/.config/doom/init.el`:

```emacs-lisp
:tools
(lsp +eglot)
direnv

:lang
(org
 +agenda
 +roam
 +journal
 +gnuplot
 +graphviz
 +pandoc
 +jupyter
 +noter
 +present
 +pretty
 +hugo)
(python +uv +lsp)
```

The flags serve different roles:

- `+uv` selects the project uv environment in Python buffers.
- `+lsp` enables Python language intelligence through the configured Eglot backend.
- `direnv` propagates the directory environment into Emacs buffers, subprocesses, and Org source-block execution.
- Org's `+jupyter` flag provides stateful Jupyter-backed Babel blocks and rich inline results.

After changing `init.el`, synchronize Doom and restart Emacs:

```bash
~/.emacs.d/bin/doom sync
```

Register the clone as a Doom project:

```text
SPC p a    add a known project
SPC p p    switch projects
```

Choose `~/langchain-academy/` when prompted.

## 6. Register the project Jupyter kernel

The uv environment contains IPython and `ipykernel`. Register it once so Emacs and Jupyter can identify it by a stable name:

```bash
uv run python -m ipykernel install --user \
  --name langchain-academy \
  --display-name "Python (langchain-academy)"
```

Verify discovery:

```bash
jupyter kernelspec list
```

If `.venv/` is deleted and recreated, repeat the kernelspec installation because the registered interpreter path may become stale.

## 7. Write an executable Org tutorial

Keep module-specific writing beside the material it explains, for example:

```text
module-1/
├── langgraph-foundations.org
├── tutorial-assets/
├── studio/
└── *.ipynb
```

Start an Org tutorial with document-wide Jupyter defaults:

```org
#+title: LangGraph Foundations
#+PROPERTY: header-args:jupyter-python :session academy
#+PROPERTY: header-args:jupyter-python+ :kernel langchain-academy
#+PROPERTY: header-args:jupyter-python+ :async yes
#+PROPERTY: header-args:jupyter-python+ :exports both
#+PROPERTY: header-args:jupyter-python+ :results replace

* Inspect the environment

#+begin_src jupyter-python
import sys
print(sys.executable)
print(sys.version)
#+end_src

* Preserve state between blocks

#+begin_src jupyter-python
values = [1, 2, 3, 4]
squares = [value**2 for value in values]
squares
#+end_src

#+begin_src jupyter-python
sum(squares)
#+end_src
```

Place point in a block and press `C-c C-c` to execute it. Blocks sharing `:session academy` use the same live kernel, so imports, variables, functions, and objects persist. Jupyter display values and plots are inserted as Org results.

Before publishing, restart the kernel and execute the document from top to bottom:

```text
M-x jupyter-org-restart-kernel-execute-buffer
```

That catches hidden dependencies on interactive execution order—the notebook equivalent of “restart and run all.” With `:exports both`, Org exporters include both source code and saved results. Doom's `+hugo`, `+pandoc`, and `+present` flags support turning the same source into blog posts, tutorials, documents, or presentations.

For deterministic publication assets, give important figures explicit paths beneath the module, such as `tutorial-assets/state-graph.png`.

## 8. Run LangGraph Studio

The Studio configurations for Modules 1–5 use Python 3.13 and inherit credentials from the root direnv environment. From a module's `studio/` directory:

```bash
cd module-1/studio
uv run langgraph dev
```

The development server normally reports:

```text
- API: http://127.0.0.1:2024
- Studio UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
- API Docs: http://127.0.0.1:2024/docs
```

Generated `.langgraph_api/` state is ignored by Git.

## 9. Original notebooks

The original `.ipynb` notebooks remain usable:

```bash
uv run jupyter notebook
```

The Org/Jupyter workflow supplements rather than removes them. It provides a Git-friendly plain-text source for commentary-heavy experiments, tutorials, and publishable articles while retaining a stateful kernel and rich output.

## API credentials

- [OpenAI API keys](https://platform.openai.com/api-keys)
- [LangSmith account and API key](https://docs.langchain.com/langsmith/create-account-api-key)
- [Tavily API](https://tavily.com/)

For the LangSmith EU instance, also set:

```dotenv
LANGSMITH_ENDPOINT=https://eu.api.smith.langchain.com
```

## Upstream resources

- [LangChain Academy](https://github.com/langchain-ai/langchain-academy)
- [LangGraph documentation](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangGraph Studio](https://docs.langchain.com/oss/python/langgraph/studio)
- [uv documentation](https://docs.astral.sh/uv/)
- [Doom Emacs](https://github.com/doomemacs/doomemacs)
