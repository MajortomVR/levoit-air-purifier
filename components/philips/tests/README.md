# Compile tests

Self-contained ESPHome configs that build the `philips` component without
needing a `secrets.yaml` — Wi-Fi and API credentials are inline dummies. They
point at `../..` as the external component source, so they always build the
working copy.

Run by [`.github/workflows/compile-tests.yml`](../../../.github/workflows/compile-tests.yml)
on every pull request that touches `components/`, and locally with:

```bash
cd components/philips/tests
esphome compile test-all-entities.yaml
```

## What each one is for

| Config | Catches |
|--------|---------|
| `test-all-entities.yaml` | Codegen for **every entity type on every platform** (`AC0951`, the model with the most entities). Anything added to a `TYPE_MAP` without working codegen fails here. |
| `test-fan-only.yaml` | Sources that include a platform header **without guarding on `USE_PHILIPS_*`**. With only a fan configured, ESPHome copies no `switch/`, `select/`, `number/` … directories, so an unguarded include cannot resolve. |
| `test-foreign-platforms.yaml` | The subtler variant: `template` entities supply the platforms, so the **generic** `USE_SWITCH` / `USE_SELECT` / … macros are defined while no philips platform directory is copied. Guarding on those generic macros instead of `USE_PHILIPS_*` breaks exactly here. |

The last one is the regression test for a bug that was real: before the
`USE_PHILIPS_*` guards, an `AC0650` config with an unrelated `template` switch
failed to compile with `fatal error: switch/philips_switch.h: No such file`.

## Why not one config per model

`model:` is a runtime enum — every model's C++ branch compiles regardless of
which one a config selects — so per-model configs would add build time without
adding compile coverage. What varies at compile time is *which platforms are
present*, which is what these three configs span.

Runtime behaviour per model is not covered here; that still needs hardware, and
the `AC0950` in particular has never been tested on a device.

## Adding a type

Add it to the platform's `TYPE_MAP`, then add the same entry to
`test-all-entities.yaml`.
