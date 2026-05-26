# CS Pathway Integration Guide

## Overview

This guide explains how to integrate the CS Pathway with larger game ecosystems and how to use the **game-in-game** technique for embedding smaller challenges within levels. It also covers how to connect class selection from the profile page with menu updates in the game.

## Integration Architecture

### The Problem

The CS Pathway is a complete learning experience, but it exists within a larger ecosystem:
- **Profile Page**: Student account, course selection, achievements
- **Main Game World**: Larger platformers or adventure games
- **Class Management System**: Teacher dashboards, roster management

These systems need to:
1. Share student profile data
2. Sync course selection changes
3. Update UI across multiple contexts
4. Track progress across game boundaries

### The Solution: Unified Profile Management

```javascript
// Single source of truth: ProfileManager
class ProfileManager {
  // All game components delegate to ProfileManager
  async getProfile() { /* returns profile */ }
  async saveProfile(data) { /* persists changes */ }
  async markObjectiveComplete(id) { /* updates progress */ }
}

// Usage across all systems:
// Profile Page:
const pm = new ProfileManager();
pm.saveCourseSelection('CSSE');

// Game Level:
const pm = new ProfileManager();
const course = (await pm.getProfile()).course;

// Game-in-Game:
const pm = new ProfileManager();
pm.markObjectiveComplete('minigame-1');
```

---

## Game-in-Game Integration (Alex & Travis Technique)

### What is Game-in-Game?

Game-in-Game is a technique where one game is **embedded within another game** without losing context or progress. The outer game pauses, the inner game runs, and upon completion, the player returns to the outer game with updated state.

**Example from Kirby Minigames:**
```
Aquatic Level (outer game)
    ↓ Player enters portal
    ├── Seek Minigame (inner game #1)
    │   ↓ Complete minigame
    ├── Basketball Challenge (inner game #2)
    │   ↓ Complete challenge
    ↓ Return to Aquatic Level (with progress saved)
```

### Implementation Pattern

#### Step 1: Outer Level Defines Entry Point

```javascript
// In GameLevelCsPath1Way (outer level)
class GameLevelCsPath1Way extends GameLevelCsPathIdentity {
  async handlePortalInteraction() {
    // 1. Save outer level state
    const outerState = {
      playerPosition: this.player.position,
      collectedItems: this.inventory,
      objectives: this.levelState.completedObjectives,
    };
    
    LocalProfile.save({
      ...outerState,
      lastLevel: 'wayfinding-world',
    });
    
    // 2. Create and load inner game
    const minigame = new CodeHubMinigame(this.gameEnv);
    this.gameEnv.setLevel(minigame);
  }
}
```

#### Step 2: Inner Game Runs Independently

```javascript
// Minigame has its own logic, assets, mechanics
class CodeHubMinigame {
  constructor(gameEnv) {
    this.gameEnv = gameEnv;
    this.score = 0;
    this.isComplete = false;
  }
  
  async onGameComplete() {
    // Save minigame results
    this.profileManager.recordMinigameCompletion({
      gameId: 'codehub-minigame',
      score: this.score,
      completedAt: Date.now(),
    });
    
    // Signal completion
    this.isComplete = true;
    this.returnToOuterLevel();
  }
  
  returnToOuterLevel() {
    // Don't create new instance — restore from saved state
    this.gameEnv.setLevel('restore:wayfinding-world');
  }
}
```

#### Step 3: Restore Outer Level with Updated State

```javascript
// In GameEngine
class GameEngine {
  async setLevel(levelSpec) {
    // Handle special "restore:" prefix
    if (typeof levelSpec === 'string' && levelSpec.startsWith('restore:')) {
      const levelId = levelSpec.split(':')[1];
      const savedState = LocalProfile.load();
      
      // Restore previous level
      const Level = LEVEL_REGISTRY[levelId];
      const restoredLevel = new Level(this);
      
      // Apply saved state (position, inventory, etc.)
      restoredLevel.restoreState(savedState);
      this.currentLevel = restoredLevel;
      return;
    }
    
    // Normal level creation
    const level = levelSpec instanceof GameLevel
      ? levelSpec
      : new levelSpec(this);
    
    this.currentLevel = level;
  }
}
```

