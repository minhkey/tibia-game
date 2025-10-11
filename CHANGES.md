# Tibia 7.7 Server Changes

## Party Experience Sharing Implementation

### Overview
Modified the experience distribution system to support true party experience sharing, where all nearby party members receive equal experience regardless of individual damage contribution.

### Files Modified
- `src/crcombat.cc`: Modified `DistributeExperiencePoints()` function and added `FindNearbyPartyMembers()` helper function
- `src/crmain.cc`: Enhanced death logging to include killer information

### Changes Made

#### 1. Added Helper Function: `FindNearbyPartyMembers`
```cpp
static void FindNearbyPartyMembers(uint32 PartyLeader, int CenterX, int CenterY, int CenterZ,
                                   uint32 *PartyMembers, int *MemberCount, int MaxMembers)
```
- **Purpose**: Find all party members within 8 tiles of the kill location
- **Range check**: Only includes players on the same floor (`CenterZ`)
- **Party validation**: Verifies players share the same party leader

#### 2. Modified `DistributeExperiencePoints` Algorithm

**Before (Original System)**:
- **Damage-proportional**: Each player gets experience based on their damage contribution
- **Party exclusion**: Party members get NO experience from each other's deaths (PvP protection)
- **Solo focus**: No party-wide sharing mechanism

**After (New System)**:
- **Party Detection & Tracking**: Track processed parties to avoid giving experience multiple times to the same party
- **Party Experience Calculation**:
  1. **Total party damage**: Sum damage from all party members who participated
  2. **Party experience pool**: `(Total Exp × Party Damage) / Total Combat Damage`
  3. **Even distribution**: Divide party experience equally among ALL nearby party members
  4. **Participation not required**: Party members get experience even if they dealt 0 damage

