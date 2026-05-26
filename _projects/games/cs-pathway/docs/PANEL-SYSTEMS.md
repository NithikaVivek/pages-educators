# CS Pathway Panel Systems Documentation

## Overview

The CS Pathway uses a unified panel system for displaying game state, progress, and interactive information. Panels are rendered as persistent HUD elements that update dynamically as the player progresses through levels.

## Panel Architecture

### Core Components

#### 1. **StatusPanel** (View Layer)
Located at: `@assets/js/GameEnginev1.1/essentials/StatusPanel.js`

The `StatusPanel` is the visual container that renders game state on-screen. It's a reusable component that displays information in a structured grid format.

```javascript
import StatusPanel from '@assets/js/GameEnginev1.1/essentials/StatusPanel.js';

// Create a panel instance
const panel = new StatusPanel({
  id: 'my-panel-id',                    // Unique DOM ID
  title: 'PANEL TITLE',                 // Header text
  fields: [                              // Field definitions
    { key: 'course', label: 'Course', emptyValue: '—' },
    { key: 'score', label: 'Score', emptyValue: '0' },
    { type: 'section', title: 'Progress', marginTop: '10px' }, // Section divider
    { key: 'level1', label: 'Level 1', emptyValue: '✓' },
  ],
  theme: {                               // Styling
    background: 'rgba(13,13,26,0.92)',
    borderColor: '#4ecca3',
    textColor: '#e0e0e0',
    accentColor: '#4ecca3',
  },
  position: { top: '16px', left: '16px' },
  width: '260px',
  padding: '12px 14px',
  zIndex: '10000',
  fontFamily: '"Courier New", monospace',
});

// Render panel to DOM
panel.render();

// Update panel data
panel.update({
  course: 'CSSE',
  score: '42',
  level1: '✓',
});
```

#### 2. **Present** (Presentation Manager)
Located at: `levels/Present.js`

The `Present` class manages all transient and persistent UI elements for a level. It combines:
- **Panels**: Persistent HUD (StatusPanel)
- **Toasts**: Temporary floating messages (auto-dismiss)
- **Alerts**: Zone-level warnings and info
- **Score**: Temporary score popups

```javascript
import Present from './Present.js';

// Initialize in level constructor
this.present = new Present(this, {
  toastDuration: 2200,                  // Toast visibility time (ms)
  ignoreToasts: ['Press E to interact'], // Messages to suppress
  isActiveLevel: () => this.gameEnv?.currentLevel === this,
});

// Show different UI types
this.present.panel('Status update');   // Persistent (bottom right)
this.present.toast('Achievement!');    // Temporary (center)
this.present.alerts('Warning: Lost?');  // Zone alert (top)
this.present.score('+100');             // Score popup (center)

// Clear UI elements
this.present.clearPanel();
this.present.clearToast();
this.present.clearAlerts();
this.present.clearScore();
```

### Panel Formatting Standards

#### Field Types

```javascript
fields: [
  // Text field
  { key: 'username', label: 'Name', emptyValue: '—' },
  
  // Section divider
  { type: 'section', title: 'Progress Tracking', marginTop: '10px' },
  
  // Completion indicator
  { key: 'complete', label: 'Status', emptyValue: '✗' },
]
```

#### Theme Variables (CSS Custom Properties)

The panel system respects these CSS variables for styling consistency:

```css
--ocs-game-panel-bg: rgba(13,13,26,0.92);      /* Panel background */
--ocs-game-accent: #4ecca3;                     /* Highlight color */
--ocs-game-text: #e0e0e0;                       /* Text color */
--ocs-game-surface-alt: #1a1a2e;               /* Button background */
```

## Panel Update Pattern

### Real-Time Data Binding (Example: Wayfinding World)

```javascript
class GameLevelCsPath1Way extends GameLevelCsPathIdentity {
  constructor(gameEnv) {
    super(gameEnv);
    
    // 1. Define panel structure
    this.profilePanelView = new StatusPanel({
      id: 'csse-wayfinding-panel',
      title: 'WAYFINDING WORLD',
      fields: [
        { key: 'course', label: 'Course', emptyValue: '—' },
        { key: 'persona', label: 'Persona', emptyValue: '—' },
        { key: 'skill', label: 'Skill', emptyValue: '—' },
        { type: 'section', title: 'Completion Status' },
        { key: 'completionIdentityForge', label: 'Identity Forge', emptyValue: '✓' },
        { key: 'completionWayfindingWorld', label: 'Wayfinding World', emptyValue: '—' },
      ],
    });
    
    // 2. Render panel to DOM
    this.profilePanelView.render();
    
    // 3. Initial state
    this.profilePanelView.update({
      course: '—',
      persona: '—',
      ...this._getCompletionPanelValues(),
    });
  }
  
  // 4. Update panel on state changes
  async onPlayerInteraction(interactionType) {
    const updatedData = await this._fetchCurrentProgress();
    
    this.profilePanelView.update({
      persona: updatedData.persona,
      skill: updatedData.skill,
      ...this._getCompletionPanelValues(),
    });
  }
}
```

### State-Driven Updates

All panel updates follow this pattern:

1. **Fetch data** from ProfileManager or game state
2. **Format data** for display (✓ for complete, — for incomplete)
3. **Call `panel.update()`** with the new data object
4. **DOM updates automatically** (StatusPanel handles rendering)

