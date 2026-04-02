# 9sense

`9sense` is a fork point for the tiling window manager described in `TILEWM-SPEC.md`.

This directory is cloned from `rio` so it keeps Rio's working 9P window file-server core (`fsys.c`, `xfid.c`, `wctl.c`), which the spec calls out as critical to preserve.

## Install (user-managed, not baked into system)

By default, the `mkfile` installs into a per-user path:

- `BIN=/usr/$user/bin/$objtype`

So running `mk install` installs `9sense` for the current user, instead of replacing system `rio` in `/$objtype/bin`.

```sh
cd /sys/src/cmd/9sense
mk
mk install
```

To install elsewhere, override `BIN` at install time:

```sh
mk install BIN=/some/other/path
```
