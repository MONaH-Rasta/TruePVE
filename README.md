# TruePVE

Oxide plugin for Rust. Better PVE/PVP implementation.

**True PVE** is a damage control plugin originally intended to improve the default server PVE mode (server.pve = true) for servers who wish to truly be PVE. This plugin can be used to fine-tune PVP behavior as well, enabling a range of damage control configurations to be implemented to customize PVP, PVE, and anything in between.

**Note:** TruePVE is meant to be used with `server.pve false` (PVP mode ON)!  Running TruePVE with `server.pve true` can have unintended effects.

Before you download any new version of this plugin, please read the _update notes_ to see what changed!  Important update information is normally included in these notes, and will let you know if you need to do anything, and what changes you can expect to see in the new version.

## Commands

### Console Commands

* `tpve.def` - Wipe and create default configuration/data
* `tpve.sched [enable|disable]` - Enable/disable the schedule
* `tpve.trace` - Toggle tracing; automatically disabled after 5m (hard-coded) to prevent accidental log overfilling. See below for more details about tracing.
* `tpve.usage` - Show command usage info

### Chat Commands

* `/tpve_prod` - Print out the Type and Prefab name of the entity being looked at (for entity groups)
* `/tpve map [name] <target>` - Create/update/delete a mapping.  [name] is the name of the mapping or the zone ID you are mapping.  `<target>` is an optional parameter defining either the RuleSet name you wish to map to or "exclude" to skip processing.  Leaving `<target>` empty will delete the mapping for [name]

## Configuration

* **Config Version** - Do not change
* **Default RuleSet** - Name of the default RuleSet to use
* **Configuration Options** - Global configuration options
  * **handleDamage** - Enables TruePVE damage handling
  * **useZones** - Enables use of zone-specific damage configurations (requires ZoneManager)
* **Mappings** - Maps a zone name (LiteZones) or name/ID (ZoneManager) to a RuleSet name, or simply a RuleSet name to itself.  Can be used to map multiple zones to the same RuleSet.  Also can be used to create exclusion zones (zones with default Rust behavior) by mapping to "exclude".  Example:

```json
  "Mappings": {
     "default": "default",
     "66499587": "killall",
     "62819081": "exclude"
  },
```

* **Schedule** - Schedule RuleSet changes
  * **enabled** - Enables schedule usage
  * **useRealtime** - Enables using realtime (server time)
  * **broadcast** - Enables broadcast messages to be sent on scheduled RuleSet changes (but no message is broadcast if there is no message set for the schedule entry)
  * **entries** - Schedule Entries - See below for details
RuleSets - Defined damage configurations - _See below for details_
Entity Groups - Defined entity groupings used in rules - _See below for details_

### Debug

Tracing turns on basic debug logging to help with debugging and identifying issues with RuleSet, rule, and EntityGroup configurations.  Tracing _should_ be manually toggled off after damage logging is captured, but will automatically disable after 5min (hard-coded) - this is to prevent the the log from overflowing if accidentally left on.  The trace results are output to **`./oxide/logs/TruePVE/truepve_ruletrace-[date].txt`**.

Trace text identifies:

* Initiator type and prefab name
* Target type and prefab name
* Whether an exclusion group is hit
* Which special logic blocks are hit
* Which RuleSet is used
* Which EntityGroups are selected
* Which rules are evaluated, and the final result (true: allow damage, false: block damage, null: Rust default damage handling)

Example output:

```log
======================
==  STARTING TRACE  ==
==  15:09:18.38210  ==
======================
From: BasePlayer, player
To: Workbench, workbench3.deployed
   No shared locations (empty location) - no exclusions
No exclusion found - lookup RuleSet
Using RuleSet "default"
No match in pre-checks; evaluating RuleSet rules...
  Initator EntityGroup matches: players
  Target EntityGroup matches: none
   Evaluating Rules...
    Checking direct initiator->target rules...
    No direct match rules found; continuing...
     Evaluating "players->any"...
      No match found
    No matching initiator->any rules found; continuing...
    No matching any->target rules found; returning default value: False
======================
==  STARTING TRACE  ==
==  15:09:18.69712  ==
======================
From: BasePlayer, player
To: VendingMachine, vendingmachine.deployed
   No shared locations (empty location) - no exclusions
No exclusion found - lookup RuleSet
Using RuleSet "default"
Door/StorageContainer detected with immortal flag; lock check results: null; continue checks
Initiator is player with authorization over non-player target; allow and return
```

### Entity Groups

Entity Groups are containers that, not surprisingly, define a group of entities.

The group **name** is used as a reference in Rules, and the **members** and **exclusions** define which entities are within the group.  Entity Groups are shared between all RuleSets, so you don't need to create multiple versions of the same group for different RuleSets.

