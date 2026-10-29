# Package-provided UE4SS runtimes

Status: implementation guide for package-based custom runtime selection.

The client ships a standard UE4SS distribution. A game that needs a custom build can deliver it
as assets in a support package, without adding that build to every client installation.

The support package declares a top-level manifest property:

```json
"ue4ss": { "frameworkRoot": "assets/ue4ss" }
```

That package-relative directory contains `.frameworks/ue4ss/profile.json` and the distribution
files referenced by its hashes. Feature packages depend on the support package using ordinary
package dependencies. The support package's host-Lua enable/disable entrypoints can be empty;
selecting the distribution does not itself inject UE4SS.

Before the first Unreal runnable, the engine resolves declarations in the applied package set,
verifies the assets, stages its bridge and injects the selected DLL. Studio resolves the same
declaration from the saved game workspace before inspection. No additional runtime download
occurs during execution; the files arrived with package installation and remain available locally.

Without a declaration, the app's standard runtime is selected. Conflicting profiles are rejected.
An existing process connection can be shared by editable and applied copies with the same profile
hash. Switching a loaded distribution requires restarting the game, including when the DLL bytes
match but the settings or layouts differ. Studio's explicit framework path is a development override.

Use `ProcessEngine/scripts/prepare-ue4ss-package.ps1` in HexModClient to prepare the package from
a release ZIP. It retains the DLL, settings, signature overrides, layout overrides, license and
shared Lua helpers. Proxy loaders, debug symbols, tools and example mods are omitted. Server
validation checks the declared profile and asset hashes without executing them; DLL metadata goes
through the existing first-party native publication policy.

Implementation: [runtime selection](https://github.com/HexintonDev/HexModClient/blob/main/ProcessEngine/src/host/ue4ss_package_profile.cpp),
[package preparation](https://github.com/HexintonDev/HexModClient/blob/main/ProcessEngine/scripts/prepare-ue4ss-package.ps1).
