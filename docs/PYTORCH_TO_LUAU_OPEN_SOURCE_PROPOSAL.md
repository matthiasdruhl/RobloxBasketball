# PyTorch-to-Luau: Open-Source Inference Compiler Proposal

## Summary
Build an open-source tool that converts a **restricted subset of trained PyTorch models** into self-contained **Luau ModuleScripts** for inference in Roblox experiences. Training stays in Python/PyTorch; inference runs locally in Luau without a Python runtime or per-prediction HTTP request.

**Primary initial use case:** a compact model that ranks legal basketball movement/animation transitions based on character state, input intent, and defender context.

**Scope boundary:** This is an inference exporter/compiler, **not** a complete PyTorch implementation, and it does not permit importing compiled C++ libraries into Roblox.

## Motivation
- Roblox gameplay scripting uses Luau; native arbitrary C++ modules and Python/PyTorch execution are not supported in published experiences.
- External model APIs introduce latency, cost, reliability, and networking constraints for interactive movement.
- A small, portable model could support motion selection, contact reaction ranking, or NPC decisions.
- Roblox documents MLService/MLSession APIs, but general custom-model upload availability, compatibility, and production suitability should be verified before investing in a compiler. **Do not assume native inference is unavailable.**

## Proposed pipeline

```text
PyTorch model + representative inputs
          |
          v
 torch.export graph + model parameters
          |
          v
 Supported-op validation / graph lowering
          |
          v
 Optimization and parameter serialization
          |
          v
 Generated Luau ModuleScript(s)
          |
          v
 Roblox: require(Model).predict(state)
```

## MVP supported features
- Fixed-shape, inference-only feedforward MLPs.
- Linear/dense layers; ReLU and Tanh; elementwise addition (as needed).
- Static parameter export (weights and biases), fixed input/output shapes, shape validation.
- Deterministic inference interface: `predict(input: {number}): {number}`.
- Automated numerical comparisons against PyTorch on representative and randomized inputs with documented tolerances.
- Performance benchmarks in Roblox Studio and, where possible, desktop/mobile clients.

**Explicitly out of scope for MVP:** arbitrary Python control flow, dynamic shapes, backpropagation, training in Roblox, attention/transformers, convolutions, GPU kernels, and broad ONNX compatibility.

## Example target model

```python
import torch
import torch.nn as nn

model = nn.Sequential(
    nn.Linear(32, 64),
    nn.ReLU(),
    nn.Linear(64, 64),
    nn.ReLU(),
    nn.Linear(64, 8),
)

exported = torch.export.export(model.eval(), (torch.randn(1, 32),))
```

This architecture has **6,792 parameters** (32×64+64 + 64×64+64 + 64×8+8), or approximately **26.5 KiB** of raw float32 parameter data, excluding representation and runtime overhead.

Illustrative generated Luau interface:

```lua
local Model = require(game.ReplicatedStorage.MovementModel)
local scores = Model.predict(featureVector)
-- Game code filters/ranks only legally available actions.
```

The compiler must define a stable convention for batch dimensions, tensor layout, activation semantics, and output ordering.

## Compiler components

### 1. Frontend
- Accept an explicitly exported PyTorch graph and associated weights, rather than assuming any `.pt` file is a safe portable format.
- Inspect graph nodes, parameters, dimensions, dtypes, and dependencies.
- Fail clearly on unsupported operations or dynamic shapes.

### 2. Intermediate representation / lowering
- Convert the supported graph to a small typed operation set (e.g., `Linear`, `ReLU`, `Tanh`, `Add`).
- Preserve tensor shapes and operation order.
- Optionally fold constants and eliminate unused nodes.

### 3. Luau backend
- Generate code for fixed-shape inference.
- Serialize parameters into Luau-compatible data; benchmark text tables versus packed buffers if appropriate.
- Minimize intermediate allocations and reuse scratch arrays when safe.
- Keep generated modules small enough to be practical in Roblox workflows.

### 4. Verification
- Run the same input vectors through PyTorch and generated Luau.
- Report maximum absolute error, relative error, and prediction/ranking agreement.
- Test edge cases: zero vectors, large values, negative values, and near-tied outputs.
- Validate runtime compatibility and behavior in Roblox Studio.