The **members** and **exclusions** of the group can contain both Types and Prefab names (provided via /tpve_prod command) - these fields are case sensitive, and generally the Type is camel case while the Prefab is lowercase.  Also, generally, a Type can contain many prefabs, but a prefab is always the same Type, so you can define a Type as a member, and exclude unwanted individual Prefabs by defining them in the exclusions.

```json
// Example entity group
{
  "name": "players",
  "members": "BasePlayer",
  "exclusions": ""
}
```

### RuleSets

A RuleSet is, also not surprisingly, a set of rules.

The **name** of the RuleSet is used as a reference when scheduling RuleSet changes, or zone-specific configurations.

The **defaultAllowDamage** option defines what the standard behavior of the RuleSet is - that is, whether it allows or blocks damage overall. This should always be set **false** for PVE rulesets.

The **flags** option allows you to define some built-in rules (rules that require more specialized coding behind the scenes).  Only the defined flags are enabled, while any flags not defined are naturally disabled.  A list and description of available flags is below.

The **rules** section is a list of quasi-semantically accurate rules (no more links!).  They define a ruleset and its behavior toward another ruleset.
The format is: **`[RuleSet 1] [behavior] [RuleSet 2]`**, however currently the RuleSets are pulled off the ends of the rule and only a few behaviors have any impact on the rule, so you can pretty much say anything in between RuleSet 1 and 2 and it will be assumed to mean "allow damage".  Exceptions of this are if the words "cannot" or "can't" appear in the behavior, then the rule will be negated and assumed to mean "prevent damage".

Also, there are generalized RuleSet names available to define a broader application - the words "_anything_", "_nothing_", "_everything_", "_all_", "_any_", and "_none_" can be used either for RuleSet 1 or RuleSet 2.  Again though, semantics are taken into account, so "nothing" or "none" will effectively invert the rule meaning again.  So using double-negatives like "players can't hurt nothing" will translate to "players can hurt everything", as well as likely prevent you from joining any yacht clubs.

#### Rule Priorities

There is a certain priority that must be taken into account when writing rules.  Generally, the more specific rules override the broad ones (rules with "anything" or "nothing", etc).  If you have two rules: "anything can hurt players" and "barricades cannot hurt players", then the barricades rule will always override the "anything" rule.

### Schedule Entries

Schedule entries define scheduled global RuleSet changes, and have been rolled into a single line with three (3) parts separated by spaces:

1. **Time** - The time the scheduled entry will take effect.  For in-game time, the format is "_hh:mm_" where hh is hours (in 24-hour format), and mm is minutes.  Realtime schedule entries, however, should be entered as "_d.hh:mm_", where d is the day of the week as 0-6 (Sunday-Saturday).  The day of week also now accepts `*` (asterisk) as a wildcard to indicate daily, so a realtime entry of "*.08:00" will fire daily at 0800 (8:00am).  Note that for realtime, if you do not provide a day of week, it will be assumed to be 0 (Sunday), and your schedule entries will only fire on Sundays!
2. **RuleSet** - The RuleSet name to set globally at the specified time
3. **Message** - All text after the RuleSet name is used as a broadcast message to be sent to all players when the RuleSet changes.  This message is also sent to any player who logs on during the duration of the schedule entry.  Shockingly, if the message is empty, no message will be broadcast.

```json
// Example schedule entries using realtime
"*.12:00 default PVE enabled!" // at 12:00 daily, set RuleSet "default" and broadcast "PVE enabled!"
"*.18:00 pvp PVP time!" // at 18:00 (6pm) daily, set RuleSet "pvp" and broadcast "PVP time!"
```

### RuleSet Flags

**Note** - Most of these flags were carried over from previous configuration options, but some have changed functionality slightly.

**Overrides Rules:** - All flags ignore evaluation of rules if in use, with the exception of TrapsIgnorePlayers, TurretsIgnorePlayers, TurretsIgnoreScientist, StaticSamSitesIgnorePlayers and PlayerSamSitesIgnorePlayers that allow exceptions in entity groups only

**Ignores All Rules:** - NoHeliDamage, NoHeliDamagePlayer, NoHeliDamageQuarry is required to handle Heli damage. Not using a heli flag allows the damage by default. Either way, rules are never evaluated.

* Decay damage - TruePVE does NOT handle decay damage
* Looting - TruePVE does NOT handle looting. Use the Prevent Looting plugin
* Animal damage - Rules are not evaluated - All damage is allowed to and from this object
* AdvancedChristmasLights - Rules are not evaluated - you must be able to build to damage this object
* GrowableEntity - Rules are not evaluated - you must be able to build, or be the owner, to damage this object

