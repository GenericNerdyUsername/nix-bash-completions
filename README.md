# nix-bash-completions
Bash completion for the [Nix](https://nixos.org/nix/), [NixOS](https://nixos.org/) and [NixOps](https://nixos.org/nixops/) command line tools.

A lot of the boilerplate (options etc.) comes from [nix-zsh-completions](https://github.com/spwhitt/nix-zsh-completions).

## Usage

Just source the `_nix` file: `. _nix`, and start tabbing.

You need [bash-completion](https://github.com/scop/bash-completion) for it to work (which is the case if most commands already provides completions). On NixOS this is done by setting `programs.bash.enableCompletion`.

The script also depends on `sed` being in the path.

## Attribute path completion

Completion of attribute paths is context aware, so supplying eg. `-f some/path/` will make `-A ` complete paths in `some/path/default.nix`, and will pick up `default.nix` or `shell.nix` in the current directory for `nix-build` and `nix-shell`.

## Implementation

The script runs on top of `_parser` which is a bare bones implementation of zsh's [`_arguments`](http://zsh.sourceforge.net/Doc/Release/Completion-System.html#Completion-Functions) with some minor modifications to the syntax, and a bunch of stuff not implemented. A brief description of the `_parser` syntax follows, for anyone interested in reading the code, or using `_parser` for other completion scripts.

### `_parser` syntax
`_parser` takes as arguments specifications of options and normal arguments.

The spec looks like this at the moment (`[]` denotes optional syntax):

Argument spec:
- `:action`

Argument specs tell `_parser` how to handle normal arguments (arguments not required by options). The first argument will be handled by the first `:action` passed to `_parser`, the second by the second argument spec passed, and so on. Actions starting with a `*` will handle all further arguments. `_parser` puts all the encountered normal arguments in the array `$line` for easy lookup when handling further completion.

There's two types of actions:

- `[*]->string`
- `[*]_function`

When completing an argument with a `->string` action `_parser` will set `$state` to `string`, return `1` and hand over control to the calling function which then need to handle the actual completion.

The `[*]_function_` spec causes `_function` to be called, making it handle completion. At the moment this isn't that useful as passing options to `_function` isn't supported. `->string` actions together with a case statement is much more flexible.

An example using the different actions:
```shell
local -a line
_parser ':_known_hosts' ':*->FILE' && return
case "$state" in
    FILE)
        COMPREPLY=($(compgen -f $cur))
esac
```
Here the function `_known_hosts` will handle completion for the first argument and then all further arguments will complete files through the `$state` case.

The options spec have a few more pieces on top of actions:
- `[(pattern|pattern ...)][*]--option[:action[:action2] ... ]`
The simplest case just being:
- `--option`

Which tells `_parser` to add `--option` when completing options.

If the option is present on the command line further option completion will exclude options matching any of the `pattern`s. By default `--option` will exclude itself. If we want an option to be repeatable we add the `*` prefix to the option.

Adding actions to an option spec tells `_parser` that the option takes arguments. These actions are specified in the exact same way as the argument spec. Option arguments can be looked up in the `$opt_args` associative array using the option as a key.

A reduced version of `nix-shell` spec:
```shell
local -a line
local -A opt_args
_parser ':*->FILE' '(--attr|-A)'{--attr,-A}':->ATTR_PATH' \
       '(--packages|-p|shell)'{--packages,-p}':*->PACKAGE_ATTR_PATH') && return
case "$state"
```

At the moment `_parser` only understands long options, eg. `--option`, and stackable short options, eg. `-f`. Syntax like `-some-option` is not supported.

## Issues

- Only the first short option is completed, but eg. `nix-env -iA` is recognized, and `-i` won't be offered on any new option completion. Completing stacked options is probably not easily doable in bash.
