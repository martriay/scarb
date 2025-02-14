<script setup>
import { data as rel } from "../../../github.data";
</script>

# Starknet Contract Target

The `starknet-contract` target allows to build the package as a [Starknet Contract](https://docs.starknet.io/documentation/getting_started/intro/).
It searches for all contract classes in the package, and builds a separate compiled JSON file each found class.
Generated file will be named with following pattern: `[target name]_[contract name].sierra.json`.

This target accepts several configuration arguments, beside ones [shared among all targets](../../reference/targets#configuring-a-target), with default values for the default profile:

```toml
[[target.starknet-contract]]
# Enable Sierra codegen.
sierra = true

# Enable CASM codegen.
casm = false
# Emit Python-powered hints in order to run compiled CASM class with legacy Cairo VM.
casm-add-pythonic-hints = false

# Enable allowed libfuncs validation.
allowed-libfuncs = true
# Raise errors instead of warnings if disallowed libfuncs are found.
allowed-libfuncs-deny = false
# Reference to the libfuncs allowlist used for validation.
# - Use `allowed-libfuncs-list.name` to use built-in named allowlist.
# - Use `allowed-libfuncs-list.path` to read an allowlist from file.
allowed-libfuncs-list = {} # Cairo compiler defined

# Emit Starknet artifacts for contracts defined in dependencies.
build-external-contracts = []
```

## Usage

To enable Starknet contract compilation for a package, write a following line in `Scarb.toml`:

```toml
[[target.starknet-contract]]
```

Then, declare a dependency on the [`starknet` package](./starknet-package).
Its version is coupled to Cairo version included in Scarb.

```toml-vue
[dependencies]
starknet = "{{ rel.stable.starknetPackageVersionReq }}"
```

## Sierra contract class generation

The enabled by default property `sierra` determines whether this target builds a Sierra
[Contract Class](https://docs.starknet.io/documentation/architecture_and_concepts/Contracts/contract-classes/) file.

## CASM contract class generation

Historically, contract classes have been defined in terms of Cairo assembly, or CASM for short (the class definition also included more information needed for execution, e.g., hint data).
The novelty of Cairo 1.0 is the introduction of Sierra, an intermediate layer between Cairo 1.0 and CASM.

When executing a contract on Starknet, the Sequencer downloads a [Contract Class] which contains Sierra bytecode.
It is a role of the Sequencer to compile it to CASM, which is a language that you can physically execute and generate proof of such execution.
If for any reason, there is a need to compile Sierra contract to CASM locally, Scarb can do that by turning the off by default property `casm`.
If enabled, Scarb will emit the CASM Contract Class file in the target directory to a file named with following pattern: `[package name]_[contract name].casm.json`.

Historically, Cairo used to use Python as the language powering the [Hints](https://www.cairo-lang.org/docs/how_cairo_works/hints.html).
CASM contract classes can be still executed on the legacy Python-based Cairo VM, under condition that they include Python version of hints generated by Sierra, which now is an optional feature.
The off by default `casm-add-pythonic-hints` property enables Scarb to add it to produced artifacts.

## Compiling external contracts

While compiling the Scarb project, by default no artifacts are emitted for contracts defined in dependencies.
To override this behaviour for certain contracts from other packages, you can use `build-external-contracts` property.
It accepts a list of strings, each of which is a reference to a contract defined in a dependency.
The package that implements this contracts need to be declared as a dependency of the project in `[dependencies]` section.
The reference to a contract is a full cairo path to the contract module.
External contracts will be built in the same way as the contracts defined in the project.
The artifacts will be emitted under `[target name]_[contract name].[sierra|casm].json` names.
In case there is a contract name collision , those colliding contract names will be replaced with full cairo paths.

For example, to build `Account` contract defined in `openzeppelin` package, add following definitions to the `Scarb.toml`:

```toml-vue
[dependencies]
starknet = "{{ rel.stable.starknetPackageVersionReq }}"
openzeppelin = { git = "https://github.com/OpenZeppelin/cairo-contracts.git", branch = "cairo-2" }

[[target.starknet-contract]]
build-external-contracts = ["openzeppelin::account::account::Account"]
```

## Starknet Artifacts

As part of building Starknet contracts, contract target generates a `[target_name].starknet_artifacts.json` file
containing a machine-readable data about built artifacts.

Version 1 of this file has a structure like this:

```json
{
  "version": 1,
  "contracts": [
    {
      "id": "<opaque>",
      "package_name": "mypackage",
      "contract_name": "Contract2",
      "artifacts": {
        "sierra": "mypackage_Contract2.sierra.json",
        "casm": null
      }
    },
    {
      "id": "<opaque>",
      "package_name": "mypackage",
      "contract_name": "Contract1",
      "artifacts": {
        "sierra": "mypackage_Contract1.sierra.json",
        "casm": null
      }
    }
  ]
}
```

- `id` is an identifier of the item in `"contracts"` list. Use it to reference items, as it has the highest chance of being a unique value.
- `package_name` is the name of the package in which the contract has been implemented.
- `contract_name` is the name of the contract module.
- `artifacts` paths are relative to this file path.
  Depending on the targets defined in `[[target.starknet-contract]]` section of the `Scarb.toml`,
  some of the values might be `null`.

## Allowed libfuncs validation

Not all Sierra libfuncs emitted by the Cairo compiler can be deployed to Starknet, as some are not audited yet,
or others are meant for development use and would be unsafe when run in contract context.
Therefore, the Starknet contract target runs a pass after compilation that checks if every emitted libfunc is present
in a provided _allowed libfuncs list_.
By default, this pass emits compilation warnings when it spots unexpected libfuncs.

This validation pass can be disabled entirely by writing `allowed-libfuncs = false` in target configuration.
On the other hand, its severity can be elevated to compilation errors, by writing `allowed-libfuncs-deny = true`.

The default allow-list is compiler-provided and corresponds to list of all audited libfuncs allowed on Starknet.

### Built-in allow-lists

The Starknet contract compiler contains several built-in allowed libfuncs lists, that can be identified by name via
the `allowed-libfuncs-list.name` property.
For example, to use the `experimental` allow-list, type the following:

```toml
[[target.starknet-contract]]
allowed-libfuncs-list.name = "experimental"
```

All built-in lists can be located in the [Cairo repository](https://github.com/starkware-libs/cairo/tree/main/crates/cairo-lang-starknet/src/allowed_libfuncs_lists).

### External allow-lists

It is also possible to use a user-provided allow-list, via the `allowed-libfuncs-list.path` property:

```toml
[[target.starknet-contract]]
allowed-libfuncs-list.path = "path/to/list.json"
```

User-provided lists should follow the same format as built-in ones.
For reference, consult the built-in [`audited`](https://github.com/starkware-libs/cairo/blob/82d75ae81da57010ef44a945c0387835dfed9a0e/crates/cairo-lang-starknet/src/allowed_libfuncs_lists/audited.json) list.
