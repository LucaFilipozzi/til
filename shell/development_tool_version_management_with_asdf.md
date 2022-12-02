# development tool version management with `asdf`

It runs counter to my Debian upbringing (package all the things; install once; leverage security updates) but,
unfortunatly, the global development community has eschewed OS pakcage managers (the packages aren't up to date,
they cry!) in favour of utilities such as `pip` to manage local installations of libraries / dependencies for
development tools (`python` in the case of `pip`).

`asdf` extends this paradigm to the development tools themselves.

[asdf-vm/asdf][1], an extensible development tool version manager with support for many languages (Erlang, Node.js,
Python, Ruby, etc. to name a few). Coupled with [direnv/direnv][2], `asdf` provies a 
consistent and predictable (per directory) approach to environment management. `direnv`, itself should be
managed with asdf via [asdf-community/asdf-direnv][3].

# example

## install and activate direnv

```
# install the latest direnv
asdf plugin add direnv
asdf install direnv latest

# use direnv in the project
cd /path/to/project/directory
asdf local direnv latest
```

## install and activate python

```
# install the latest python
asdf plugin add python
asdf install python latest

# use python in the project
cd /path/to/project/directory/
asdf local python latest
```

## .envrc

Now use `pyenv` or, better `layout python3` in .envrc to get a python environment for the project.

```
# .envrc
use asdf
layout python3
```

(It'd be elegant if the second directive were `use python`.)

---
Copyright (c) 2022 Luca Filipozzi

[1]: https://github.com/asdf-vm/asdf
[2]: https://github.com/direnv/direnv
[3]: https://github.com/asdf-community/asdf-direnv
[4]: https://gitlab.com/gitlab-org/gitlab-development-kit/-/blob/main/doc/migrate_to_asdf.md
