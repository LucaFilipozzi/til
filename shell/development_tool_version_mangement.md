# Development Tool Version Management

It runs counter to my Debian upbringing (package all the things; install once; leverage security updates) but,
unfortunatly, the global development community has eschewed OS pakcage managers (the packages aren't up to date,
they cry!) in favour of utilities such as `nvm`, `pyenv`, `rbenv`, etc. to manage local installations of
development tools. The GitLab Development Kit has [migrated to `asdf`][4], for example.

Although they deliver the same basic functionatliy -- development version management -- each of these utilities
has adifferent command line interface and idiosyncracies.

Enter [asdf-vm/asdf][1], an extensible version manager with support for many languages (Erlang, Node.js,
Python, Ruby, etc. to name a few) & other development tools. Coupled with [direnv/direnv][2], `asdf` provies a 
consistent and predictable (per directory) approach to environment management. `direnv`, itself should be
managed with asdf via [asdf-community/asdf-direnv][3].

[1]: https://github.com/asdf-vm/asdf
[2]: https://github.com/direnv/direnv
[3]: https://github.com/asdf-community/asdf-direnv
[4]: https://gitlab.com/gitlab-org/gitlab-development-kit/-/blob/main/doc/migrate_to_asdf.md
