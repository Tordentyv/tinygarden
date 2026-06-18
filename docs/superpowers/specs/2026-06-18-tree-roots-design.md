# Tree Roots Growing Beneath the Earth

## Overview

Add visible root systems that grow underground as trees develop. Roots spread slowly during the trunk growth phase, forming subtle single-pixel tendrils beneath the soil. When roots from different trees grow near each other, they connect, creating an interconnected underground network.

## Growth Behavior

### Initialization
- When a tree plant is created, seed 2-4 root tendrils at the tree's base (x position, at terrain level)
- Each tendril has an angle biased downward: between 100-260 degrees (mostly down and outward)
- Tendrils stored as `roots: [{x, y, angle, len, maxLen, children}]` on the tree plant object

### Growth Rate
- Each game tick during trunk phase (and continuing through canopy/life), a tree has ~0.02 chance per tick of extending one of its roots by 1 pixel
- This is much slower than trunk growth (~0.07-0.11 grow rate) — roots are barely perceptible
- A root tendril picks its next pixel position based on its angle, with slight random wobble (+/- 0.2 radians)

### Branching
- When a root extends, ~0.08 chance of spawning a child branch at the current tip
- Child branches have slightly more horizontal angles than parent
- Max branch depth: 2 (root -> child -> grandchild)
- Each tendril has a maxLen of 6-12 pixels; children are shorter (3-6 pixels)

### Depth Limits
- Roots cannot grow above terrain level (stay underground)
- Max depth: 12 pixels below the terrain surface at that x position
- Roots stay within canvas bounds

### Connection Detection
- After each root extension, check if the new tip is within 3 pixels of any root pixel belonging to a different tree
- If detected, curve the root toward the nearest foreign root pixel (adjust angle toward it)
- Once the root reaches the adjacent pixel, stop growing that tendril (connection made)
- No special visual treatment at connection points — just two tendrils meeting

## Visual Style

### Colors
Root pixel colors are subtle dirt-adjacent browns, chosen randomly per pixel:
- `[65, 42, 25]` — dark root brown
- `[72, 48, 28]` — medium root brown  
- `[58, 38, 22]` — deep root brown

These sit between dirt `[80, 50, 30]` and dark earth `[55, 35, 20]`, making roots visible but not prominent.

### Rendering
- Roots are drawn via the existing `setPixel` system with the tree's `plantId` for ownership
- Single-pixel width always (no thickening near base)
- Roots overwrite dirt pixels but are tracked in the pixel ownership stack
- When a tree dies/falls, its roots are removed via `removePlantPixels` (same as above-ground parts)

## Data Structure

```javascript
// Added to tree plant objects during creation:
roots: [
  {
    x: Number,        // current tip x
    y: Number,        // current tip y
    angle: Number,    // growth direction in radians
    len: Number,      // current length
    maxLen: Number,   // max length (6-12)
    depth: Number,    // branch depth (0=main, 1=child, 2=grandchild)
    children: [],     // child branches
    connected: false, // true if reached another tree's root
  }
]
```

## Integration Points

- `growTree()` function: add root growth logic alongside trunk/canopy phases
- Root growth happens regardless of phase (trunk or canopy), just at very low probability
- New helper function `growRoots(p)` called from `growTree()`
- Need a spatial lookup to find nearby foreign roots efficiently — a simple array scan of all trees' root tip positions is fine given the low tick rate

## Edge Cases

- Trees that die: roots removed via existing `removePlantPixels` mechanism
- Trees hit by lightning: roots removed along with the rest
- Roots near canvas bottom edge: stop growing if y >= H
- Roots that would enter sky (above terrain): clamp to terrain[x] + 1
