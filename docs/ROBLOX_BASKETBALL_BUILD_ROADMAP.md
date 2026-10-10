# Roblox Basketball — Development Roadmap

**Status:** Working plan, October 9, 2026  
**Scope:** Four-dribble, guard-focused 1v1 basketball game; eventual single-player AI and multiplayer.  
**Principle:** Build a testable basketball simulation first, develop ML-assisted motion second, and polish the Roblox game third. Keep a playable Roblox prototype available early.

## Guiding decisions

- AI coding agents (e.g., Codex) can implement much of the code, but gameplay feel, correctness, and motion quality require human evaluation.
- Deterministic, server-authoritative game rules and collision resolution remain the source of truth. ML may propose motion, select animation transitions, and adapt contact reactions within physical constraints.
- Plan for networking from the beginning, even if public multiplayer launches later.
- Build repeatable tests and scripted bots early. Competitive self-play is not a substitute for coverage-driven tests.
- Use pretrained computer vision and pose estimation first; train basketball-specific extraction models only when labeled errors justify them.
- TNC-style four-dribble 1v1 footage is the target source for studying high-level guard movement, but commercial use requires appropriate footage rights. Verify quality and reconstruction feasibility before scaling collection.
- Keep the game playable without ML so each learned component can be compared with a deterministic baseline.

## Phase 1 — Core basketball simulation

**Rough estimate:** 2–4 weeks. **Priority:** Highest.

Build the court and basic two-player environment; a responsive custom guard controller with acceleration, deceleration, turning, body orientation, and stance; basic collisions; basketball and possession state; four-dribble rules; gather and footwork state; basic shooting and scoring. Use placeholder animations.

**Architecture requirements:** Define a shared state/action schema and authoritative simulation contract. Separate player input, movement constraints, ball state, collision resolution, rules, and presentation. Structure for client prediction/server reconciliation even before full multiplayer.

**Exit criterion:** Two controllable characters can complete repeatable, legal 1v1 possessions with plausible movement and contact.

## Phase 2 — Automated testing foundation

**Rough estimate:** 1–2 weeks.

Add deterministic input replay, state and event logging, collision/interpenetration checks, acceleration bounds, dribble-count and gather validation, animation-transition diagnostics, foot-sliding metrics, and ball-hand alignment checks where available. Implement simple scripted offensive and defensive bots and a scenario runner.

**Exit criterion:** Automated tests can repeatedly exercise common drives, stops, changes of direction, and contact events; regressions can be reproduced from recorded inputs.

## Phase 3 — Footage-to-motion extraction

**Rough estimate:** 3–6 weeks; research uncertainty is high.

Pilot a small number of permitted TNC-style 1v1 possessions. Detect and track both players, estimate 2D/3D body poses, calibrate movement against the court, segment possessions, and label dribble, gather, contact, and movement events. Review and correct reconstructions manually. Preserve full possessions and shorter labeled segments, including pre-contact and recovery frames.

**Important limitations:** Single-camera video does not reveal hidden limbs or true contact forces. Ball occlusion and close body contact can corrupt reconstructed skeletons. Do not assume automatic labeling is accurate without review.

**Exit criterion:** A small verified set of 3D offensive and defensive sequences with usable foot placement, timing, and contact context.

## Phase 4 — Animation library and motion matching

**Rough estimate:** 3–6 weeks.

Clean and retarget extracted motions onto the Roblox character rig. Build a reusable motion library for initiation, live-dribble movement, stops, gathers, finishes, defensive reactions, and contact. Implement motion matching or state-conditioned selection, smooth blending, foot IK, and ball-hand coordination. Support differing guard proportions where practical.

**Exit criterion:** A compact library of high-quality motions produces responsive, recognizable guard movement with minimal sliding and believable transitions.

## Phase 5 — Learned movement and contact

**Rough estimate:** 4–8 weeks; research uncertainty is high.

Train a lightweight movement-selection model using current pose and velocity, foot-contact state, controller intent, player attributes, dribble state, and defender context. Add paired offensive/defensive contact-reaction selection and optionally learned balance or motion adjustments. Keep collision response and rule enforcement deterministic and authoritative.

