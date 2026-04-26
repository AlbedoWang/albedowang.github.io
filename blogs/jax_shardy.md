# A Walk Through the JAX Compiler Stack and Shardy: A Tools & Usage Field Manual

When we train or serve a large model in JAX, *"how does sharding actually take effect?"* is the source of nearly every performance question. This blog post does not talk about any specific business model. It only stitches together a single line: what the JAX compilation pipeline looks like, what Shardy is, how to call it in-process, what it does and does not do, and which API gotchas matter — as a reference manual for any future sharding-related work.

---

## 1. The JAX Compilation Pipeline at a Glance

`jax.jit(f)(...)` looks like one line. Underneath, it is a full lowering pipeline:

```
Python f(*args)
   │  jit + tracing
   ▼
Jaxpr (JAX IR)
   │  jax._src.interpreters.mlir
   ▼
StableHLO  (MLIR module text)
   │  jaxlib MLIR passes
   ▼
SDY/Shardy propagation + explicit reshard materialization
   │
   ▼
SDY/StableHLO IR with explicit reshard-derived collectives
   │  XLA (C++) SPMD lowering / backend compilation
   ▼
HLO → device bytecode
```

Note: the SDY/Shardy stage above does **not** materialize every collective the program will actually run. Reductions implied by op semantics (e.g. a contracting dim sharded on both sides of `dot_general`) are still inserted later by XLA's C++ SPMD lowering. We come back to this in §3.

You can stop at every stage, take the intermediate artifact out, look at it, mutate it, and feed it back in:

| What you want to inspect | How |
|---|---|
| Jaxpr | `jax.make_jaxpr(f)(...)` |
| StableHLO text | `jax.jit(f).lower(...).as_text()` |
| StableHLO with explicit `in_shardings` | pass `in_shardings=...` to `lower` |
| IR after sharding | open your own `ir.Context` and run sdy passes via `PassManager` |
| Final compile artifact (HLO) | `lowered.compile().as_text()` (expensive — actual compile) |

---

## 2. Where Shardy Sits, and What It Does

Shardy (the `sdy.*` dialect) is OpenXLA's MLIR-based, axis-name-based SPMD partitioning / propagation system. In modern JAX, Shardy is the migration path intended to replace GSPMD. During the transition window, the toggle is:

```python
jax.config.update("jax_use_shardy_partitioner", True)   # or False to opt out
```

Check the exact default for your JAX version — recent JAX docs flag Shardy as the sole partitioner once the migration completes, with `jax_use_shardy_partitioner=False` available as a temporary opt-out.

Its work splits into two phases.

### 2.1 Propagation — diffuse local sharding to the whole graph

Input: a *small* number of `stablehlo.*` ops carry an `sdy.sharding_per_value` attribute (coming from `with_sharding_constraint` / input `in_shardings`).
Output: shardings are inferred for unannotated tensors and propagated bidirectionally through the use-def graph until a fixed point — telling you "what shape and sharding the result of each op has."

Empirical observation in the workloads I tested: after propagation, almost every rank ≥ 1 compute op ends up annotated; the unannotated cases were mostly rank-0 scalar ops and ops inside reduce-region bodies. (Treat this as a useful invariant for downstream analyses, not a formal API guarantee from Shardy.)

### 2.2 Partitioner — materialize sharding boundaries as communication ops

