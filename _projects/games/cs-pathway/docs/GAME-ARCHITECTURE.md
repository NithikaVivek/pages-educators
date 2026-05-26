# CS Pathway Game Architecture & Development Guide

## Overview

The CS Pathway is a multi-level game that guides students through Computer Science onboarding via gamified experiences. Each level (`GameLevelCsPath*`) is independent but shares common infrastructure for progression tracking, profile persistence, and UI management.

## Game Structure

### Level Progression

```
GameLevelCsPath0Forge       → Level 0: Identity Forge
    ↓ (player completes)
GameLevelCsPath1Way         → Level 1: Wayfinding World
    ↓
GameLevelCsPath1CodeHub     → Level 1: Code Hub (side quest)
    ↓
GameLevelCsPath2Mission     → Level 2: Mission Tooling
    ↓
GameLevelCsPath3Analytics   → Level 3: Analytics
```

Each level is self-contained but can transition to others via:
- **Gatekeepers**: NPCs that unlock the next level
- **Completion Checks**: Verifying objectives complete before advancing
- **Menu System**: Dropdown to select level from profile page

### File Organization

```
_projects/games/cs-pathway/
├── levels/                       # Game level implementations
│   ├── GameLevelCsPath0Forge.js
│   ├── GameLevelCsPath1Way.js
│   ├── GameLevelCsPath1CodeHub.js
│   ├── GameLevelCsPath2Mission.js
│   ├── GameLevelCsPath3Analytics.js
│   ├── GameLevelCsPathIdentity.js      # Base class (shared code)
│   ├── Present.js                      # UI presentation manager
│   ├── CourseEnlistmentTrial.js        # Course selection modal
│   ├── PersonaHallTrial.js             # Persona selection
│   ├── AboutMeBuilder.js               # Blogging interface
│   └── SkillPassport.js                # Skill tracking
├── model/                        # Data persistence (MVC model)
│   ├── ProfileManager.js                # Main profile API
│   ├── localProfile.js                  # localStorage with user namespacing
│   ├── persistentProfile.js             # Backend API integration
│   └── LoginManager.js
└── docs/                         # Documentation
    ├── CS-PATHWAY.md                   # High-level design
    ├── CS-PATHWAY-SCENARIOS.md         # Storage architecture
    ├── PANEL-SYSTEMS.md                # Panel/UI guide
    └── GAME-ARCHITECTURE.md             # This file
```

## Level Switching Implementation

### Option 1: Gatekeeper NPCs (Default)

Each level contains "gatekeepers" that let players transition to the next level:

```javascript
// In GameLevelCsPath1Way.js
const missionToolsGatekeeperPos = {
  x: width * 0.53,
  y: height * 0.21,
};

const createGatekeeperData = ({
  id: 'mission-tools-portal',
  onInteract: async (player) => {
    // Check if player completed prerequisites
    const completion = this._getCompletion();
    
    if (!completion.wayfindingWorld) {
      this.showToast('Complete Wayfinding World first');
      return;
    }
    
    // Load next level
    const nextLevel = new GameLevelCsPath2Mission(this.gameEnv);
    this.gameEnv.setLevel(nextLevel);
  },
});
```

### Option 2: Profile Page Dropdown Menu

The profile page has a dropdown to select any unlocked level:

```javascript
// Pseudo-code: profile page dropdown
<select id="level-selector" onchange="switchLevel(this.value)">
  <option value="">Select Level</option>
  <option value="identity-forge">Identity Forge</option>
  <option value="wayfinding-world" selected>Wayfinding World</option>
  <option value="mission-tools">Mission Tools</option>
</select>

function switchLevel(levelId) {
  const levels = {
    'identity-forge': GameLevelCsPath0Forge,
    'wayfinding-world': GameLevelCsPath1Way,
    'mission-tools': GameLevelCsPath2Mission,
  };
  
  const Level = levels[levelId];
  if (Level) {
    const newLevel = new Level(gameEnv);
    gameEnv.setLevel(newLevel);
  }
}
```

### Option 3: Menu Updates (Recommended Integration)

**This is what Xavier should connect:**

The profile page menu should:
1. Track current course selection (CSSE, CSP, CSA, CSH)
2. Update the level dropdown based on selected course
3. Sync selected course to game profile
4. Reflect level completion status in menu UI

```javascript
// Profile page menu update logic
class ProfileMenu {
  async onCourseSelected(courseId) {
    // 1. Update profile with course
    this.profile.course = courseId;
    await this.profileManager.saveProfile();
    
    // 2. Update available levels based on course
    this.updateLevelDropdown(courseId);
    
    // 3. Notify game (if open) to reflect course change
    if (window.gameEnv?.currentLevel) {
      window.gameEnv.currentLevel.onCourseChange(courseId);
    }
  }
  
  updateLevelDropdown(courseId) {
    // Show course-specific levels
    const courseLevels = {
      'CSSE': ['Identity Forge', 'Wayfinding World', 'Mission Tools'],
      'CSP': ['Identity Forge', 'Creative Pathways'],
      'CSA': ['Identity Forge', 'Advanced Journey'],
    };
    
    // Populate dropdown with course levels
    this.levelDropdown.innerHTML = courseLevels[courseId]
      .map(level => `<option>${level}</option>`)
      .join('');
  }
}
```

