+++
title = "Nix whacks"
date = 2025-04-28
[taxonomies]
tags = ["nix"]
+++

A loose collection of useful nix snippets.

## Patch-in a PR while building a package

Say, we want to include the [147th PR to done.fish](https://github.com/franciscolourenco/done/pull/147)

* append `.diff` or `.patch` to the url, which will redirect you to [https://patch-diff.githubusercontent.com/raw/franciscolourenco/done/pull/147.diff](https://patch-diff.githubusercontent.com/raw/franciscolourenco/done/pull/147.diff)
* pull and apply the patch via nix:

```nix
(fishPlugins.done.overrideAttrs (final: prev: {
  patches = [(fetchpatch {
    url = "https://patch-diff.githubusercontent.com/raw/franciscolourenco/done/pull/147.diff";
    hash = "sha256-tIXVfXLSLwc54Z3HZcfwgGv1vEzAaBVNAjeq4vVNxNI=";
  })];
}))
```

[[Source](https://discourse.nixos.org/t/easiest-way-to-include-nixpkgs-pull-request/39307/3)]

## Overriding non-trivial builders

Default `overrideAttrs` is applied to the result of various `buildRustPackage`s and `buildGoModule`s, so if you want to override some builder-specific stuff (like `cargoHash`), you need to `override` the builder instead:

```nix
(tectonic-unwrapped.override (old: {
  rustPlatform = old.rustPlatform // {
    buildRustPackage = args: old.rustPlatform.buildRustPackage (args // {
      # override src/cargoHash/buildFeatures here
    });
  };
}))
```

[[Source](https://discourse.nixos.org/t/is-it-possible-to-override-cargosha256-in-buildrustpackage/4393/9)]
