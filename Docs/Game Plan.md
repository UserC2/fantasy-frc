## Skill System
* Robots can be customized by adjusting different "skills", which each effect how the robot plays the game in different ways.
* The amount of skill points the player has to distribute is affected by the difficulty level.
* There are three types of skills:
  * Decision: Affects the chance of making the *correct* decision
  * Action: Affects the chance of succeeding at an action
  * Time: Affects the time necessary to take an action

### Levels & Points
* 5 levels
* Starts at 1st level
* Costs 1 point to go up one level
* For some non-essential skills, can go down to level 0 (incapable of performing skill) but you get 2 skill points in return
  * Failed reliability check can cause any skill to drop to level 0, regardless of how essential it is
* Time multiplier: (multiplier to be adjusted according to playtesting)
  * 0: Robot cannot perform action
  * 1: 1.5x
  * 2: 1.2x
  * 3: 1x
  * 4: 0.8x
  * 5: 0.6x
  * TODO: levels should have a descriptive name
* Decision/Action success chance: (% to be adjusted according to playtesting)
  * 0: 0%
  * 1: 40%
  * 2: 60%
  * 3: 75%
  * 4: 85%
  * 5: 95%
  * TODO: levels should have a descriptive name
  * Decision and Action skills are stored as a number between 0 and 1.

### Robot Skill Types
Global:
* Reliability: Affects chance to break down within match
  * Reduces random *robot* skill (i.e. pneumatics failure should not affect drive team coordination) by certain amount of points depending on severity
* Speed: Affects transit time
* Agility: Affects chance to avoid defense
* Coordination: Affects alliance delay time and makes decisions between alliance members more effective
  * e.g. if one robot is trying to score amp and there's already one piece, no other robot is going to try to score amp, will try to do speaker if coordination is high
* Automation: Affects scoring, intake, and end game time
* Shooting: Affects scoring *transit time* (can score pieces from far away)

Autonomous:
* Autonomous: Affects points scored in autonomous period

