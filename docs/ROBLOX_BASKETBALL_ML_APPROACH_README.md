# Roblox Basketball — ML Movement & Contact Approach

**Project design snapshot — October 9, 2026**  
**Status:** Proposed architecture and implementation plan; not yet validated by experiments.

## 1. Product scope and guiding decisions

We are building a realistic, guard-focused **Roblox 1v1 basketball game with a four-dribble limit**. The aim is expressive footwork and body movement, realistic offensive/defensive interactions, and physical contact without sacrificing responsive user controls.

- Start with **1v1**; do not model help defense, switches, screens, passing, team rotations, or off-ball five-on-five behavior at launch.
- Initial guard presets: **shifty/technical** and **explosive/slashing**, implemented as continuous attribute combinations rather than completely separate movement systems.
- Support eventual customization of height, weight, wingspan, torso/leg proportions, and athletic/skill attributes. Keep body geometry distinct from skill ratings.
- **Four dribbles maximum** per offensive possession. Players may gather earlier; do not assume they must use all four. Define precisely what counts as a dribble and how the fourth dribble transitions into a legal gather, pivot, shot, or finish.
- Prioritize high-quality foot plants, hip turns, deceleration, balance, ball-hand synchronization, gathers, and responsive collision interactions.

## 2. Central ML thesis

**Use real high-level 1v1 footage to build a library of realistic motion and paired contact responses. At runtime, a small attribute-conditioned model selects feasible motion transitions based on the player's current physical state and controller input, while deterministic game systems enforce physics, ball control, collisions, and rules.**

Crucial separation:

1. **User intent:** Controller input specifies intended movement, attack, dribble, gather, etc. ML must not override the user's direction simply because another basketball decision looks better.
2. **Feasibility:** Deterministic constraints filter out impossible or illegal transitions (foot contact, acceleration limits, dribble state, ball reach, collision geometry).
3. **Motion selection:** A compact ML model ranks valid animations/transitions using body state, input, player attributes/style, defender geometry, and possession/dribble context.
4. **Authoritative simulation:** Physics/rules control positions, velocity, legal footwork, ball state, and collision outcomes; animations must match the simulated result.
5. **AI opponent (later):** A separate defensive decision policy may learn *what* a defender should do; it is distinct from the animation selector.

**Do not make a neural network the authoritative collision physics engine.** Contact-response ML can select coordinated reactions, not invent forces or arbitrarily teleport players.

## 3. Primary footage strategy: TNC-style 1v1

Use continuous, high-quality **The Next Chapter (TNC)** 1v1 possessions as the primary candidate footage source because the 1v1 format, frequent contact, and four-dribble constraint closely match our game.

- Prefer stationary or consistent camera angles, visible feet, full-body views, continuous plays, minimal editing, and high resolution.
- Include both offensive and defensive players, with **complete possessions** preserved alongside shorter movement segments.
- Record the specific video's rules and count dribbles explicitly; do not assume every possession uses all four.
- Sample diverse high-level guards, matchups, movement styles, and defenders; avoid overfitting to one athlete or one camera.
- Consider supplemental footage for rare movements not represented in TNC.
- **Rights:** Publicly viewable YouTube videos are not automatically licensed for downloading, ML training, or commercial Roblox use. Seek appropriate permissions/licensing, ideally original source footage, before committing to a production dataset.

### Initial dataset milestones (engineering targets, not proven minimums)

| Milestone | Target | Purpose |
|---|---:|---|
| Reconstruction feasibility | 10 continuous possessions | Verify two-player tracking, 3D motion, foot plants, and multi-angle replay |
| Initial extraction evaluation | 50–100 manually reviewed possessions | Find errors in possession boundaries, ball/dribble detection, and motion labels |
| First broader motion set | 100–200 varied possessions | Test segmentation, transition library, and initial selector feasibility |
| Contact study | ~200–400 reviewed contact events **if available** | Explore contact classes, paired reactions, and near-contact negatives |

These numbers are **planning hypotheses**, not claims that the resulting models will be accurate. Expand only after measuring reconstruction quality and diversity.

## 4. Offline video-to-motion extraction pipeline

```text
Licensed TNC-style video
    ↓
Possession discovery / candidate clipping
    ↓
Player + ball detection and tracking
    ↓
2D pose estimation for both players
    ↓
3D pose/motion reconstruction + court calibration
    ↓
Temporal event detection (dribbles, hand contact, gathers, foot plants)
    ↓
Movement and contact segmentation/classification
    ↓
Human review, correction, and quality filtering
    ↓
Structured motion/contact dataset + animation candidates
    ↓
Retargeting / cleanup / foot locking / ball-hand alignment
    ↓
Roblox-ready animation library and ML training examples
```

### Initial automation vs. custom learning

