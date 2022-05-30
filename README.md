# Nixinate 🕶️

Nixinate is a proof of concept that generates a deployment script for each
`nixosConfiguration` you already have in your flake, which can be ran via `nix
run`, thanks to the `apps` attribute of the [flake
schema](https://nixos.wiki/wiki/Flakes#Flake_schema).

## Usage

To add and configure `nixinate` in your own flake, you need to:

1. Add the result of `nixinate self` to the `apps` attribute of your flake.
2. Add and configure `_module.args.nixinate` to the `nixosConfigurations` you want to deploy

Below is a minimal example:

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-21.11";
    nixinate.url = "github:matthewcroughan/nixinate";
  };

  outputs = { self, nixpkgs, nixinate }: {
    apps = nixinate.nixinate.x86_64-linux self;
    nixosConfigurations = {
      myMachine = nixpkgs.lib.nixosSystem {
        modules = [
          (import ./my-configuration.nix)
          {
            _module.args.nixinate = {
              host = "itchy.scratchy.com";
              sshUser = "matthew";
              buildOn = "remote"; # valid args are "local" or "remote"
	            hermetic = false;
            };
          }
          # ... other configuration ...
        ];
      };
    };
  };
}
```

Each `nixosConfiguration` you have configured should have a deployment script in
`apps.nixinate`, visible in `nix flake show` like this:

```
$ nix flake show 
git+file:///etc/nixos
├───apps
│   └───nixinate
│       └───myMachine: app
└───nixosConfigurations
    └───myMachine: NixOS configuration
```

To finally execute the deployment script, use `nix run .#apps.nixinate.myMachine`

#### Example Run

```
[root@myMachine:/etc/nixos]# nix run .#apps.nixinate.myMachine
🚀 Deploying nixosConfigurations.myMachine from /nix/store/279p8aaclmng8kc3mdmrmi6q3n76r1i7-source
👤 SSH User: matthew
🌐 SSH Host: itchy.scratchy.com
🚀 Sending flake to myMachine via nix copy:
(matthew@itchy.scratchy.com) Password: 
🤞 Activating configuration on myMachine via ssh:
(matthew@itchy.scratchy.com) Password: 
[sudo] password for matthew: 
building the system configuration...
activating the configuration...
setting up /etc...
reloading user units for matthew...
setting up tmpfiles
Connection to itchy.scratchy.com closed.
```

# Available arguments via `_module.args.nixinate`

- `host` *`string`*

   A string representing the hostname or IP address of a machine to connect to
   via ssh.

- `sshUser` *`string`*

   A string representing the username a machine to connect to via ssh.

- `buildOn` *`"remote"`* or *`"local"`*

  - `"remote"`

    Push the flake to the remote, build and activate entirely remotely,
    returning logs via SSH.

  - `"local"`

    Build the system closure locally, copy to the remote and activate.

- `hermetic` *`bool`*

  Whether to copy Nix to the remote for usage when building and activating,
  instead of using the Nix which is already installed on the remote.