## Updating a Level

### Minimal Change Pattern (SRP)

Keep each level focused on ONE responsibility:

```javascript
// ✓ GOOD: Single responsibility
class GameLevelCsPath1Way {
  // Only handles Wayfinding World gameplay
  // No profile persistence logic mixed in
  
  constructor(gameEnv) {
    this.gameEnv = gameEnv;
    this.profileManager = new ProfileManager(); // Delegate to model
  }
  
  async onPlayerCompleteObjective() {
    // 1. Update model
    await this.profileManager.markObjectiveComplete('objective-1');
    
    // 2. Update view (panel)
    this._syncCompletionPanel();
    
    // 3. Show feedback
    this.showToast('✓ Objective complete!');
  }
}

// ✗ BAD: Multiple responsibilities
class GameLevelMixedUp {
  // Don't mix UI, persistence, and game logic
  async onPlayerCompleteObjective() {
    // Loading/saving directly in level ✗
    const data = localStorage.getItem('profile');
    const profile = JSON.parse(data);
    profile.objectives.push('objective-1');
    localStorage.setItem('profile', JSON.stringify(profile));
    
    // Posting to backend directly ✗
    await fetch('/api/profile', { method: 'POST', body: JSON.stringify(profile) });
  }
}
```

### Adding a New Objective

```javascript
// 1. Define objective in level
const objectiveData = {
  id: 'collect-10-coins',
  title: 'Collect 10 Coins',
  condition: () => this.player.coins >= 10,
  onComplete: () => {
    this.showToast('✓ Coins collected!');
    this.score('+50');
  }
};

// 2. Check condition in update loop
update() {
  if (objectiveData.condition()) {
    this.markObjectiveComplete(objectiveData.id);
  }
}

// 3. Persist to profile
async markObjectiveComplete(objectiveId) {
  const profile = await this.profileManager.getProfile();
  profile.completedObjectives = profile.completedObjectives || [];
  profile.completedObjectives.push(objectiveId);
  
  await this.profileManager.saveProfile();
  this._syncCompletionPanel(); // Update UI
}
```

### Adding a New NPC

```javascript
import Npc from '@assets/js/GameEnginev1.1/essentials/Npc.js';

const tutorNpcData = {
  id: 'mentor-sage',
  src: 'path/to/mentor-sprite.png',
  INIT_POSITION: { x: 400, y: 300 },
  SCALE_FACTOR: 4,
  // ... animation config
};

this.mentorNpc = new Npc(tutorNpcData);

// Register dialogue
this.mentorNpc.dialogue = [
  { text: 'Welcome to the level!', duration: 2000 },
  { text: 'Collect those coins to proceed.', duration: 2000 },
];

// Handle interaction
this.mentorNpc.onInteract = () => {
  this.dialogue = this.mentorNpc.dialogue;
  this.showDialogue();
};
```

## Key Tips on Single Responsibility Principle (SRP)

The CS Pathway strictly separates concerns:

### MVC Architecture

```
MODEL (model/)
├── ProfileManager.js      # Data persistence API
├── localProfile.js        # localStorage layer
└── persistentProfile.js   # Backend API layer

VIEW (levels/)
├── Present.js             # UI rendering (panels, toasts)
├── StatusPanel (imported) # Visual component
└── Dialogue system        # Conversation UI

CONTROLLER (levels/)
├── GameLevelCsPath*.js    # Game logic & state management
├── Npc/Player             # Game entities
└── Event handlers         # User input → model/view updates
```

### Why SRP Matters

1. **Testability**: Each class has one reason to change
2. **Reusability**: ProfileManager works in any level without modification
3. **Maintainability**: Bug fixes don't cascade across unrelated code
4. **Scalability**: New levels reuse existing components

### Example: Violating SRP

```javascript
// ✗ BAD: GameLevel doing everything
class GameLevelMixed extends Npc {
  update() {
    // Game logic mixed with...
    // ...UI rendering...
    document.getElementById('panel').innerHTML = /* complex template */;
    
    // ...persistence...
    localStorage.setItem('data', JSON.stringify(this.state));
    
    // ...and network requests
    fetch('/api/save').then(...);
  }
}

// ✓ GOOD: Separated concerns
class GameLevelCsPath1Way {
  update() {
    // Only game logic
    this.updatePlayerPosition();
    this.checkObjectiveCompletion();
  }
  
  async onObjectiveComplete() {
    // Delegate to model
    await this.profileManager.markObjectiveComplete(id);
    
    // Delegate to view
    this._syncCompletionPanel();
  }
}
```

## Easy Coding Patterns for CS Pathway

### 1. Data Flow Pattern

