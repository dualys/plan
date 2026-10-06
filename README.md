# plans

Process manifest: defines layers and capabilities for sandboxed runs.

A `Plan` is a bounded stack of layers attached to a directory [`Noun`](https://docs.rs/noun) and an AWQ branch (`0..=15`). `seal` content-addresses that stack. `Sceau` is the birth record a runner weighs a plan against before executing it.

`no_std`. Layers live in `heapless::Vec` with room for `MAX_LAYERS` (64). Names and descriptions are `&'static str`.

## Install

```toml
[dependencies]
plans = "0.1.0"
```

```sh
cargo add plans
```

Direct dependencies: `noun`, `blake3` (`default-features = false`), `heapless`.

## Example

```rust
use plans::layer::{Capabilities, Layer, Layers};
use plans::sceau::{Sceau, Verdict};
use plans::Plan;
use noun::Noun;

let dir = Noun::of(b"root");
let mut phoenix = Layers::new(dir.clone());
phoenix
    .add_layer(Layer::new("gamma", 1, "", Noun::of(b"g"), Capabilities::Read))
    .unwrap();

let mut plan = Plan::new(dir, &mut phoenix, 0).unwrap();
plan.add_layer(Layer::new(
    "tui",
    1,
    "",
    Noun::of(b"tui"),
    Capabilities::Read,
))
.unwrap();

assert_eq!(plan.effective_capabilities(), Capabilities::Read);
assert_eq!(plan.seal(), plan.seal());

let sceau = Sceau::birth(&plan, 100);
assert_eq!(sceau.weigh(&plan), Verdict::Accept);
```

`Plan::new` refuses a null directory, a branch above 15, and a phoenix whose `layers` vec is empty.

## Layers

`layer::Layer` is a name, a `u32` version, a description, a root `Noun`, and a `Capabilities` value.

`Capabilities` is a closed enum: `None`, `Read`, `Write`, `Execute`, `ReadWrite`, `ReadExecute`, `WriteExecute`, `ReadWriteExecute`, `All`.

`Layers` keeps two vecs. `layers` is the live stack. `phoenix` is the snapshot restored by `reborn`. `init` moves a vec into `phoenix` and clears the live stack. `LibraryLayers` is the same stack shape, keyed by a directory noun.

`Plan::effective_capabilities` is the last layer's capability, or `None` if the plan stack is empty.

## Seal

`Plan::seal` is BLAKE3 over, in order:

1. directory noun bytes
2. `branch_index`
3. tag of the effective capability
4. layer count as `u32` little-endian
5. each layer: name, version (`u32` LE), description, root bytes, capability tag

The digest is wrapped with `Noun::from_bytes`. Same contents produce the same noun across boots.

`fork` clones the plan, clears `should_quit`, and replaces `directory` with that seal.

`merge_with` unions layers by root noun. The first plan seen wins a duplicate root. The result is sorted by root bytes, then by name. `branch_index` is the minimum of the two. `directory` becomes the seal of the union. `layer_face` and the phoenix snapshot are copied from `self`.

## Sceau

`Sceau::birth(plan, budget)` records `plan.seal()`, the effective capability, the caller budget, and the layer count.

`weigh` returns `Verdict::Accept` or `Verdict::Refuse`. It refuses when:

| Check | Reason |
| --- | --- |
| directory noun is null | `"null directory"` |
| `plan.seal()` ≠ `sceau.root` | `"noun mismatch"` |
| layer count ≠ the sealed count | `"layer count"` |
| capability rank above the seal | `"caps exceed seal"` |

Rank is not set inclusion. `Read`, `Write`, and `Execute` share rank 1, so a `Write` plan passes a `Read` seal. Pairs share rank 2. `ReadWriteExecute` is 3. `All` is 4. `None` is 0.

## Documentation

API docs are on [docs.rs/plans](https://docs.rs/plans).

## License

[AGPL-3.0-or-later](LICENSE).
