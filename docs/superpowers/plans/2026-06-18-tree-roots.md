# Tree Root Networks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add subtle underground root systems that grow slowly as trees develop, connecting when they approach roots from neighboring trees.

**Architecture:** Add a `growRoots(p)` function called from `growTree()`. Each tree gets a `roots` array of tendril objects seeded at creation. A separate spatial check connects nearby foreign roots. All rendering uses the existing `setPixel` with plant ownership.

**Tech Stack:** Vanilla JS, inline in `index.html`. No dependencies.

---

### Task 1: Seed root tendrils on tree creation

**Files:**
- Modify: `index.html` — inside the tree cases in `growNewPlant()` (around line 661-689)

- [ ] **Step 1: Add root initialization to all tree plant creation**

In the `growNewPlant()` function, every tree case creates a plant object with spread `...base`. Add root seeding after the plant is pushed. Find the section starting at line 661 (the `case 'oak':` block) and add a helper function before `growNewPlant()`, then call it after each tree push.

Add this function before `growNewPlant()` (around line 635):

```javascript
function seedRoots(plant) {
  plant.roots = [];
  const numTendrils = 2 + Math.floor(Math.random() * 3); // 2-4
  for (let i = 0; i < numTendrils; i++) {
    // Angles biased downward: 100-260 degrees (in radians ~1.75 - 4.54)
    const angle = (100 + Math.random() * 160) * Math.PI / 180;
    const maxLen = 6 + Math.floor(Math.random() * 7); // 6-12
    plant.roots.push({
      x: plant.x,
      y: plant.ground,
      angle,
      len: 0,
      maxLen,
      depth: 0,
      children: [],
      connected: false,
    });
  }
}
```

- [ ] **Step 2: Call seedRoots after each tree plant is pushed**

After each `plants.push(...)` call for tree types (oak, pine, birch, willow, cherry, maple, cypress, palm, baobab, juniper — lines 661-689), add:

```javascript
seedRoots(plants[plants.length - 1]);
```

So for example the oak case becomes:

```javascript
case 'oak':
  plants.push({ ...base, type: 'tree', treeType: 'oak', maxHeight: 55 + Math.floor(Math.random() * 80), canopySize: 8 + Math.floor(Math.random() * 10), canopyShape: ['wide','spreading','round','asymmetric'][Math.floor(Math.random()*4)], growRate: 0.07 });
  seedRoots(plants[plants.length - 1]);
  break;
```

Repeat for all 10 tree cases (pine, birch, willow, cherry, maple, cypress, palm, baobab, juniper).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: seed root tendrils on tree creation"
```

---

### Task 2: Implement root growth logic

**Files:**
- Modify: `index.html` — add `growRoots()` function and call it from `growTree()`

- [ ] **Step 1: Add root color palette constant**

Add after the `COLORS` object (around line 103), inside or just after it:

```javascript
const ROOT_COLORS = [
  [65, 42, 25],
  [72, 48, 28],
  [58, 38, 22],
];
```

- [ ] **Step 2: Add the growRoots function**

Add this function after `seedRoots` (before `growNewPlant`):

```javascript
function growRoots(p) {
  if (!p.roots) return;
  const id = p.id;

  function extendTendril(tendril) {
    if (tendril.connected || tendril.len >= tendril.maxLen) return;

    if (Math.random() < 0.02) {
      // Wobble the angle slightly
      tendril.angle += (Math.random() - 0.5) * 0.4;

      // Calculate next pixel
      const nx = Math.round(tendril.x + Math.cos(tendril.angle));
      const ny = Math.round(tendril.y + Math.sin(tendril.angle));

      // Bounds: stay underground, within canvas, max 12px deep
      const localTerrain = (nx >= 0 && nx < W) ? terrain[nx] : p.ground;
      if (nx < 0 || nx >= W || ny >= H || ny < localTerrain + 1 || ny > localTerrain + 12) return;

      // Draw the root pixel
      const col = ROOT_COLORS[Math.floor(Math.random() * ROOT_COLORS.length)];
      setPixel(nx, ny, col, id);

      tendril.x = nx;
      tendril.y = ny;
      tendril.len++;

      // Chance to branch
      if (tendril.depth < 2 && Math.random() < 0.08 && tendril.len > 2) {
        const childAngle = tendril.angle + (Math.random() - 0.5) * 1.2;
        const childMaxLen = 3 + Math.floor(Math.random() * 4); // 3-6
        tendril.children.push({
          x: nx,
          y: ny,
          angle: childAngle,
          len: 0,
          maxLen: childMaxLen,
          depth: tendril.depth + 1,
          children: [],
          connected: false,
        });
      }
    }

    // Recursively grow children
    for (const child of tendril.children) {
      extendTendril(child);
    }
  }

  for (const root of p.roots) {
    extendTendril(root);
  }
}
```

- [ ] **Step 3: Call growRoots from growTree**

In the `growTree(p)` function, add a call to `growRoots(p)` at the very end of the function (after all the phase logic, just before the closing `}`). Around line 1055:

```javascript
  // Grow roots underground (all phases)
  growRoots(p);
}
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: implement root growth with branching"
```

---

### Task 3: Implement root connection detection

**Files:**
- Modify: `index.html` — add connection logic inside `growRoots`

- [ ] **Step 1: Add a helper to collect all root tip positions for a tree**

Add this function after `growRoots`:

```javascript
function collectRootTips(p) {
  if (!p.roots) return [];
  const tips = [];
  function collect(tendril) {
    if (tendril.len > 0) {
      tips.push({ x: tendril.x, y: tendril.y, id: p.id });
    }
    for (const child of tendril.children) collect(child);
  }
  for (const root of p.roots) collect(root);
  return tips;
}
```

- [ ] **Step 2: Add connection check inside extendTendril**

In the `extendTendril` function inside `growRoots`, after the line `tendril.len++;` and before the branching check, add:

```javascript
      // Check for nearby foreign roots
      for (const other of plants) {
        if (other === p || other.type !== 'tree' || !other.roots) continue;
        const otherTips = collectRootTips(other);
        for (const tip of otherTips) {
          const dx = nx - tip.x;
          const dy = ny - tip.y;
          if (dx * dx + dy * dy <= 9) { // within 3 pixels
            // Curve toward the foreign root
            tendril.angle = Math.atan2(tip.y - ny, tip.x - nx);
            if (dx * dx + dy * dy <= 2) { // adjacent — connected
              tendril.connected = true;
            }
            return;
          }
        }
      }
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: connect roots between nearby trees"
```

---

### Task 4: Verify visually and adjust

**Files:**
- Modify: `index.html` (if adjustments needed)

- [ ] **Step 1: Start the dev server and verify**

Open the page in a browser. Set speed to 10x and watch for a minute or two. Verify:
- Roots appear as subtle brown pixels below the terrain surface
- They grow slowly — not flooding the underground
- Trees near each other eventually show roots meeting
- When a tree is struck by lightning or dies, its root pixels disappear

- [ ] **Step 2: Adjust if needed**

Potential tuning:
- If roots are too rare, increase the 0.02 probability slightly
- If roots grow too deep, reduce the max depth from 12
- If connection detection is slow, limit the `plants` scan to trees within 30px horizontally

- [ ] **Step 3: Final commit**

```bash
git add index.html
git commit -m "feat: tune root growth parameters"
```

---

## Notes

- No test framework exists in this project — verification is visual
- The pixel ownership system already handles cleanup when trees die (`removePlantPixels`)
- Root pixels are drawn with the tree's `plantId`, so they participate in the ownership stack naturally
- Performance: the 0.02 probability per tendril per tick keeps computation minimal even with many trees
