# CS Pathway Documentation Index

Complete guides for developing, maintaining, and integrating the CS Pathway game.

## Core Documentation

### [PANEL-SYSTEMS.md](PANEL-SYSTEMS.md)
**How the panel works: updates, formatting, and customization**

- StatusPanel component overview
- Present manager (UI presentation)
- Real-time data binding patterns
- Panel formatting standards (fields, themes, positions)
- Event-driven updates
- Styling and accessibility
- Best practices and troubleshooting

**When to use:** You're building a new level and need to display game state in panels.

---

### [GAME-ARCHITECTURE.md](GAME-ARCHITECTURE.md)
**How the game overall works: structure, level switching, and coding patterns**

- Level progression tree (how levels connect)
- File organization
- Level switching mechanisms (gatekeepers, dropdowns, menus)
- How to update a level without breaking things
- Single Responsibility Principle (SRP) explained
- Easy coding patterns (data flow, state management, events)
- Common patterns to avoid
- Testing and debugging tips
- Performance considerations

**When to use:** You're adding features to a level, creating a new level, or confused about architecture.

---

### [INTEGRATION-GUIDE.md](INTEGRATION-GUIDE.md)
**How to integrate: game-in-game technique and class selection sync**

- Integration architecture overview (unified profile management)
- Game-in-Game pattern from Alex & Travis
  - How outer level pauses, inner game runs, state restored
  - Step-by-step implementation
  - Data flow diagrams
  - Benefits and use cases
- Class Selection Integration (Xavier's Task)
  - Connect profile page dropdown to game menu
  - Course change notifications
  - Level completion updates
  - Full event flow diagrams
- Implementation checklist
- Best practices (ProfileManager, events, namespacing, validation)

**When to use:** You're embedding a minigame, connecting the profile page menu, or syncing across systems.

---

## Quick Start by Task

| Task | Read |
|------|------|
| Create a new level | GAME-ARCHITECTURE → Updating a Level section |
| Add a status panel | PANEL-SYSTEMS → Core Components section |
| Embed a minigame | INTEGRATION-GUIDE → Game-in-Game pattern |
| Connect course selector | INTEGRATION-GUIDE → Class Selection Integration |
| Debug panel not updating | PANEL-SYSTEMS → Troubleshooting section |
| Understand folder structure | GAME-ARCHITECTURE → File Organization |
| Add an NPC with dialogue | GAME-ARCHITECTURE → Adding a New NPC |
| Test a level | GAME-ARCHITECTURE → Testing Tips |

---

## Related Documentation

- **[CS-PATHWAY.md](CS-PATHWAY.md)**: High-level design philosophy and pedagogical principles
- **[CS-PATHWAY-SCENARIOS.md](CS-PATHWAY-SCENARIOS.md)**: Storage architecture and data migration
- **[AGENTS.md](../../AGENTS.md)**: Development guidelines for all projects

---

## Key Concepts

### Single Responsibility Principle (SRP)
Each class/file has one job. Games shouldn't handle persistence; persistence shouldn't handle UI rendering.

```
MODEL (ProfileManager, localProfile, persistentProfile)
  ↓
CONTROLLER (GameLevelCsPath*.js)
  ↓
VIEW (Present, StatusPanel, DialogueSystem)
```

### Data Flow
```
Game Logic → ProfileManager → localStorage/backend → View Updates
```

### Panel Update Pattern
1. Game detects event
2. Update ProfileManager
3. Call panel.update()
4. DOM updates automatically

### Level Switching
- **Gatekeeper NPCs**: In-game portals to next level
- **Profile Menu**: Dropdown selector from profile page
- **Minigame Return**: Save state, load minigame, restore when done

---

## Important Files

| File | Purpose |
|------|---------|
| `ProfileManager.js` | Core API for all profile operations |
| `Present.js` | UI management (panels, toasts, alerts, scores) |
| `GameLevelCsPath*.js` | Individual level implementations |
| `CourseEnlistmentTrial.js` | Course selection modal |
| `localProfile.js` | localStorage with user namespacing |

---

## Development Workflow

1. **Start here**: GAME-ARCHITECTURE (understand structure)
2. **Update a level**: Read "Updating a Level" section
3. **Add a panel**: Read PANEL-SYSTEMS (formatting, updates)
4. **Test in game**: Use console logs to trace execution
5. **Integrate with profile**: Read INTEGRATION-GUIDE (events, data sync)

---

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| Panel not showing | Check `zIndex`, verify `render()` called, inspect DOM |
| Panel data stale | Use `panel.update()` after profile change |
| Level won't load | Check ProfileManager initialized; verify class imported |
| Course change not syncing | Verify event listener attached; check localStorage |
| Minigame state lost | Use `LocalProfile.save()` before switching levels |

---

## Quick Reference: API

### ProfileManager
```javascript
const pm = new ProfileManager();
await pm.initialize();
const profile = await pm.getProfile();
await pm.saveProfile(data);
await pm.markObjectiveComplete(id);
```

### StatusPanel
```javascript
const panel = new StatusPanel({ id, title, fields, theme, ... });
panel.render();
panel.update({ key1: 'value1', key2: 'value2' });
panel.destroy();
```

### Present
```javascript
this.present.panel('message');    // Persistent
this.present.toast('message');    // Auto-dismiss
this.present.alerts('message');   // Zone alert
this.present.score('message');    // Score popup
this.present.clear*();            // Remove elements
```

### LocalProfile
```javascript
LocalProfile.setUserContext(uid);
LocalProfile.save(data);
const data = LocalProfile.load();
LocalProfile.clear(uid);
```

---

## Contributing

When adding new features:
1. ✅ Maintain SRP — one responsibility per class
2. ✅ Use ProfileManager for all persistence
3. ✅ Update panels with `panel.update()`, not DOM manipulation
4. ✅ Use events for cross-system communication
5. ✅ Add console logs for debugging
6. ✅ Clean up listeners in `destroy()`
7. ✅ Test with guest + authenticated users

---

Last Updated: 2026-05-26
