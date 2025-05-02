
## Instructions

**Generating `age` Keys**
For this tutorial, we will be using [age](https://github.com/FiloSottile/age) for encrypting and decrypting our secrets. We will need one set of `age` keys for ourselves as the administrator, and another one for each host.

To generate our own `age` keys, we will use the `age-keygen` command, available from the `age` package on most distributions. First, we generate our private key:
```sh
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

Then we derive our public key from the private key:
```sh
age-keygen -y ~/.config/sops/age/keys.txt
```

The `age` public key for each host can be generated from its SSH host key using the following command:
```sh
ssh-keyscan myhost | ssh-to-age
```

**Setting up Directory Structure**
Directory structure is something often left to personal taste or team policy. The purpose of this article is not to dictate how your repository must be laid out, it is to inform of how to use `sops-nix`. Therefore I will provide my structure as an example and use it to explain the meanings behind `sops-nix` configuration.

```
.
├── README.md
├── flake.lock
├── flake.nix
├── hosts
│   ├── host1
│   │   ├── configuration.nix
│   │   ├── hardware-configuration.nix
│   │   ├── hostModules
│   │   │   ├── hostModule1.nix
│   │   │   └── hostModule2.nix
│   │   └── hostSecrets
│   │       ├── default.nix
│   │       └── hostSecret1.yaml
│   └── host2
│       ├── configuration.nix
│       ├── hardware-configuration.nix
│       ├── hostModules
│       │   └── hostModule1.nix
│       └── hostSecrets
│           ├── default.nix
│           └── hostSecret1.yaml
├── modules
│   ├── default.nix
│   └── globalModule1
│       └── default.nix
└── secrets
    ├── default.nix
    └── globalSecret1.yaml
```

In this directory structure, the configuration for each host is kept in a separate subdirectory under the `hosts` directory in the root of the repository. The `modules` and `secrets` root directories are used for NixOS modules and `sops` secrets which will be globally available to all hosts, while the `hostModules` and `hostSecrets` directories inside of each host's subdirectory contain NixOS modules and `sops` secrets used exclusively by that host.

This allows for a neat separation of configuration and secrets. `host1` won't be able to access the secrets of `host2`, and vice-versa, but they can both access a global secret. None of the secrets, however, are available in the repository in plaintext: they are all protected by `sops`. 

**Writing the `.sops.yaml`**
In order to achieve these, we need to have `.sops.yaml` file which configures `sops` to follow our layout.
```
keys:
  # Admin keys
  - &admin age1FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF

  # Host keys
  - &host1 age1AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
  - &host2 age1BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
creation_rules:
  - path_regex: secrets/[^/]+\.(yaml|json|env|ini)$
    key_groups:
    - age:
      - *admin
      - *host1
      - *host2
  - path_regex: host1/hostSecrets/[^/]+\.(yaml|json|env|ini)$
    key_groups:
    - age:
      - *admin
      - *host1
  - path_regex: host2/hostSecrets/[^/]+\.(yaml|json|env|ini)$
    key_groups:
    - age:
      - *admin
      - *host1
```

Here, the value used for `&admin` is the public key generated from our private key during the previous step, while the values for `host1` and `host2` are their `age` public keys, respectively.