#### Step 4: Restore Method in Outer Level

```javascript
// In GameLevelCsPath1Way
class GameLevelCsPath1Way {
  async restoreState(savedState) {
    // Restore player position
    this.player.setPosition(
      savedState.playerPosition.x,
      savedState.playerPosition.y
    );
    
    // Restore inventory
    this.inventory = savedState.collectedItems;
    
    // Restore objective completion
    this.levelState.completedObjectives = savedState.objectives;
    
    // Sync panel with restored state
    this._syncCompletionPanel();
    
    // Show toast about minigame completion
    const minigameResult = savedState.lastMinigameResult;
    if (minigameResult) {
      this.showToast(`✓ Minigame complete! Score: ${minigameResult.score}`);
    }
  }
}
```

### Game-in-Game Data Flow

```
┌─────────────────────────────────────────────────┐
│ Outer Level (Wayfinding World)                  │
│                                                   │
│  Player Position: (400, 300)                    │
│  Objectives: ['intro', 'collect-items']        │
│  Inventory: ['coin-1', 'coin-2']               │
└────────────┬────────────────────────────────────┘
             │ Save state to localStorage
             ↓
┌─────────────────────────────────────────────────┐
│ LocalProfile.save(outerState)                    │
│                                                   │
│ localStorage['ocs_profile_guest'] = {            │
│   playerPosition: { x: 400, y: 300 },          │
│   objectives: [...],                            │
│   inventory: [...],                             │
│   lastLevel: 'wayfinding-world'                │
│ }                                               │
└────────────┬────────────────────────────────────┘
             │ Switch to minigame
             ↓
┌─────────────────────────────────────────────────┐
│ Minigame (Code Hub)                              │
│                                                   │
│  Independent score tracking                     │
│  Separate gameplay mechanics                    │
│  New UI/HUD                                     │
└────────────┬────────────────────────────────────┘
             │ Minigame complete
             │ Save results
             ↓
┌─────────────────────────────────────────────────┐
│ ProfileManager.recordMinigameCompletion()        │
│                                                   │
│ Updates profile:                                │
│ profile.minigames.codehub = {                  │
│   completed: true,                              │
│   score: 850                                    │
│ }                                               │
└────────────┬────────────────────────────────────┘
             │ Return to outer level
             │ (restore from localStorage)
             ↓
┌─────────────────────────────────────────────────┐
│ Outer Level Restored                            │
│                                                   │
│  Player Position: (400, 300) ← restored        │
│  Objectives: ['intro', 'collect-items'] ← restored
│  Inventory: [...] ← restored                    │
│  Achievement: "Code Hub Completed"              │
└─────────────────────────────────────────────────┘
```

### Benefits of This Approach

1. **Clean State Management**: Each game is isolated; outer level isn't modified
2. **Reusable Minigames**: Same minigame can be embedded in multiple outer levels
3. **Progress Persistence**: Minigame results automatically sync to profile
4. **Smooth UX**: No full page reload; just level transition
5. **Easy Testing**: Minigames can be tested independently

---

## Class Selection Integration (Xavier's Task)

### Current System

The profile page has:
1. Course selection dropdown (CSSE, CSP, CSA, CSH)
2. Student roster view
3. Progress dashboard

The game currently:
1. Stores selected course in profile
2. Shows course info in panel
3. Has level-specific UI

**What's Missing:** The menu dropdown and course selection aren't wired to update the game in real-time.

### Integration Goal

When a student:
1. Selects a different course on the profile page
2. The game level menu should update to show course-specific levels
3. The panel should show the new course
4. Any open minigames should reflect the course change

### Implementation Steps

#### Step 1: Profile Page Course Selection Handler

