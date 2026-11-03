# glycin

Ruby bindings for glycin, sandboxed image loading for GNOME.

```ruby
require "glycin"

Gly.constants.size  # the whole namespace, from the typelib
```

## How this works, and what it costs to install

The library ships a GObject Introspection typelib — `Gly-2.typelib` — describing
every class, method, property and signal it has. `Gly::Loader`, about twenty
lines, hands that to ruby-gnome's `GObjectIntrospection::Loader`, which defines
the Ruby classes from it at `require` time.

So this gem is pure Ruby: no extension of its own, no Cargo workspace, no
per-platform binary. You get the whole library rather than the subset somebody
got round to wrapping by hand, and it does not drift when the library gains API.

Be precise about install, though: `gem install glycin` still compiles C, because
`gobject-introspection` and `glib2` are themselves native extensions built
against glib, so a compiler and glib's development headers are needed. What
this gem adds on top is nothing — those two are what every ruby-gnome
application already has, since `gtk4` depends on them.

## The one thing you must install

The typelib is a C-library artefact, so it belongs to your distribution and
this gem cannot install it:

| | |
|---|---|
| Debian / Ubuntu | `libglycin-1 (typelib in gir1.2-gly-2)` |
| Fedora | `libglycin` |
| Arch | `libglycin` |
| nix | `pkgs.libglycin` |

If it is not in the default search path, point `GI_TYPELIB_PATH` at the
directory holding `Gly-2.typelib`. The dev shell in `flake.nix` does this.

**One trap worth knowing.** The typelib must come from the same glib build as
the `gobject-introspection` gem. Mixing two registers GIO twice in one process
and fails with `cannot register existing type 'GTlsCertificate'`. Under nix,
take both from one nixpkgs — and note that glib's typelibs live in its `out`
output, not the default one.

## Using it from a flake

The gem needs `Gly-2.typelib` and everything that typelib declares, resolved
transitively by the loader at `require` time. A missing one surfaces as
`Typelib file for namespace 'X' not found`, one at a time, so the whole chain
is listed here rather than discovered by repeated failure:

- `pkgs.libglycin` — its typelib, which is where `Gly-2.typelib` lives
- `pkgs.glib` — GLib-2.0, GObject-2.0 and Gio-2.0, which nearly every GNOME typelib declares

Note the `.out`: a package's default output is often `bin`, which carries no
typelibs. `makeSearchPath` takes the first output, so it has to be asked for
`out` explicitly — this is the single most common way to get a search path that
looks right and finds nothing.

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

  outputs = { nixpkgs, ... }:
    let
      system = "x86_64-linux";
      pkgs = nixpkgs.legacyPackages.${system};

      typelibPath = pkgs.lib.makeSearchPath "lib/girepository-1.0"
        (map (drv: drv.out or drv) [ pkgs.libglycin pkgs.glib ]);
    in
    {
      devShells.${system}.default = pkgs.mkShell {
        packages = [ pkgs.ruby_3_4 pkgs.bundler ] ++ [ pkgs.libglycin pkgs.glib ];

        shellHook = ''
          export GI_TYPELIB_PATH="${typelibPath}''${GI_TYPELIB_PATH:+:$GI_TYPELIB_PATH}"
        '';
      };
    };
}
```

Everything must come from **one** nixpkgs. A typelib built against a different
glib than the `gobject-introspection` gem registers GIO twice in the process
and fails with `cannot register existing type 'GTlsCertificate'`.

If you are packaging an application rather than entering a shell, the same
`typelibPath` goes on the wrapper:

```nix
makeWrapper ${ruby}/bin/ruby $out/bin/my-app \
  --set GI_TYPELIB_PATH "${typelibPath}"
```

## Development

```sh
nix develop
bin/test
```

## Releasing

```sh
gem kit release
```

That is the whole thing. It gates on the changelog having an entry for this
version, on no deprecation deadline coming due, and on a clean working tree;
then builds, pushes and tags.

There is nothing to wait for in CI. This gem is pure Ruby, so `gem build`
produces one `glycin-X.Y.Z.gem` that is the same artefact wherever it is built —
no extension, no per-platform gems, nothing for a workflow to hand back.


## Licence

Apache-2.0. The library itself keeps its own licence; this gem links nothing
and copies no code from it — it reads a description of it at runtime.
