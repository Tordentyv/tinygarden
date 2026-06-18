# Swinging Monkeys

## Overview

Add monkeys that swing from tree to tree across the full screen width once the forest is mature enough. They appear rarely, travel edge-to-edge using pendulum arcs between tree canopies, and are drawn on the overlay layer like other animals.

## Spawn Conditions

- Monkeys can only spawn when there are 10+ mature trees (`type === 'tree' && done === true`) in the `plants` array
- Maximum 2 monkeys on screen simultaneously
- Spawn chance: ~0.01 per tick (similar rarity to bird flocks)
- Spawn position: just off-screen on one edge (x = -5 or x = W + 5)
- Direction: random left-to-right or right-to-left, fixed for the monkey's lifetime

## Movement — Pendulum Swing

### Finding grip points
- A grip point is the top of a mature tree's canopy: `(tree.x, tree.ground - tree.height)`
- Monkey always targets the next mature tree in its travel direction
- "In range" means within 60px horizontal distance ahead of current position
- Trees are searched in order of proximity in the travel direction

### Swing arc
- From a grip point, the monkey swings in a pendulum arc to the next grip point
- Parametric motion over a fixed number of frames (8-12 frames per swing depending on distance):
  - `x` moves linearly from current grip to next grip
  - `y` follows a downward arc: `gripY + sag * sin(PI * progress)` where `sag` = 30-50% of horizontal distance, capped at 15px
- At arc completion, monkey arrives at the next grip point

### Pause between swings
- Brief pause at each grip point: 3-5 ticks
- During pause, monkey rendered in compact "gripping" pose

### Gap bridging
- If no tree is within 60px ahead, extend search to 100px and use a longer, flatter arc (sag = 20% of distance)
- If still no tree found and monkey hasn't reached the opposite screen edge, despawn (this handles sparse forests gracefully)

### Completion
- Monkey continues swinging until its x position moves past the opposite screen edge
- Then removed from the animals array

## Visual Appearance (5-7 pixels)

### Colors
- Body: `[140, 90, 40]` — warm brown
- Tail: `[110, 70, 30]` — darker brown
- Arm: `[140, 90, 40]` — same as body

### Swing pose (during arc)
- Body: 2 pixels at monkey position (horizontal pair)
- Arm: 1 pixel extending upward toward the grip point (above and behind)
- Tail: 2 pixels trailing behind/below (offset opposite to travel direction, curving down)

### Grip pose (during pause)
- Body: 2 pixels (vertical pair, hanging from grip)
- Arm: 1 pixel directly above (at grip point)
- Tail: 2 pixels curling below

## Data Structure

```javascript
{
  type: 'monkey',
  x: Number,           // current position
  y: Number,
  dir: 1 | -1,         // travel direction
  state: 'swinging' | 'gripping',
  gripX: Number,       // current/target grip point x
  gripY: Number,       // current/target grip point y
  nextGripX: Number,   // next swing target x
  nextGripY: Number,   // next swing target y
  swingFrame: Number,  // current frame in swing arc
  swingTotal: Number,  // total frames for this swing
  pauseTimer: Number,  // ticks remaining in grip pause
  life: Number,        // not used for timeout, just for compat with animal cleanup
}
```

## Integration

- Monkeys live in the existing `animals` array
- Drawn via `setOverlay` (same as mice and birds)
- Updated in `updateAnimals()` alongside existing animal logic
- New functions: `spawnMonkey()`, `drawMonkey(m)`, and update logic within the animal loop
- Spawn check uses: `plants.filter(p => p.type === 'tree' && p.done).length >= 10`
