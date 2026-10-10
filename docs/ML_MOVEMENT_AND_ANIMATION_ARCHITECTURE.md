# Roblox 1v1 Basketball — ML Movement and Animation Architecture

Status: Proposed technical direction, October 9, 2026. Research hypotheses and phased implementation plan, not proven capabilities.

## Core thesis
Build a **player-directed, state-conditioned hybrid motion controller**. The player determines movement intent; a rules/physics layer limits what is legal and physically possible; an ML selector eventually ranks, blends and chains appropriate motion segments from a curated animation library. Procedural corrections maintain plausible feet, body and ball alignment. ML should not choose basketball strategy on the player's behalf.

## Runtime architecture
1. Input abstraction: left stick movement, right stick orientation, L2 dribble timing, R2 pickup/fake/shoot, context-specific modifiers.
2. Player state: possession and dribble phase/count, pivot/support foot, ball location/velocity, body facing, current velocity/acceleration, animation phase, movement mode, opponent-relative position.
3. Deterministic rules and physical constraints: traveling, dribble legality, gather steps, contact restrictions, reachable positions and transition eligibility.
4. Candidate generator: shortlist feasible motion clips/segments and transition windows.
5. Motion selector: begin with deterministic motion matching; later replace/augment ranking with a small learned model conditioned on intent, state and style.
6. Animation renderer: authored clips, blending, speed adjustment, inverse kinematics and procedural joint/foot correction.
7. Movement/collision controller: maintain authoritative world displacement, constrain overlap and reconcile visual motion.

Maintain **desired**, **allowed**, and **actual** velocity/displacement as separate quantities. The animation selector needs both intent and actual movement, so collisions can be added without rewriting locomotion.

## Animation representation
- Keep a **small number of high-level control states**: stationary footwork, live dribble, gather/finish, defense.
- Store many **short reusable motion segments**: load, plant, jab extend/recover, pivot, first step, decelerate, crossover, defensive slide, gather, finish, etc.
- Each segment has start/end pose, root trajectory, body orientation, foot support/contact, ball-hand/phase, required legal state, valid transition windows, and possible successor families.
- Hand-authored constraints specify *which transitions are legal*; ML chooses *which valid transition best executes the user's request*.
- A smooth animation blend alone does not guarantee correct foot planting or collision response; explicitly enforce these.
- Prefer authored motions plus procedural adaptation initially over generating arbitrary joint poses from a neural model.

## Training from real basketball footage
Proposed offline pipeline:
1. Acquire rights-cleared basketball clips and/or motion-capture data, ideally 1v1, post footwork, drives, defense and finishes.
2. Estimate 3D skeleton/joint rotations, angular velocity, root velocity/acceleration, body facing, foot contact, approximate center of mass, ball location/phase and defender geometry. Record confidence/occlusion.
3. Normalize to player-relative coordinates and optionally proportions, retaining body size and stride traits separately.
4. Segment motion **sequences**, not just individual frames, at meaningful foot-contact/action transitions.
5. Learn temporal motion embeddings; use clustering (e.g. HDBSCAN/k-means) to discover related patterns within interpretable action families.
6. Preserve original full sequences, not only cluster labels. Use learned similarities to build motion search indices and candidate transitions.
7. Train/evaluate a compact state-and-intent-conditioned motion ranking model, starting from a deterministic motion-matching baseline.

**Data caveats:** broadcast video lacks exact depth, contacts, control intentions and clean views; monocular 3D pose estimation is noisy. Observed trajectories are not direct labels for player button intent. Use mocap/controlled captures where possible and measure reconstruction quality. Publicly accessible video is not necessarily licensed for commercial training.

## Archetypes and customization
- Prefer **one shared model conditioned on continuous style vectors** rather than separate networks per archetype initially.
- Separate movement capability (size, speed, braking, agility) from stylistic preferences (rhythm, stance, stride, pivot patterns, transition preferences).
- Cluster movements first; derive player movement profiles from distributions of motions, timing and execution. Do not pre-label all clips into archetypes before learning movement structure.
- Example inspirations: shifty guard, deliberate post/stepback scorer, long-stride slasher, technical post scorer. These are illustrative, not measured clusters.
- Players could eventually combine compatible movement styles; archetype selection should not bypass physical attributes or competitive balance.

## Roblox integration and performance
- Roblox animation playback/blending and inverse kinematics may support the visual layer, but the custom movement/rules controller must enforce legal planted feet and real displacement.
- Begin with R15-compatible rig; test foot planting, pivots, deceleration and blends before deciding on a custom rig.
- Explore Roblox's documented ML inference facilities only after checking availability, model-format support, client platform support, latency and memory. Do not assume a full PyTorch runtime inside Roblox.
- Train offline (e.g. Python/PyTorch), precompute embeddings/transition candidates, and keep runtime inference compact. No per-frame external AI API calls.
- Ensure responsive local prediction, server authority for rules/collisions and a deterministic fallback on weak/unsupported devices. Benchmark networking and inference.

## Contact as an optional layer
**Agreed development priority: open movement first; contact later.** Build contact-awareness interfaces from the start without making movement quality depend on advanced contact animation.
- Stage A: open movement with no opponent collision.
- Stage B: non-contact opponent to test attack/defense interaction.
- Stage C: basic collision volumes prevent walking through defenders; existing movement animations can continue with simple speed/IK correction.
- Stage D: authored contextual bracing, bump, stumble, post-pressure and recovery animations.
- Stage E (optional): learned contact-aware animation selection, if data and testing justify it.

Collision detection/allowed displacement should be deterministic/authoritative. ML may select **visual responses**, not decide whether collision exists or permit illegal overlap. Basic collision is a shippable fallback if learned contact proves unreliable.

## Recommended implementation milestones
1. **Technical proof of movement:** left-stick locomotion, right-stick facing, triple threat, planted pivot, jab, first-step drive and deceleration. No ML, no contact, possibly no ball.
2. **Ball and legality:** L2 hold/float, R2 pickup/fake/shoot, four-dribble tracking, gather and pivot enforcement.
3. **Motion library:** consistent rig, metadata, segment boundaries, transition compatibility and IK corrections.
4. **Deterministic motion matching:** state-conditioned candidate selection, measurable responsiveness and foot sliding.
5. **ML selector:** train on extracted/captured motions, compare against deterministic baseline for latency, legality, responsiveness and transition quality.
6. **Archetype conditioning:** shared model plus style vectors, test variety versus competitive fairness.
7. **Defender and collision:** non-contact, then basic blocking, then optional richer reactions.

## Success criteria / risks
- Inputs feel immediate; player intent is preserved, including manual dribble timing and gathers.
- No illegal footwork or dribble resets; foot contacts and ball-hand timing look plausible.
- Smooth transitions under direction changes; stable framerate on target Roblox devices.
- Contact can be disabled without breaking open movement.
- ML must demonstrate a measurable improvement over simple motion matching before being made a runtime dependency.
- Main risks: quality/licensing of motion data, foot sliding, ball/hand synchronization, latency, network reconciliation, and excessive control complexity.