## Optimization research opportunities
1. **Model-specific code generation:** Specialize matrix operations to known layer dimensions.
2. **Operator fusion:** Reduce intermediate arrays and dispatch overhead.
3. **Buffer reuse:** Preallocate temporary vectors.
4. **Quantization:** Explore smaller weight representations; benchmark decompression and numerical degradation rather than assuming speedups.
5. **Code size vs. runtime:** Compare unrolled arithmetic, generic loops, and packed parameter formats.

## Benchmarks and success criteria
Compare against a straightforward hand-written Luau MLP implementation and, if usable, Roblox's native ML inference service.

Measure:
- Inference latency distribution (median, p95) on representative devices.
- Memory use and serialized module/model size.
- Numerical error and top-action agreement with PyTorch.
- Scaling with network width/depth and number of agents.
- Frame-time impact during realistic gameplay.

**Success criterion for MVP:** export at least one useful small MLP, match PyTorch outputs within documented tolerances, and show measured Roblox runtime performance sufficient for event-driven movement selection. Do not promise specific millisecond latency without testing.

## Suggested roadmap

### Phase 0 — Deployment feasibility spike
- Test Roblox MLService custom-model availability and deployment constraints.
- Hand-port one small trained MLP into Luau and benchmark it.
- Decide whether a compiler is justified compared with native inference.

### Phase 1 — Working exporter
- Implement a restricted graph parser and parameter extractor.
- Generate Luau for Linear/ReLU/Tanh fixed-shape networks.
- Add Python-side golden test vectors and a Roblox test harness.

### Phase 2 — Production usability
- Add CLI, clear error messages, documentation, example project, CI, and release packaging.
- Benchmark desktop and mobile; optimize storage and allocations.

### Phase 3 — Advanced capabilities (only if justified)
- Quantization, more operators, recurrent policies, operator fusion, broader model support.
- Research evaluation of model-specific code generation for constrained scripting runtimes.

## Potential CLI (illustrative, not implemented)

```bash
pytorch-luau export \
  --exported-program movement_model.pt2 \
  --output MovementModel.luau
```

## Integration with RobloxBasketball
- Train a motion-transition ranking model in PyTorch using state features and reviewed motion data.
- Export it to Luau.
- Call it only at useful decision points (foot plant, dribble transition, gather, contact onset), initially avoiding inference on every frame.
- Generate legal candidate transitions with deterministic rules **before** ranking them.
- Keep physics, dribble limits, collision validation, and authoritative multiplayer state independent of the model.
- Use a backend abstraction so the game can switch among native MLService, generated Luau, and deterministic fallback.

## Open-source positioning
A narrow, well-tested PyTorch-to-Luau inference compiler for Roblox developers. Value comes from reliable deployment, reproducible benchmarks, and ergonomic tooling—not claiming support for arbitrary PyTorch models.

Possible repository structure:

```text
pytorch-luau/
  compiler/          # PyTorch export parsing, IR, Luau generation
  runtime/           # Luau helpers / optional shared kernels
  examples/          # Small classifiers and Roblox usage
  tests/             # PyTorch equivalence and Luau tests
  benchmarks/        # Performance harnesses and results
  docs/              # Supported ops, limitations, integration
```

## Risks and decisions to revisit
- Roblox native ML capabilities may make a custom compiler unnecessary for some models.
- Luau CPU inference may be too slow for larger networks or frequent calls.
- Generated code size, asset limits, and memory overhead may constrain model complexity.
- PyTorch-to-Luau numerical behavior requires testing; exact bitwise equivalence is not expected.
- Native Roblox API capabilities can change; recheck documentation when implementing.
- Commercial game code, datasets, and third-party model licenses should be kept compatible with open-source distribution.

## First actionable experiment
Train a tiny 32→64→64→8 MLP on synthetic movement states, manually export weights, implement the forward pass in Luau, and benchmark it in Roblox Studio. Compare output vectors to PyTorch and investigate MLService model-upload feasibility. **Only then** decide whether to build the reusable compiler.