```javascript
// On pages/educators/index.html or profile view
class StudentProfile {
  async onCourseSelection(courseId) {
    // 1. Update profile
    const profile = await this.profileManager.getProfile();
    profile.course = courseId;
    await this.profileManager.saveProfile(profile);
    
    // 2. Update UI dropdown
    this.renderLevelDropdown(courseId);
    
    // 3. Notify game if open
    this.notifyGameOfCourseChange(courseId);
  }
  
  notifyGameOfCourseChange(courseId) {
    // Dispatch event to game context
    if (window.gameEnv?.currentLevel) {
      window.gameEnv.currentLevel.onCourseChange?.(courseId);
    }
    
    // Or use custom event
    const event = new CustomEvent('course-selected', {
      detail: { courseId }
    });
    window.dispatchEvent(event);
  }
  
  renderLevelDropdown(courseId) {
    const levels = {
      'CSSE': [
        { id: 'identity-forge', name: 'Identity Forge' },
        { id: 'wayfinding-world', name: 'Wayfinding World' },
        { id: 'mission-tools', name: 'Mission Tooling' },
      ],
      'CSP': [
        { id: 'identity-forge', name: 'Identity Forge' },
        { id: 'creative-path', name: 'Creative Pathways' },
      ],
      'CSA': [
        { id: 'identity-forge', name: 'Identity Forge' },
        { id: 'advanced-journey', name: 'Advanced Journey' },
      ],
    };
    
    const courseLevels = levels[courseId] || [];
    const dropdown = document.getElementById('level-selector');
    
    dropdown.innerHTML = courseLevels
      .map(level => `
        <option value="${level.id}" 
                data-completion="${this.getLevelCompletion(level.id)}">
          ${level.name}
        </option>
      `)
      .join('');
  }
}
```

#### Step 2: Game Level Listens for Course Changes

```javascript
// In GameLevelCsPath1Way (or base class)
class GameLevelCsPath1Way {
  constructor(gameEnv) {
    super(gameEnv);
    
    // Listen for course selection changes
    window.addEventListener('course-selected', 
      this.onCourseChangeFromProfile.bind(this)
    );
  }
  
  async onCourseChangeFromProfile(event) {
    const { courseId } = event.detail;
    
    // 1. Update profile
    const profile = await this.profileManager.getProfile();
    profile.course = courseId;
    await this.profileManager.saveProfile(profile);
    
    // 2. Update panel immediately
    this.profilePanelView.update({
      course: courseId,
    });
    
    // 3. Show feedback
    this.showToast(`Course changed to: ${courseId}`);
    
    // 4. Update available minigames/quests
    this.updateAvailableContent(courseId);
  }
  
  updateAvailableContent(courseId) {
    // Show/hide quests based on course
    const courseQuests = {
      'CSSE': ['quest-intro', 'quest-coding'],
      'CSP': ['quest-intro', 'quest-creative'],
      'CSA': ['quest-intro', 'quest-advanced'],
    };
    
    const quests = courseQuests[courseId] || [];
    
    // Update available npcs/quests
    this.npcs.forEach(npc => {
      npc.isAvailable = quests.includes(npc.questId);
      npc.updateVisibility();
    });
  }
  
  destroy() {
    window.removeEventListener('course-selected', 
      this.onCourseChangeFromProfile.bind(this)
    );
  }
}
```

#### Step 3: Profile Page Level Dropdown Syncs with Game

```javascript
// In profile page dropdown change handler
async function onLevelSelected(levelId) {
  // 1. Check if user can access level
  const profile = await profileManager.getProfile();
  const completion = profile.completedObjectives || [];
  
  const levelRequirements = {
    'identity-forge': { requires: [] },
    'wayfinding-world': { requires: ['identity-forge'] },
    'mission-tools': { requires: ['wayfinding-world'] },
  };
  
  const requirements = levelRequirements[levelId];
  const canAccess = requirements.requires.every(req => 
    completion.includes(req)
  );
  
  if (!canAccess) {
    alert('You must complete the previous level first!');
    return;
  }
  
  // 2. Switch level in game
  const Level = LEVEL_REGISTRY[levelId];
  const newLevel = new Level(gameEnv);
  gameEnv.setLevel(newLevel);
  
  // 3. Update dropdown selection
  document.getElementById('level-selector').value = levelId;
}
```