#### 3. Preserved Features
- **PvP level restrictions**: Still apply (using highest party member's level)
- **Soul regeneration**: Still triggers for individual members
- **Solo players**: Still use original damage-based system
- **Range limitations**: Only nearby party members benefit (8-tile radius)

### Example Scenario
**Monster worth 1000 EXP killed by party + solo player:**

**Original system**:
- Party member A (500 damage): 0 EXP (party exclusion)
- Party member B (300 damage): 0 EXP (party exclusion)
- Party member C (0 damage, nearby): 0 EXP (no damage, party exclusion)
- Solo player D (200 damage): 1000 EXP (all of it)

**New system**:
- Party gets: `(800 damage / 1000 total) × 1000 = 800 EXP`
- Party member A: `800 ÷ 3 = 266 EXP` (equal share)
- Party member B: `800 ÷ 3 = 266 EXP` (equal share)
- Party member C: `800 ÷ 3 = 266 EXP` (equal share, despite 0 damage)
- Solo player D: `(200 damage / 1000 total) × 1000 = 200 EXP`

### Technical Details

#### Party Processing Logic
1. **Single Processing**: Each party is processed only once, even if multiple members dealt damage
2. **Nearby Requirement**: Only party members within 8 tiles and on the same floor receive experience
3. **Equal Distribution**: Experience is divided equally among all nearby party members
4. **Level Restrictions**: PvP level restrictions apply using the highest-level party member

#### Range and Limitations
- **Search Radius**: 8 tiles (same as other game mechanics)
- **Floor Restriction**: Party members must be on the same floor (`posz`)
- **Maximum Party Size**: Supports up to 50 party members (configurable)
- **Maximum Concurrent Parties**: Supports up to 20 different parties in one combat (configurable)

#### Debug Output
- Added debug messages for party experience distribution
- Shows individual member experience amounts
- Indicates when level restrictions reduce party experience

### Benefits
1. **True Party Play**: Encourages group cooperation without damage competition
2. **Support Role Friendly**: Healers and support players get equal experience
3. **Balanced**: Solo players maintain competitive experience rates
4. **Fair Distribution**: No more "kill stealing" within parties
5. **Proximity Required**: Prevents abuse by requiring physical presence

### Compatibility
- **Backward Compatible**: Solo players experience no change in mechanics
- **Party System**: Uses existing party infrastructure (`InPartyWith`, `GetPartyLeader`)
- **PvP Protection**: Maintains original PvP level restriction mechanics
- **Soul System**: Preserves soul regeneration triggers for all participants

This implementation creates a modern MMO-style party experience system while preserving the original game's balance mechanisms and preventing common abuse scenarios.

## Improved Skill Training Rate

### Overview
Enhanced the skill advancement system with a configurable skill rate multiplier, allowing server administrators to customize skill progression speed to match their gameplay preferences.

### Files Modified
- `src/config.hh`: Added `SkillRateMultiplier` variable declaration
- `src/config.cc`: Added skill rate multiplier configuration parsing and default value
- `src/crskill.cc`: Modified `TSkillProbe::Increase()` function to apply multiplier to experience gains

### Changes Made

#### 1. Configuration System Enhancement
**New Config Variable**: `SkillRateMultiplier`
- **Default Value**: 2 (provides 2x faster skill advancement)
- **Configuration File**: Add `skillratemultiplier = X` to `.tibia` config file
- **Purpose**: Allows server-wide adjustment of skill training speed

#### 2. Skill Experience Calculation
**Before (Original System)**:
- **Base Rate**: `this->Exp += Amount;`
- **Training**: Skills advance at original Tibia 7.7 rates
- **Balance**: Slow progression encourages long-term play

**After (Enhanced System)**:
- **Multiplied Rate**: `this->Exp += Amount * SkillRateMultiplier;`
- **Configurable**: Server administrators can adjust training speed
- **Balanced**: Maintains skill progression curves while reducing time investment

#### 3. Implementation Details
- **Universal Application**: Affects all skills including magic level (combat, magic, distance, shielding, etc.)
- **Magic Level Integration**: Magic level advances 2x faster through mana spending on spells
- **Experience Source Independence**: Works with any experience gain method (combat, magic use, etc.)
- **Maintains Ratios**: Preserves relative skill advancement rates between different skills
- **Level Requirements**: Does not affect level-based restrictions or requirements

### Example Scenarios
**Original System (Multiplier = 1)**:
- Sword skill gains 1 experience point per successful hit
- Magic level gains experience equal to mana spent (100 mana = 100 magic level experience)
- Takes considerable time to advance from skill 70 to 71

**Enhanced System (Multiplier = 2)**:
- Sword skill gains 2 experience points per successful hit
- Magic level gains 2x experience from mana spent (100 mana = 200 magic level experience)
- All skill advancement occurs twice as fast while maintaining progression curves

### Technical Notes
- Applied in `TSkillProbe::Increase()`: `this->Exp += Amount * SkillRateMultiplier`
- Default multiplier: 2 (configurable via `skillratemultiplier = X` in .tibia config)
- Affects all skills including magic level advancement through mana spending
- Requires server restart to change multiplier value

## Enhanced Death Logging

### Overview
Enhanced the creature death logging system to include information about who killed the creature, improving server administration and debugging capabilities.

### Files Modified
- `src/crmain.cc`: Modified death logging in `TCreature::Death()` function

### Changes Made

#### Enhanced Log Format
**Before:**
```
Tod von Dragon: LoseInventory=1.
```

**After:**
```
Tod von Dragon: LoseInventory=1, Killer=PlayerName.
Tod von Rat: LoseInventory=0, Killer=NA.
```

#### Implementation Details
- **Killer Detection**: Uses existing `Murderer` field which tracks the creature responsible for the kill
- **Fallback Handling**: Shows "Killer=NA" when no murderer is recorded (environmental deaths, etc.)
- **Consistent Format**: Maintains existing log structure while adding killer information
- **All Creature Types**: Applies to all creatures (players, monsters, NPCs)

#### Benefits
1. **Server Administration**: Easier to track player activities and monster kills
2. **Debugging**: Helps identify issues with combat mechanics and kill attribution
3. **Statistics**: Enables better analysis of player vs creature interactions
4. **Anti-Cheat**: Assists in detecting unusual kill patterns

#### Log Examples
```
Tod von Ancient Dragon: LoseInventory=1, Killer=WarriorBob.
Tod von Demon: LoseInventory=1, Killer=MageSally.
Tod von PlayerX: LoseInventory=1, Killer=PK_Hunter.
Tod von Rat: LoseInventory=0, Killer=NA.
```

The enhanced logging provides valuable insights into server activity while maintaining the original log format's readability.

## CSV Kill Log

### Overview
Added a separate CSV-formatted kill log for easy analysis of PvP and combat statistics with coordinates and timestamps.

### Files Modified
- `src/main.cc`: Added kill log initialization and CSV header creation
- `src/crmain.cc`: Added CSV kill log entry generation in death function

### Changes Made
**New Log File**: `${LOGPATH}/kills.log`
- **Format**: CSV with headers: `datetime,timestamp,killer,victim,level,x,y,z`
- **Automatic Initialization**: Creates header on server startup
- **Kill Tracking**: Records all kills where a murderer is identified

**Example Output**:
```csv
datetime,timestamp,killer,victim,level,x,y,z
23.10.2025 15:30:45,1698065445,WarriorBob,Dragon,50,123,456,7
23.10.2025 15:31:12,1698065472,MageSally,Demon,75,200,300,8
23.10.2025 15:32:05,1698065525,PK_Hunter,PlayerX,45,150,250,6
```

### Technical Notes
- Only logs kills with identified murderers (excludes environmental deaths)
- Includes victim level and exact coordinates for spatial analysis
- Uses same datetime format as game.log for consistency
- Thread-safe logging via existing infrastructure
- Separate from game.log for specialized analysis tools

## Improved Potion Consumption

### Overview
Modified potion consumption behavior to destroy bottles completely after use, eliminating empty bottle inventory clutter and providing a cleaner user experience.

### Files Modified
- `src/magic.cc`: Modified `DrinkPotion()` function

### Changes Made

#### Enhanced Consumption Behavior
**Before:**
```cpp
Change(Obj, CONTAINERLIQUIDTYPE, LIQUID_NONE);  // Empty the bottle
```

**After:**
```cpp
Delete(Obj, -1);  // Destroy the bottle completely
```

#### Implementation Details
- **Consistent with Runes**: Now matches the behavior of depleted runes, which are also completely destroyed
- **Inventory Management**: Eliminates the need to carry empty bottles back to stores for refunds
- **Cleaner Experience**: Players no longer accumulate empty bottles in their inventory
- **Same Function**: Uses the same `Delete(Obj, -1)` call that runes use when charges are depleted

#### Benefits
1. **Inventory Space**: No more empty bottles cluttering player inventories
2. **User Experience**: Streamlined potion usage without micromanagement
3. **Consistency**: Matches the consumption pattern of other consumables (runes)
4. **Simplicity**: Removes the refund/return bottle mechanic complexity

#### Technical Notes
- **Safety**: Uses the standard object destruction function used throughout the codebase
- **No Side Effects**: Does not affect potion effectiveness, only post-consumption cleanup
- **Backward Compatible**: No impact on existing game mechanics beyond bottle handling

This change modernizes the potion system by removing tedious inventory management while maintaining all original functionality.

## New Mortal Strike Rune

### Overview
Added a knight-accessible ranged damage rune using the same damage calculation as berserk (exori) spell.

### Files Modified
- `src/magic.cc`: Added spell definition (#36) and damage calculation case

### Changes Made
**Spell #36**: "ad hur mort"
- **Requirements**: Level 30, Magic Level 3, 400 mana, 2 soul points, 3 charges per rune
- **Damage**: Level-only scaling: `(Level × ComputeDamage(NULL, 0, 80, 20)) / 25`
- **Effect**: Single-target ranged with EFFECT_DEATH, DAMAGE_PHYSICAL
- **RuneGr/RuneNr**: 79/99

### Technical Notes
- Uses previously unused spell slot #36
- Level-based damage scaling (no magic level dependency)
- Requires corresponding sprite in .dat file

## Life Potion to Great Mana Potion Conversion

### Overview
Converted life potions (LIQUID_LIFE) to provide enhanced mana restoration instead of health recovery.

### Files Modified
- `src/magic.cc`: Modified `DrinkPotion()` function LIQUID_LIFE case

### Changes Made
**Before**: `ComputeDamage(NULL, 0, 50, 25)` + `Heal()` → 50±25 health
**After**: `ComputeDamage(NULL, 0, 300, 150)` + `RefreshMana()` → 300±150 mana

**Comparison**: Regular mana potion provides ~100 mana, Great Mana Potion provides ~300 mana (3x)

### Technical Notes
- Uses existing LIQUID_LIFE item IDs and sprites
- Maintains bottle destruction behavior from previous potion improvements
- No changes to item data structure required

## Enhanced Rune Damage

### Overview
Increased damage output of Heavy Magic Missile, Great Fireball, and Explosion runes by 1.5x.

### Files Modified
- `src/magic.cc`: Modified damage calculations in `UseMagicItem()` for spells 8, 16, and 18

### Changes Made
**Heavy Magic Missile (Spell 8)**:
- Before: `ComputeDamage(Actor, SpellNr, 30, 10)` → 30±10 damage
- After: `ComputeDamage(Actor, SpellNr, 45, 15)` → 45±15 damage

**Great Fireball (Spell 16)**:
- Before: `ComputeDamage(Actor, SpellNr, 50, 15)` → 50±15 damage
- After: `ComputeDamage(Actor, SpellNr, 75, 23)` → 75±23 damage

**Explosion (Spell 18)**:
- Before: `ComputeDamage(Actor, SpellNr, 60, 40)` → 60±40 damage
- After: `ComputeDamage(Actor, SpellNr, 90, 60)` → 90±60 damage

### Technical Notes
- Proportional 1.5x increase applied to both base damage and damage variation
- Preserves original visual effects and damage types
- No changes to area of effect radii or targeting mechanics

## New Great Energy Ball Rune

### Overview
Added an energy-based area damage rune that mirrors Great Fireball mechanics but uses energy damage, effects, and animations instead of fire.

### Files Modified
- `src/magic.cc`: Added spell definition (#43) and damage calculation case

### Changes Made
**Spell #43**: "ad evo gran vis"
- **Requirements**: Level 23, Magic Level 4, 480 mana, 3 soul points, 2 charges per rune
- **Damage**: `ComputeDamage(Actor, SpellNr, 75, 23)` → 75±23 energy damage
- **Effect**: Uses MassCombat with EFFECT_ENERGY, DAMAGE_ENERGY, ANIMATION_ENERGY
- **Area**: 4-tile radius (same as Great Fireball)
- **RuneGr/RuneNr**: 79/45

### Technical Notes
- Uses previously unused spell slot #43
- Identical mechanics to Great Fireball except for damage type and visual effects
- Requires corresponding sprite in .dat file

## Poison Storm to Great Mass Healing Conversion

### Overview
Converted the underused Poison Storm spell into Great Mass Healing, providing druids with an enhanced area healing spell.

### Files Modified
- `src/magic.cc`: Modified spell definition and damage calculation for spell #56

### Changes Made
**Spell #56**: "ex evo gran mas res" (formerly "ex evo gran mas pox")
- **Name**: "Poison Storm" → "Great Mass Healing"
- **Function**: Poison damage → Area healing
- **Healing**: `ComputeDamage(Actor, SpellNr, 800, 160)` → 800±160 healing
- **Radius**: Increased from 8 to 6 tiles (optimized for healing vs damage)
- **Requirements**: Level 50, 600 mana (maintained druid-only high-level spell)

**Comparison with Regular Mass Healing**:
- **Regular Mass Healing**: 200±40 healing, 4-tile radius, 150 mana, level 36
- **Great Mass Healing**: 800±160 healing, 6-tile radius, 600 mana, level 50

### Technical Notes
- Uses existing MassHeal function instead of MassCombat
- Enhanced healing amount (4x regular mass healing for balanced mana efficiency)
- Larger healing radius for better group support
- Maintains original mana cost and level requirements for balance

## New Great Fire Wave Spell

### Overview
Added a powerful fire-based wave spell that mirrors Energy Wave mechanics but uses fire damage and effects.

### Files Modified
- `src/magic.cc`: Added spell definition (#104) and damage calculation case

### Changes Made
**Spell #104**: "ex evo gran flam hur"
- **Requirements**: Level 38, 250 mana (identical to Energy Wave)
- **Damage**: `ComputeDamage(Actor, SpellNr, 150, 50)` → 150±50 fire damage
- **Effect**: Uses AngleCombat with EFFECT_FIRE, DAMAGE_FIRE
- **Area**: 5-tile range, 30-degree angle (identical to Energy Wave)
- **Flags**: 1 (aggressive spell)

### Technical Notes
- Uses previously unused spell slot #104
- Identical mechanics to Energy Wave (spell #13) except for damage type and visual effects
- Distinguishable from existing Fire Wave (spell #19) which is much weaker (30±10 damage, level 18)
- Provides high-level fire alternative to Energy Wave for balanced elemental options