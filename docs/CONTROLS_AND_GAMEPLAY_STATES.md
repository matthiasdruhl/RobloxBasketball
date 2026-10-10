# Roblox 1v1 Basketball — Controls and Gameplay States

Status: Working design notes, October 9, 2026. Distinguishes agreed direction from unresolved proposals. No implementation yet.

## Game format
- Competitive half-court, isolation-focused 1v1 inspired by The Next Chapter (TNC) and Ballislife.
- Design around a **four-dribble limit** (event rules may vary). Each bounce is a strategic resource.
- Emphasize jabs, pivots, hip swivels, first-step acceleration, deceleration, post play, legal gathers, finishing, defensive positioning and reaction.
- Rules such as scoring, possession changes, foul handling and exact dribble counting need independent confirmation before being locked.

## Controls agreed in discussion
- **Left analog stick:** desired movement direction.
- **Right analog stick:** intended overall body facing/orientation. Hips and shoulders share one primary orientation input; small natural torso differences can be animated automatically. Do not give separate shoulder/hip controls.
- **L2 / LT:** dribble control. **Hold** to keep triggering legal successive bounces; **release** to stop commanding further bounces and permit a legal float/hesitation. Releasing does *not* automatically gather. Current bounce finishes normally.
- **R2 / RT:** combined pickup / shot fake / shooting button. Tap/hold distinction; detailed context mapping remains to be tested (see below).
- **Offense and defense have different mappings**, while both retain left-stick movement and right-stick orientation.
- Controller-first mechanics are being explored, but keyboard/mouse and mobile translation have not been finalized.

## R2 / RT behavior — unresolved conflict to prototype
Original user proposal:
- Tap R2 while **holding L2** = deliberate dribble pickup.
- Tap R2 while **not holding L2** = shot fake.
- Hold R2 = shooting motion / shot timing.

Alternative suggested in conversation (not accepted as final): possession-state-driven mapping:
- Ball secured before dribble: tap = shot fake, hold = shoot.
- Live dribble (including a float with L2 released): tap = gather/pickup, hold = gather into shot.
- Dribble already picked up: tap = shot fake, hold = shoot.

**Critical unresolved case:** If L2 is released during a live float and R2 is tapped, is that a gather only, or a gather plus shot fake? Avoid illegal fake-without-gather behavior. Shooting may initiate an automatic legal gather. Early release of a held shot could produce a fake before a commitment point. Accidental pickups must be minimized.

## Proposed control states
These are high-level **control modes**, not the full legal state machine:
1. **Stationary footwork:** triple threat, jabs, pivots, post footwork, shot fakes. Used before the dribble and after pickup, with different permitted actions.
2. **Live dribble:** drives, rhythm control, floats/hesitations, hand changes, retreats, dribble backdowns.
3. **Gather / finish:** transition into layup, euro step, hop step, pull-up, release; track legal step sequence.
4. **Defense:** slides, retreats, hip turns, reaches, contests, positioning; separate button meanings.

Posting up / ball protection / explosive drive can be **movement submodes**, not necessarily exclusive possession states.

## Independent legal state
Track possession and legality separately from control modes:
- Before first dribble: ball secured, legal pivot and ability to begin dribble.
- Live dribble: ball still live, count bounces, permit continuing or legal hesitation.
- Picked up / dribble ended: cannot initiate another dribble; can pivot, fake, shoot or step through if legal.
- Gather in progress: track gather instant, support foot, step count, airborne state and ball release.
- Defense / no possession.

Do not represent legality using only a `canDribbleAgain` boolean. Track planted/pivot foot, possession phase and actual ball contact. A float cannot suspend the ball indefinitely or allow carrying.

## Movement and footwork intent
- Players command movement and orientation; controller/animation system handles detailed foot placement, joint movement and realistic recovery.
- Jab: directional command (potentially a footwork modifier), pivot foot remains anchored, free foot extends with variable depth and direction.
- Hip swivel / pivot: right-stick body rotation constrained by planted foot and stance; procedural torso nuance, not independent shoulder control.
- Post: pressure and position, pivots, drop steps and spins depend on planted foot, facing and defender geometry.
- Gather / euro: directional step-by-step choices should matter; not a one-button canned euro.
- Input should feel responsive; animation should not arbitrarily override the requested action.

## Buttons not yet decided
- L1/LB, R1/RB, four face buttons, D-pad, L3/R3.
- Footwork modifier, stance depth, ball hand manipulation, post engagement, explosive acceleration, defensive reach/contest/contact are *candidates*, not confirmed assignments.
- Avoid overloading right stick with incompatible simultaneous tasks; test reachability while both thumbs remain on sticks.

## Next design decisions
1. Finalize R2 tap/hold behavior in every possession phase, especially live float.
2. Define exactly how a jab/pivot/post move is requested with currently free buttons.
3. Specify dribble count semantics and illegal pickup/carry/travel behavior under chosen 1v1 rules.
4. Define offense-to-defense transitions and held-button suppression on possession changes.
5. Prototype ergonomics, timing windows, input buffering, and keyboard/mobile adaptations.