```javascript
// Unidirectional: Game Logic → Model → View

// Step 1: Game logic detects event
if (player.position.x === npc.position.x) {
  this.onPlayerInteractWithNpc(npc);
}

// Step 2: Update model
async onPlayerInteractWithNpc(npc) {
  await this.profileManager.recordInteraction(npc.id);
}

// Step 3: View automatically updates
// (panel listens to model changes)
```

### 2. State Management Pattern

```javascript
// Keep level state flat and immutable

const levelState = {
  playerPosition: { x: 100, y: 200 },
  collectedItems: ['coin-1', 'coin-2'],
  completedObjectives: ['intro-tutorial'],
  currentDialogue: null,
  isPaused: false,
};

// Update state
levelState.collectedItems.push('coin-3');

// Notify view
this._syncCompletionPanel();
```

### 3. Event Handler Pattern

```javascript
// Use clear event names and callbacks

const eventHandlers = {
  'player-move': (x, y) => {
    this.player.setPosition(x, y);
    this.checkCollisions();
  },
  
  'npc-interact': (npcId) => {
    const npc = this.npcs[npcId];
    npc.showDialogue();
    this.profileManager.recordInteraction(npcId);
  },
  
  'objective-complete': (objectiveId) => {
    this.markObjectiveComplete(objectiveId);
    this.showToast(`✓ ${objectiveId} complete!`);
  },
};

// Trigger events
eventHandlers['player-move'](playerX, playerY);
```

### 4. Conditional Rendering Pattern

```javascript
// Show UI only when relevant

const shouldShowTutorialPanel = 
  !this.profile.completedObjectives.includes('tutorial') &&
  this.levelState.playerPosition.x < 50;

if (shouldShowTutorialPanel) {
  this.showToast('← Explore this direction');
}
```

### 5. Async Loading Pattern

```javascript
// Queue async work, show loading state, complete gracefully

async initialize() {
  this.beginLoadingScreen();
  
  try {
    // Queue multiple async operations
    this.queueLoadingWork('profile');
    const profile = await this.profileManager.initialize();
    this.finishLoadingWork('profile');
    
    this.queueLoadingWork('assets');
    await this.preloadAssets();
    this.finishLoadingWork('assets');
    
    this.finishLoadingScreen();
    this.startGameplay();
  } catch (error) {
    console.error('Init failed:', error);
    this.showToast('Loading failed. Try refreshing.');
  }
}
```

## Common Patterns to Avoid

```javascript
// ✗ Don't: Mix concerns
async update() {
  // Game logic
  this.player.update();
  
  // Model persistence
  localStorage.setItem('state', JSON.stringify(this.gameState));
  
  // View rendering
  document.getElementById('panel').innerHTML = this.renderPanel();
}

// ✓ Do: Separate concerns
update() {
  this.player.update();          // Game logic only
  this.checkCompletionConditions();
}

async saveState() {
  await this.profileManager.saveProfile(this.gameState); // Model layer
}

updateUI() {
  this._syncCompletionPanel();   // View layer
}
```

## Testing Tips

Keep tests focused on single responsibility:

```javascript
// Test game logic independently
describe('GameLevelCsPath1Way', () => {
  it('should mark objective complete when condition met', async () => {
    const level = new GameLevelCsPath1Way(mockGameEnv);
    
    // Don't mock ProfileManager - that's tested separately
    // Just test the condition and behavior
    await level.markObjectiveComplete('test-objective');
    
    expect(level.levelState.completedObjectives)
      .toContain('test-objective');
  });
});

// Test model independently
describe('ProfileManager', () => {
  it('should save profile to localStorage', async () => {
    const manager = new ProfileManager();
    await manager.saveProfile({ test: 'data' });
    
    const stored = localStorage.getItem('ocs_profile_guest');
    expect(JSON.parse(stored)).toEqual({ test: 'data' });
  });
});
```

## Debugging Tips

1. **Check Console Logs**: Each level logs initialization steps
   ```
   ProfileManager: user context set to guest
   ProfileManager: loaded guest profile from localStorage
   Wayfinding World: level initialized
   ```

2. **Inspect Panel Data**: Open browser DevTools → find StatusPanel DOM
   ```javascript
   // In console:
   document.getElementById('csse-wayfinding-panel').textContent
   ```

3. **Check localStorage**: 
   ```javascript
   // In console:
   localStorage.getItem('ocs_profile_guest')
   ```

4. **Trace Profile Sync**: 
   ```javascript
   // ProfileManager logs all sync attempts:
   // "ProfileManager: background sync failed (non-blocking)"
   ```

## Performance Considerations

1. **Debounce Panel Updates**: Don't call `panel.update()` every frame
   ```javascript
   let lastUpdateTime = 0;
   update() {
     if (Date.now() - lastUpdateTime > 1000) {
       this._syncCompletionPanel();
       lastUpdateTime = Date.now();
     }
   }
   ```

2. **Lazy Load Levels**: Only initialize next level when needed
3. **Memory Cleanup**: Call `level.destroy()` when switching levels
4. **Async Profile Sync**: Backend sync happens non-blocking