Teleop:
* Defense: Affects chance to delay opponent robot during scoring
* Strategy: Affects chance to make more efficient gameplay decisions
  * Decisions also affected by robot capabilities (e.g. a robot with high strategy isn't going to try to score on an amp if it has a level 1 or 2 low goal skill)
  * e.g. Crescendo: whether to score note on speaker or amp
    * e.g. if amplified and high strategy %, score on speaker
    * e.g. if need few more points and not amplified, score on speaker
    * e.g. if not amplified but early in game, score on amp

Game Elements:
* When a game year is chosen, these skills are renamed to the actual action they enable or enhance in the current game.
  * e.g. "high goal" when year = 2024 -> "speaker"
* High Goal: Affects chance of successful scoring on more efficient goal
* Low Goal: Affects chance of sucessful scoring on less efficient goal
* Special Piece: Affects chance of scoring special piece
  * Not always applicable, some games don't have one (e.g. Charged Up or Rapid React)
  * e.g. Gear in Steamworks or High Note in Crescendo
* End Game: Sequence of tasks, affects chance of success on each step. End game sequence stops at first failed task.
  * Certain stages are affected by coordination depending on game.
  * e.g. Crescendo:
    * Park
    * Trap
    * Climb
    * Harmony -> Affected by coordination
  * e.g. Charged Up:
    * Park
    * Docked
    * Engaged -> Affected by coordination
  * e.g. Rapid React:
    * Low
    * Mid
    * High
    * Traverse -> Affected by coordination

### Skill Checks
* A "skill check" is an expression that generates a random number and then checks it against a robot's skill to determine if an action or decision succeeded.

Algorithm
```
success if: skill >= pseudo-randomly generated number between 0 and 1
```

## Match Gameplay
### Autonomous
* 15 seconds long
* Autonomous skill check for Mobility
  * Always succeeds if Autonomous skill is > 0
* Scoring check for preloaded pieces
* With remaining time, robot can try to ground pickup and score as many pieces as possible
  * If Autonomous skill check is successful:
    * Robot randomly decides on game piece to grab
    * *If* robot attempts to grab same game piece as another robot, perform Coordination skill check:
      * Success: Robot grabs a random *unused* game piece (it should not attempt to grab a game piece that another robot is targeting (causing another Coordination check) if the first Coordination check succeeded)
        * If no remaining game pieces, robot takes no more actions for the remainder of autonomous
      * Fail: Robot is delayed by alliance member and cannot take an action until other robot picks up game piece
    * Once game piece is scored, continue attempting to ground pickup and score game pieces
    * If the autonomous period ends before the game piece is scored, the robot retains the game piece
  * If it fails, the robot takes no more actions for the remainder of autonomous

### Teleop
* 135 seconds long (2:15)

Scoring Loop:
* If game piece is in robot:
  * Perform Strategy check for low or high goal:
    * Perform efficiency check on low/high goal
    * If successful: Take more efficient option
    * If unsuccessful: Take less efficient option
  * When taking option
    (defense)
* If game piece is not in robot:
  * Perform Strategy check for ground pickup or station pickup:
    * Perform efficiency check on ground/station pickup
    * If the Strategy check succeeds, choose the more efficient option
    * If the Strategy check fails, choose the less efficient option

Endgame:
* Once the teleop timer reaches 30s remaining (climb time), make a Strategy check to stop scoring and begin endgame:
  * Total endgame time = endgame transit time + endgame time + 1s safety margin:
  * When teleop timer = 2x endgame time, make a Strategy check:
    * On success: Wait
    * On failure: Start endgame (too early)
  * When teleop timer = 1.5x endgame time, make a Strategy check:
    * On success: Wait
    * On failure: Start endgame (too early)
  * When teleop timer = endgame time, make a Strategy check:
    * On success: Start endgame (correct timing)
    * On failure: Don't start endgame (now impossible to complete endgame)

### Shared
* Scoring:
  * Transit time + scoring time + defense delay + alliance delay
  * Success check
* Station pickup:
  * As many pieces as desired
  * Transit time + pickup time + defense delay + alliance delay
  * Success check
* Ground pickup:
  * Limited amount of game pieces
    * Can be replenished as pieces are scored depending on game (e.g. Rapid React)
  * Transit time + (less) pickup time + defense delay
  * Success check

### Efficiency Checks
Given a list of actions, decide which one is more effective for the robot to attempt to perform.
* Each option has an efficiency, calculated with the following equation:
```
input: <option>

alliance delay = 
  if alliance members at <option> >= max alliance members at or going to <option> (excluding self):
    get time remaining for <option> for each alliance member at or going to <option>
      choose the smallest amount of time
  else:
    0

action time =
  if applicable action time skill(s) > 0:
    = applicable action time skill(s) * action time for <option>
  else:
    Infinity

transit time = 
  if applicable transit time skill(s) > 0:
    = applicable transit time skills(s) * transit time from current position to action (see "Robot States + Transit Time" section)
  else:
    Infinity

total time = action time + transit time + alliance delay
efficiency = applicable skill for <option> / total time
```
* If the robot's strategy check succeeds, the option with a greater efficiency is chosen. If it fails, the option with the second-highest efficiency is chosen.

### Options
* Time for each action is customizable according to game.
* Low Goal
* High Goal
* Special Goal
* Ground Pickup
* Station Pickup
* Endgame

### Robot States + Transit Time
Possible Positions:
* Starting
* Close ground pickup
* Far ground pickup
* Station pickup
* Low goal
* High goal
* Special goal (if applicable)
* End game
* Defense
* In transit

Transit Times:
* All transit times fall under one of these categories:
  * Far: 6s
    * Starting -> Station Pickup
  * Middle: 4s
    * Starting -> Far Ground Pickup
    * Starting -> Defense
  * Close: 2s
    * Starting -> Close Ground Pickup
    * Starting -> End Game
    * Starting -> Low/High/Special Goal
  * Not Allowed:
    * Any position -> Itself
    * Any position -> Starting
* TODO: add other ones

Position Limits:
* Configured by Game:
* e.g. for Crescendo:
  * Starting: No limit
  * Ground pickup: # of pieces left on the ground
  * Station pickup: 2
  * Low goal: 1
  * High goal: No limit
  * End game: 3
  * In transit: Infinite

## Features
* Prepare robots before match
* Save robots/alliances for later (team #, name, skill distribution + difficulty, win/loss ratio maybe?)
* Full event structure
* Autogenerated robots?
* 1 v 1 matches
* alliance v alliance matches (quals, playoffs)

* For alliance skill points, maybe it should be calculated from the skill points of each robot that makes it up? That would be more realistic (and more interesting since you'd actually have to choose good robots that have good drive teams too)

## Robot Configuration
* Player can configure as many as they want (or choose presets), can leave computer to randomly choose presets or randomly configure robots
* Player can configure
* Player can select from presets
* Computer can auto-configure

## Alliance Configuration
* Player can configure (single match + full event (but only their alliance))
* Computer can auto-configure randomly (single match)
* Computer can auto-configure according to RP & robot stats during a full event

### Full Event
* Alliance selection proceeds according to

## Robot vs. Robot Match
* Two robots compete against each other to get the most points.
  * RP not considered.

## Alliance vs. Alliance Match
* Two alliances (of three (or four) robots) compete against each other to get the most points.
* RP is calculated if current match is a practice or qualification match

## Scouting
* Maybe robot skill points are actually team skill points
* Some skill points can be put towards scouting, gives you more accurate information on opponent robots (normally, you get some randomly skewed statistics)
* scouting should start with certain # of points that gives you an average estimate of robot skills
  * if you are playing a full event, you can add these points to your robot's stats (and suffer worse scouting) or you can use some robot points to enhance your scouting
  * if you are playing a single match, the starting scouting points are not able to be used

## Full Event
### Properties
* Year
* Number of robots, used to calculate:
  * Number of practice matches
  * Number of qualification matches
  * Number of alliances in playoffs
  * Playoff match lineup
  * There must be at least six robots (must have at least two alliances of three robots)

### Robot Configuration
* Player can configure as many robots as they want, and can leave computer to generate or select the rest.

### Structure
* Every robot only plays a portion of the matches in each section, so some matches cannot be played by the player. These matches are played by the computer and the results are displayed to the player.
* Between each match, the player can choose to:
  * Fix a broken robot
  * Improve a robot
  * Train drive team
  * (Playoff intermission) Train alliance/Discuss strategy

Practice Matches
* Randomly matched to alliance partners
* No effect on a team's RP

Qualification Matches
* Randomly matched to alliance partners
* Match results contribute to team's RP

Alliance Selection
* Player & computer select alliance partners for playoffs

Intermission
* Can train alliance, improve robot functionality, or repair broken robots.
* Worth more than regular intermission

Playoff Matches
* Follow playoff double-elimination structure, determines event winners

## Environments
* So, either field faults become part of the rules (easier, more entertaining in the short term), or the competition year can be changed (way harder, more interesting).
* Different games are much more interesting to program and to play, so we'll do that.
* Arena faults are fun so we'll still do those too.
  * Game pieces being utterly destroyed aren't counted as arena faults; this game will consider them to be.

## Proposed Features
* "Bias" system for certain skill checks to make them easier or more difficult
  * e.g. driver should know if their robot cannot score on high goal, and should have a VERY LOW (or zero) chance of attempting it if it can't instead of the current random selection based on skill
* A higher high goal skill should be more difficult to obtain than a higher low goal skill, otherwise players would never contribute points towards low goals (since they award points less efficiently)