```javascript
// Get completion values from shared storage
_getCompletionPanelValues() {
  const completion = this._getCompletion();
  const score = this._getOverallScore();
  
  return {
    completionIdentityForge: completion.identityForge ? '✓' : '—',
    completionWayfindingWorld: completion.wayfindingWorld ? '✓' : '—',
    completionMissionTools: completion.missionTools ? '✓' : '—',
    completionOverallScore: score.toFixed(2),
  };
}

// Sync panel when state changes
_syncCompletionPanel() {
  this.profilePanelView.update(this._getCompletionPanelValues());
}
```

## How Panels Update During Gameplay

### Event-Driven Updates

Panels update reactively to game events:

```javascript
// When player completes an objective
completeObjective() {
  // Update profile storage
  this.profileManager.markObjectiveComplete('objective-1');
  
  // Sync panel immediately
  this._syncCompletionPanel();
  
  // Show toast
  this.showToast('✓ Objective completed!');
}
```

### Lifecycle Hooks

```javascript
// Called when level initializes
async initialize() {
  await this.profileManager.initialize();
  const profile = this.profileManager.getRestoredState();
  
  // Set initial panel data from profile
  this.profilePanelView.update({
    persona: profile.persona || '—',
    course: profile.course || '—',
  });
}

// Called during level update (frame-by-frame)
update() {
  // Fast polling-based updates (avoid if possible)
  // Better: Update only on state changes
}

// Called when level unloads
destroy() {
  this.profilePanelView = null;
  this.present.destroy();
}
```

## Styling Panels

### Built-in Theme System

```javascript
const panel = new StatusPanel({
  // ... other options
  theme: {
    background: 'var(--ocs-game-panel-bg, rgba(13,13,26,0.92))',
    borderColor: 'var(--ocs-game-accent, #4ecca3)',
    textColor: 'var(--ocs-game-text, #e0e0e0)',
    accentColor: 'var(--ocs-game-accent, #4ecca3)',
    secondaryButtonBackground: 'var(--ocs-game-surface-alt, #1a1a2e)',
    secondaryButtonTextColor: 'var(--ocs-game-text, #e0e0e0)',
  },
});
```

### Position Variants

```javascript
// Top-left (default)
position: { top: '16px', left: '16px' }

// Top-right
position: { top: '16px', right: '16px' }

// Bottom-left
position: { bottom: '16px', left: '16px' }

// Bottom-right
position: { bottom: '16px', right: '16px' }

// Centered
position: { top: '50%', left: '50%', transform: 'translate(-50%, -50%)' }
```

## Accessibility & Display

### Toast Messages (Temporary)

Appear in center, auto-dismiss after timeout:
- Ideal for achievements, quick feedback
- Ignored when level is inactive (prevent spam)

### Alerts (Zone-Level)

Appear at top, persist until cleared:
- Use for navigation hints, warnings
- Clear with `present.clearAlerts()`

### Panels (Persistent)

Appear in corner, always visible:
- Use for status/progress tracking
- Update on any state change
- Positioned to avoid blocking gameplay

### Scores (Temporary)

Appear in center, fade quickly:
- Use for combat/collection feedback
- Cumulative score display

## Best Practices

1. **Single Source of Truth**: All panel data comes from ProfileManager or game state (never duplicate)
2. **Debounce Updates**: Don't call `panel.update()` every frame—only on state changes
3. **Meaningful Empty Values**: Use '—' or '✓' consistently across all panels
4. **Theme Inheritance**: Use CSS custom properties, not hardcoded colors
5. **Responsive Sizing**: Use relative positions, not fixed pixel coordinates
6. **Cleanup**: Call `destroy()` or `clearPanel()` when level unloads

## Common Panel Patterns

### Progress Tracking Panel

```javascript
const panel = new StatusPanel({
  id: 'progress-panel',
  title: 'PROGRESS',
  fields: [
    { key: 'currentLevel', label: 'Current', emptyValue: '?' },
    { key: 'completionPercent', label: 'Completion', emptyValue: '0%' },
    { type: 'section', title: 'Milestones' },
    { key: 'milestone1', label: 'Intro', emptyValue: '—' },
    { key: 'milestone2', label: 'Mid-game', emptyValue: '—' },
    { key: 'milestone3', label: 'End-game', emptyValue: '—' },
  ],
});
```

### Score/Stats Panel

```javascript
const panel = new StatusPanel({
  id: 'stats-panel',
  title: 'STATS',
  fields: [
    { key: 'health', label: 'Health', emptyValue: '100' },
    { key: 'mana', label: 'Mana', emptyValue: '100' },
    { key: 'score', label: 'Score', emptyValue: '0' },
    { key: 'combo', label: 'Combo', emptyValue: '0x' },
  ],
});
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Panel not visible | Check `zIndex` is high enough; ensure `render()` called |
| Data not updating | Verify `panel.update()` is called; check key names match field definitions |
| Styling looks wrong | Verify CSS custom properties are defined in parent document |
| Toast doesn't dismiss | Check `toastDuration` is set; verify `isActiveLevel()` returns true |
| Memory leak | Call `destroy()` in level's cleanup; remove event listeners |

