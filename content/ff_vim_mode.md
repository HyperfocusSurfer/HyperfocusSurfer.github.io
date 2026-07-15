+++
title = "Perfect keyboard-driven web browsing on NixOS"
date = 2026-07-15
[taxonomies]
tags = ["nix" "vim" "firefox"]
+++

In the life of every long term vim user there comes a moment when they start wanting
to bring modal editing to almost every program they often use. While trivial for
shells and many cli applications, browsers tend to be a bit more problematic in that regard.
Here's my descent into that rabbithole.

## Setup #1, switch browsers

There are projects like [vimb](https://github.com/fanglingsu/vimb) and [qutebrowser](https://github.com/qutebrowser/qutebrowser),
and they indeed do shine at being modal and keyboard-driven, but the ultimate deal-breaker for me was the lack of extensions,
at least when I tested those. Note that it happened like 8 years ago, so now the situation may be quite different.

Basically, after a few months I went back to good old firefox, since the ease of use and the power of
ublock, violentmonkey, sponsorblock and alike was too much to give up.

## Setup #2, webextensions

Naturally, I then looked into making firefox modal. First, I tried [vimium](https://github.com/philc/vimium), and was generally happy
up until I discovered [tridactyl](https://github.com/tridactyl/tridactyl) after a few more months. I more or less loved it at first
sight due to a great extensibility and the existence of the native client, which, among other things, provides the ability to open
protected browser pages from tridactyl's cmdline. However, there's a big "but": tridactyl itself still doesn't load on said pages.

For some pages, it can be mitigated, but others, like the built-in pdf viewer, various settings pages or even not-yet-loaded tabs remain out of its reach. Nevertheless, with occasional swearing I've been using it for the last couple of years.

## Setup #3, perfection

Finally, a couple of months ago I've stumbled upon [VimFx](https://github.com/akhodakivskiy/VimFx), a *legacy* firefox extension that seems to avoid these issues by not using the webextensions api at all, instead relying on the [LegacyFox](https://gir.st/blog/legacyfox.htm) compatibility shim, that brings the older API back. Installing it on NixOS, however, is a bit different from both the nix way of installing regular extensions and the guides provided in the repos. Anyways, here's what I came up with.

The snippet here targets librewolf + home manager, since it's my main browser of choice at the moment, but it shouldn't be too different from firefox-esr and, possibly, the regular firefox (not the bin version, tho, since it for doesn't support disabling xpi signature checks, IIRC). 

```nix
programs.librewolf = {
  enable = true;
  package = (
    pkgs.librewolf.overrideAttrs (
      prev:
      let
        legacyFox = pkgs.fetchgit {
          url = "https://git.gir.st/LegacyFox.git";
          hash = "sha256-vCRIiYdl7t3I5asndJBjSRVFu9ADBfSEkyKdlgbMxww=";
        };
        vimfx = pkgs.fetchurl {
          url = "https://github.com/akhodakivskiy/VimFx/releases/download/v0.27.6/VimFx.xpi";
          hash = "sha256-tC/98IKleMBZPdtEYVY8H2KmGKz3CyRhKjGSK2yUdx8=";
        };
      in
      {
        buildCommand = prev.buildCommand + ''
          libDir=$out/lib/librewolf
          cp -r ${legacyFox}/{legacy.manifest,legacy} $libDir/
          cp ${vimfx} $libDir/distribution/extensions/
          echo "// legacyFox installation" >> $libDir/mozilla.cfg
          cat ${legacyFox}/config.js >> $libDir/mozilla.cfg
        '';
      }
    )
  );
};

home.file.".librewolf/librewolf.overrides.cfg" = {
  text = ''
    // First line must be a comment
    pref("extensions.VimFx.config_file_directory", "/home/fl42v/.config/vimfx")
    pref("security.sandbox.content.read_path_whitelist", "/home/fl42v/.config/vimfx")
    pref("general.config.sandbox_enabled", false);
  '';
};
```

Basically, what it does is modifies the `pkgs.wrapFirefox` (which `pkgs.librewolf` is an instance of) to also install LegacyFox.
Then we use the fact that librewolf's mozilla.cfg includes `~/.librewolf/librewolf.overrides.cfg` to do the job of `defaults/pref/config-prefs.js`. Since this is librewolf-specific, the same lines can be added to mozilla.cfg or even the original location. I just found this way more simple to test.

### Zen-browser (and troubleshooting)

Since I also use zen mostly due to eyecandy and its better implementation of vertical tabs, I wanted to VimFx it as well, but the installation turned out to be more involved:

```nix
programs.Zen-browser = {
  enable = true;
  package = (pkgs.wrapFirefox (
    inputs.zen-browser.packages."${pkgs.stdenv.hostPlatform.system}".beta-unwrapped.overrideAttrs (
      prev:
      let
        legacyFox = pkgs.fetchgit {
          url = "https://git.gir.st/LegacyFox.git";
          hash = "sha256-vCRIiYdl7t3I5asndJBjSRVFu9ADBfSEkyKdlgbMxww=";
        };
        vimfx = pkgs.fetchurl {
          url = "https://github.com/akhodakivskiy/VimFx/releases/download/v0.27.6/VimFx.xpi";
          hash = "sha256-tC/98IKleMBZPdtEYVY8H2KmGKz3CyRhKjGSK2yUdx8=";
        };
      in
      {
        postInstall = ''
          libDir=$(find "$out/lib" -maxdepth 1 -name "zen-bin-*")
          chmod +w $libDir
          mkdir -p $libDir/distribution/extensions/
          chmod +w $libDir/distribution/extensions/
          mkdir -p $libDir/defaults/pref/
          chmod +w $libDir/defaults/pref/
          touch $libDir/legacyfox.js
          chmod +w $libDir/legacyfox.js
          touch $libDir/defaults/pref/config-prefs.js
          chmod +w $libDir/defaults/pref/config-prefs.js

          cp -r ${legacyFox}/{legacy.manifest,legacy} $libDir/
          cp ${vimfx} $libDir/distribution/extensions/
          # 'cause zen folks are too smart to follow the firefox versioning
          sed -i 's/115.0/1.0.0/' $libDir/distribution/extensions/*-VimFx.xpi
          echo "// legacyFox installation" >> $libDir/legacyfox.js
          cat ${legacyFox}/config.js >> $libDir/legacyfox.js

          sed 's/config\.js/legacyfox.js/' ${legacyFox}/defaults/pref/config-prefs.js >> $libDir/defaults/pref/config-prefs.js

        '';
      }
    )
  ) {});
};
```

Right off the bat, you can see the 1st difference: it's the unwrapped package that's overridden and only then wrapped with wrapFirefox.
The reason is both simple and not: for some reason even after being wrapped, zen tries to locate the `legacy.manifest` in the `-unwrapped` derivation. This might be zen-specific or applicable to all binary releases (I'm too lazy to check).

Now, the troubleshooting steps that led me to that finding. First of all, I checked if the installation of LegacyFox was successful by trying to navigate to the resource://legacy/ url. If it was, I should've seen a reguar-ish filesystem listing page, similar to what one sees on file:///home/<username>/some/directory/ urls. If the page is empty, the installation has failed.

After making sure the settings from `config-prefs.js` were visible in about:config and checking that the files were indeed there (nushell: `which zen-beta  | get path.0 | readlink $in | readlink $in | path dirname --num-levels 2 | glob $"($in)/lib/zen-bin-*" | get 0 | ls $in`), I decided to trace the manifest loading manually as follows:

1. <C-S-I> to open DevTools
2. F1 to open DevTools' settings
3. check the `Enable remote debugging` option
4. close DevTools
5. <C-A-S-I> to open DevTools for the browser itself, then allow the connection
6. go to console tab, then paste the `config.js`' contents line-by-line

The last step shoed me an error when the registering the manifest along with the path firefox tried to load it from.

The 2nd notable change is that the mozilla.cfg is replaced with `legacyfox.js`. The reason here is that wrapFirefox tries to create one and fails when it already exists. Using this name instead of `config.js` is a personal preference and can be disregarded.

Finally, here's using `sed` on VimFx.xpi. This is highly zen-specific and is basically a workaround for its devs deciding to reinvent the wheel [again](https://github.com/zen-browser/desktop/discussions/12025) by not following firefox' versioning for no reason, hence the addon fails to install due to its target firefox 115 seeming quite a bit newer than zen's 1.2.something. Also, I'm not entirely sure if placing the addon in distribution/extensions actually installs it. It may happen on the 2nd start of the browser or not at all, in which case you can always install it manually (not declarative, but at least it works).

## Next steps

While VimFx gives you a better way to interact with the browser itself, it doesn't tackle the text input. This can be mitigated with the help of [firenvim](https://github.com/glacambre/firenvim), which turns the browser into an actual neovim client (pretty much embeds nvim into input fields when triggered).
