---
feature: config-option
start-date: 2019-03-10
author: Silvan Mosberger
co-authors: (find a buddy later to help our with the RFC)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

A lot of NixOS options exist to specify single settings of configuration files. Along with such options come multiple disadvantages, such as having to synchronize with upstream changes, option bloat and more. An alternative is to provide options for passing the configuration as a string, such as the common `configFile` or `extraConfig` options, but this also comes with a set of disadvantages, including difficulty to override values and bad modularity.

This RFC proposes aims to solve these problems by encouraging NixOS module authors to provide an option for specifying the programs configuration file as a Nix value, providing a set of utility functions for writing these options conveniently, and updating the documentation to recommend this way of doing program configuration.

# Motivation
[motivation]: #motivation

NixOS commonly has 2 models of specifying configuration for programs, each with their own set of problems. This RFC aims to solve all of them.

## Single option for every setting

Having a single option for every setting in the configuration file, this often gets combined with an `extraConfig` option to provide greater flexibility. Problems:

- Coupling to upstream
  - When upstream adds or removes settings, the NixOS module needs to be updated to reflect that.
    - Upstream adds a setting: If the module has an `extraConfig` option people might set the new setting there. But if we ever add it as a NixOS option, we'll have trouble merging the values together with what the user already specified in `extraConfig`
    - Upstream removes a setting (backwards incompatible): The NixOS module is straight up broken in nixpkgs until somebody fixes it, end users can't fix it themselves (unless the module provides a `configFile` option which can override the generated one). The same can also happen when the user uses overlays or `disabledModules` to cause a module/package version mismatch.
  - The documentation of the upstream settings needs to be copied over to the NixOS options, which means it might get out of sync with changes
  - The upstream defaults are being copied to the NixOS modules, so we need to also update the defaults whenever upstream changes them. This can be solved by using `nullOr` to allow for a `null` value to indicate that the upstream default shall be used, but that's not very nice.