**Exit criterion:** Learned selection measurably improves realism or transition quality over motion matching/rules without hurting input responsiveness, stability, or multiplayer consistency.

## Phase 6 — AI opponents and self-play

**Rough estimate:** 3–6 weeks.

Use the scripted bots from Phase 2 as baselines. Develop offensive and defensive policies for four-dribble 1v1, initially in a fast headless training simulation if useful. Explore reinforcement learning and self-play, export small policies or distilled behavior to Roblox, and validate against actual Roblox gameplay. Separately build exploration/stress agents that deliberately search for collisions, animation failures, rule exploits, and rare input sequences. Retain deterministic regression replay.

**Exit criterion:** A usable single-player opponent and automated adversarial testing that finds and reproduces meaningful failures.

## Phase 7 — Multiplayer, customization, and public-alpha polish

**Rough estimate:** 4–8 weeks.

Harden client prediction, server reconciliation, latency behavior, and exploit resistance. Add matchmaking, progression, cosmetics, guard customization, camera and input polish, mobile/controller support as appropriate, performance profiling, and model inference optimization.

**Exit criterion:** Stable public alpha supporting human-versus-human and human-versus-AI 1v1.

## ML priorities

| Capability | Priority | Planned stage |
|---|---|---|
| Pretrained video pose reconstruction | Essential | Phase 3 |
| Motion cleanup, retargeting, and quality review | Essential | Phases 3–4 |
| Motion matching and procedural IK | Essential baseline | Phase 4 |
| Learned animation selection | High | Phase 5 |
| Learned paired contact reactions | High | Phase 5 |
| Competitive offense/defense agents | Medium–high | Phase 6 |
| Exploration and regression agents | High | Phases 2 and 6 |
| Neural motion generation | Experimental | After Phase 5 |
| Fully learned collision physics | Low / research only | Not a launch dependency |

## Parallel work and dependencies

- **Start video reconstruction experiments while Phases 1–2 are underway.** A short feasibility pilot can expose footage and rigging limitations before substantial dataset work.
- **Design multiplayer-aware simulation interfaces in Phase 1.** Do not wait until Phase 7 to consider server authority and synchronization.
- **Use Python for offline computer vision and training.** Roblox remains the game runtime and final validation target.
- **If using a separate headless training simulator, continuously compare its behavior with Roblox.** Differences in collision and movement dynamics can make trained policies fail to transfer.
- **Don't gate early playtesting on neural networks.** Scripted and motion-matching baselines should remain available.

## First four weeks — aspirational milestones

| Week | Target |
|---|---|
| 1 | Roblox court, basic guard controller, shared game-state/action schema |
| 2 | Ball handling, four-dribble rules, gathers, basic shooting, simple contact |
| 3 | Scripted offense/defense bots, deterministic replay, physics/rules test suite |
| 4 | First permitted TNC-style video-to-3D reconstruction and initial retargeting experiment |

**Month-one success:** A basic, repeatably testable basketball simulation **and** at least one real movement sequence reconstructed into an inspectable 3D skeleton. The schedule is a planning target, not a guaranteed completion estimate.

## Key risks and evaluation gates

1. **Footage quality and rights:** Secure appropriate use rights and verify full-body visibility before bulk processing.
2. **Reconstruction accuracy:** Inspect feet, hip orientation, occlusion, ball-hand contact, and paired contact from multiple virtual viewpoints.
3. **Responsiveness:** Measure input-to-action delay and transition behavior before adding learned models.
4. **Physical consistency:** Enforce legal footwork, dribble limits, and collision constraints outside ML.
5. **Training-to-Roblox mismatch:** Compare trajectories, contact outcomes, and rules across simulation environments.
6. **Scope:** Only expand to generative motion or learned physics when a measurable deficiency in the baseline justifies it.

## Definition of the project

A realistic, responsive, four-dribble 1v1 basketball game centered on guard footwork, ball handling, body contact, and player-specific movement. ML is a tool for extracting, selecting, and eventually synthesizing realistic motion, and for developing AI opponents and automated testers—not a replacement for sound game physics or deterministic rules.