| Extraction task | Start with | Later improvement |
|---|---|---|
| Player detection/tracking | Pretrained detector + multi-object tracker | Fine-tune for overlaps/contact |
| 2D joint detection | Pretrained human pose model | Basketball-specific refinement |
| 3D reconstruction | Pretrained 3D motion estimator | Corrected motion / multi-view footage |
| Court/world calibration | Geometry + manual calibration | Automatic calibration |
| Ball detection/tracking | Pretrained detector plus manual QA | Fine-tune small-ball tracker |
| Dribble/contact event detection | Trajectory heuristics + manual correction | Small temporal event classifier |
| Gather/footwork identification | Heuristics + explicit review | Trained temporal classifier |
| Contact region/intensity labeling | Proximity and motion features + review | Specialized contact event model |
| Motion quality | Physical checks + visual review | Learned quality scoring |

Do **not** train a large end-to-end video model from scratch. Reuse pretrained vision models, and train small specialized classifiers only after collecting reliable corrected labels. Exact dribble and gather timing, especially with occlusion, will be challenging.

### Human review interface

Build a simple gallery/replay tool that shows original footage beside reconstructed 3D skeletons from multiple virtual camera angles. Reviewers should be able to approve/reject, correct dribble and gather frames, mark contact events, fix player identity swaps, and flag sliding feet or implausible poses. Use corrections as labeled data; eventually prioritize uncertain cases via active learning.

**Important:** A rotated single-camera 3D reconstruction is an *estimate* of hidden anatomy, not ground truth. Heavy contact and overlapping bodies are particularly error-prone. Visible video cannot directly recover exact contact forces or physical impulses.

## 5. Motion categories and four-dribble context

Capture and label:

- **Initiation:** triple threat, jab, pivot, first step, stance changes.
- **Live-dribble:** straight-line acceleration, lateral cuts, crossovers, hesitations, deceleration, hip swivels, change of hand.
- **Gather:** early/late pickup, stride gather, jump stop, foot planting and transition to finish.
- **Finishing:** layups, extended steps, floaters, pull-ups, step-backs, contests.
- **Defense:** lateral slides, hip turns, cutoffs, retreats, recovery steps, body positioning, contests.
- **Contact:** paired offensive/defensive reactions throughout the above phases.

Store the **dribble count/state** with every frame or event, but do not prescribe that dribbles 1–4 must map to fixed tactical actions. Keep zero-, one-, two-, three-, and four-dribble possessions.

### Example structured state (illustrative)

```python
state = {
    "dribbles_used": 3,
    "ball_state": "live_dribble",
    "ball_hand": "right",
    "player_position": [0.0, 0.0],
    "player_velocity": [3.2, 0.0],
    "facing_angle": 0.0,
    "left_foot_planted": True,
    "right_foot_planted": False,
    "defender_relative_position": [0.8, 0.2],
    "defender_relative_velocity": [-0.4, 0.1],
    "body_attributes": {"height": 1.83, "mass": 79.0},
    "controller_intent": "drive_left",
}
```

Units and values above are examples; define canonical Roblox/world units and coordinate frames in implementation.

## 6. Contact: include in dataset now; staged runtime integration

**Decision:** Capture and label contact from the beginning. Do not initially train a model to determine authoritative collision forces or outcomes.

### Contact classes

1. Shoulder/body bumps during drives.
2. Chest-to-body cutoffs and pressure.
3. Hip-to-hip contact while turning the corner.
4. Arm/hand interactions, shielding, reaching, and contests.
5. Balance disruption, stumbling, and recovery.
6. Contact during gathers and finishes.

For each event, preserve pre-contact approach, impact window, and post-contact recovery for **both players**. Record estimated contact region, relative angle/speed, posture, planted feet, ball state, dribble count, and visible outcomes. Include near-contact and no-contact examples to reduce false triggers. Label uncertainty when bodies overlap or contact is visually ambiguous.

### Runtime contact architecture

```text
Offense state + input ─┐
                       ├─> deterministic collision detection and constraints
Defense state + input ─┘                 ↓
                             contact geometry / legal feasibility
                                         ↓
                          paired reaction candidate generation
                                         ↓
                           ML or rule-based reaction ranking
                                         ↓
                    authoritative physics + synchronized animation
                                         ↓
                         updated state for both players and ball
```

**Paired reactions matter:** A shoulder bump should not independently trigger two incompatible canned animations. The animation system must respect the actual collision resolution, maintain control responsiveness, and support differing player sizes and abilities.

### Contact implementation phases

- **Phase A (MVP):** Deterministic collisions + hand-authored or cleaned reaction animations; rules select plausible paired reactions.
- **Phase B:** Train a compact contact-event/reaction classifier or ranker from verified data to choose among feasible paired responses.
- **Phase C (optional):** More adaptive motion blending/retargeting conditioned on approach angle, speed, stance, and body geometry, while physics remains authoritative.

Start by supporting shoulder bump on a drive, chest-up defensive cutoff, and hip-to-hip turn-the-corner contact.

## 7. Runtime ML architecture

**Initial runtime model:** One small **attribute-conditioned motion-transition ranker**, shared across player archetypes. Inputs may include:

- Current locomotion/animation phase, velocity, acceleration, heading, stance, foot contacts, and balance.
- Player controller input and desired direction/action.
- Ball hand, dribble/gather state, dribbles remaining, and shot/finish intent.
- Relative defender position, orientation, velocity, and contact state.
- Player dimensions, physical capabilities, skill attributes, and optional learned style embedding.

**Output:** Scores over a **pre-filtered set of feasible animation transitions**, optionally including blend parameters. The model does not choose illegal actions or replace the user's tactical intent.

Keep collision detection, physics response, dribble legality, footwork rules, and ball authority deterministic. For multiplayer, use server-authoritative state with client-side prediction/reconciliation; animation selection should be lightweight and predictable. Deploy a compact model or lookup approximation in Roblox Luau, rather than a cloud inference request every frame.

Potential later models: paired contact-response ranker, defensive AI policy, motion adaptation/retargeting. Do not add these until there is a measured need.

## 8. Body customization and animation quality

A shared movement library must adapt to different guard builds. Retarget skeletons to Roblox rigs, then use procedural IK/foot locking and ball-hand constraints to correct stride length, reach, and planted-foot stability. Avoid treating height or weight as arbitrary speed multipliers; tune physical attributes separately and validate believable movement envelopes.

Prioritize:

- No obvious foot skating or penetration through the floor.
- Continuous motion at transition boundaries; minimal snapping.
- Accurate dribble-hand and gather-ball timing.
- Contact reactions consistent with displacement and both players' momentum.
- Responsive controls and stable network behavior.

## 9. Proposed development sequence

1. **Choose footage with suitable rights.** Inspect camera quality and identify 10 continuous 1v1 possessions.
2. **Prove extraction.** Track both players, estimate 2D/3D pose, calibrate court, and render rotatable skeleton replay. Assess feet and occlusion errors.
3. **Create reviewer workflow.** Approve/reject and correct dribbles, gathers, player tracks, contact, and motion labels.
4. **Build motion segments.** Extract basic initiations, lateral movement, cuts, stops, gathers, and defensive slides; clean/retarget a small high-quality set.
5. **Build deterministic Roblox controller.** Implement four-dribble state machine, ball possession, movement constraints, collisions, and basic footwork legality before introducing ML.
6. **Add animation candidate filtering and selection.** Start with rules; benchmark against a small trained ranker using the same feasible candidate set.
7. **Implement three contact interactions.** Shoulder drive, chest cutoff, hip-to-hip turn with paired animations and physics resolution.
8. **Train contact selection if justified.** Use reviewed contact examples and explicit negative examples; measure whether ML improves over rules.
9. **Expand footage diversity and customization.** Add guards, defensive styles, player attributes, retargeting, and eventually AI defense.

## 10. Success criteria and tests

Do not judge success by how many clips are collected. Establish measurable evaluations:

- **Extraction:** player identity consistency, 2D/3D pose plausibility, foot-contact timing, dribble/gather event accuracy, contact precision/recall.
- **Motion:** foot skating, transition continuity, ball-hand alignment, responsiveness to abrupt controller changes.
- **Contact:** no body interpenetration, plausible paired reactions, stable ball possession, predictable outcomes, minimal input lockout.
- **Gameplay:** dribble/gather legality, user control fidelity, server/client consistency, performance under real multiplayer latency.
- **Generalization:** evaluate on players, matchups, body types, and videos **not present in training**.
- **Ablation:** compare deterministic rule selection versus ML ranking to prove the model adds value.

## 11. Known risks and open decisions

- **Video rights:** secure permissions/licensing for the intended commercial use.
- **Single-view reconstruction:** feet, hands, and body contact can be hidden; no guaranteed physically accurate 3D ground truth.
- **Intent ambiguity:** real footage does not contain controller inputs. Infer coarse intent where possible and generate scripted/simulated input-to-state training examples; avoid pretending observed actions reveal exact controls.
- **Data volume:** proposed possession/contact counts are pilot targets, not established training thresholds.
- **Four-dribble rules:** finalize bounce counting, ball pickup, gathers, pivots, and what happens after the fourth dribble.
- **Contact/fouls:** determine desired physicality, foul rules, offensive pushing, and how often contact disrupts ball control.
- **Runtime feasibility:** benchmark inference and replication on Roblox/mobile/controller devices before committing to model complexity.
- **Animation coverage:** sparse transitions may require procedural motion blending and manual animation work even with ML.

## 12. Decisions explicitly *not* made

- No commitment yet to a particular pose-estimation model, neural architecture, training framework, or hosted GPU provider.
- No assertion that TNC has granted permission to use its footage.
- No guarantee that an automatic pipeline will achieve a particular accuracy or automation percentage.
- No commitment to end-to-end learned physics or a separate ML model for each guard type.

---

**One-sentence north star:** *Learn the style and timing of elite four-dribble 1v1 movement and contact from video; preserve player control and basketball legality through deterministic simulation; use lightweight ML only where it demonstrably improves animation and reaction selection.*