#### Step 4: Game Updates Profile Page Menu

When a level is completed in the game, the profile page menu should update:

```javascript
// In GameLevelCsPath (base class)
class GameLevelCsPathBase {
  async markLevelComplete() {
    // 1. Save to profile
    const profile = await this.profileManager.getProfile();
    profile.completedObjectives = profile.completedObjectives || [];
    profile.completedObjectives.push(`level:${this.levelId}`);
    await this.profileManager.saveProfile(profile);
    
    // 2. Dispatch event to profile page
    window.dispatchEvent(new CustomEvent('level-completed', {
      detail: { levelId: this.levelId }
    }));
  }
}

// In profile page, listen for completion
window.addEventListener('level-completed', (event) => {
  const { levelId } = event.detail;
  
  // Update UI to show checkmark
  const levelOption = document.querySelector(
    `option[value="${levelId}"]`
  );
  if (levelOption) {
    levelOption.dataset.completed = 'true';
    levelOption.text += ' ✓';
  }
  
  // Re-render level dropdown with new options
  updateLevelDropdown(profile.course);
});
```

### Full Event Flow

```
Student clicks course selector
    ↓
Profile: "Course selected"
    ├── Update localStorage
    ├── Dispatch 'course-selected' event
    ├── Re-render level dropdown
    │
    └→ Game receives event
        ├── Update panel (show new course)
        ├── Update available quests
        └── Sync profile

Student selects level from dropdown
    ↓
Profile: "Level selected"
    ├── Check completion requirements
    ├── Dispatch 'level-switch' event
    │
    └→ Game receives event
        ├── Save outer level state
        └── Load new level

Student completes level
    ↓
Game: "Level completed"
    ├── Update profile
    ├── Dispatch 'level-completed' event
    │
    └→ Profile receives event
        ├── Mark level as complete (✓)
        └── Unlock next level
```

### Implementation Checklist

- [ ] Add `onCourseChange()` handler to GameLevelCsPath base class
- [ ] Add course-selected event listener in level constructor
- [ ] Add `updateLevelDropdown()` function to profile page
- [ ] Add completion requirement checking to profile page
- [ ] Add 'level-completed' event dispatch in game
- [ ] Add listener to profile page for level-completed events
- [ ] Test course change while game is open
- [ ] Test level selection from profile dropdown
- [ ] Test level completion updates profile menu
- [ ] Test minigame completion syncs back to profile

---

## Best Practices for Integration

1. **Always Use ProfileManager**: Never access localStorage directly from integrations
   ```javascript
   // ✗ DON'T
   const profile = JSON.parse(localStorage.getItem('ocs_profile_guest'));
   
   // ✓ DO
   const pm = new ProfileManager();
   const profile = await pm.getProfile();
   ```

2. **Use Events for Communication**: Games shouldn't directly reference profile page code
   ```javascript
   // ✗ DON'T
   window.studentProfile.updateUI(); // Tight coupling
   
   // ✓ DO
   window.dispatchEvent(new CustomEvent('level-completed'));
   // Profile page listens independently
   ```

3. **Namespace Data**: Prevent collisions when multiple games are open
   ```javascript
   // Store minigame results separately
   profile.minigames = {
     'codehub': { completed: true, score: 850 },
     'seek': { completed: true, score: 920 },
   };
   ```

4. **Validate State**: Always validate data when switching levels
   ```javascript
   // Before restoring, check integrity
   if (savedState.playerPosition && savedState.inventory) {
     this.restoreState(savedState);
   } else {
     this.showToast('State corrupted; starting fresh');
     this.resetState();
   }
   ```

5. **Clean Up Listeners**: Remove event listeners when levels are destroyed
   ```javascript
   destroy() {
     window.removeEventListener('course-selected', this.handler);
     this.profilePanelView = null;
   }
   ```