- Option bloat: <s>NixOS evaluation time is still linear in the number of *available* options for all users, even though everybody only uses a fraction of them. This means that when a module adds 100 new options, this leads to a direct increase in evaluation time for every `nixos-rebuild switch` of everybody using NixOS.</s> With high confidence debunked in [#57477](https://github.com/NixOS/nixpkgs/issues/57477).
- Timeconsuming to implement, tedious to review and hard to maintain
  - It takes a non-zero amount of time to write all the wanted options out into a NixOS module
  - Reviewing is tedious because people need to make sure types are correct, descriptions are fitting, defaults are acceptable, for every option added. Due to the size and repetitiveness, people are also less willing to thoroughly review the code.
  - The bigger the NixOS module is the harder it is to maintain, and the less people want to actually maintain it. 
- Responsibility for backwards compatibility: By adding such options, we obligate ourselves to handle backwards incompatibilites on our side. We will have to support these options for a long time and can't remove them without at least one person being annoyed about it.

## `configFile` or `extraConfig` option

An option for specifying the contents of the configuration file directly with `configFile` or `extraConfig`. Problems:

- Not very modular at all
  - In case of json or a `configFile` option, you can only assign the option once, merging is impossible
  - In case of ini (or similar), assigning a single option multiple times to make use of list concatenation, ordering or priorities is impossible, so in general you can't e.g. override a default value set somewhere else.
- No syntax checking: Users will have to know the syntax of the configuration language and encode their values properly, any syntax error will only be realized when the program errors at runtime.

## Occurences of problems

- Because prometheus uses options to encode every possible setting, [#56017](https://github.com/NixOS/nixpkgs/pull/56017) is needed to allow users to set a part of the configuration that wasn't encoded yet.
- Because strongswan-ctl uses options to encode its full configuration, changes like [#49197](https://github.com/NixOS/nixpkgs/pull/49197) are needed to update our options with upstream changes.
- Pull requests like [#57036](https://github.com/NixOS/nixpkgs/pull/57036) or [#38324](https://github.com/NixOS/nixpkgs/pull/38324) are needed because users wish to have more configuration options than the ones provided.

## Previous discussions

- https://github.com/NixOS/nixpkgs/pull/44923#issuecomment-412393196
- https://github.com/NixOS/nixpkgs/pull/55957#issuecomment-464561483 -> https://github.com/NixOS/nixpkgs/pull/57716

## Implementations

This idea has been implemented already in some places:
- [#45470](https://github.com/NixOS/nixpkgs/pull/45470)
- [#52096](https://github.com/NixOS/nixpkgs/pull/52096)
- [My Murmur module](https://github.com/Infinisil/system/blob/45c3ea36651a2f4328c8a7474148f1c5ecb18e0a/config/new-modules/murmur.nix)

# Detailed design
[design]: #detailed-design

For specifying configuration files to programs in NixOS options, there should be a main single option called `config` which represents the configuration of the program as a Nix value, which can then be converted to the configuration file format the program expects. This may look as follows:

```nix
{ config, lib, ... }: with lib;
let

  cfg = config.services.foo;

  configText = configGen.json cfg.config;

in {

  options.services.foo = {
    enable = mkEnableOption "foo service";

    config = mkOption {
      type = lib.types.config.json;
      default = {};
      description = ''
        Configuration for foo. Refer to <link xlink:href="https://example.com/docs/foo"/>
        for documentation on the supported values.
      '';
    };
  };
  
  config = mkIf cfg.enable {
  
    # Set minimal config to get service working by default
    services.foo.config = {
      # We don't use mkDefault, as this module requires this value in order to function
      dataPath = "/var/lib/foo";
      logLevel = mkDefault "WARN";
      # Upstream default, needed for us to open the firewall
      port = mkDefault 2546;
    };
  
    environment.etc."foo.json".text = configText;
    
    networking.firewall.allowedTCPPorts = [ cfg.config.port ];
    # ...
  };
}
```

This approach solves all* of the above mentioned problems. In addition we have the following properties that work with this approach out of the box:
- Ability to easily query arbitrary configuration values with `nix-instantiate --eval '<nixpkgs/nixos>' -A config.services.foo.config`
- The configuration file is well formatted with the right amount of indentation everywhere
- Usually hardcoded defaults can now be replaced by simple assignment of the `config` option, which also allows people to override those values. See the [Defaults](#defaults) section for more details.

*: The only problem it doesn't solve is the coupling to upstream defaults for required defaults, see the [Defaults](#defaults) section.

### Defaults

Depending on how default settings matter, we need to set them differently and for different reasons:
- If the module needs a specific value for a setting because of how the module or NixOS works (e.g. `logger = "systemd"`, because NixOS uses `systemd` for logging), then the value should *not* use `mkDefault`. This way a user can't easily override this setting (which would break the module in some way) and will have to use `mkForce` instead to change it. This also indicates that they are leaving supported territory, and will probably have to change something else to make it work again (e.g. if they set `logger = mkForce "/var/log/foo"` they'll have to change their workflow of where to look for logs).
- If the program needs a setting to be present in the configuration file because otherwise it would fail at runtime and demand a value, the module should set this value *with* a `mkDefault` to the default upstream value, which will then be the equivalent of a starter configuration file. This allows users to easily change the value, but also enables a smooth first use of the module without having to manually set such defaults to get it to a working state. Optimally modules should Just Work (tm) by setting their `enable` option to true.
- If the module itself needs to know the value of a configuration setting at evaluation time in order to influence other options (e.g. opening the firewall for a services port), we may set upstream's default with a `mkDefault`, even though the program might start just fine without it. This allows the module to use the configuration setting directly without having to worry whether it is set at all at evaluation time.

If all of the above points don't apply to a configuration setting, that is the module doesn't care about the value, the program doesn't care about the setting being present and we don't need the value at evaluation time, there should be no need to specify any default value.

### Configuration checking

One general downside of this approach is that the module system is not able to check the types and values of the configuration file, which would be fast, simple and give good error messages by default. While it would be possible to use `types.addCheck` for the type of the `config` option, this sounds more painful than it's worth and would lead to bad error messages, so we'll ignore this here. Here are some alternatives.

#### Configuration checking tools

A number of programs have tools for checking their configuration that don't need to start the program itself. We can use this to verify the configuration at **build time** by running the tool for a derivation build. While this is not as fast as if we had the module system do these checks, which would be at evaluation time already, it is faster than the program failing at runtime due to misconfiguration. These tools however are also more powerful than the module system and can integrate tightly with the program itself, allowing for more thorough checks. If a configuration checking tool is available, optimally by the program itself, it should be used if possible, as it can greatly improve user experience. The following illustrates an example of how this might look like

```nix
{ config, pkgs, lib, ... }: with lib;

let

  configText = configGen.json cfg.config;

  configFile = pkgs.runCommand "foo-config.json" {
    # Because this program will be run at build time, we need `nativeBuildInputs` instead of `buildInputs`
    nativeBuildInputs = [ pkgs.foo ];

    inherit configText;
    passAsFile = [ "configText" ];
  } ''
    foo check-config $configTextPath
    cp $configTextPath $out
  '';

in { /* ... */ }
```

#### Ad-hoc checks with assertions

While not as optimal as a configuration checker tool, assertions can be used to add flexible ad-hoc checks for type or other properties at **evaluation time**. It should only be used to ensure important properties that break the service in ways that are otherwise hard or slow to detect (and easy to detect for the module system), not for things that make the service fail to start anyways (unless there's a good reason for it). The following example only demonstrates how assertions can be used for checks, but any reasonable program should bail out early in such cases, which would make these assertions redundant, and only add more coupling to upstream, which we're trying to avoid.

```nix
{ config, lib, ... }: with lib; {
  # ...
  config = mkIf cfg.enable {
    # Examples only for demonstration purposes, don't actually add assertions for such properties
    assertions = [
      {
        assertion = cfg.config.enableLogging or true -> cfg.config ? logLevel;
        message = "You also need to set `services.foo.config.logLevel` if `services.foo.config.enableLogging` is turned on.";
      }
      {
        assertion = cfg.config ? port -> types.port.check cfg.config.port;
        message = "${toString cfg.config.port} is not a valid port number for `services.foo.config.port`.";
      }
    ]
  };
}
```

TODO: Are there any good examples of using assertions for configuration checks at all?

### Implementation

The implementation consists of three separate parts

#### Configuration types

A set of types for common configuration formats should be provided in `lib.types.config`. Such a type should encode what values can be set in files of this configuration format as a Nix value, with the module system being able to merge multiple values correctly. This is the part that checks whether the user set an encodeable value. This can be extended over time, but could include the following as a start:
- JSON
- YAML, which is probably the same as JSON
- INI
- A simple `key=value` format
- A recursive `key.subkey.subsubkey=value` format

Sometimes programs have their own configuration formats, in which case the type should be implemented in the program's module directly.

#### Configuration format writers

To convert the Nix value into the configuration string, a set of configuration format writers should be provided under `lib.configGen`. These should make sure that the resulting text is somewhat properly formatted with readable indentation. Things like `builtins.toJSON` are therefore not optimal as it doesn't add any spacing for readability. These writers will have to include ones for all of the above-mentioned configuration types. As with the type, if the program has its own configuration format, the writer should be implemented in its module directly.

#### Documentation

The nixpkgs manual should be updated to recommend this way of doing program configuration in modules, along with examples.

## Limitations

- Limited to configuration file formats representable conveniently in Nix, such as JSON, YAML, INI, key-value files, or similar formats. Examples of unsuitable configuration formats are Haskell, Lisp, Lua or other generic programming languages. For those it is recommended to not hardcode anything and provide a `config` option with type `types.str` or `types.lines` if it makes sense to merge multiple assignments of it.

## Additional config options
Sometimes it makes sense to have an additional NixOS option for a specific configuration setting. In general this should be discussed on a case-by-case basis to judge whether it makes sense. However keep in mind that it's always possible to add more options later on, but you can't as easily remove existing options. Instances of where it can make sense are:
- Settings that are necessary for the module to work and are different for every user, so they can't have a default. Examples are `services.dd-agent.api_key`, `services.davmail.url` or `services.hydra.hydraURL`. Having a separate option for these settings can give a much better error message when you don't set them (instead of failing at runtime or having to encode the requirement in an assertion) and better discoverability.
- Password settings: Some program's configuration files require passwords in them directly. Since we try to avoid having passwords in the Nix store, it is advisable to provide a `passwordFile` option as well, which would replace a placeholder password in the configuration file at runtime.

This `config` approach described here is very flexible for these kind of additions, here's an example where we add a `domain` setting to the above service as a separate NixOS option:
```nix
{ config, lib, ... }: with lib; {

  options.services.foo = {
    # ...
    
    domain = mkOption {
      type = types.str;
      description = "Domain this service operates on.";
    };
  };
  
  config = mkIf cfg.enable {
    services.foo.config = {
      # ...
      domain = mkDefault cfg.domain;
    };
  };
}
```

Even though we have two ways of specifying this configuration setting now, the user is free to choose either way.

# Drawbacks
[drawbacks]: #drawbacks

There are some disadvantages to this approach:
- If there is no configuration checking tool as explained in [this section](#configuration-checking-tools), the types of configuration settings can't be checked as easily, which can lead to packages failing at runtime instead of evaluation time. Refer to [Configuration checking](#configuration-checking) for more info.
- Documentation for the configuration settings will not be available in the central NixOS manual, instead the upstream documentation has to be used, which can be unfamiliar and harder to read.

# Alternatives
[alternatives]: #alternatives

See [Motivation](#motivation)

# Unresolved questions
[unresolved]: #unresolved-questions

Ctrl-F for TODO

# Future work
[future]: #future-work
