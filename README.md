# Corkey

Corkey is [Corgi](https://github.com/corgi-emacs/corgi)'s key binding system.

The main ideas behind Corkey:

- Define key bindings as "just data", in dedicated key binding files
- Never define keys implicitly just by loading a package
- Unify evil and plain emacs bindings
- Provide which-key style discoverability, with docstrings being first class
- Decouple conceptual operations (e.g. "eval current form") from concrete and mode-specific implementations

The last point is achieved to a Corkey-specific concept: signals, which we'll elaborate on further down.

## Getting started

Create a file named `user-keys.el` and place it in your Emacs user directory
(`user-emacs-directory`), or anywhere on Emacs's `load-path`.

```emacs-lisp
(bindings
 ;; "global" bindings are always active regardless of Evil's "state" (= vim mode)
 ;; If you don't provide this the default is `normal'.
 (global
  ("M-RET" projectile-switch-project))

 ;; Bindings for commands are usually only active in normal and visual state.
 (normal|visual
  ("SPC"
   ("o" "Open"
    ("u" "Open URL at point" browse-url-at-point)
    ("s" "Edit string at point" string-edit-at-point)))))
```

Now start Corkey:

``` emacs-lisp
(corkey-mode)
(corkey/load-and-watch '(user-keys))
```

`corkey-mode` is a globalized minor mode which will enable Corkey in all buffers.

Bindings are nested, e.g. `("SPC" ("b" ("k" kill-buffer)))` means that "space"
followed by "b" and then "k" will invoke `M-x kill-buffer`. Instead of a prefix
key (string) you can use an evil state name (what vim calls a "mode", should be
a symbol). `global` is a special case, it means "these bindings should work
regardless of the evil state". To apply bindings to multiple states, separate
them with a `|`.

You can add a descriptions before the command, this will show up in a pop-up
when you press the prefix key and wait a bit. (This uses which-key)

Now `M-RET` should work anywhere for switching to another project, and `SPC o u`
/ `SPC o s` will work when you are in evil's `normal` or `visual` state. If you
press `SPC o` and wait you will get a popup showing your options, with
human-readable descriptions.

To add or change bindings, simply update and save `user-keys.el`, and Corkey
will reload it.

## Signals

Instead of a concrete command like `kill-buffer` or `projectile-switch-project`
you can put a keyword like `:eval/buffer` (notice the leading colon). Corkey
calls this a "signal". It's a way of saying "conceptually what this key does is
evaluate the current buffer, but what that does in a given instance is
context-specific".

``` emacs-lisp
;; user-keys.el
(bindings
 ("," ("e" ("b" "Evaluate buffer" :eval/buffer))))
```

At this point this doesn't do anything yet, Corkey will not create a `, e e`
binding. For that you need to define what this signal does.

``` emacs-lisp
;; user-signals.el
((emacs-lisp-mode (:eval/buffer eval-buffer))
 (clojure-mode    (:eval/buffer cider-eval-buffer)))
```

Now install this signals file, the optional second argument to
`corkey/load-and-watch` is a list of signal file names:

``` emacs-lisp
(corkey/load-and-watch '(user-keys) '(user-signals))
```

Now it will stitch the two together, in Emacs Lisp buffers `, e e` will call
eval-buffer, whereas in Clojure buffers it will call `cider-eval-buffer`.

You can bind signals per major or minor mode. Use `default` to provide a
fallback value, if no specific mode applies.

Having this indirection promotes a degree of consistency that in other configs
is achieved by manually ensuring analogous bindings across modes, which is
difficult to enforce and maintain, and where it's often left to contributors to
try to deduce the conventions used by other modes.

## Layering

Note that the arguments to `corkey/load-and-watch` are both lists, it is
possible to provide multiple key files, and multiple signal files, for instance
you can load both the `corgi-keys` and `user-keys` keys files, and both the
`corgi-signals` and `user-signals` signal files. In fact this is the default
when calling `corkey/load-and-watch` without arguments.

``` emacs-lisp
(corkey/load-and-watch '(corgi-keys user-keys) '(corgi-signals user-signals))
```

This will cause Corkey to layer later files over earlier ones, in other words:
any definitions in `user-keys` and `user-signals` will take precedence over
`corgi-keys` and `corgi-signals`.

Typically the `corgi-*` files are provided by the `corgi-bindings` package,
installed via straight. These provide the base set of bindings for Corgi. The
`user-*` files are files you can place in your `emacs-user-directory`, to add
your own customizations. 

For any file name `emacs-user-directory` will always be searched first, followed
by scanning the emacs `load-path`. This means you can also copy the `corgi-*`
files to your `emacs-user-directory` to customize them directly.

File names that Corkey can't find are currently silently ignored. This means you
can use the default values even if you don't have a `user-keys.el` or
`user-signals.el`. If you decide to create them later they will be picked up
then.

## Signal and third party packages

We've already explained the main purpose of signals, to allow mode-specific
bindings. But signals become really interesting when considering the
implications for third party packages.

### Providing signal bindings

Say someone wants to create a Corgi integration for a programming language that
isn't covered yet by the base setup. That means they just need to create a
package that contains a `my-lang-signals.el`

```emacs-lisp
((my-lang-mode (:eval/last-sexp my-lang-eval-last-sexp
                :eval/buffer my-lang-eval-buffer
                :eval/region my-lang-eval-region
                :repl/connect my-lang-connect-repl
                :repl/jack-in my-lang-start-repl)))
```

Now a Corkey user can load this file

```emacs-lisp
(corkey/load-and-watch '(corgi-keys user-keys) '(corgi-signals my-lang-signals user-signals))
```

And get all their familiar Corgi bindings available for said language. But
what's more, if they have user-specific bindings (maybe they have a different
preference for how to eval a form), then this will also "just work" for the new
language.

### Providing alternate key maps

Say someone has particularly fond memories of using Notepad, and wants to create
a keys file, so that others can enjoy it too.

```clj
;; notepad-keys.el
(bindings 
 (global
  ("C-o" :file/open)
  ("C-s" :file/save)
  ...))
```

Now a Corkey user can load this file

```emacs-lisp
(corkey/load-and-watch '(notepad-keys) '(corgi-signals user-signals))
```

Now currently (this is might change) Corgi uses `counsel-find-file` to open a
file, but maybe the user has decided to use something else than Counsel, so they
can declare that in their `user-signals`, and the `C-o` binding will now honor
that.

## Undefined commands

If Corkey can not find a listed command, i.e. if the package that provides it is
not loaded when your key bindings are loaded, then this will simply be ignored,
and no binding for the given keys will be made.

This is intentional, this allows us to define bindings in Corgi-bindings for
various packages, even though not everyone may load all of those packages.

## keys-file grammar

```
BINDINGS  := '(bindings' <def>+ ')'
<def>     := '(' <key> <doc> <target> ')' | '(' <prefix> <def> + ')'
<target>  := <signal> | <command>
<prefix>  := <state> | <key>
<state>   := 'normal' | 'insert' | 'visual' | 'emacs' | 'motion' | 'global'
<key>     := stringp
<doc>     := stringp
<signal>  := keywordp
<command> := symbolp
```

<!-- license -->
## License

Copyright &copy; 2020-2022 Arne Brasseur and Contributors

Licensed under the term of the GNU General Public License, version 3. 
<!-- /license -->
