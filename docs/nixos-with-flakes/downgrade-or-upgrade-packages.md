# Downgrading or Upgrading Packages

After using Flakes, most people are currently using the `nixos-unstable` branch of nixpkgs. However, sometimes you may encounter bugs, such as the [chrome/vscode crash problem](https://github.com/swaywm/sway/issues/7562).

To resolve these issues, you may need to downgrade or upgrade some packages. In Flakes, all package versions and hash values correspond one-to-one to the git commit of their flake input. Therefore, to downgrade or upgrade a package, you need to lock the git commit of its flake input.

For example, you can add multiple nixpkgs, each using a different git commit or branch:

```nix
{
  description = "NixOS configuration of Ryan Yin"

  inputs = {
    # default to nixos-unstable branch
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

    # the latest stable branch of nixpkgs, used to rollback the version of some packages
    # the current latest version is 22.11
    nixpkgs-stable.url = "github:nixos/nixpkgs/nixos-22.11";

    # we can also use git commit hash to lock the version
    nixpkgs-fd40cef8d.url = "github:nixos/nixpkgs/fd40cef8d797670e203a27a91e4b8e6decf0b90c";
  outputs = inputs@{
    self,
    nixpkgs,
    nixpkgs-stable,
    nixpkgs-fd40cef8d,
    ...
  }: {
    nixosConfigurations = {
      nixos-test = nixpkgs.lib.nixosSystem rec {
        system = "x86_64-linux";

        # The core parameter, which passes the non-default nixpkgs instances to other nix modules
        specialArgs = {
          # To use packages from nixpkgs-stable, we need to configure some parameters for it first
          pkgs-stable = import nixpkgs-stable {
            system = system;  # refer the `system` parameter form outer scope recursively
            # To use chrome, we need to allow the installation of non-free software
            config.allowUnfree = true;
          };
          pkgs-fd40cef8d = import nixpkgs-fd40cef8d {
            system = system;
            config.allowUnfree = true;
          };
        };
        modules = [
          ./hosts/nixos-test

          # omit other configuration...
        ];
      };
    };
  };
}
```

And then refer to the packages from `pkgs-stable` or `pkgs-fd40cef8d` in your sub module, a home manager's sub module as an example:

```nix
{
  pkgs,
  config,
  # nix will search and jnject this parameter from specialArgs in flake.nix
  pkgs-stable,
  # pkgs-fd40cef8d,
  ...
}:

{
  # refer packages from pkgs-stable instead of pkgs
  home.packages = with pkgs-stable; [
    firefox-wayland

    # chrome wayland support was broken on nixos-unstable branch, so fallback to stable branch for now
    # https://github.com/swaywm/sway/issues/7562
    google-chrome
  ];

  programs.vscode = {
    enable = true;
    package = pkgs-stable.vscode;  # refer vscode from pkgs-stable instead of pkgs
  };
}
```

After adjusting the configuration, deploy it with `sudo nixos-rebuild switch`. This will downgrade your Firefox/Chrome/VSCode to the version corresponding to `nixpkgs-stable` or `nixpkgs-fd40cef8d`.

>According to [1000 instances of nixpkgs](https://discourse.nixos.org/t/1000-instances-of-nixpkgs/17347), it's not a good practice to use `import` in submodules to customize `nixpkgs`. Each `import` creates a new instance of nixpkgs, which increases build time and memory usage as the configuration grows. To avoid this problem, we create all nixpkgs instances in `flake.nix`.