After propagation, the IR only says *"each op should have this sharding"*; it does not yet contain real communication. The partitioner phase materializes communication in two steps: `sdy-insert-explicit-reshards` first inserts `sdy.reshard` ops at incompatible sharding boundaries; `sdy-reshard-to-collectives` then rewrites those reshards into concrete `sdy.all_*` / collective ops where possible (after this pass, no non-redundant `sdy.reshard` should remain). Note that this only materializes *reshard-derived* collectives — collectives implied by op semantics (e.g. a row-parallel matmul's all-reduce) are still added later by XLA's C++ SPMD lowering.

In jaxlib 0.9 the full pipeline looks like this:

```
sdy-propagation-pipeline                            # inference
sdy-close-shardings                                 # close open axes
func.func(sdy-insert-explicit-reshards)             # insert sdy.reshard at boundaries
func.func(sdy-reshard-to-collectives)               # lower reshard → concrete collective
```

After this runs, every place where the sharding ought to change becomes an explicit `sdy.all_*` op, and the downstream IR topology is stable.

> ✅ Memorize this pipeline string for direct reuse:
> ```
> builtin.module(sdy-propagation-pipeline,
>   sdy-close-shardings,
>   func.func(sdy-insert-explicit-reshards),
>   func.func(sdy-reshard-to-collectives))
> ```

---

## 3. What Shardy **Does Not** Do: the Truth About Implicit Collectives

This is a point that's easy to misjudge. Shardy's `sdy-reshard-to-collectives` does exactly one thing: **rewrite existing `sdy.reshard` ops into concrete `sdy.all_*` ops**.

It does **not** synthesize new collectives based on op semantics. For example:

- `stablehlo.dot_general` with the contracting dim sharded along `tensor` on both sides → mathematically the result is a partial sum and needs an `all-reduce`, but no all-reduce will appear in the IR.
- `stablehlo.reduce` along a sharded dim → also needs a cross-device sum; will not appear as a collective.

So who inserts those? XLA's C++ SPMD partitioner. The flow is: `xla::sdy::ShardyXLA` is the bridge that hands the SDY IR off to XLA, and from there the **GSPMD** partitioner (e.g. `PartitionDotGroupOnContractingImpl` in `xla/service/spmd/dot_handler.cc`) is what actually emits the contracting-dim all-reduce. (Per the OpenXLA Shardy overview, the long-term plan is a new MLIR-based SPMD partitioner, but the short-term implementation reuses the existing GSPMD partitioner.) It only fires when you call `lowered.compile()`.

Two practical consequences:

1. A region of IR with no `sdy.all_*` nodes is **not** necessarily local. **Any honest cost model has to account for these implicit collectives separately** — contracting / reduce dim sharded → reduction is required.
2. `unreduced=` / `sharded_to_unreduced` only show up if the *user* explicitly writes `unreduced={...}` in a sharding annotation. Shardy never auto-emits them for row-parallel matmul.

If you want to *see* implicit collectives in the IR, either call `compile()` and let XLA partition (expensive — seconds per config), or roll alpa-style inference rules over your own graph data structure (milliseconds per config).

---

## 4. The Minimal In-Process Recipe

Here is the canonical template; almost every sdy experiment is a variant of it:

```python
import jax
jax.config.update("jax_use_shardy_partitioner", True)

from jaxlib.mlir import ir, passmanager
from jaxlib.mlir.dialects import stablehlo, sdy   # MUST import — registers dialects
from jax._src.interpreters import mlir as jax_mlir

# 1. lower once, get StableHLO text
text = jax.jit(f, in_shardings=in_shardings).lower(*shaped_args).as_text()

# 2. parse into your own ir.Context (key: use jax_mlir.make_ir_context())
ctx = jax_mlir.make_ir_context()
ctx.allow_unregistered_dialects = True
module = ir.Module.parse(text, context=ctx)

# 3. run the sdy pass pipeline
PIPELINE = ("builtin.module(sdy-propagation-pipeline,"
            "sdy-close-shardings,"
            "func.func(sdy-insert-explicit-reshards),"
            "func.func(sdy-reshard-to-collectives))")
with ctx:
    passmanager.PassManager.parse(PIPELINE, context=ctx).run(module.operation)

# 4. read the annotations
for op in walk(module):
    print(op.name, op.attributes.get("sdy.sharding"))
```

Several **easy-to-trip-over** details:

- A bare `ir.Context()` does not register stablehlo / sdy / chlo. `Module.parse` will fail with "unregistered dialect." **Use `jax._src.interpreters.mlir.make_ir_context()`** which registers all dialects JAX needs.
- Even `from jaxlib.mlir.dialects import stablehlo, sdy` (with `noqa`) cannot be omitted — dialect registration relies on import side effects.
- `OpView.name` is an attribute (e.g. `"stablehlo.add"`); `Operation.name` is the same string. Mixing both is fine, but `OpView` has methods `Operation` does not, so when iterating do `op = op.operation if hasattr(op, "operation") else op`.
- If you are injecting your own sharding annotations and re-running propagation, **make sure the module has a top-level `sdy.mesh @mesh = <"a"=2, ...>`** declaration — otherwise propagation silently does nothing.

---

## 5. Abstract Lowering: Allocate Zero Real Tensors

A frequently overlooked fact: **the entire lower → propagate → partition chain needs no real data, no real mesh, no real weights**. This is critical for any kind of search-based work.

The two pillars of going abstract:

```python
# 1. flax.nnx models can be abstracted via eval_shape
graphdef, params_abs, rest_abs = nnx.eval_shape(build_block)

# 2. inputs use ShapeDtypeStruct, not jnp.zeros
hs_abs = jax.ShapeDtypeStruct((batch, seq_len, dim), jnp.bfloat16)

# 3. lower (note: ShapeDtypeStruct, not real arrays)
lowered = jax.jit(forward).lower(params_abs, rest_abs, hs_abs, ...)
```

Throughout, `jax.live_arrays()` stays at 0 — not a single byte of HBM is allocated. Why this matters:

- A 14B model's lowering analysis runs on a single v4 chip without OOM.
- Re-lowering after changing `logical_axis_rules` / mesh shape costs only tens of ms.

---

## 6. AbstractMesh and Its API Quirks

If you want to abstract the mesh too — e.g. to run a unified "logical mesh, every axis size=2" lowering — use `jax.sharding.AbstractMesh`:

```python
# Modern JAX expects axis_types as the third argument; older versions accepted
# just (shape, axis_names). Check the AbstractMesh signature for your JAX version.
a_mesh = jax.sharding.AbstractMesh(
    (2,) * len(logical_axes),                                    # axis_sizes
    tuple(logical_axes),                                          # axis_names
    (jax.sharding.AxisType.Explicit,) * len(logical_axes),        # axis_types
)
```

(Note: `jax.make_mesh`'s default `axis_types` shifted to `Explicit` around 0.9.0 — omitting it is a migration footgun. The same caution applies to `AbstractMesh`.)

But jax 0.9 has several **counter-intuitive** API choices. Once you've stepped on them, you won't again:

| What you want | Recommended form | Notes |
|---|---|---|
| Enter abstract-mesh context inside a jit'd function | `with jax.sharding.use_abstract_mesh(a_mesh):` | `jax.set_mesh` is the public global setter and is oriented around a concrete device `Mesh`; for swapping to a *different* `AbstractMesh` mid-trace, `use_abstract_mesh` is the documented entry point (paired with `get_abstract_mesh()`). |
| Lower with no real backend, or pin a target platform | `jax.jit(f).trace(*args).lower(lowering_platforms=("tpu",))` | `jax.jit(f).lower(*args)` against `ShapeDtypeStruct` + `AbstractMesh` is usually fine when a backend for the target platform is already initialized in-process. The two-stage form is what you need when there is no such backend, or when you are exporting cross-platform. |

Whether `shard_map` (`sdy.manual_computation`) lowers and partitions cleanly under AbstractMesh is an implementation detail. If your attention kernel reads `mesh.shape["context"]` directly (i.e. uses **physical** axis names), AbstractMesh raises KeyError. A robust workaround when doing "universal lowering": switch attention to a pure-op implementation like `dot_product`, and model fused kernels (flash / ring) as a separate boundary cost.

---

## 7. Reading the IR: Op Classification and Abstract Graph

After the partitioner runs, the ops in the IR fall into roughly four kinds:

| Kind | Typical ops | Meaning |
|---|---|---|
| `compute` | `stablehlo.*`, `chlo.*` (excluding `stablehlo.constant`) | computation, carries `sdy.sharding` |
| `comm` | `sdy.{all_slice, all_gather, all_to_all, all_reduce, reduce_scatter, collective_permute, reshard}` | explicit collective communication |
| `boundary` | `sdy.manual_computation` (= `shard_map`) | user-written SPMD boundary, body is opaque |
| `constant` | `sdy.constant`, `stablehlo.constant` | constants |

Build a simple `Node` / `CommGraph` over these four kinds (each node carries `kind / op_name / shape / dtype / sharding / operand_ids`), and you have a clean input for downstream analyses (capacity estimation, ILP search, α-β communication cost modeling, ...).

A few details that are easy to miss in practice:

- For the purposes of a CommGraph cost model, treat the region body of `manual_computation` as an opaque boundary — otherwise you double-count ops inside `shard_map`. Note this is an analysis choice, not a Shardy invariant: per the Shardy docs, the body is local with respect to `manual_axes`, but propagation still flows through the body for any *free* axes you didn't list as manual.
- After partitioning, `stablehlo.while` may also contain `stablehlo.*` ops in its body, but semantically it is an iteration — decide based on the analysis whether to walk in.
- `collective_axes` on multi-axis meshes is a list-per-tensor-dim of axis-name groups, not a flat list.
- Dim-spec format: Shardy uses `[{a,b},{},{c}]` — "tensor dim 0 is sharded along axes `a` and `b`, dim 1 is replicated, dim 2 is sharded along `c`." Axis-name sets can have multiple elements, which corresponds to nested-ring cost models.

---

## 8. Performance: How Long Each Stage Takes

Before putting these tools inside a search loop, you need a clear picture of per-stage cost. Below are typical timings for a 1B-scale transformer block (seq=128) on v4-8:

| Stage | Time |
|---|---|
| `nnx.eval_shape` (abstract build) | ~5 ms |
| `jax.jit(...).lower(...)` | ~65 ms |
| `lowered.as_text()` | ~3 ms |
| New `ir.Context` + `Module.parse` | ~5 ms |
| `sdy-propagation-pipeline` | ~25 ms |
| Three-stage partitioner | ~30 ms |
| Walking the module + extracting sharding | ~50 ms |
| **Total** | **~180 ms / config (warm)** |

Some non-obvious observations:

1. **Propagation itself is only ~1/8 of the total.** The bottleneck is the Python-side MLIR walk (`str(op)`, attribute parsing).
2. **Propagation is insensitive to the number of injected annotations**: 0 vs 14 differs by less than 1 ms. A single propagation pass over the full graph is enough; no need to re-run per candidate.
3. **Don't call `compile()` carelessly.** A real XLA compile is in the seconds range — invoking it inside a search loop ruins everything.
4. **One-time cold-start cost** (jaxlib import + TPU backend init + dialect registration) is around 4–5 seconds. Any benchmark must exclude this.

---

## 9. Shardy / GSPMD / alpa on a Single Chart

| Aspect | GSPMD (legacy) | **Shardy** | alpa (research prototype) |
|---|---|---|---|
| Annotation form | `tile_assignment_dimensions=[2,1,4]` (mesh-axis index) | `[{a,b},{},{c}]` (mesh-axis name) | `tile_assignment_dimensions` |
| Multi-axis dims | awkward | natural, supports nesting | 2D mesh, restricted |
| When are collectives inserted | XLA C++ partitioner | partial (reshard kind) + XLA C++ (contracting / reduce kinds) | solver virtually inserts during cost modeling |
| Search-friendly | no | no, but **axis names are symbolic** — propagation depends on names, not sizes | yes, ILP over sharding choices |
| Usage core | `in_shardings` + `with_sharding_constraint` | `in_shardings` + `with_sharding_constraint`; `sdy.sharding` annotations are directly readable | own IR + solver |

There is one insight here worth highlighting on its own:

> **Shardy's sharding representation is axis-name-based.** If you alpha-rename mesh axes (e.g. `data` → `L_batch`, `tensor` → `L_hidden`) while keeping axis sizes, axis types, manual axes, and divisibility/legalization conditions unchanged, propagation in practice produces an IR that is isomorphic up to names — same op signatures, same collective list, same per-op dim-specs.

Treat this as a useful **modeling assumption to validate per workload**, not a compiler contract. Once axis sizes, axis types, divisibility constraints, or backend legalization rules vary, the post-partition collective structure can shift. With that caveat, you can still treat mesh axes as **free variables**: run propagation+partition once on a "logical mesh" (one mesh axis per logical axis name, size=2 each) to get a CommGraph that encodes the full topology, and verify on real configs that the topology is stable before relying on it.

---

## 10. Mental Models Worth Memorizing

To compress this whole stack into a few mental models:

1. **`jit.lower()` is free; `compile()` is expensive.** To see "what the IR looks like after sharding," use the former. To get the real partitioner artifact, use the latter — and accept that it costs seconds.
2. **Shardy makes sharding annotations readable.** To debug "why didn't this matmul shard the way I expected," the fastest path is: lower → run propagation yourself → print the `sdy.sharding` of every op.
3. **Always model implicit collectives separately.** Shardy's IR contains no contracting-dim all-reduce / reduce-dim all-reduce, but they really exist. Any cost model has to add them back explicitly.
4. **Abstract everything you can.** `nnx.eval_shape` + `ShapeDtypeStruct` + `AbstractMesh` together drop the cost of "try one sharding config" from seconds to ~200 ms.
5. **Logical axis names are a powerful design primitive.** Beyond maxdiffusion's `logical_axis_rules` — any work that wants sharding search or mesh-config portability should adopt this symbolic decoupling layer.
6. **Walking MLIR is not free.** On large modules, `str(op)` and repeated walks are substantially slower than the propagation pass itself. If performance bites, either reduce walks (one traversal that does everything), or push the analysis into native code.

---

## 11. Further Reading

- The OpenXLA Shardy repository / dialect docs: `shardy/` (full `sdy.*` op set and pass descriptions).
- There is no clean built-in pass-list dump in jaxlib's MLIR Python bindings. To probe what's registered in your jaxlib, the empirical recipe is to try parsing each candidate pipeline string and see whether it raises:
  ```python
  from jaxlib.mlir import passmanager
  from jaxlib.mlir.dialects import sdy, stablehlo  # noqa: F401  (register dialects)
  from jax._src.interpreters import mlir as jax_mlir
  ctx = jax_mlir.make_ir_context()
  for p in ["sdy-propagation-pipeline", "sdy-close-shardings",
            "sdy-insert-explicit-reshards", "sdy-reshard-to-collectives"]:
      try:
          passmanager.PassManager.parse(f"builtin.module({p})", context=ctx)
          print("OK  ", p)
      except Exception as e:
          print("FAIL", p, str(e)[:80])
  ```
- `flax.linen.partitioning.axis_rules` — the entry point for logical → mesh axis resolution.
- `jax.experimental.shard_map` — explicit SPMD boundary; appears as `sdy.manual_computation` in the IR.
- alpa's `auto_sharding_solver/` (a playground-style, pure-Python prototype) — the best reading material for seeing how "logical sharding choice + α-β communication cost" fit together.

Stitch all of this together, and JAX sharding behavior stops being "a black box that disappears into XLA." It becomes a sequence of MLIR pipelines you can inspect, mutate, re-run, and parameterize from Python. That is the biggest experiential upgrade Shardy delivers.