* **AuthorizedDamage** is a very niche flag and is grossly misunderstood. It allows players to damage entities they own, or have cupboard authorization over. When paired with **CupboardOwnership** it will allow damage when no tool cupboard is protecting the entity. **AuthorizedDamageRequiresOwnership** helps round it out by allowing damage when players own the entity, are an ally, or attack an entity not protected by a tool cupboard.

* **AuthorizeDamage** overrides rules except when the rules apply to mounts or samsites. **AuthorizedDamageRequiresOwnership** overrides rules except when the player is an ally and the rules apply to mounts. Rules in this case will continue evaluation if the damage is not blocked. If the player is not an ally then the rules may override samsites in addition to mounts.

* **SuicideBlocked** - Block suicide - Does not use rules
* **SelfDamage** - Allow player (usually) to damage themselves, e.g. with C4 or BeanCans, etc.
* **CupboardOwnership** - When enabled with _AuthorizedDamage_ will treat entities outside of cupboard range as unowned, and entities inside cupboard range will require authorization.
* **TwigDamage** - Allows players to damage any twig building blocks regardless of authorization (to encourage sound building practices).  This currently requires that the AuthorizedDamage flag also be set pending a rewrite.
* **NoHeliDamage** - Disables heli damage (use NoHeliDamageQuarry for quarries, and NoHeliDamagePlayer for players)
* **NoHeliDamagePlayer** - Prevents heli from hurting players
* **NoHeliDamageQuarry** - Prevents heli from damaging quarries
* **HeliDamageLocked** - Allows heli to damage locked boxes/doors (requires LockedBoxesImmortal or LockedDoorsImmortal)
* **HumanNPCDamage** - Enables HumanNPC damage
* **LockedBoxesImmortal** - Locked boxes are immortal (`HeliDamageLocked` overrides this)
* **LockedDoorsImmortal** - Locked doors are immortal (`HeliDamageLocked` overrides this)
* **AdminsHurtSleepers** - Admins can hurt sleepers
* **ProtectedSleepers** - Sleepers are protected from NPC damage
* **TrapsIgnorePlayers** - Players don't trigger traps (not working for bear/snap traps)
* **TurretsIgnorePlayers** - Players don't trigger turrets (not working for flame turret)
* **TurretsIgnoreScientists** - Scientists and all other npcs don't trigger turrets
* **StaticSamSitesIgnorePlayers** - Static sam sites ignore all players, such as from Launch Site or a plugin that has set SamSite.staticRespawn to true.
* **PlayerSamSitesIgnorePlayers** - Player sam sites ignore all players, such as from deployed sam sites. If another plugin sets SamSite.staticRespawn to true then this flag will not work for that sam site.
* **MiniCopterIsImmuneToCollision** - REMOVED - use rule `mini cannot hurt mini` with `MiniCopter` as a member in an entity group
* **MiniCannotHurtPlayers** - REMOVED - use rule `mini cannot hurt players` with `MiniCopter` as a member in an entity group
* **CarsImmunity** - REMOVED - see default config as the rule and entity group for this one larger.
* **NoTurretDamagePlayer** - REMOVED - use TurretsIgnorePlayers flag
* **NoTurretDamageScientist** - REMOVED - use TurretsIgnoreScientists flag

## For Developers

Hooks are available to external plugins for adding, updating and removing mappings:

```csharp
// add or update a mapping - returns true if successful
bool AddOrUpdateMapping(string key, string ruleset);

// remove a mapping - returns true if successful
bool RemoveMapping(string key);
```

```csharp
//Get the current ruleset name
TruePVE.Call<string>("CurrentRuleSetName");
```

### External API Calls

We call the following hook while processing OnEntityTakeDamage();

**CanEntityTakeDamage**:  Returning _true_/_false_ will allow/disallow damage and skip normal TruePVE damage evaluation.

```csharp
object CanEntityTakeDamage(BaseCombatEntity entity, HitInfo hitinfo)
```

We call the following hook while processing CanBeTargeted();

**CanEntityBeTargeted**:  Returning _true_/_false_ will allow/disallow targeting and skip normal TruePVE target evaluation.

```csharp
object CanEntityBeTargeted(BasePlayer player, BaseEntity turret)
```

We call the following hook while processing OnTrapTrigger();

**CanEntityTrapTrigger**:  Returning _true_/_false_ will allow/disallow trigger event and skip normal TruePVE trigger evaluation.

```csharp
object CanEntityTrapTrigger(BaseTrap trap, BasePlayer player)
```

## Zone Integration

As of ZoneManager 2.4.61, you no longer need to modify ZoneManager!
ZoneManager 3.0.0 will not return zoneIds for us.  Any version above or below should work.

## Credits

* **ignignokt84**, the original author of this plugin
* **rfc1920**, for helping maintain the plugin
* **nivex**, for helping maintain the plugin
