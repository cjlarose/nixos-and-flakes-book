# Modularize Your NixOS Configuration

At this point, the skeleton of the entire system is configured. The current configuration structure in `/etc/nixos` should be as follows:

```
$ tree
.
├── flake.lock
├── flake.nix
├── home.nix
└── configuration.nix
```

The functions of these four files are:

- `flake.lock`: An automatically generated version-lock file that records all input sources, hash values, and version numbers of the entire flake to ensure reproducibility.
- `flake.nix`: The entry file that will be recognized and deployed when executing `sudo nixos-rebuild switch`. See [Flakes - NixOS Wiki](https://nixos.wiki/wiki/Flakes) for all options of flake.nix.
- `configuration.nix`: Imported as a Nix module in flake.nix, all system-level configuration is currently written here. See [Configuration - NixOS Manual](https://nixos.org/manual/nixos/unstable/index.html#ch-configuration) for all options of configuration.nix.
- `home.nix`: Imported by Home-Manager as the configuration of the user `ryan` in flake.nix, containing all of `ryan`'s configuration and managing `ryan`'s home folder. See [Appendix A. Configuration Options - Home-Manager](https://nix-community.github.io/home-manager/options.html) for all options of home.nix.

By modifying these files, you can declaratively change the system and home directory status.

However, as the configuration grows, relying solely on `configuration.nix` and `home.nix` can lead to bloated and difficult-to-maintain files. A better solution is to use the Nix module system to split the configuration into multiple Nix modules and write them in a classified manner.

The Nix module system provides a parameter, `imports`, which accepts a list of `.nix` files and merges all the configuration defined in these files into the current Nix module. Note that `imports` will not simply overwrite duplicate configuration but handle it more reasonably. For example, if `program.packages = [...]` is defined in multiple modules, then `imports` will merge all `program.packages` defined in all Nix modules into one list. Attribute sets can also be merged correctly. The specific behavior can be explored by yourself.

> I only found a description of `imports` in [Nixpkgs-Unstable Official Manual - evalModules Parameters](https://nixos.org/manual/nixpkgs/unstable/#module-system-lib-evalModules-parameters): `A list of modules. These are merged together to form the final configuration.` It's a bit ambiguous...

With the help of `imports`, we can split `home.nix` and `configuration.nix` into multiple Nix modules defined in different `.nix` files. Lets look at an example module `packages.nix`:

```nix
{
  config,
  pkgs,
  ...
}: {
  imports = [
    (import ./special-fonts-1.nix {inherit config pkgs}) # (1)
    ./special-fonts-2.nix # (2)
  ];

  fontconfig.enable = true;
}
```

This modules loads two other modules in the imports section, namely `special-fonts-1.nix` and `special-fonts-2.nix`. Both files are modules them self and look similar to this.

```nix
{ config, pkgs, ...}: {
    # Configuration stuff ...
}
```

Both import statements above are equivalent in the parameters they receive:

- Statement `(1)` imports the function in `special-fonts-1.nix` and calls it by passing `{config = config; pkgs = pkgs}`. Basically using the return value of the call (another partial configuration [attritbute set]) inside the `imports` list.

- Statement `(2)` defines a path to a module, whose function Nix will load _automatically_ when assembling the configuration `config`. It will pass all matching arguments from the function in `packages.nix` to the loaded function in `special-fonts-2.nix` which results in `import ./special-fonts-2.nix {config = config; pkgs = pkgs}`.

Here is a nice starter example of modularizing the configuration, Highly recommended:

- [Misterio77/nix-starter-configs](https://github.com/Misterio77/nix-starter-configs)

A more complicated example, [ryan4yin/nix-config/i3-kickstarter](https://github.com/ryan4yin/nix-config/tree/i3-kickstarter) is the configuration of my previous NixOS system with the i3 window manager. Its structure is as follows:

```shell
├── flake.lock
├── flake.nix
├── home
│   ├── default.nix         # here we import all submodules by imports = [...]
│   ├── fcitx5              # fcitx5 input method's configuration
│   │   ├── default.nix
│   │   └── rime-data-flypy
│   ├── i3                  # i3 window manager's configuration
│   │   ├── config
│   │   ├── default.nix
│   │   ├── i3blocks.conf
│   │   ├── keybindings
│   │   └── scripts
│   ├── programs
│   │   ├── browsers.nix
│   │   ├── common.nix
│   │   ├── default.nix   # here we import all modules in programs folder by imports = [...]
│   │   ├── git.nix
│   │   ├── media.nix
│   │   ├── vscode.nix
│   │   └── xdg.nix
│   ├── rofi              #  rofi launcher's configuration
│   │   ├── configs
│   │   │   ├── arc_dark_colors.rasi
│   │   │   ├── arc_dark_transparent_colors.rasi
│   │   │   ├── power-profiles.rasi
│   │   │   ├── powermenu.rasi
│   │   │   ├── rofidmenu.rasi
│   │   │   └── rofikeyhint.rasi
│   │   └── default.nix
│   └── shell             # shell/terminal related configuration
│       ├── common.nix
│       ├── default.nix
│       ├── nushell
│       │   ├── config.nu
│       │   ├── default.nix
│       │   └── env.nu
│       ├── starship.nix
│       └── terminals.nix
├── hosts
│   ├── msi-rtx4090      # My main machine's configuration
│   │   ├── default.nix  # This is the old configuration.nix, but most of the content has been split out to modules.
│   │   └── hardware-configuration.nix  # hardware & disk related configuration, autogenerated by nixos
│   └── nixos-test       # my test machine's configuration
│       ├── default.nix
│       └── hardware-configuration.nix
├── modules          # some common NixOS modules that can be reused
│   ├── i3.nix
│   └── system.nix
└── wallpaper.jpg    # wallpaper
```

There is no need to follow the above structure, you can organize your configuration in any way you like. The key is to use `imports` to import all the submodules into the main module.

## `lib.mkOverride`, `lib.mkDefault`, and `lib.mkForce`

In Nix, some people use `lib.mkDefault` and `lib.mkForce` to define values. These functions are designed to set default values or force values of options.

You can explore the source code of `lib.mkDefault` and `lib.mkForce` by running `nix repl -f '<nixpkgs>'` and then entering `:e lib.mkDefault`. To learn more about `nix repl`, type `:?` for the help information.

Here's the source code:

```nix
  # ......

  mkOverride = priority: content:
    { _type = "override";
      inherit priority content;
    };

  mkOptionDefault = mkOverride 1500; # priority of option defaults
  mkDefault = mkOverride 1000; # used in config sections of non-user modules to set a default
  mkImageMediaOverride = mkOverride 60; # image media profiles can be derived by inclusion into host config, hence needing to override host config, but do allow user to mkForce
  mkForce = mkOverride 50;
  mkVMOverride = mkOverride 10; # used by ‘nixos-rebuild build-vm’

  # ......
```

In summary, `lib.mkDefault` is used to set default values of options with a priority of 1000 internally, and `lib.mkForce` is used to force values of options with a priority of 50 internally. If you set a value of an option directly, it will be set with a default priority of 1000, the same as `lib.mkDefault`.

The lower the `priority` value, the higher the actual priority. As a result, `lib.mkForce` has a higher priority than `lib.mkDefault`. If you define multiple values with the same priority, Nix will throw an error.

Using these functions can be very helpful for modularizing the configuration. You can set default values in a low-level module (base module) and force values in a high-level module.

For example, in my configuration at [ryan4yin/nix-config/blob/c515ea9/modules/nixos/core-server.nix](https://github.com/ryan4yin/nix-config/blob/c515ea9/modules/nixos/core-server.nix#L32), I define default values like this:

```nix{6}
{ lib, pkgs, ... }:

{
  # ......

  nixpkgs.config.allowUnfree = lib.mkDefault false;

  # ......
}
```

Then, for my desktop machine, I override the value in [ryan4yin/nix-config/blob/c515ea9/modules/nixos/core-desktop.nix](https://github.com/ryan4yin/nix-config/blob/c515ea9/modules/nixos/core-desktop.nix#L18) like this:

```nix{10}
{ lib, pkgs, ... }:

{
  # import the base module
  imports = [
    ./core-server.nix
  ];

  # override the default value defined in the base module
  nixpkgs.config.allowUnfree = lib.mkForce true;

  # ......
}
```

## `lib.mkOrder`, `lib.mkBefore`, and `lib.mkAfter`

In addition to `lib.mkDefault` and `lib.mkForce`, there are also `lib.mkBefore` and `lib.mkAfter`, which are used to set the merge order of **list-type options**. These functions further contribute to the modularization of the configuration.

As mentioned earlier, when you define multiple values with the same **override priority**, Nix will throw an error. However, by using `lib.mkOrder`, `lib.mkBefore`, or `lib.mkAfter`, you can define multiple values with the same override priority, and they will be merged in the order you specify.

To examine the source code of `lib.mkBefore`, you can run `nix repl -f '<nixpkgs>'` and then enter `:e lib.mkBefore`. To learn more about `nix repl`, type `:?` for the help information:

```nix
  # ......

  mkOrder = priority: content:
    { _type = "order";
      inherit priority content;
    };

  mkBefore = mkOrder 500;
  mkAfter = mkOrder 1500;

  # The default priority for things that don't have a priority specified.
  defaultPriority = 100;

  # ......
```

Therefore, `lib.mkBefore` is a shorthand for `lib.mkOrder 500`, and `lib.mkAfter` is a shorthand for `lib.mkOrder 1500`.

To test the usage of `lib.mkBefore` and `lib.mkAfter`, let's create a simple Flake project:

```shell{16-29}
# Create flake.nix with the following content
› cat <<EOF | sudo tee flake.nix
{
  description = "Ryan's NixOS Flake";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.05";
  };

  outputs = { self, nixpkgs, ... }@inputs: {
    nixosConfigurations = {
      "nixos-test" = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";

        modules = [
          # Demo module 1: insert 'git' at the head of the list
          ({lib, pkgs, ...}: {
            environment.systemPackages = lib.mkBefore [pkgs.git];
          })

          # Demo module 2: insert 'vim' at the tail of the list
          ({lib, pkgs, ...}: {
            environment.systemPackages = lib.mkAfter [pkgs.vim];
          })

          # Demo module 3: simply add 'curl' to the list
          ({lib, pkgs, ...}: {
            environment.systemPackages = with pkgs; [curl];
          })
        ];
      };
    };
  };
}
EOF

# Create flake.lock
› nix flake update

# Enter the nix repl environment
› nix repl
Welcome to Nix 2.13.3. Type :? for help.

# Load the flake we just created
nix-repl> :lf .
Added 9 variables.

# Check the order of systemPackages
nix-repl> outputs.nixosConfigurations.nixos-test.config.environment.systemPackages
[ «derivation /nix/store/0xvn7ssrwa0ax646gl4hwn8cpi05zl9j-git-2.40.1.drv»
  «derivation /nix/store/7x8qmbvfai68sf73zq9szs5q78mc0kny-curl-8.1.1.drv»
  «derivation /nix/store/bly81l03kh0dfly9ix2ysps6kyn1hrjl-nixos-container.drv»
  ......
  ......
  «derivation /nix/store/qpmpv

q5azka70lvamsca4g4sf55j8994-vim-9.0.1441.drv» ]
```

As you can see, the order of `systemPackages` is `git -> curl -> default packages -> vim`, which matches the order we defined in `flake.nix`.

> Although adjusting the order of `systemPackages` may not be useful in practice, it can be helpful in other scenarios.

## References

- [Nix modules: Improving Nix's discoverability and usability](https://cfp.nixcon.org/nixcon2020/talk/K89WJY/)
- [Module System - Nixpkgs](https://github.com/NixOS/nixpkgs/blob/eab660d/doc/module-system/module-system.chapter.md)
