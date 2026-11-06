# State of Decay (2013) co-op: progress and handoff

Single source of truth for the current state of the co-op work. Read this first, then the relevant parts of
[NOTES.md](NOTES.md) (full research log, by phase) and [README.md](README.md) (public summary).

**Rule for every agent:** after each discovery, fix or verified result, add a dated entry at the top of the
[Findings log](#findings-log) and update [Current state](#current-state) if it changed. Keep it free of private
data (no machine names, user folders, IP addresses, account names).

## Current state

Last updated: 2026-10-28.

Two copies of the game run side by side (host = Marcus Campbell, joining player = Ed Jones since 2026-10-28;
before that the joining player was "Maya"). Everything is applied at
runtime by Python scripts under the gitignored `local/` folder; no game file is modified except the loose
`Game/scripts/gamerules/singleplayer.lua` (extra RMIs) in both test copies.

| Area | State |
|---|---|
| Join, walking, camera, HUD, minimap, mission banner on the joining copy | Works |
| Attacks, dodges, jumps replicate both ways | Works |
| Zombies visible on both machines, clothed, animated on the client (no T-pose), host's hit reactions shown | Works (verified 2026-10-24) |
| Zombies target, chase and damage her, also with him 300-450 m away | Works (soaks 2026-10-24) |
| Her melee hits and kills zombies; her kills counted for her, not the host | Works (final build 2026-10-26: her +1, host +0) |
| Host player's own weapon kills zombies, counted for him | Works (final build 2026-10-26: his +4, hers +0) |
| Zombies attack the host player | Works (final build 2026-10-26: 90 -> 1 in 15 s) |
| Zombies attack her with the host's player near (20 m) | Works (final build 2026-10-26: 100 -> 64 in 9 s) |
| Zombies attack her with the host's player far (300 m) | Works, weaker (fresh boot 2026-10-26: 100 -> 86 / 72 in 15 s vs 100 -> 17 near; fewer zombies around her) |
| Weapons: her starter hammer, cycling from community storage (middle mouse), host's journal equip | Works |
| Host stability (hangs / silent deaths) | **Fixed**: minimap name-table race (2026-10-24) and a per-call buffer leak in our tooling (2026-10-26) |
| Looting / doors / searching for the joining player | Doors work on her machine: state, door and player animations, collision (`maya_record.py`, `door_sync.py`). Her search runs and completes on the host with her own search manager (`search_ctx.py`); her loot screen opens on her machine with her own backpack, and Take/Use apply to her backpack on the host (`loot_relay.py`) |
| Getting hurt, going down and getting up (her) | Works: health replicated, HUD follows, down at 1 HP, mash Space to get up (`downed.py`) |
| Starting a session from the game's main menu | Works: "Co-op Play" item with Host Game (Continue / New Story) and Join Game (hosts found on the local network); `coop_agent.py` per copy runs the co-op setup on each side (2026-10-28) |
| Characters | New story: host Marcus, player 2 Ed (Ed's own survivor record, his NPC body hidden everywhere), players 3-4 random generic survivors no record uses, never a story character (`maya_record.pick_def`) |
| Up to 4 players | Works (2026-10-28): host + 3 joining players through the menu; every co-op module handles every joining player; 20-minute 4-player soak 10/10 fight + search/loot cycles; a player leaving mid-game leaves the others playing |
| Leaving and coming back | Works (2026-10-29): a player who left frees its slots everywhere; one who comes back (or the next one) takes the free slots and the departed player's character (Ed stays Ed) |
| Separate PCs | Works (2026-10-29): a second PC on the LAN joined through Co-op Play > Join Game as Ed; its moves arrive on the host. Files for it: `local/make_coop_bundle.py` (scripts + README), `coop_firewall.ps1` on the host PC, `coop_hosts.txt` when broadcasts don't get through; a VPN on either PC must leave the LAN out |
| Session stability | Asserts and missing-entity updates no longer kill the host (`crash_guards.py`); 25-minute play soak 19/19 cycles without a crash |
| Time of day | Works (2026-11-02): the host's clock runs (it stood still at midnight as a server), joining players follow the host's time, `COOP_HOUR=12` or `tod_sync.py host 12` for daytime (`tod_sync.py`) |
| Vehicles | Untested in co-op: the only vehicle at the story start is locked for both players |
| Missions, community, journal data for the joining player | Not started (she now has a community record on the host; nothing mirrored to her machine) |

## How to run a session

1. Never start or modify the Steam install. Test copies live in `local/test-install` (host) and
   `local/client-install` (client); both gitignored.
2. Keep the PC awake and the display on: `python local/keep_awake.py 4`. With the display off a new game
   process finds no Direct3D device ("Could not find any compatible Direct3D devices"); waking from Modern
   Standby crashes both renderers. A locked screen also blocks booting.
3. From the game's menu: `local/play_menu.sh` starts both copies through `local/coop_agent.py` (one agent per
   copy; on two PCs run one `coop_agent.py` per PC). In one window Co-op Play > Host Game > Continue Story or
   New Story; in the other Co-op Play > Join Game > the host's entry. Steps 4-5 are then done by the agents.
   Scripted alternative: steps 4-5.
4. `local/restart_session.sh` (kills only the test copies, boots host + client, OKs the "Unsupported video card"
   dialog via `local/dismiss_gpu.ps1`).
5. `python local/coop_setup.py` installs everything on both machines and starts `local/hud_relay.py`
   (must keep running on the host).
6. Optional diagnostics: `Proc.trap_faults()` on both processes (in-process crash catcher), exit watchers
   (`local/exit_watch.py host|client`).

Use a real Python 3 (the Windows Store stub does not work); `capstone` and `pillow` are needed by the tools.

## Co-op layer (local/, installed by coop_setup.py)

| Module | What it does |
|---|---|
| `sodlib.py` | Process access, mailbox calls on the game's main thread (PeekMessage hook), hooks, Lua/console exec, hardware breakpoints (`count_exec`, `watch_writes`), `trap_exits`, `trap_faults` |
| `prep_client.py`, `client_tick.py` | Joining copy: UI gate, front-end/HUD update replacement |
| `player_sm.py`, `peer_input.py` | UL state machines for both players on both machines; input sent as chat messages (`coop:`/`cooh:`) |
| `peer_melee.py` | Her hits applied to zombies on the host (CombatDamage with her params) |
| `zombie_targets.py` | Zombies pick her as a target; contact/strike on her |
| `zm_focus.py` | Zombie manager: focus point for her, cull uses the nearer player, population pass around her (<=300 m) |
| `kill_counts.py` | Her kills counted for her (chat `kilh:`); HUD banner off (`SHOW_BANNERS`) |
| `weapon_relay.py` | Melee weapon hash relay both ways, starter hammer, cycling |
| `pause_sync.py`, `remote_fx.py`, `turn_clamp.py`, `sync_models.py` | Shared pause, no host screen FX on client, turning, NPC models |
| `hud_relay.py` + `zombie_sync.py` | Host -> client over RMIs: banner, minimap icons, zombie bodies + animations + visibility (matched by position), split into <=400-char parts |
| `coop_menu.py`, `coop_agent.py` | Main-menu "Co-op Play" item and submenus (native hooks on the menu filler, title lookup and back input); per-copy agent: LAN discovery (UDP 64091), control channel (TCP 64092), host/join flows, coop_setup steps split by machine |
| `crash_guards.py` | Null guards, NaN-position guard, **radar_cache** (stability fix) |

Main test tools: `zm_soak.py` (long stress cycles), `soak_ab.sh`, `zm_freeze_test.py`, `kill_verify.py`,
`goal_check.py`, `path_hit.py`, `zombie_spawn.py`, `dbg_watch.py`, `exit_watch.py`, `anim_twice.py`,
`zm_dupes.py`, `use_test.py`, `door_pose.py`, `use_conds.py`, `cond_snap.py`, `far_strikes.py`, `anim_rate.py`,
`zcomb_conds.py`, `reaction_ab.py`, `contact_ab.py`.

## Next steps

1. Optional: more zombies around her when the host's player is far (population pass density, `zm_focus`), so
   far fights are as dense as near ones. Zombies near her spend much time in `Reaction.Basic` (not from her
   attacks; cause unknown).
2. Her UL character data on her machine (journal, HUD item slot, using backpack items outside loot screens).
   The loot screen's backpack panel already shows how to mirror UI from the host (`loot_relay.py`).
3. Missions and community for the joining player; vehicles once a save has them unlocked.
4. Commit/PR only when the user asks.

## Findings log

Newest first. One entry per discovery; link the NOTES section for details.

### 2026-11-06 - walk-stop glide found and fixed: a stopped copy stands still
Owner: the glide happens on the host's screen when the joining player stops after walking, and the other way round.
(The previous entries measured sprints; a walk has no relayed state, so none of that touched this.)
- walk_stop.py reproduced it both ways: the copy ran about 0.1 m past its player's end point, its idle clip started
  at +0.17 s, and from +0.3 to +0.8 s the body slid back those 0.1 m with the idle feet standing still; it came to
  rest at +0.76-0.82 s, its player at +0.3 s. The received velocity was already 0 by then.
- Cause: the copy's desired velocity (0x169ED70) is (target - position) normalised, times +0x8C = clamp(distance /
  0.2 s, 0.15, 12 m/s), with no regard for its player having stopped.
- `rest_deadzone.py` (both machines, from coop_setup): with a received velocity under 0.3 m/s and the copy within
  0.2 m of its target, that function returns zero. The error left goes with the next step.
- Verified from a fresh session, both ways, 4 runs each: the copy rests at +0.23-0.28 s, its player at +0.29-0.33 s,
  no movement back; it stands 0.09-0.16 m past the exact point until the next step. Sprint stops unchanged.
- Corrected reading of the controller: the +0x8C speed in that function is what moves the copy, not only the
  received velocity. Sprint speed wobble (5-8 m/s against 7) is still open.
- Tools: foot_slide.py (foot speed on the ground, not run yet), side_view.py (host screen frames of a crossing run).

### 2026-11-04 - sprint snap-back and stop glide fixed: copies play their own clips, natively
Owner: the glide is still there; while sprinting the joining player's model on the host lags a bit, goes back a bit
and then moves forward.
- The drawn body, not the entity: a Lua timer recording the pelvis bone in world space (pelvis_track.py) showed the
  copy's body running up to 3.9 m ahead of its own entity at 12-16 m/s and jumping back ~2 m every 0.6 s, while
  the entity moved smoothly; on the player's own screen the pelvis stays within 0.35 m. The cause was the clip
  relay: _SPRINT_NW and the stop clip started on the copy with Lua StartAnimation keep their root motion in the
  pose while the network moves the entity too; each loop restart snapped the body back. Clips are no longer
  relayed (own_state_relay.RELAY_CLIPS): the copy's own sprint state plays _SPRINT_NW with its motion extracted.
- The stop, natively: a requested Sprint.Stop.Normal played no clip on the copy (or Dodge_roll_normal). Two causes:
  a context tag (actor +0x10F0, 8 x {u16 tag, u8 count}) left on the copy by a state whose end never ran there
  (0x84; it selected the roll), and the stop clip comes with the transition, not the state. Now the player's machine
  sends its tags with its state every 0.5 s and the host sets the copy's once it is in that state; and
  copy_moves_block lets the copy take a relayed stop itself for 0.15 s (allow cell) before it is requested.
- After the stop the copy went to its weapon idle (held away from Observer): held, it can no longer pick
  Combat.Idle.* / Combat.BaseLogic.*, and it is sent back to Observer when its player goes there.
- Sprint start: started 90 ms late, the copy jogged on for ~0.25 s, fell 1.0-1.4 m behind and rushed at 10-12 m/s
  (its net input asks for distance to target / 0.2 s, up to 12); a relayed sprint start now plays at once.
- Verified both ways (host copy of the joining player, joining machine's copy of the host player): same clips in
  the same order 0.1-0.15 s later (walk, sprint, Test_hero_sprint_stop_nw, relaxed_idle_nw); no backward pelvis
  steps; pelvis-to-entity offset like the player's own (-0.35..+0.19 m); stop overshoot 0.02-0.16 m, ends within
  0.02 m; after the stop clip starts the copy moves 2.1-2.4 m against its player's 2.6.
- The joining player's own stamina stopped regenerating twice after being knocked down by zombies (no stamina
  change calls for it at all). stamina_mirror.RegenGuard (joining agent): in Observer, below its maximum and
  unchanged for 1 s, it adds the normal rate (5.9/s) until the game's own regeneration moves it again.

### 2026-11-04 - why a sprinting copy lags, snaps and rushes (not fixed yet)
Owner: the tired bug is gone, the glide is still there; while the joining player sprints, its model on the host
lags a bit, goes back a bit and then moves forward.
- How a copy moves: the engine's update (SetState 0x169E0B0) copies the received state into the copy's net input
  (+0xC velocity, as a fraction of pl_netSerialiseMaxSpeed 12; +0x3C position); every frame 0x169E9C0 aims at
  position + velocity * t (+0x58) and asks the copy's own locomotion for a move toward it, faster the further away.
- What arrives: updates about every 30 ms (both games at ~30 FPS on one PC), gaps up to 63 ms, positions 0.12-0.48 m
  behind the player, unevenly (one update moves 0.11 m, the next 0.43 m). The copy's time lag (100 ms smoothed)
  swings between about 25 and 110 ms during a steady run, and it catches up at up to 16 m/s (sprint_jerk.py,
  lag_swing.py). No backward position steps were measured.
- Tried: `pose_relay.py` streams the player's own position and velocity every frame over the control channel; the
  copy's machine replays them 50 ms late through a buffer every 4 ms, and a hook after SetState's copy keeps the
  engine's own updates from moving the target. Time-lag swing with it 68-95 ms against 53-205 ms without: no clear
  gain, so it is not started by the agents. The copy's own locomotion (speed set from the distance to the target)
  seems to be the limit; next is driving the copy's position directly while keeping its gait speed.
- Seen once: after a zombie grab, with its health low, the joining player's own stamina stopped regenerating (no
  stamina change calls for it at all); not reproduced in a fresh session (regenerates from 50 and from 0.5).

### 2026-11-04 - stamina mirrored both ways; copies' states played as late as their positions (glide)
Owner: both bugs still there on both screens; Ed is still tired, the glide is visible for both characters.
- Stamina is a field of the player actor (current +0x1678, maximum +0x1680), stored by its SetStamina 0x1462020.
  The host's copy of the joining player was never set: 0 of 100, so its walk graph played exhausted_idle after every
  stop (its player: relaxed_idle_nw). The joining machine's copy of the host player ran its own stamina simulation
  with its own maximum (83 while the host player had 63). `stamina_mirror.py`: each agent sends its own player's
  stamina every 50 ms ("stam"); the receiver writes it into the copy, and a hook at the start of SetStamina replaces
  the value for copies, so the machine's own simulation does not move it. Verified both ways: within about 1 point
  through sprint and rest, relaxed_idle on the copy when its player plays it.
- Glide: after its stop clip started, the copy's body moved 0.2-0.6 m further than its player's (2.8-3.2 m against
  2.6 m; stop_slide.py): the clip came about 30 ms after its player's but the position trails about 90 ms, so the
  copy played the stop while still sliding up. own_state_relay now plays relayed states and clips DELAY_S (0.09 s)
  after arrival, and runs both ways (the host agent streams the host player's states; the joining agent applies them
  to its copy of the host player; copy_moves_block on any machine covers every player but its own). After: the
  copy moves 1.8-2.5 m after its stop clip against its player's 2.0-2.75 m, in both directions; states follow one
  to one, 110-130 ms behind.
- Left: the joining machine's copy of the host player once took a landing (Jump.Forward.Land.Running) of its own.
  With three or more players, a joining machine's copies of the other joining players get neither relay.

### 2026-11-04 - the copy's tired look came from its own locomotion state, not stamina
Owner: the stamina bug is still there, and the glide; try new approaches (networking).
- The joining machine has no stamina object at the local-player global (0x1EE2638 reads 0), and the host
  player's stamina stayed at 8500 through four sprints of the joining player: the copy's tired clips were not
  stamina. Its states showed why: the relay put it into Sprint.RunForward and 30-40 ms later its own picker took
  it back to GraphDriven.Observer, where it ran the whole sprint on the walk graph (_WALK_INJURED during the
  sprint, TestUnshared_exhausted_idle_enter / exhausted_idle_nw after it).
- `copy_moves_block.py`: a hold cell per copy slot; while set, the copy also takes no transition to
  GraphDriven.Observer or MachineDriven.ObserverLogic.*. own_state_relay sets it when its player enters a mirrored
  state and clears it (then requests Observer) when the player leaves them.
- Verified (four sprints, two with stops, several landings): the copy's states follow its player's one to one,
  10-40 ms behind, no tired clips at all.
- Networking: the control channel now has TCP_NODELAY. A prototype fed the joining player's own position and
  velocity into the copy's net input every 5 ms (pose_proto.py). With the look-ahead cap at 0.15 it stops as
  cleanly as the engine's own data (overshoot 0.00-0.07 m against 0.01-0.12 m) but trails more (170-180 ms against
  80-130 ms), so it is not used.

### 2026-11-04 - the copy's sprint no longer twitches; no false tiredness; its own movement chosen by its player
Owner: the glide is still there; after sprinting the joining player's model on the host bends over as if tired,
while on its own screen it is not (its stamina had not run down that far).
- With the sprint requested on it, the copy's own state machine left it for Sprint.Stop.Normal / HardTurn every
  20-40 ms and was pulled back each time (a twitch through stop and turn clips for the whole sprint), and once rolled
  (Dodge_roll_normal) at a stop its player did not dodge at. `copy_moves_block.py` (host): the transition picker
  reports no transition when a copy's record would go to GraphDriven.Sprint.RunForward, MachineDriven.Sprint.* or
  MachineDriven.Dodge.*; those come only from its player (own_state_relay requests, also Dodge.* now). The hold loop
  is gone. A stop's clip comes with the transition, so the joining agent also sends its own player's clip starts in
  those states ("cclip") and the host starts them on the copy.
- Tired: the stamina actions (StaminaPenaltyMode 0x84, StaminaTankModify 0x85, StaminaTankRateMode 0x86) act on
  the local player's stamina (global 0x1EE2638): the copy's sprint drained the host player's stamina (and the
  joining machine's copy of the host player drained the joining player's), and the copy then played
  exhausted_idle / _WALK_INJURED. remote_fx now wraps them for remote players' records like the camera actions.
- Verified: the copy sprints without a single stop or turn in between, plays Test_hero_sprint_stop_nw 0.01-0.02 s
  after its player, no exhausted clips in the 8 s after four sprints. Position overshoot at the stop 0.04-0.23 m.

### 2026-11-04 - the joining player's sprint, sprint stop and landings play on its host copy
Owner: the glide did not feel different after the look-ahead cap.
- What showed as gliding: at a sprint stop the joining player's own machine plays Sprint.Stop.Normal (clip
  Test_hero_sprint_stop_nw); its host copy stayed in Observer the whole time (never in the sprint state, so no
  _SPRINT_NW either: the "slow mo" of a sprint), kept its running pose while the interpolation slowed it down and
  snapped to idle 0.77 s later.
- `own_state_relay.py`: the joining agent polls its own player's state (record +0xC, every 5 ms) and sends each
  entry into GraphDriven.Sprint.RunForward / MachineDriven.Sprint.* / MachineDriven.Jump.* (and leaving them) as an
  unsolicited "cstate" message on the control channel; falls and ObserverLogic are neutral (its little drops when
  running downhill). The host agent's channel now has a reader thread (Channel.start_reader: streams to handlers,
  replies to recv) and requests the state on its copy (pending +0xE, flag +0x18: entry actions run, so the gait
  and clips do; the state tables are the same on both machines). The copy's own state machine left a requested
  sprint for a stop or turn of its own: the sprint is held (re-requested every 20 ms when the copy drifts).
- Verified (state log): the copy enters the sprint 0.03-0.04 s after the player with _SPRINT_NW, keeps it through
  the player's falls, lands with it, enters Sprint.Stop.Normal 0.01 s after the player and returns to Observer
  with it. Position overshoot at the stop is still 0.06-0.12 m (the copy now plays the stop animation over it).

### 2026-11-04 - remote copies' look-ahead capped (less glide-back, feet keep their speed)
- CNetPlayerInput's interpolation (0x169E9C0) aims a remote copy at the last received position plus the received
  velocity times t, t = time since that update + 0.2 s (constant at 0x1978238), clamped at 1 s (0x169EB24..42):
  even fresh data is projected 0.2 s ahead, so a sprinting player's copy ran ~0.2 m past the stop and slid back.
- `interp_cap.py` (both machines, coop_setup) caps t (minss with a cell; the t > 1 branch takes the cap too).
  Sprint stop, per cap: 0 s -> overshoot 0.01-0.02 m, trailing 1.5 m; 0.05 s -> 0.01-0.04 m, 1.2 m; 0.1 s ->
  0-0.08 m, 0.8 m; 0.15 s (kept) -> 0.05-0.12 m, 0.4-0.5 m (default: 0.18-0.26 m, 0.15-0.4 m). The copy keeps the
  received velocity, so its animation is not slowed (unlike pl_velocityInterpAlwaysSnap).

### 2026-11-04 - run-turn loop fixed; the snap setting reverted (slow-motion feet)
Owner: with the snap the joining player's model on the host moves "in slow mo a bit"; running one way it keeps
playing an animation as if trying to turn around, indefinitely.
- Turn loop: running downhill, the joining player's own machine goes Sprint.RunForward -> Fall -> Land about every
  0.7 s, and each time its host copy found a transition into AnimationTest.TurnTest.Right.ToRunRight (clip
  Test_hero_run_turnright), an animation test state single player never enters; uphill nothing. It does not
  depend on either camera (`turn_loop.py`: 6-7 turn clips per 2.5 s sprint at four host camera headings).
  `turn_test_block.py` (both machines, coop_setup): the transition picker drops AnimationTest.Test01-04 and
  TurnTest.* targets for records of the player state machine (chained in front of last_stand). After: 0-1 turn
  clips per sprint.
- Slow motion: `stride_test.py` (each machine records the left foot's forward offset every 20 ms, stride
  frequency from it; noisy when the player falls or lands): with pl_velocityInterpAlwaysSnap 1 the copy stepped at
  0.19-0.20 strides/s against the player's 1.6-2.2 (it moves at full speed, its animation gets no speed); with it
  off 1.14 against 0.98. The snap is off again, so the glide-back overshoot is back (0.18-0.25 m in a fresh
  session). pl_netSerialiseMaxSpeed 12 (set in NET_RATES and the test copies' system.cfg) did not change it in a
  fresh session.
- Next for the glide: position the copy from the player's own stream and give its animation the real speed,
  instead of the engine's interpolation.

### 2026-11-04 - a remote player no longer glides back after stopping
Owner: on the host, after the joining player stops, its model still glides a bit back.
- `glide_test.py` (sprint 2 s with Shift+W in the moving player's window, stop, sample both machines every 10 ms):
  the host copy trailed 0.39-0.77 m while sprinting, ran 0.18-0.26 m past the stopping point and slid back over
  ~1.1-1.3 s. CryEngine's networked-player input steers a remote copy towards each received position from the
  last velocity (velocity interpolation), which overshoots a stop.
- `pl_velocityInterpAlwaysSnap 1` (both machines, coop_setup.NET_RATES with the packet rates): the copy takes each
  received position. Joining player on the host: overshoot 0.00 m in 4/4 runs, trailing 0.22-0.35 m; the host's
  player on the joining machine: overshoot 0.00 m in 4/4, trailing 0.33-0.37 m. The copy still arrives ~0.85 s
  after the key is released, the time the moving player itself needs to decelerate.

### 2026-11-03 - nobody could search containers since 10-29: a broken crash guard
Owner: in a tent the host sees a highlighted suitcase, the joining player does not; neither gets a prompt, E does
nothing.
- The search candidate check (0x14569A0, from the interaction finder 0x1464000) never ran: the player record's
  search slot ([rec+0x204 + slot*0x6C]+0x28, slot byte rec+0x1528, active bit 0 of rec+0x11A0+slot) stayed empty on
  both machines, because the query action RTSResourceNodeQueryMode (type 0x72, fire handler 0x170B340) returned
  nothing for any player.
- Cause: `crash_guards.rts_query_guard` (2026-10-29) wrote "xor eax,eax; pop edi; pop esi; ret" over 0x170B3A5..AA,
  but the handler's found path jumps to 0x170B3A7, into those bytes: every found record returned without writing
  its result. (The 10-29 entry already noted that no player, the host included, got a search context at the known
  containers any more.)
- Fix: the original bytes stay; the lookup's checks at 0x170B38A jump to a page that returns on a failed lookup and
  goes on to 0x170B3A7 otherwise. rts_query_guard also undoes the broken version in a running game.
- Verified live: the joining player at container 0x5BD: Search.Logic.ContextTest -> Ground.Start -> Loop, the
  manager completes after 7 s, Stop, back to Observer.
- Ruled out on the way (reverted and retested, no change): the two query guards, the purge-loop guard, last_stand,
  the effects relay hooks; the query pool had 497/505 of 512 free.

### 2026-11-03 - being grabbed by a zombie shows on the joining screen; another null-query crash guarded
Owner: the client crashed; when a zombie grabs the joining player, its model moves around uncontrollably.
- Crash (joining game, dump 21:11:30): access violation at 0x1522FE4 in the entity-db query result reader
  (0x1522FE0, cdecl (out id, out flag, query); bit 2 of query+0xD8 = results ready) with a null query, called from
  a UL script action (0x17317C1) on that machine. `crash_guards.query_read_guard` (both machines, at boot) sends a
  null query down the function's own "not ready" path (outputs 0, returns 0).
- Grabbed: the host copy goes into MachineDriven.ReactionGrab.ByBasicZombie (clips Grab_struggle_bite_back_start /
  _loop); nothing of that family was mirrored, so the joining player's own machine kept its own control and played
  a fall, a landing and an idle while the host pulled its position back (0.2-0.7 m apart). door_sync now mirrors
  ReactionGrab.* like a finisher (state, clip, the host copy's place and facing), and re-sends the state every
  0.15 s while the copy is held (its own state machine left a written grab state at once).
- Verified with a staged grab (`grab_force.py`: host copy requested into ByBasicZombie): its own machine entered
  it 0.17 s later, held it until the host copy left it and left 0.14 s after; the player did not move (0.00 m).
  A natural grab did not happen again in more than 10 minutes of tests.
- Test note: goal_check.stand_up (writes the idle state directly) leaves the joining player marked down for the
  zombies (they use their TargetDown strikes on it standing, never grab it); fresh sessions do not have that.

### 2026-11-03 - puppet zombies: exact pairing by name, corpses get bodies, corrections no longer fight the network
- Identity: a zombie's network copy on the joining machine has the same entity NAME as its host zombie (the entity
  ids are the same in one session and off by 1-2 in the next, so ids cannot be used). The host's zombie list now
  carries each zombie's name (and id); the joining machine pairs by name first (the nearer by lagged place when the
  game left two of a name), then falls back to position. Position matching had crossed two zombies 1 m apart and
  kept them crossed. Each entry also carries where the host zombie was 0.22 s ago (host position history sampled
  every 50 ms): that is where its network copy is now, and position fallbacks and corrections use it.
- Corrections: the glide compared against the host's current place, and every running zombie (1.2-1.5 m behind)
  was corrected about 30 times a second, fighting the engine's position updates (2 m errors in a crowd). Now it
  measures against and glides to the lagged place: 8 glides in a whole session.
- Verified in a crowd around the joining player (`twin_fit.py`: per zombie, the host zombie whose path it follows
  best): all 4 paired with their best fit, 0.01-0.02 m, including two 0.9 m apart.
- Corpses without a body: the game swaps a killed zombie for a new corpse entity (same index, new salt), which
  reached the joining machine without a character, so the body vanished. The host now lists the nearest corpses
  (25 m, up to 6) with loop "*"; a corpse entity the joining machine never dressed is matched to a corpse entry
  (corpses only with corpses) and gets its body, parts and death clip. `corpse_bodies.py` after kills: 2 of 2
  corpses with bodies, lying (pelvis to foot 0.67-0.69 m), a head-shot one without its head; 8 revived in place,
  3 reloaded.
- `engage_test.py` in a fresh session: PASS (11 strikes standing, 24 strikes and 31 Down.Hit reactions down,
  0 staggers). It failed once after long test runs in one session (zombies used their downed-target strikes on the
  standing player): test state, not reproduced fresh.
- Both games crashed once at 0x11F9775, a second after Windows woke from Modern Standby (Kernel-Power 506 at
  20:15, 507 at 20:38): test sessions now run with `keep_awake.py`.

### 2026-11-03 - puppet zombies: collapsed corpses revived in place (no reload, no blink)
- The joining machine's corpse fix reloaded the character when the local death ragdoll collapsed, and its parts
  then went back on behind a 0.25 s hide: a blink per kill. The ragdoll comes from native code (wrapping the
  zombie classes' script TurnRagdoll / RagDollize / OnDeath / Kill and the actor binding's RagDollize: none called;
  g_hitDeathReactions_disableRagdoll 1: still collapsed); physics profiles ("alive", "unragdoll", "animated",
  "frozen", "none", "sleep") left the bones collapsed.
- The actor binding's Revive ends the ragdoll: the skeleton is animated again, health stays 0, all five parts
  stay bound. The corpse check now calls Revive and holds the down idle's first frame (lying face down); the reload
  is kept as a fallback when the corpse is still collapsed a second later. Verified on a collapsed corpse (head
  2 cm from its origin): revived, head 0.65 m out, lying, parts Legs/Skin/body/hair/head, no reload.
- Exact identity (host zombie = this zombie, without position matching): the network's entity-to-net-id table
  (CNetContext::m_mNetIDs) has no code reference in this build (its logging is compiled out); matching stays by
  position with stickiness (0-2 model reloads in 30 s, from recycled entities). SinglePlayer.coopNoSettle = true
  pauses the corpse check (tests).

### 2026-11-03 - puppet zombies: host-driven gait sooner, glides instead of jumps
Owner's plan, step 3: the joining machine's zombies as pure puppets of the host's. What they already are: network
entities whose positions the engine keeps 0.2-0.25 s behind the host (mean error 0.06-0.26 m in
`zombie_lag.py`), no state machine of their own (records inactive), clips, outfit, corpse pose and effects from the
host, sticky matching (2 reloads in 30 s, from the game recycling entities), corpses settled when the local ragdoll
collapses, and no last stand for the joining player. Changed now:
- Gait: only "important" clips (hits, grabs, deaths, climbs, strikes) went out at once; walk/run/idle changes
  waited for the regular list (up to 0.5 s of the wrong gait). zombie_sync counts every clip (State.any) and
  hud_relay's fast path sends a list within 0.15 s of any new clip (ZOMBIES_ANY_S), important ones within 0.05 s.
- Corrections: a matched zombie more than 1.5 m from its host zombie was put there in one jump; it now glides
  there in 5 steps of 50 ms (a paired grab or finisher start still takes the exact place at once). 2 glides in
  20 s near the joining player.
- Checked: `engage_test.py` PASS (standing: 14 strikes and bites, 0 staggers; down: 132 strikes, 65 Down.Hit
  reactions), 6 swings with prediction, no errors, no dumps.
- Not changed: matching stays by position (the entity ids are allocated per machine; no network-id mapping
  found yet), and the corpse fix still reloads the character (a 0.25 s blink while its parts go back on), since
  the death ragdoll's trigger sits in packed game scripts that could not be read.

### 2026-11-03 - instant local hits: the joining player's swings land on its own screen at once
Owner: for the joining player, attacks come late.
- Its machine plays its own swing locally (clip logs on both machines: own swing clip 0.06-0.27 s after the click,
  the host copy's at the same time or later), but the hit is decided on the host, and the zombie's reaction came
  back with the zombie list: 0.8 s and more after the click on the joining screen, against about 0.5 s on the host.
  Its own state machine finds no targets (CombatDamage never fires there: its zombies are not in the combat
  system), so there is no local hit to hook.
- `hit_predict.py` (joining agent): zombie_sync's passive clip hooks also run on the joining machine; when its own
  player starts an attack swing, 0.30 s later the zombie handler's coopPredictHit picks the nearest live shown
  zombie within 2 m in front, starts a light reaction on it and marks it; the effects runner plays a flesh splat
  there (a copy of the host's zedflesh_splat block) and drops the host's own splat for that hit. For 0.7 s the
  zombie handler starts none of the host's clips on that zombie but knock-downs, deaths, grabs and finishers (the
  host's own reaction, or the walk it had started before the hit reached it, cut the shown reaction short).
- Verified (one zombie at a time, others parked): reactions on the joining screen 0.42-0.46 s after the click (host
  0.5-0.6 s), on the zombie in front, played out; then the host's clips again. No errors, no dumps.
- A swing that misses on the host still shows a short reaction there.

### 2026-11-03 - effects relay: blood, impacts and head explosions on the joining screens
Owner: on the joining machine blood splashes sometimes do not happen and heads do not explode on the final blow.
- Only the host runs the zombies' state machines and the combat code, so their effects existed on its screen only.
  In a 40 s fight the host ran CMaterialEffects::ExecuteEffect (0x6DA190, thiscall (uint16 id,
  SMFXRunTimeEffectParams*)) 255 times (zombie claw hits c3fxlib_combat/zed_clawattack_body, zedflesh_splat, body
  falls zed_bf_dirt_*), the joining machine 0-16 times. A finisher also runs the UL actions Dismember (core
  0x138FAA0 (out, entity id, part mask, a, b); mask 4 = head: the head and Skin attachments come off) and
  FxSpawnParticleEffect (fire handler 0x170A8C0 (context, params 0x7C, effect name inline at +0x19)).
- The effect tables are identical on both machines (783 effects, same ids), and the parameter block (0xB8 bytes,
  constructor 0x6D72F0) holds only values and pointers into the exe image, so a copy can be replayed.
- `effects_relay.py`: host detours copy effects, dismemberments and particle actions into rings; a host-agent
  thread sends what is new every 30 ms as an "fx" message on each joining player's control channel (no reply; the
  channel's send now takes a lock). The joining agent maps entity ids (zombies: cvar coop_zmap "host id=own id",
  written by the zombie handler from the dress keys; players by name) and runs each batch in one native call:
  ExecuteEffect on its own effects manager, the dismember core, the particle handler with a stand-in context whose
  record points at its own UL actor (0x1462350). Parts a dismemberment took off are kept off by zombie_dress
  (CUT), which otherwise dresses whatever is missing.
- Verified: 4 swings by the joining player: 130 effects, 5 dismemberments, 1 particle effect played on the
  joining machine; a head-shot zombie there lost head and Skin exactly like its host zombie; no errors, no dumps.

### 2026-11-03 - joining player's camera no longer stays zoomed in after getting up
Owner: after being downed and getting up, the joining player's camera stays zoomed in on the player; Tab or Shift
zooms it back out, but it comes back.
- The joining player's own state machine, 0.1 s after its mirrored Down.Core.Idle, found its own transition to
  Reaction.BlazeOfGlory.Start.From_Stunned (last stand) true and went there, while its host copy stayed down.
  Last stand's CameraMode action (handlers: begin 0x170CF90 pushes a camera mode on [0x1634090()]+0x28 and keeps
  the handle in the action's data, end 0x1703B30 removes it) pushed a close camera (1.9 m, 45 degree view). The
  mirrored states written over it skip exit actions, so the end never ran: the close camera stayed on the stack
  for the rest of the session, under Tab's and Shift's own camera modes.
- The writer was found with a write watch on the record's state words: 0x1711450 is the transition picker (the
  first transition whose condition holds becomes the pending state +0xE, flag +0x18). `last_stand.py`
  (coop_setup.client_main, joining machines only) drops a last-stand target (the BlazeOfGlory states, one
  contiguous index range) for that machine's own player and reports no transition.
- Verified in a fresh session, two down / get-up cycles: before 3.8 m and 45 degrees, down 5.9 m (the normal
  downed view), 1.7 s after standing back at 3.8 m and 45 degrees; screenshots before and after match. Its own
  machine stays in Stunned.Basic.Down.Core.Idle while down.

### 2026-11-03 - finisher crash fixed; corpses no longer collapse on the joining machine; dress list read fixed
Owner: zombies still deform on death, and the game crashed after a finisher.
- Crash (joining game, dump 14:47:05): access violation at 0x1522EE7 in the entity-db query free (0x1522EB0,
  UL\EntityDb\EntityDb.cpp; also the destructor of a UL script's query variables, thunk 0x17308E0), reached from a
  state frame's teardown (0x17066A0 loop). The handle was garbage (0xBCCB5F44): the finisher mirrored onto the
  joining player's state machine left an uninitialised query variable in a frame. `crash_guards.query_free_guard`
  (at boot) ignores a handle that is not a slot of the pool (512 x 0x108 at 0x1EC00B8; free bitmap 0x1EE10C0).
  The pool stayed healthy afterwards (host 442/512 free, client 505/512 after the tests).
- Deformed bodies: the joining machine turns its zombie corpses into ragdolls, and the ragdoll of a skeleton
  dressed from parts collapses: every bone falls onto the entity origin (head 2 cm from it, measured with
  GetBonePos), so the body is a crumpled ball or invisible. Clips no longer move it (physics drives the skeleton).
  Loading the character again makes it animated: the zombie handler checks each corpse at its clip's end and
  then every second for 5 s; a collapsed one is reloaded (zombie_dress puts its parts back on) and held in the
  first frame of ZombieReact_DownFace, lying face down. (SetAnimationTime 0.98 on a finisher clip left the body
  crouching mid-fall, so the end frame is not used.)
- Dressing silently failed: `zombie_dress._cvar_text` read the dress list with one 16000-byte read, and a read
  that crosses the end of the string's heap region fails as a whole (here 4000 bytes failed, 2000 worked), so the
  list read as empty and no zombie got (re)dressed. It now reads a page at a time up to the NUL.
- Verified: finisher kill by the joining player, then on its machine the corpse lies flat, dressed (5 parts
  bound), head 0.67 m from the origin; all shown corpses intact (2 ok, 0 collapsed; 7 reloads in the session);
  no new dumps on either machine.
- Owner's Steam stats file: `UserStats.cfg` in the Steam save folder was written at 09:00:54 today. Steam was not
  running after the 08:55 boot; a test copy launched by the agent at 09:00:29 started it (Steam up at 09:00:34),
  and the file was written on that boot. All later boots (Steam already running) left it alone, and the save
  (`Class3.0.ulsave`, 2026-10-28 22:42:12) is untouched. The agent now refuses to start a test copy while Steam is
  not running.
- Still flaky in tests: the joining player's finisher approach often aborts (Combat.BaseLogic.Idle), 1 in 3 to 1
  in 9 tries; the host's own finisher at the same spot works first time.

### 2026-11-03 - finishers line up on the joining machine; by-id zombie clips relayed; corpses keep their pose
Owner: Z finishers work but the player's and the zombie's animations play apart; killed zombies' bodies deform.
- Clips started by id were never relayed: CSkeletonAnim::StartAnimationById (0xAEFF10, thiscall (id, params);
  StartAnimation by name is vtable +0x24, by id +0x28 of the skeleton = character instance +0x108) starts the
  finisher (player Grab_finisher_golfswing, zombie Grab_finisher_golfswing_slot2...) and most zombie clips (wander,
  creep, idle, Environmental_window_* climbs). Our clip hook only saw the UL by-name helper 0x820BC0.
  `zombie_sync.install_byid` hooks it too (layer-0 starts into a second ring); `State.read_events` maps skeleton to
  entity and id to name (animation set [[skeleton+0x3BC]+0x90]+0x108, vtable +0x18 GetNameByAnimID; cached) and
  merges them with the by-name clips. Only hits, grabs, finishers, downs, deaths, climbs and strikes trigger an
  immediate list (`IMPORTANT`), wander/idle clips go with the regular 0.5 s list.
- Finisher alignment: the host lines the player up with the downed zombie; the joining machine kept its own spot
  and its zombie copy, lying, got no position updates (2.8 m off). Now: player states (door_sync.PeerStates) also go
  out from the relay's fast thread (every 30 ms; the clip went with the round, up to ~0.3 s after the zombie's), a
  grab clip carries the host copy's position and facing (entity rotation quaternion +0x30) and the player's machine
  takes it; the zombie list carries each zombie's facing, and a matched zombie more than 1.5 m off, or one starting
  a paired grab clip, is put at the host's place. Measured: the player's machine enters the finisher 0.1 s after the
  host and stays in it for its length, the player 0.1-0.2 m from its host copy, the zombie copy 0.0 m from the host
  one; screenshot: the golf-swing finisher on the joining screen.
- Corpse poses: Lua StartAnimation cannot ask for "repeat last key" (the host's one-shots use flags 0x4084), so a
  corpse's last one-shot clip ended and its skeleton collapsed into a ~0.5 m ball (some floating). The zombie
  handler now holds a corpse's last clip at its end (SetAnimationSpeed 0 just before it ends; restarted at 0.98 if
  it already ended). Verified: a finisher kill's corpse stays lying (0.84 x 1.50 m, top 0.56 m above ground) for
  3.5 s and on.
- Crash guard: the joining game once died in a UL purge loop (0x146A662) that walked a stale blackboard whose list
  count (+0x1300) was a smeared 0x028F8D8B, while the player was down and being hit. `crash_guards.
  blackboard_list_guard` (at boot) treats a count above 64 as an empty list.
- Checks after these changes (`engage_test.py`, `slide_watch.py` 120 s): no staggers, zombies strike the player
  standing and down, no client zombie more than 0.7 m from its host one, no new dumps.
- Not done: the approach walk before the finisher does not show on the joining machine (only the finisher).

### 2026-11-03 - corpses stay on the joining machine, Z finishers show there; hit timing re-measured
Owner: a zombie hit by the joining player may slide backwards endlessly; downed zombies fall through the ground;
the joining player can't finish a downed zombie with Z.
- Corpses: a zombie's UL record goes inactive at death, so the host relay stopped listing it and the joining
  machine hid it two lists later; on the host it lies there (most corpses also leave the record pool, so the host
  cannot list them by record). Health is synced, so the joining machine's zombie handler now treats a zombie with
  health below 1 as a corpse: shown where the network puts it, in the pose of its last clip
  (ZombieReact_DownFace...), and out of the matching. Verified in a session with 7 corpses: all shown on both
  machines, world boxes 0.4-0.6 m high at ground level (host the same), one half-lying (1.4 m).
- Sliding: not reproduced (a physics push moves a client zombie 0.4 m and it stops; 60 s fights with the joining
  player swinging: no client zombie more than 0.7 m from a host zombie). Likely cause, now gone: dead client
  zombies stayed in the matching and could take a live host zombie's entry, i.e. a lying body playing a run loop.
- Z finisher: on the host it works for the joining player (its host copy runs Grab.BluntLight.Ground.GolfSwing.
  Approach -> Finisher.FaceFromHead, the zombie ReactionGrab.ByHero and dies; `local/finisher_test.py` puts a
  zombie into Reaction.Down.Front.Idle.Normal and presses Z), but its own machine showed nothing and kept its
  position, so the approach was undone and the finisher sometimes aborted. `door_sync.PeerStates` now mirrors the
  player's "MachineDriven.Grab." states and clips to its own machine as well: 2/2 finishers, its machine plays
  Grab...Finisher.FaceFromHead with the host.
- Hit timing, corrected: `hit_latency.py` read the joining machine's clock before focusing its window, which
  added 0.3 s or more to every "joining screen" figure (the 0.36 / 0.53 s in the entry below). With the clock read
  right before the click: its host copy starts the swing 0.05-0.1 s after the click, the zombie reacts on the host
  0.4-0.5 s after the click (the swing reaching it), and on the joining screen ~0.07 s after that.
- The host's start time (COOP_HOUR) did not always hold: right after the load the save's time was applied after
  ours (read back 16.94 instead of 12, so tests ran into the evening). The agent now sets it again until it holds.
- Seen, not fixed: a zombie climbing through a window on the host is ~1.3 m lower on the joining machine.

### 2026-11-03 - a downed joining player reacts to hits on its own screen; dressing no longer races the renderer
- Owner: when the joining player is down and zombies hit him, he lies still on his own screen. His host copy plays
  Reaction.Stunned.Basic.Down.Hit.Front/Left/Right/Back every 0.3-0.6 s (clips React_stunned_hit_*), his own
  machine stayed in Down.Core.Idle. `door_sync.PeerStates` now mirrors the whole "Reaction.Stunned.Basic.Down."
  family to the player's own machine with its clips (as for door and Reaction.Core states); the down idle is
  mirrored too, else the return from a hit would be sent as "idle" and stand him up there. Getting up
  (Reaction.Stunned.ToStand) still runs on his own machine from his input. Verified (`local/down_hits_test.py`):
  20 s down, 28 Down.Hit on the host copy, 26 on his machine; screenshot: recoiling on the ground; after mashing
  Space both copies are up and stay up.
- Owner: the joining game crashed after a few Win+Tab. Dump (client-install/USER): access violation reading
  [0+8] at 0xB55085 (attachment code) in the render thread: the zombie dressing (zombie_dress.py) removed and
  added attachments of zombies that were being drawn. Now a zombie is hidden while it waits for its parts (the
  handler hides it when it loads the bare skeleton; the dressing loop hides any zombie whose parts are missing)
  and its attachments change only after it has been hidden for 0.25 s; then it is shown (after 2 s it is shown
  anyway). Verified: new zombies dressed while the joining window was minimized and restored 20 times: no new
  dump, both games responding, 9/9 visible zombies dressed like the host.

### 2026-11-03 - zombies on the joining machine: no more "teleports", they fight a downed player, hits show sooner
Owner's report: zombies lag and teleport now and then; a downed joining player is not hit, zombies just stagger
around; hitting zombies shows a visible latency.
- Positions are fine: the engine's own entity sync keeps a moving client zombie 0.1-0.25 s behind the host one
  with a few cm of error after that shift (`local/zombie_track.py`, `zombie_lag.py`); 50+ m jumps happen on the
  host too (the game relocates zombies).
- "Teleports": the joining machine's zombie handler re-matched every client zombie to the nearest host entry on
  every list, so zombies in a crowd swapped identities: 283 model reloads in 180 s around the joining player
  (each a bind-pose flash and a restarted animation) plus 20 hides/shows (`local/zombie_pops.py`). Now a client
  zombie keeps the host zombie it had ("@key" models are unique) while that one is within 6 m, the others match
  the nearest within 4 m: 0 reloads in 60-180 s. The game also recycles zombie entities (same id, new outfit): the
  host reads each dress again every 5 s and the dress key carries a checksum, and the joining machine removes an
  attachment of the same name before attaching the new part (vtable +0x14 RemoveAttachmentByName of the
  attachment manager); its dressing is one native call per zombie (separate calls per part let a reloaded model
  free the character in between: "Pure function call" crash in the attachment code).
- Staggering: `zombie_targets` added a "Hero" contact event to every zombie within 1.4 m of the joining player on
  every frame (in the game a contact is one event per touch); those zombies looped Reaction.Logic.BasicReaction.
  Contact events are off now (`zombie_targets.CONTACT = 0`); the strike damage part stays. Fresh sessions, host's
  player 20 m away (`local/engage_test.py`): with contacts 5 staggers / 5 strikes in 30 s, health 100 -> 92, not
  knocked down in 40 s; without: 0 staggers, 10-11 strikes and bites, 100 -> 27, down at once, then 29-84 strikes
  at the downed player and 37-60 Down.Hit reactions in 30 s (before: zombies around a downed joining player
  howled and walked in place). Zombies blocked by the base's walls also "walk in place" (not a co-op issue).
- Hit latency: the joining player's swing starts on its host copy 50-100 ms after the click. The zombie list then
  reached the joining machine's handler after 66 ms median / 277 ms 90th percentile / 618 ms max
  (`local/relay_latency.py`, via a "#n" message number and the joining machine's cvar coop_zseq), and the relay's
  fast path only ran between rounds (80 ms average, up to ~0.35 s). Now: packet rates 60 on both ends
  (`coop_setup.net_rates`: sv_packetRate, cl_packetRate, net_packetsendrate; before 30/30 and 20/10), the zombie
  fast path in its own thread (checks every 10 ms, sends within 0.05 s of a new clip, ~12 ms per send; a lock
  around the zombie clip log, mailbox calls stay serialized by sodlib): 43-50 ms median / ~125-130 ms 90th
  percentile. One measured hit: host reaction -> joining screen 0.53 s before, 0.36 s with the rates only (before
  the thread). A 3-minute fight with the thread: no crash, no fast-path errors.
- Not done: playing the hit reaction on the joining machine before the host confirms the hit (prediction).
  Tools: `zombie_track.py`, `zombie_lag.py`, `zombie_pops.py`, `down_watch.py`, `down_test.py`, `engage_test.py`,
  `relay_latency.py` (needs the relay started with COOP_RELAY_TIMING=1), `hit_latency.py` (aiming the joining
  player's camera at a zombie is unreliable in it).

### 2026-11-03 - zombies on joining machines wear the host's detailed parts (no more low-detail bodies)
- Before: each zombie on a joining machine got one whole low-detail body per gender (`hud_relay.ZOMBIE_BODIES`,
  `.../low/low.cdf`). The host builds every zombie from the bare zombie skeleton plus five skin attachments chosen
  per zombie: body, head, hair, Skin, Legs (e.g. `body/tshirt/tshirt.chr`, `heads/head01/head01.chr`; "fresh"
  zombies use human clothing parts).
- Where the parts are (host): character instance (entity GET_CHARACTER 0xBC66F0) +0x60 is its attachment manager
  (vtable +0x18 GetInterfaceByName, +0xC CreateAttachment), the attachment array at +0x68 (count at [array-4] &
  0x7FFFFFFF); per attachment +0xA0 name, +0xB4 type (2 = skin), +0xA4 bound object, whose +4 is the skin's
  character instance (file at +0xB8). The game dresses zombies with 0x1391600 (thiscall on the character
  instance: name, .chr path, 1): CreateAttachment, load the skin through the character manager, bind it, own
  material copy. Lua `CreateSkinAttachment(slot, name)` only creates an empty attachment (a path argument is
  ignored).
- Fix (`local/zombie_dress.py`): the host's zombie relay (`zombie_sync.relay_message`) reads each zombie's parts
  and sends "=key:g:name~part+..." once per zombie and every 5 s (key = host entity id; part = path without
  `characters/<gender>/zombie/` and `.chr`); the zombie's entry refers to it as "@key". The joining machine's zombie
  handler (`zombie_sync.CLIENT_LUA`, now installed at runtime by the joining agent over the one in
  `singleplayer.lua`) loads the bare skeleton on the matched zombie and lists its parts in a local cvar
  `coop_zdress`; a loop in the joining player's agent (`zombie_dress.run_client`) calls 0x1391600 for each part of
  each listed zombie whose bound parts differ. Special zombies not built from the skeleton keep their own model.
- Pitfalls found: keys by entity id & 0xFFFF collided (the game reuses entity slots with a different high part), so
  a new zombie overwrote an old dress; a reloaded character can sit at the old instance's address, so the client
  loop checks the bound parts instead of remembering what it did.
- Verified (`local/zombie_dress_test.py`, fresh session, zombies spawned near the players): every visible zombie on
  the joining machine has the same skeleton and the same five part files as the host zombie it was dressed from
  (3/3, again 20 s later); screenshot: a detailed, animated zombie attacking Ed.

### 2026-11-02 - journal model, radio lock and quitting fixed
- Journal (Tab) on a joining machine: the player raised its weapon instead of the journal book. Opening the
  journal runs UL action SelectItem "Journal" (type 0x7A, fire handler 0x1703960; params = the item class name)
  and closing it SelectItem "Fists" (the host's own player then gets his weapon back from his UL character).
  `weapon_relay.guard_selects` drops UL's select actions for players with no UL character (else they lose their
  weapon), so the journal was never selected. Now, for the machine's own player, "Journal" goes through, and the
  next SelectItem after it selects the player's weapon again by name (0x13E7F20, actor thiscall (class, keep
  history)) instead of the Fists. The client Lua weapon report skips "Journal".
- The other machines showed the reading player idle: the journal states run only on the reading player's own
  machine (the host's state machine leaves a journal state written into its copy at once). The selected item does
  replicate, so `local/journal_sync.py` (a Lua timer on every machine, installed by `coop_setup`) plays the
  journal clip `Item_journal_idle` on animation layer 1 of every other player holding the Journal and stops it when
  the item changes. Verified both ways (`local/journal_test.py`: each player's item on both machines, the clip on
  the other one, weapon back after closing; screenshots).
- Radio Assistance on joining machines: it is locked by the story through UL value Game.RTS.NoRadio (0x4020018;
  the open function 0x15FC580 returns at once while it is set). The host has it from the save, a joining game loads
  none, so it was 0 there and the menu opened (also by itself). `local/rts_flags.py`: the host's agent sends the
  story locks (NoRadio, NoOutposts, NoJournal Expertise/Home/Challenge/People) with its time of day, on join and
  every 30 s, and the joining agent sets them (getter 0x13F4630, setter 0x13F4A10). Verified: before, the menu
  opened on the joining machine; after joining, NoRadio/NoOutposts arrived and it stays closed.
- Quitting: after a session, Exit Game in the main menu left the window "not responding". Two causes:
  - joining machines: `prep_client.prep` (run by `coop_setup`) installed `Proc.trap_exits`, a diagnostic that
    makes ExitProcess spin forever. Now opt-in (`exit_trap=True`); the game closes in about 1.4 s.
  - the host: after hosting, its main thread stays in the engine's teardown after ISystem::Quit (keeps drawing
    frames, never reaches ExitProcess); not found inside. The agent now watches the quit flag (CSystem +0x388)
    and ends the process when it is still there 8 s after the player chose Exit (the player is back at the main
    menu by then, nothing is lost). The agents and `hud_relay.py` now stop when their game ends.
  Verified (`local/exit_test.py`, in-game Exit Game -> Yes -> main menu Exit Game): joining copy closed after
  1.4 s, host after 8.4 s, no agent or relay left running.
- Tools: `tab_probe.py` (a player's state names on every machine during a key press), `action_probe.py` with
  target `*` (all records), `radio_test.py`, `park_zombies.py`, `thread_eips.py` (busy threads and their EIPs).

### 2026-11-02 - rejoin suite rerun: rejoining works, searching on the host does not start
- `rejoin_suite.sh` (fresh 4-player New Story, `COOP_HOUR=12`): Ed left and came back with his agent run from
  the second-PC bundle and got his old slot and character back (ed_jones.cdf), and the host's time right after
  joining. The 4th player left and came back into its freed slot. All four stayed alive through the soak.
- Simultaneous loot: FAIL, but not the wrong-panel race. No search starts at all: in `loot_two.py` neither
  search completed, and in the soak 5 of 5 searches were "incomplete". The host's copy of a joining player
  standing at node 0x5BD stays in `GraphDriven.Observer` with interaction kind 0 and no focused node (its
  manager +0x40 = -1), even with nobody else near and the zombies parked. Ed's own screen shows "Search (Hold)"
  there. Use only gave a push kick.
- On the host the interaction finder 0x1464000 runs for all four players (64 calls each in 2 s), but its node
  candidate check 0x14569A0 (called only from 0x14646BA) never runs. The gate before it needs a blackboard slot
  index at +0x1528 whose flag at +0x11A0 + slot has bit 0 set. All players had slot 31 with the flag 0, and Ed's
  own machine shows the same, so this gate may not be the search path. Open. Same symptom as the 10-29 note (no
  search context at these containers after that day); it needs a fresh session with the host's player at a
  fresh container to tell a broken host search from a used-up spot.
- Second PC: its share still returns "Permission denied" from this PC, so nothing new was copied or tested there.

### 2026-11-02 - time of day: the host's clock runs again, players follow it, daytime on request
- Why it was always night on the host: as a listen server the host runs CryAction's server time-of-day rule
  (0x6E9FD0) every frame. With `sv_timeofdayenable 0` (the default) it still calls the engine's
  `ITimeOfDay::SetAdvancedInfo` (vtable +0x24, 0x4AA3F0), but with start/end taken from uninitialised stack and
  speed 0. The clock (time-of-day object +0x24; start/end/speed at +0x2C/+0x30/+0x34) was then clamped to a junk
  start (5.9e-39, i.e. midnight) every frame, whatever the save said. The single-player game never runs as a
  server, so this only shows in co-op. It also explains the earlier failed tries (`e_TimeOfDay`, `rts_tod`).
- Fix (`local/tod_sync.py`, called by `coop_agent.py` before the host loads): `sv_timeofdaylength 1.9608` and
  `sv_timeofdayenable 1`. The rule then passes start 0, end 24 and speed (1/150)/length = 0.0034 h/s, the game's
  own speed (a game day takes 1.96 real hours). It does not set the time itself, so the save's time is kept
  (verified: a save written at 12:14 loaded at 12:14).
- A joining player's game loads no save, so its clock started at midnight and ran on its own. The host's agent
  now sends its time over the agents' control channel when a player joins and every 30 s; the player's agent
  sets it when it is more than 3 game minutes off (`tod_sync.follow`). Verified: host 18.61 / Ed 18.61 and
  back to 12.11 / 12.11 within one interval after a change on the host.
- Setting the time: `ITimeOfDay::SetTime(float hour, bool force)` (vtable +0x14) through
  `tod_sync.set_hour`. Daytime for tests: `COOP_HOUR=12 local/play_menu.sh` (or `coop_agent.py --hour 12`),
  or during a session `python local/tod_sync.py host 12`. Verified on both screens (full daylight, shadows).
- Avoid `rts_tod`: its argument parsing writes a garbage time.

### 2026-11-02 - joining player's sprint falls (open), notes
- In a fresh session 4 of 6 sprints on the base's parking lot were cut by Environmental.Fall.* -> Land (the
  host's own player: 0 of 8). Not a condition flag: comparing the flags ([0x1F36D60 + id], snapshot after pass 1
  each frame via `cond_probe.py`) in the frames right before 9 falls with ~3000 steady-sprint frames showed no flag
  that differs (`fall_cond.py`), so the fall comes another way (an event?). The engine's own in-air timer stays 0
  at the fall (player+0xBD4) and the height does not drop. Open.
- In one long session (after many probes) the joining player's machine-driven states stopped ending
  (Sprint.Stop.Normal, Land.Normal held for minutes while the host copy was idle; the record was ticked but did
  not change). Requesting idle via pending state +0xE / flag +0x18 got it out once. Not seen after a restart.
- Tools: `sprint_check.py` (walk vs sprint distance and states), `cond_probe.py`, `fall_cond.py`,
  `sprint_door_cond.py`, `order_probe.py` (frame order of two functions), `action_probe.py` (action begin/end
  counts per player; named types only).

### 2026-11-02 - daytime for testing: not found yet
- `e_TimeOfDay` set from the console does not stick (UL's clock overwrites it); the host and a joining machine
  showed different times (18:09 vs 21:20), so the time of day is not synced either.
- `rts_tod <value>` is UL's "Set time of day" command: "0.5", "720", "1200" are taken as e_TimeOfDay as given
  (720 stayed, but the scene stayed night), "12", "12.0", "12:00", "12 0" left it at 5.9e-39; unit and range
  unknown. A float at 0x1DE1B60 that crept up ~0.0024/s (6.27 = 6:16?) was not the clock (writing 12.0 there
  changed nothing visible). Open; the host's clock in that session may be off after these tries (test save only).
- The Radio Assistance menu was open on the joining machine again (twice during these tests), still unexplained.

### 2026-11-02 - sprint after doors fixed
- Owner confirmed: sprint works after opening a door (locked door / break-in not tried yet).
- Cause: on the joining machine `door_sync`'s stub mirrors the players' door states from the host by writing the
  state index (entry/exit actions skipped). Going back to idle that way never ended the door state's modes: during
  one door use the joining player's state machine began ContextTagMode x2, StateMachineAnimationMode and
  AnimationControlsRotationMode and never ended them (`action_probe.py`, now limited to the named action types:
  swapping the unnamed entries 0xA3+ crashed the game), and its graph-driven Sprint never started again.
- Fix: the return to idle is now requested like a native transition (pending state +0xE, flag +0x18; applied by
  0x1633DB0 -> 0x1633B20 in the next tick, which runs the door state's exit actions and switches the
  "StateMachine" graph input off). Door states themselves are still written directly (their entry actions need the
  host's door context).
- Verified on one PC: fresh sprint 15.7 m; after opening a door 15.5 m (Sprint.RunForward); after closing it
  15.8 m. Before the fix: after any door the player never entered Sprint again (Shift held: 8-9 m, plain run).
  Locked-door use not re-tested after the fix. The separate intermittent Fall while sprinting up a small rise is
  still open.

### 2026-11-02 - zombie animations reach the joining machine sooner
- Zombie clips (hit reactions, falls) went to the joining machines at most every 0.5 s from a relay loop that
  sleeps 0.25 s between rounds, i.e. up to ~0.75 s late plus network (owner: "zombies react later to the hit").
  `hud_relay` now watches zombie_sync's clip counter every 30 ms between rounds and sends the zombie list as soon
  as a new clip started (at most every 0.1 s); the other relays keep their pace. The relay runs; not checked
  against real hits yet (no live zombie near the players in this save at the time).
- Dead zombies sinking into the ground on the joining machine: not investigated yet (needs a kill).

### 2026-11-02 - joining player's sprint: not fixed yet (the tick-order change is reverted)
- Owner: Ed's sprint works until he opens any door, then never again. Reproduced on one PC: after door use
  (and fights) the joining player never enters Sprint.RunForward; pressing Shift while already running stays
  GraphDriven.Observer (8 m in 2 s, plain run). In a fresh session it sprints (15-16 m in 2.5 s).
- Not the cause: the joining machine's game clock 0x1EE7F44 (0 there; kept running by hand, no change), a model
  swap (SetActorModel on the host's player: still 8/8 clean), the combat tags (removed: still no sprint), the
  actor's stamina (actor+0x1678 is 100 on both machines). The HUD's stamina arc on the joining machine is almost
  empty while the host's record for that player shows 110; the joining machine has no "current player record"
  ([0x1EE2638] = 0), so whatever its sprint test and HUD read is not kept up there. Transition mechanics found:
  conditions are flags in [0x1F36D60 + id] (0x16FF340), chosen by 0x1711450, applied by 0x1633DB0 -> 0x1633B20.
- Also seen: the Radio Assistance menu opening by itself on the joining machine (blocks its input); a crash of the
  host test copy from SetActorModel with an empty model path (my test).
- Correction to the entry below: moving the joining machine's state-machine passes after CPlayer::Update made the
  porch work but the joining player then fell mid-sprint on flat ground. Measured on the host (`order_probe.py`,
  frame-stamped hooks): its state-machine manager (world update 0x145E23E -> 0x1636A40) runs before
  CPlayer::Update in every frame (240/240), i.e. the original placement already matched; reverted.
- With either order the joining player's sprint is cut by spurious falls (about 2 in 5 sprints on the base's
  parking lot; `sprint_check.py`): Sprint.RunForward -> Environmental.Fall.FromRunning -> Land.Running while it
  steps up a small rise (height 229.375 -> 229.54), never with the host's own player on the same path (4/4 clean).
  The host runs four UL systems in its world update right before the state machines (0x1613D10, 0x1613530,
  0x1435140, 0x1611B00); a joining machine runs none of them (the world update does not run there at all). Next:
  find which one keeps the player's ground contact, or what the Fall transition reads (`fall_probe.py`).
- Menu driver / agent: on a DLC left selected in the title carousel the "has a save" test looked at the DLC's
  save, so Continue Story was missing and `menu_flow_test.py` started a new story (the game stopped at its
  "overwrite your save?" prompt). The agent now selects the main game before building the Host Game menu, and
  the menu driver picks Continue/New by name.

### 2026-11-02 - joining player's sprint at obstacles fixed; intermittent host start hang
- Sprint "falling" loop: pressed against an obstacle (porch next to Door_4), the joining player's height jitters
  by ~0.1 m and its own state machine flipped Fall.FromRunning <-> Land.Running every few frames, while its host
  copy and the host's own player at the same spot did not. On a joining machine `player_sm` ticks the player's
  state machine itself (the game only does it in the host's world tick), and it ran the three passes at the start
  of CPlayer::Update, i.e. on the previous frame's movement state. The passes now run after CPlayer::Update
  returns (the detour swaps the return address for a post-update block; input is still sent before). Verified on
  one PC: 3/3 sprints at that porch go Observer -> Sprint.RunForward (stopped by the obstacle like the host's
  player); crouch, door open and melee swings still match on both machines.
- Host start hung three times today and twice on 10-29 right after `sv_bind`, before "connection requested ...
  <local>:64087" and before the game port was bound; the main thread sat in a poll + Sleep(1) loop (0x1222C28).
  Not the save (a save that loaded on 10-29 hung too, the next start with it worked), not the VPN (owner: on all
  along), not the co-op code (nothing of it runs that early besides the boot guards). Intermittent; kept the hung
  saves (test-saves/host/*.stuck-*). Open.

### 2026-11-02 - sprint "falling" loop narrowed down; radio menu on the joining machine
- The joining player's sprint loop (Fall.FromRunning <-> Land.Running without falling, owner: "after a door")
  is tied to a spot, not to doors: at a porch next to Door_4 (~3173,3358, z 226.2) it loops before and after the
  door is opened and also with every door's collision removed on the joining machine; the host's own player
  sprinting from the same spot and facing does not loop. On flat ground the joining player sprints normally
  (22 m in 3 s, three runs). So the joining machine's ground check / physics differs from the host's at such
  spots; open.
- The Radio Assistance menu was found open twice on the joining machine (its input blocked: the player could not
  move) without anyone opening it there; not reproduced with the host's R, Q, Tab, T or F, nor with Alt/Shift on
  the joining machine. Open.

### 2026-11-02 - flashlight: each player's T switches only its own light
- UL_Flashlight's compiled script (string constants via string.dump): OnInit calls self:OnReset(), which places a
  projector light (LoadLight, flashlight1.tif) by g_localActor, sets the blackboard "Game.Player.LightOn" and plays
  flashlight_on/off, or frees the light slot. g_localActor is the machine's own player, so every replicated
  flashlight entity switched the local player's light: the host's T lit the joining player's too, and a joining
  player's T (its host copy spawns the entity on the host) switched the host's.
- New `flashlight_fix.py` (installed by coop_setup on the host and on every joining machine): UL_Flashlight.OnReset
  (looked up by name from OnInit; OnInit itself is cached by the engine, a wrapper on it never ran) runs only when
  the entity's nearest player is this machine's own. Verified on one PC: host T -> host's entity runs on the host,
  skipped on the joining machine; joining player's T -> skipped on the host, runs on the joining machine. Another
  player's light is not shown on the other machines yet.
- Owner checked it in the running session: each player's T toggles only its own light, a second press turns it off.

### 2026-11-02 - owner's local test: crouch confirmed; flashlight; log flood; failed starts
- Owner confirmed the joining player's crouch now shows on the host.
- Flashlight (T): the key is part of the relayed input (bytes 0xFB.. block: 0x103/0x104 change on the host copy's
  stand-in input). The joining player's host copy runs it through its state machine: UL action ThrowEntity
  (0x8B, fire 0x1709590) spawns a UL_Flashlight (call chain 0x1709772 -> 0x1707172 -> 0x1706B41 -> spawn at
  0x1465862) at that player, and at the same moment the host's own flashlight goes out (one player flashlight
  per machine?). On the joining machine its own T cannot spawn anything (the host spawns entities), so it gets no
  light; the host's light is replicated and shown at the host's player (near the joining player it looks like
  its own). Open: give each player its own light.
  More: the spawn goes through ThrowEntity's helper 0x1707160 and gets a physics impulse after it (0x1465868..),
  i.e. a UL_Flashlight entity is thrown/placed at the player; each T press adds one more that stays where it was
  spawned (20 at the base after a few presses) and does not follow the player, so the light on the character
  itself is something else switched along with it. UL_Flashlight's Lua class has only OnInit/OnReset/
  OnPropertyChange (compiled). Asked the owner to describe what T does on each screen in the running session.
  Owner's answer: the host's T toggles its light (follows it) and also the joining player's; the joining
  player's own T toggles its own light. Each T spawns one UL_Flashlight on the host, replicated to every machine
  (+1 on both); nothing runs in the joining machine's copy of the host's state machine meanwhile
  (`local/action_probe.py`), and a Lua wrapper on UL_Flashlight.OnInit is never called, so the light is switched
  natively when the entity appears, for that machine's own player. Fix needs that native code (type names are
  stripped). A first version of action_probe.py crashed the joining test copy (its stub overwrote eax, which some
  handlers take as an argument); fixed to keep every register.
- Joining machines' logs grew by ~40 MB per 15 min: "Entity %u is not part of the community" every frame (also on
  the host). `crash_guards.quiet_community_warning` NOPs the three CryWarning calls (0x17167AD, 0x171685B,
  0x17168EB); installed at boot. Verified: client log growth 0 B in 3 s afterwards.
- A host start failed (early save redirect timed out after 60 s; the game was gone). `save_redirect.install_early`
  now ends the game process on a timeout (it must never run without the redirect: Steam folder), and
  `coop_agent.start_game` tries a second start. The owner's Steam folder was not written by it (newest file there
  from 2026-10-30).

### 2026-10-29 - host crash while a third player joined; mailbox wait
- The host crashed (dump in test-install/USER) while a third player was being set up: access violation writing
  [0+4] at 0x170B3AE, in the fire handler of UL action RTSResourceNodeQueryMode (type 0x72, 0x170B340), whose
  record lookup (0x164A910) failed for the new player's copy (record not bound yet) and wrote through null.
  `crash_guards.rts_query_guard` returns from the handler on a failed lookup; installed at boot. The next fresh
  3-player session joined cleanly.
- The crash showed as "join failed: mailbox busy" (the host's main thread sat in CryEngine's "Crash Dump Error!"
  dialog). `sodlib` calls now wait for a pending call to finish (up to the call's timeout) instead of failing at
  once, so a call that timed out during a hitch no longer fails the next one.
- `loot_two.py` can now use two different containers (search nodes 0x5BD and 0x5BB, 3.6 m apart; positions from
  [[0x1E80E80]] + 0x18 * node). In the later sessions no player, the host's included, got a search context at
  those containers any more (also after restoring the 2026-10-28 host save), so the two-container run is still
  missing; it needs another searchable spot.

### 2026-10-29 - simultaneous loot, rerun
- `loot_two.py` (both joining players search the same container at the same moment), 4 runs in a fresh 3-player
  session: both searches completed every time, but only one loot screen was queued and opened (for either
  player); the other got none. Same container = one screen, as in single player (the host's container screen
  is per container). The wrong-panel race the panel ring fixed needs two screens queued within ~0.25 s, i.e. two
  different containers searched at once; only one container spot is known to the test tools (3276,3408), so that
  case is not retested. Next: find a second container node near it (search nodes are not entities).

### 2026-10-29 - a killed joining game can come back
- A joining game that was killed (not quit) could not rejoin while the host ran on: Windows queues an ICMP "port
  unreachable" for every packet the host still sends to the dead address, and the host socket's next recvfrom
  fails with WSAECONNRESET (logged as "32769 from <old address>") instead of returning the newcomer's packet; the
  host never even logged a connection request. New `udp_connreset.py` finds the host game's UDP socket in the
  process (getsockname on each handle, port 64087, SOCK_DGRAM) and sets SIO_UDP_CONNRESET = FALSE; `coop_agent`
  does this once the host listens.
- With the reports off a killed game's player stayed in the game (no timeout; the console commands `kick` and
  `kickid` do nothing in this build). The host agent now notices a player whose agent connection closed or reset,
  turns the reports on until that player has left (the host drops it at once, "Unreachable address"), then off.
- The joining agent also keeps one UDP socket open on the port Windows would hand the game next (Windows gave the
  restarted game the same local port as the killed one).
- Verified on one PC: kill the joining game -> dropped in 1 s -> a restarted game joined at its first attempt
  (in the world 23 s after connect, same character), no leftover player; twice before that without the drop logic.

### 2026-10-29 - crouch of the joining player on the host
- Holding crouch (C) as the joining player kept its host copy in Dodge.Normal.Prep.DodgePrep <-> EvaluateNoTurbo:
  the input-queue fix (same day) ORed the newest input into the merged one, so the one-frame "pressed" bytes stayed
  set in the stand-in input until the next message and the dodge preparation restarted every tick instead of
  turning into a crouch. `peer_input` now merges everything but the newest entry and applies that one whole on
  the next tick. Verified on one PC: host copy and client both go DodgePrep -> Stealth.Basic.Idle / Movement.Creep
  / Walk; backlog stays bounded (max 6 with constant mouse movement, drained every tick).
- Hit reactions: `door_sync.PeerStates` (players' states mirrored from the host into every machine's state
  machines, with the latest clip) now also mirrors `MachineDriven.Reaction.Core.*` (light/heavy hits) and sends
  their clip whatever its name; knock-downs stay with `downed.py`. Not verified yet: no zombie came close enough
  to hit in the test spots (no live zombies; zm_spawn gave none either).

### 2026-10-29 - rucksacks on the ground on the joining machine
- Dropped supply rucksacks (class UL_Rucksack, created by UL on the host only, e.g. from the save) were missing on
  the joining machine (owner's play test). New `rucksack_sync.py`: every 2 s the host sends its visible rucksacks
  (position, yaw; RMI `ClCoopRucksacks`, handler in `singleplayer.lua` via `patch_lua_npcs.py`); the joining
  machine moves a pool of non-physical look-alikes (BasicEntity + `.../rucksack/dropped/rucksack.cgf`, needs
  `LoadObject`) onto them and hides the rest. Verified on one PC: 5 of 5 at the host's positions. Not yet: the
  world markers the host draws over them; picking one up still goes through the player's host copy (untested).

### 2026-10-29 - second play test on two PCs (updates copied to the second PC over its share)
- Joined from the second PC again (20:01), left at 20:07; a local copy then joined into the same slot (rejoin) fine.
- Locked doors: the joining player shoulder-broke a locked door; its host copy went Locked.Start -> Loop -> Slam ->
  Breach in ~2 s (state log `local/state_logger.py`).
- Emote list on the joining player's own machine opened and never closed: the list is shown while bit 2 of the
  request mask 0x1F35B2C is set and only hidden when bit 4 ("allowed") is set too (0x16513B0); bit 4 is set by
  host-side game code (0x14757B2). `coop_setup.enable_emote_list` sets it at join (0x1651B70(4)); verified: shown
  while Caps Lock is held, hidden on release, a tap no longer leaves it open.
- Still open from this test: crouch still shows standing on the host (its host copy entered Stealth only for a
  moment, between dodge rolls); after opening a door the player "flies" a bit and can't sprint (on one PC: sprinting
  then loops Fall.FromRunning <-> Land.Running without falling, also once before any door, so not proven door
  related); no hurt animation on the joining player's screen when zombies hit it (its host copy plays
  Reaction.Core states, nothing sends them back); zombie hit reactions late and dead zombies sink into the ground
  (zombie clips are replayed on the joining machine with a delay).
- Rejoin after a killed (not quit) joining game on the same PC failed twice: the host had dropped the old
  connection ("Unreachable address ...:56219") and ignored the new game's packets from the same address and port
  (warning "32769 from ...:56219"; the joining game seems to use the same local port every start). A clean quit
  (the second PC: "Remote disconnected: Nub destroyed") and a join from another address were fine. Open.
  Reproduced on one PC (20:40): a restarted joining game connects from the same local UDP port as the killed one
  (60818 both times; 56219 in the earlier case), and the host answers each attempt with warning 32769 for that
  address. Lowering the host's `net_backofftimeout` (360 s) to 5 s did not help, so the host seems to keep the dead
  connection's address rather than a timed ban. Ideas: make the joining game bind another local port, or clear
  the host's entry for the departed player's address.

### 2026-10-29 - zombies around the joining player; low-detail zombie bodies
- No zombies around the joining player until the host's player came near (second-PC test): the extra population
  pass `zm_focus` runs around each joining player shared the manager's ambient-spawn timer (+0x20) with the host's
  pass, which had just reset it after spawning, so the extra pass never reached its ambient spawns (the first half,
  32 spawn zones at 0x1E512A0, was empty). The extra pass now starts from the timer value the host's pass started
  from (and the host's value is put back after): 0 -> 1-2 zombies within 100 m of the joining player 250 m from the
  host (one PC, two spots, 60 s each).
- Beyond ~300 m from the host's camera nothing spawns even with the cap (`POP_MAX_DIST`) raised to 1000 m (no
  crash either, fault trap on): the ambient spawn candidates (128 entries at 0x1E57104, stride 0x34) come from the
  level data streamed around the host's camera. Zombies far from the host need the host to stream around the
  joining players; not done.
- Low-detail zombies on the joining machine: zombie bodies there are whole-character files picked per gender
  (`hud_relay.ZOMBIE_BODIES`, all `.../low/low.cdf`); the host assembles each zombie from detailed parts (skeleton +
  body .chr + head/hair) instead. Fix needs the host's part list per zombie sent and attached on the client (or
  CDFs built from the detailed parts; the pak's .cdf files are binary XML). Not done.

### 2026-10-29 - survivor bodies and base markers on the joining machine
- Green see-through boxes and floor outlines at bases (joining machine only): the base's `FacilitySocket` entities
  (e.g. Facility_RangerStation_Indoor) draw placeholder models that UL's base code, host-only, keeps out of sight.
  `coop_setup.hide_facility_sockets` turns their slots off on the joining machine at join; verified at the ranger
  station. (Facilities built later on the host are not handled yet.)
- Survivor NPC bodies (ULNPCnn, class ProtoAlly; same names and synced positions on every machine): new
  `npc_sync.py`. zombie_sync's clip hook also logs "ULNP..." entities (marked with a flag bit); once a second the
  host sends every body's hidden state, loop clip and last clip (RMI `ClCoopNpcs`, new handler in
  `singleplayer.lua`, `patch_lua_npcs.py` adds it to every install). Verified: hidden states match on both
  machines (15 differed before); survivors at the base play the host's clips (e.g. Interact_vigilant_low_loop)
  instead of standing in T-pose. Survivors animated another way (no clip through that helper) are not covered.
- Second-PC update: `sod-coop-pc2-update.zip` (scripts + `singleplayer.lua`); every machine needs the new Lua.

### 2026-10-29 - fixes from the second-PC play test (movement, emote list)
- Input queue overrun: each machine queues the other players' inputs (chat messages, sent whenever they change)
  and took one per frame. A sender running more frames than the receiver (a faster PC; a mouse that never stops
  moving) outran it: measured on one PC with constant mouse movement, ~42 inputs/s sent, ~36 taken, backlog 52
  after 8 s in a 16-entry ring, so moves arrived late and overwritten (likely the second PC's crouch that never
  showed on the host and the twitching). `peer_input` now takes everything queued each frame: sticks and values
  from the newest input, button bytes ORed so a one-frame press still counts. Backlog stays 0; crouch, crouch-walk,
  search and take still work (soak: 2/2 loot cycles).
- The host's emote list (Caps Lock) opened on the joining player's screen and stayed: the client's copy of the
  host's state machine ran UL action `SocialDisplayMode` (0x81). Added to `remote_fx`'s local-only actions;
  verified: the list shows only on the host now. UL action type names are read from the type table
  (0x1F378A8, 0x24 per type, name pointer at +0x20); list in `local/action_types.txt`.
- Green see-through boxes on bases on the joining player's screen (host: none) are not entities with a different
  hidden state (compared all ~1770 entities on both machines, one log line each: long log lines get cut) and no
  debug cvar differs; still open (base-building markers? reproduced at the ranger-station base).
- Same comparison: 15 survivor NPC bodies (ProtoAlly) are hidden on the host but shown on the joining machine,
  where nothing animates them; a likely part of "NPCs in T-pose" (UL's NPC body streaming only runs on the host).
- The joining player's radar shows no map at the base while the host's does (reproduced on one PC).
- A host save written during a co-op session once failed to load (Continue Story: the game stopped right after
  the network setup, before loading the level); New Story worked, a later co-op save loaded. Kept for analysis.

### 2026-10-29 - what the player on the second PC saw (owner's play test)
First real play on a separate PC, playing Ed (the joining player). Not seen in the one-PC tests (those were
checked mostly from the host and by scripts):
- Zombies: none around Ed until the host's player comes near; when they appear their models look low-detail
  (as if not fully loaded).
- Interaction goes to the host or does nothing: Ed can't shoulder-break a locked door (Marcus gets the break
  sequence); the base supply box is not there for Ed (he stands inside it, on the host he stands against it);
  walking up to an NPC shows the "greet" hint on the host's screen.
- Animations: NPCs doing actions (e.g. lying on a bed) are in T-pose for Ed; in cutscenes the characters are in
  T-pose for Ed and Ed keeps walking while the host's player is locked in the scene; Ed crouching (holding C) still
  stands on the host; Ed twitches a little on the host's screen.
- HUD: green outlines (trigger areas?) on bases; Ed's radar map vanished at some point; the host's emote wheel
  (Caps Lock) appeared on Ed's screen and stayed; Tab raises Ed's weapon instead of opening the journal; Ed has no
  inventory grid, portrait, base survivors or base state, and is named "Stranger"; the map opens but points of
  interest have no titles, map marks can be added with the left button but not removed, no controls/legend.
- Looting was not tried yet.

### 2026-10-29 - first join from a second PC
- A second Windows PC on the same LAN joined a hosted game: its agent reached the host's control port, its game
  connected, it got Ed Jones (survivor record bound, NPC body hidden), the full setup ran (`client_first`,
  `client_main`, HUD relay, turn clamp), and the host saw Ed walk ~45 m in 20 s while the host's player stood still. What the player there saw: next entry.
- What the second PC got: one folder with the joining game copy (co-op `singleplayer.lua` and `user.cfg`), the
  bundle scripts, a copy of the Python install with capstone + pillow (no install needed) and a `join.bat` that
  runs `coop_agent.py --install <game> --saves pc2`; Steam logged in there.
- Before that, a VPN client on the host PC hid the LAN from other machines (see the entry below); excluding the
  local subnets from it was needed.

### 2026-10-29 - rejoining, simultaneous loot, other PCs, Steam stats file
- Rejoining: every per-player table frees a departed player's entry (actor/entity id pair: a returning player can
  get a new actor at the old address) and reuses it (`peer_input`, `player_sm`, `search_ctx`, `downed`,
  `pause_sync` - its wish becomes "no objection", `weapon_relay`, `door_sync`); the per-player hook blocks of
  `zombie_targets`/`peer_melee`/`kill_counts` are forgotten and disabled, `remote_fx` wrappers keep the actor in a
  cell that is cleared, the turn clamp is reinstalled. The host agent tells the remaining agents
  (`player_left`); the next joiner takes the departed player's character (`host_pick(prefer=...)`).
- Crashes found while players left and came back (`crash_guards.py`): the players category list (capacity 1) got
  every network player on a joining machine and the game wrote past its arrays ("Buffer overflow in entity db"
  assert, garbage actor pointers): a full list now skips the add, as on the host (`entitydb_full_guard`, at boot);
  a category remove callback on a departing actor (`category_remove_guard`); the NPC-list entry the host keeps for
  a joining player is removed from inside the game's own EntityDbRemove, before the actor is freed.
- Simultaneous loot: the host keeps one copy of the backpack-panel capture per queued loot screen (8 slots);
  test `local/loot_two.py`: two players search the same container at once, each screen went to its own player
  with its own panel (2 of 3 runs; in the third only one search completed, nothing to compare).
- Other PCs (not testable on one PC): LAN search sends directed broadcasts for every interface's real subnet
  (from ipconfig, any mask) and probes the addresses in `coop_hosts.txt`; per-process state moved out of the
  scripts folder (%TEMP%\sod-coop, a folder per process id + creation time); `make_coop_bundle.py` packs the 40
  scripts the agent needs, `singleplayer.lua`, `user.cfg` and a README; `coop_firewall.ps1` opens UDP 64087,
  UDP 64091, TCP 64092 on private networks (not run here). A joining copy run from an extracted bundle joined,
  left and came back like the others.
- Tried a stand-in second machine: the WSL virtual machine on the same PC (its own IP on a separate virtual
  network). It cannot reach the host at all (no ping, no TCP connection to the agent's control port 64092,
  which listens on every interface): WSL's Hyper-V firewall blocks traffic coming in from the VM by default,
  and the VM has no UDP tool for the discovery probe. Testing across machines still needs a real second PC
  (or firewall changes on the host PC), with `coop_firewall.ps1` run on the host.
  Later the same day, after the owner excluded the local subnets from the VPN client running on this PC, the
  VM reached the host: TCP to the control port and UDP to the game port (64087) and the LAN-search port (64091)
  all arrived (UDP sent with BusyBox nslookup to the port). So the VPN client, not the Windows firewall, was
  blocking it; on a PC with a VPN, exclude the LAN from it. A real join still needs a second PC (the VM cannot
  run the game).
- The agent retries the connect once (one rejoin's connection was released by the game right after the server's
  settings arrived, no reason logged; the next attempt got in).
- Steam stats file: every test-copy boot wrote `UserStats.cfg` (game stats, not the save) into the owner's Steam
  save folder about 8 s after launch, before `save_redirect.install` ran (~18 s); probably since the redirect
  existed (2026-10-27). `save_redirect.install_early` now patches ISteamUser's GetUserDataFolder slot from the
  moment steam_api's interface pointer exists (no mailbox needed); the stats file then appears in
  `test-saves/<name>`, and the Steam folder's files kept their times through 6 more boots. (`Class3.0.ulsave`
  there changed on 2026-10-28 22:42: the owner's own Steam session, not a test copy.)
- Never start a test copy's exe directly (even to see whether it boots): a bare launch on 2026-10-29 had no
  redirect and wrote `UserStats.cfg` into the Steam folder again. Launch through the agent or call
  `save_redirect.install_early` right after the launch.
- Verified (one PC, four copies): fresh 4-player New Story; Ed leaves and comes back with his agent run from the
  bundle (Ed again); the 4th player leaves and comes back (same character); soak 6/6 fight + search/loot
  cycles after both rejoins, three knock-downs and get-ups, all four games running.

### 2026-10-28 - up to 4 players
- The network side already took more players: a third copy joined after one more join-crash guard
  (`patch_client.py` "join null local actor", 0x13BE162: a later joiner binds the other players before its own
  actor exists). Channels are not 2, 3, 4 in join order (a failed join used up channel 3), entity ids differ per
  machine, and player actors are named Dude, Dude(1)..Dude(3) everywhere: `local/peers.py` tells players apart by
  actor/name, own player = the client-actor getter.
- Every co-op module was single-peer ("channel 2", one baked actor/record). Now per player:
  - `peer_input.py`: a slot per remote player (queue + stand-in input), chat input routed by the sender's entity id
    (the game translates it per machine); the host passes each joining player's input on to all clients as
    "cooh:" with that player as source. Other chat handlers (pause, kills, weapon, loot) get the sender too.
  - `player_sm.py`: a joining machine ticks its own and every other player's state machine (record read from
    actor +0x1110 at run time).
  - `search_ctx.py` (a search manager + context + swap routine per player), `loot_relay.py` (screens tagged with
    the player, sent with `onClient` to that player's channel only; Take/Use by sender), `downed.py`,
    `maya_record.py` (record per player, `keep_active` table), `zm_focus.py` (focus point / cull / population
    pass per player), `pause_sync.py` (host is the authority: pause only when everybody has a menu open),
    `remote_fx.py`, `kill_counts.py` ("kilh:n:NNNN"), `weapon_relay.py`, `door_sync.py` (4 name slots; the
    client Lua `ClCoopPeerState` changed in every install's `singleplayer.lua`), `zombie_sync.py`.
  - Hooks with per-player code run as chains of blocks (`local/chain.py`): `peer_melee`, `zombie_targets`,
    kill counts.
  - `coop_setup.py`: once-per-session host part, a per-player host part, the newcomer's machine, and
    `client_player_joined` for the machines already in the game; `coop_agent.py` hosts up to 3 joiners one at a
    time (the LAN list shows "n/4 players").
- Crashes found and fixed on the way (all in `crash_guards.py` unless noted):
  - a record bound to a player whose actor UL could not resolve (record update, null body, 0x154837B /
    0x1548660); UL finds actors only through the query lists, so `zombie_targets` now registers the player first;
  - joining machines ticking other players' state machines: actions writing through a missing per-actor slot
    (0x170B19F, 0x170B3AE; joining machines give missing handles a spare slot, only for the action handlers'
    callers) and a condition on a freed zombie actor (0x17325DB; actor and entity vtables checked);
  - the Lua damage handler killed a joining player outright (health <= 0 -> ProcessDeath -> Kill) before the
    health floor applied: `singleplayer.lua` (all installs) keeps Dude(n) at 1 there;
  - a player leaving: its freed actor stayed in the NPC query list and the interaction list (both ours) and
    crashed the host within a second; the game's own EntityDbRemove now takes it out of the NPC list
    synchronously, the interaction loop and the NPC tag pass skip non-CPlayer entries, other native loops check
    the player vtable, and the host agent notices the leave.
- Verified on one PC with four copies (`local/play_menu.sh 4`, `menu_flow_test.py --new --direct`): New Story,
  Marcus / Ed / two random survivors, no NPC double of any player on any machine; 20-minute soak with the three
  joining players taking turns, 10/10 cycles (knock-downs and get-ups included), all four games running; with
  the 4th player's game killed mid-session the other three kept playing (soak 4/4 afterwards).
- Not done: a player rejoining into a slot used before; loot screens of two players opened in the same
  quarter second share one panel capture.

### 2026-10-28 - joining player is Ed Jones; no doubled characters
- Character definitions: 146 entries at `[0x1EE25E4]` (0x34 bytes, +8 model path). Story characters have models
  under `story_characters/`: 0 Marcus Campbell, 116 (swat01), 124-145 (142 Ed Jones, 145 Maya Torres); the rest
  are generic survivors (`/human/body/`).
- In a new story Ed has a community record from the start (flags 0x20029) with an NPC body (`ULNPC01`); the
  host player's own record body (`ULNPC00`, Marcus) is hidden by UL on the host only. Body names are the same on
  both machines but do not follow the record index (record 16 had `ULNPC15`), so the client gets names, not indices.
- `maya_record.py` now binds the joining player to a chosen character: `pick_def(slot)` gives Ed for player 2
  while he has a record, else (and for players 3+) a random generic survivor no record or player uses; story
  characters are never picked. Ed's existing record is bound like a saved Maya record was (record +0x58 = her
  entity, `keep_active`, bad trait stripped) and his NPC body hidden, deactivated and moved under the map.
- `coop_setup.host_first` picks the character and the host's actual model (Marcus in a new story; a continued
  save may have the host on another survivor - the old code always loaded Marcus); the client loads both, and
  `hide_player_doubles` hides every `ULNPC*` body wearing a player's model on the joining machine.
- Verified: fresh New Story started from the menu: host Marcus, joining player Ed; both NPC doubles hidden on both
  machines before and after a 5-minute play soak (3/3 search + Take, she went down at 1 HP and got up, both alive).
- Not checked yet: story missions that script Ed's NPC (his body is hidden now) further into the story.
- Only one joining player is supported by the co-op layer (channel 2 everywhere); the character rule for players
  3+ is ready in `pick_def` but has nothing to run on yet.

### 2026-10-28 - co-op from the main menu (Host / Join with LAN search)
- The main menu (front-end screen 2) keeps its items in a static array: 11 slots of 0x90 bytes at `0x1EE81A0`
  (+0 accept callback, +4 other-input callback, +0xC inline text shown as is, +0x78 1.0), count `0x1EE87D0`,
  selected `0x1EE87D4`. The filler `0x161A1C0` (menu enter and its refresh timer) fills them and tail-jumps to
  the shared builder `0x1618E10`, which sends them to Flash every frame. The shared input handler `0x16144E0`:
  6 accept calls the item's callback, 9 back pops the menu (`0x16144FE`), 11/12 down/up.
- `coop_menu.py`: the filler's tail inserts "Co-op Play" before "Exit Game"; in submenus the items come from
  per-mode tables. Every custom item has one native accept stub (navigation done natively, other choices
  posted to a request cell the agent polls); back in a submenu goes to the parent instead of popping.
- The menu title is a localization hash (`0x1EE88C4`, looked up by `0x14F1C50`); a plain string showed
  `ENGLISH_LOC_MISSING`. The builder's two lookups go through a stub that maps our own hashes to our titles.
- `coop_agent.py` (one per copy/PC): starts the game with test saves and boot guards, serves the menu. Host Game:
  force `map %s server`, accept Continue / Start a New Game through the menu's own input handler, answer LAN
  probes (UDP broadcast on every local subnet and loopback), wait for the player. Join Game: lists the hosts
  that answer, connects, then both agents run their half of `coop_setup.py` over a TCP control channel
  (`coop_setup` is now split into host_first / client_first / host_main / client_main / host_last;
  `sync_models.sync_to` and `weapon_relay.give_host/select_client` take the cross-machine data as values).
- Pitfall found on the way: `user.cfg`'s `sv_bind` held the test PC's old LAN address; after a DHCP change
  the server was not created ("Failed creating network nub") and every hosted load crashed (null current
  level at `0x143E5CB` for Continue, null `[0x19F65F4]` at `0x92D5A2` for New Game) - also with the old
  scripted boot. The agent and `session_join_leave.py` now set `sv_bind` to the current LAN address at runtime.
- The joining copy's game framework exists at the menu already, and a copy started while another holds
  `class3.log` writes `class3(N).log`: the agents detect the join from the actor list (channel 1 and 2 actors).
- Verified on one PC: both copies started by agents, host via Host Game > Continue Story, joining copy found it
  in Join Game and joined; setup finished on both sides in ~1 min; short play soak 2/2 search + Take cycles,
  both games alive.

### 2026-10-27 - final soak; vehicles and HUD consumables checked
- 25-minute play soak on a fresh boot (`local/play_soak.py 25`): 19/19 search + Take cycles, she went down
  5 times under real zombie damage and got up each time (game restored 51 HP), both games alive at the end.
  The assert guard caught two asserts ("TDBTagSet duplicate buffer overflow", tag set full at 0x13E5CA6) that
  would each have quit the host before.
- Vehicles: the only vehicle near the start (pickup ULVEH00) is locked at this point of the story for both
  players (`Vehicle.EntryLogic -> Vehicle.CannotEnter.Locked`); interaction kinds next to a vehicle are 7 (enter)
  and 9 (trunk inventory, opens the journal). Co-op driving needs a save where vehicles are unlocked.
- Keys seen while testing on the host: 1-4 emotes, C dodge, E use. Using a backpack consumable outside a loot
  screen (`ul_player_item`) needs her journal / item slot on her machine, i.e. her UL character data there (not
  done; see next steps).

### 2026-10-27 - playable sessions: she goes down instead of dying, assert/entity crashes guarded, play soak
- `local/play.sh` starts a full session (keep-awake, both copies, co-op layer, relay). `local/play_soak.py` runs
  play-like cycles: she fights a fresh zombie group (mashing jump when down), then searches and takes at a
  cabin spot. Result on a fresh boot: 5/5 cycles, real zombie damage, she went down at 1 HP once, got up and
  was given 51 HP back by the game, every search and Take worked, both games alive. Earlier 20- and 12-minute
  runs: both games alive throughout, no asserts.
- Her HUD health arc follows her health (health is replicated).
- Death: at 0 health UL's actor is dead for good; raising it again asserts "SetHealth reviving a dead actor"
  (ULActor.cpp), and the assert quit the host. `local/downed.py`:
  - her health never goes below 1 (hook on ULActor::SetHealth 0x1461E40, her actor only). Like the host's
    player, she then lies down and gets up by mashing jump (Space); her input reaches her host copy through
    peer_input, and the game restores her health on getting up;
  - a finisher's death animation (`ReactionGrab.Dying/Death_*`) becomes the down idle after 4 s.
  - Forcing her straight to idle instead (state index write) skipped the down states' exit actions and she then
    took no damage at all; the same happened to the host's player after test resets.
- Asserts: the game's assert handler 0x58DF40 calls exit(1) right away (no dump). `crash_guards.assert_guard`
  makes it record and return (count + ring of caller and message), installed at boot on both copies.
- Silent host deaths: UL's EntityDbUpdate (0x1525610, EntityDb.cpp) asserted on a registered entity id the engine
  no longer had. `crash_guards.entitydb_guard` skips that entry instead (the assert block is rewritten in place).
- Search fixes from the soak:
  - her search focus now survives 1 s without a refresh: the start animation moved her and aborted it;
  - the local player's "down, press Space to get up" byte (0x1EA6DED, read by the finder through 0x146DBD0)
    switched interactions off for everyone, so while the host's player lay down she could not search. It is
    now swapped in her search context too.
- Test-harness notes:
  - Escape on her machine with no loot screen open opens the Game Menu, which swallows her input (the first
    soak ran with it open);
  - the client crash at 0x1527E74 (null client actor) is a consequence of the host going away.

### 2026-10-27 - looting works for the joining player: her backpack on her screen, Take and Use
- Verified on her machine: search, then her loot screen shows the container on the right and HER portrait and
  backpack on the left (rifle, axe, snacks, painkillers, weight). Take moves the item into her backpack (stack
  1 -> 2, 11.9 -> 12.2 lbs, container "Nothing left"). Use consumes it (container emptied, backpack unchanged).
  Everything is applied on the host to her survivor record and the host's container.
- Her backpack panel: the left panel is drawn by 0x1653F60 from the current player's record with Flash calls
  PIShow, PIFocusSet, PICapacitySet, PIPortraitSet, PIWeightSet, PIItemSet/PIItemClear per slot, PIBackpackSet,
  PIRucksackSet. Her machine has no record, so:
  - the host runs 0x1653F60 for her record while Flash Invoke (0x6E37D0: name, variant array, flag) records the
    calls instead of making them;
  - `hud_relay.py` sends them as text (RMI `ClCoopLootPI`, in parts, before the container);
  - her machine's 0x1653F60 is replaced by a replay stub (cvar `coop_loot_pi`).
- Flash argument arrays: data with the element count at [data-4] (& 0x7FFFFFFF) and 8-byte elements
  (type info, value). Types: int 0x1970A88, bool 0x195F30C, float 0x195F32C, string 0x195F1EC (char* with a
  CryString header; refcount at -0xC, -1 = static, never freed), 0x195F2EC. The array object also holds
  converted values at +0xC and flags at +0x14: Invoke converts again only when bit 0x4 is set, so a reused
  array must set it before every call (the first replay showed every call with the first call's values). The
  object must be built with its constructor 0x5CD0A0 (a zeroed one crashed her machine inside Invoke).
- Use (flags 0x80) runs the record's UseItem (0x1548EE0), which only applies an item to a record with flag 0x4
  (+0x15C; his record has it, her survivor record does not). The host's action handler sets that bit on her
  record for the duration of a loot action and clears it again (her record stays 0x29, her state machine
  stays).
- Test notes: a host container menu closed with Escape can leave bit 0x8 of [0x1EA7FFC] set, after which
  ShowContainerMenu only updates data and opens nothing. `local/host_her_screen.py` opens the host's loot
  screen for her record (debugging), and `local/ui_capture.py` logs Flash calls.

### 2026-10-27 - Take from her loot screen works; boot crash on her saved record fixed
- Verified: on her machine Take (Space) moves the item from the host's container into HER backpack on the host
  (shown by the host's loot screen with her record as current player: her portrait, rifle, axe, snacks,
  painkillers, plus the taken snack), and her screen refreshes to "Nothing left. / Empty".
- How:
  - her machine's menu callback 0x145B520 is replaced by a stub that sends `lotc:SS:FFFF:DDDDDDDD`
    (slot, action flags, data) to the host as a chat message (new tag in `peer_input.py`);
  - the host runs UL's callback in her search context, with the client-actor getter returning her actor
    (search_ctx "force" cell) and the current-player record getter 0x14B0420 returning her record handle
    (0x14AF690 by entity);
  - the callback's own refresh (0x145CD60 with flag 1) goes back through the loot relay. Her machine passes
    flag 1 when the node is already the one shown, else an emptied container closed instead of redrawing.
- Her machine has no UL character records at all (the record table is empty, current-player handle 0xFFF8000),
  so the left panel of her loot screen shows a default survivor with placeholder slots. Next: show her
  backpack there.
- Use (Q) on her screen does not reach the menu callback (Take does). Her machine's UI probably checks the
  current player before calling back, and she has no record there. Same missing piece as the backpack panel.
- Intermittent host crash while the saved world loads (dumps 15:25, 16:07): the per-record update asks the body
  actor of her saved survivor record for its state machine (0x154832C -> 0x1461960) without a null check; the
  record names the previous session's player entity, which no longer exists. `crash_guards.record_body_guard`
  (null body skips that part) and `collision_guard` are now installed at boot by `session_join_leave.py`, right
  after the save redirect (both report True).

### 2026-10-27 - her loot screen opens on her machine
- Verified end to end: her full search on the host -> the loot screen ("Found some remaining items", Snack) opens
  on her machine, not on the host's. Still to do: the left panel shows the client's own current-player record (a
  default survivor with an empty placeholder backpack), not her backpack, and Take/Use/Close are not relayed back.
- How UL opens it: `0x145CD60(node, flag)` needs node == the last focused node [0x1970E04]. It reads the
  container entry from the table at 0x1EA64A8 (20 x 0x68 bytes: +0 node, +4 count, +8 item handles,
  +0x48 counts, +0x58 kinds), picks the status text (string ids 0x20B..0x213) and calls ShowContainerMenu
  0x148BAE0 with the menu callback 0x145B520 (Take/Use). Her manager reaches it through 0x1630C20 -> 0x145EE80
  (SetInput when the node is already searched, or right after a search completes).
- The client has the same node ids and UI. Writing the host's container entry into the client's table and
  calling the client's 0x145CD60 shows the same screen there.
- `local/loot_relay.py`:
  - host: 0x145CD60, called while her search context is in (search_ctx override cell set), copies the
    container entry into a ring and returns without the host UI;
  - `hud_relay.py` sends it as RMI `ClCoopLoot("seq:hex")`;
  - client: the Lua handler sets cvar `coop_loot`, and a stub chained on pass 1 writes the entry into the
    client's table, sets its last node and calls 0x145CD60.
  - New RMI in both `singleplayer.lua` copies (handler placed before the door marker).
- Her search-node perception: candidates come from her perception query (actor +0x1170 list, entries in the
  perception manager 0x1634090 at +0x6B1A8 + idx * 0xAC, results written to blackboard +0x204 + i * 0x6C by
  0x16493E0). The query's transform is set by a state-machine action (0x170B340). When it was stale she saw no
  node.
- Her host copy got stuck in `Combat.Idle.BluntLight` (weapon out) on fresh boots and after fights, while her
  client was back in Observer. The exit waits for combat tags (0x7E...) that the host only clears for its own
  player, and those tags (0xCD) block the search. `search_ctx.stand_down` (from `hud_relay.py`): after 3 s in a
  combat idle state with no button held on her machine, her copy returns to idle and the combat tags
  (0x2C, 0x09, 0x7E, 0xCD, 0x199) are removed. Verified: Observer about 4 s later.
- Her search could lose focus while her LowVert search animation moved her (node out of reach after ~1 s). Her focus
  now stays while her manager is searching (state 2/3).
- Crash found and fixed: `peer_melee` and `remote_fx` baked her state-machine record address into their stubs.
  After UL re-created her state machine (new address) the stub wrote through freed memory (host access
  violation in the stub). Both now read the record from her actor (+0x1110) each time. Host crash dumps are
  parsed with `local/dump_exc.py` (exception, registers, stack code addresses).
- Test notes: node state persists in the test save (a depleted node stays depleted, status 0 from 0x1451B30).
  For tests, node flags at [[0x1E80E80]+0x20]+node = 0 make it searchable again. Nearby fresh nodes: 1467
  (0x5BB) LowVert near 3281.5,3410 and 0x5BA. Zombies at the cabins knock the host player down during tests.

### 2026-10-27 - her search works on the host, completion opens the loot screen (on the host's screen)
- Verified: at a search spot she holds use on her machine, her host copy runs `Interact.Search.Logic.ContextTest
  -> Search.Normal.Ground.Start -> Loop`, progress accumulates while held, release gives `Stop -> Observer`, and a
  full search completes (manager state 4); further holds on that node stop at once. Her own screen shows her
  crouched searching (state mirroring from the door work). The loot screen then opens, but on the host's
  screen, with his portrait and backpack. Next phase: that screen on her machine.
- Why her player state machine dropped at random (the earlier "trait 492" theory was wrong): the per-record update
  0x1547920 also streams each survivor's NPC body. From distances it sets "hidden"/"inactive" and then
  releases the body's player state machine (0x1466AD0 -> 0x1636780) or creates one (0x1463C20). Her record's
  entity (+0x58) is her player actor, so depending on where she stood UL "despawned" her. `maya_record.
  keep_active` forces both flags off for her record at 0x15482C9; a lost state machine also comes back by itself.
  Caught with a logging detour on 0x1636780 (`local/sm_release_log.py`, installed by `coop_setup.py`).
- UL's search system is a singleton for the local player. The manager is the object at [0x1EED390]
  (0x7C bytes, ctor 0x162F880; +0x40 focused node, +0x6C state 0 none / 1 focused / 2,3 searching / 4 done /
  5 stopped, +0x60 time searched). Two more globals belong to it: the last node [0x1970E04] and the focus
  timeout [0x1E81F78], both written by the node candidate check 0x14569A0 that the finder runs for each player.
  - The finder runs for both players, so the player with no node in reach reset the focus (her kind flickered).
  - Only the local player's update feeds use input to the manager (0x1468A03: actor +0x10A4, null for her).
  - Tags and the search handle go to the client actor (getter 0x6683A0, used at 10 places in the manager code).
    One of them is tag 0xC6, which ContextTest needs.
- Fix `local/search_ctx.py`: she gets her own manager, focus and timeout, swapped into those globals:
  - around her finder call, then her use input (stand-in input bytes +0xFD/+0x117) is fed to her manager;
  - once per frame for her manager's tick, after the host's, including her focus timeout;
  - the HUD progress bar only for the host's manager;
  - the client-actor getter returns her actor when called from the manager code while her context is in.
- Actor tag sets: actor +0x10B0 (512 bits; tag = 64 * qword + 63 - bit, `local/tags.py`), copied to blackboard
  +0x12E0 each tick (0x1461E00); added/removed by CPlayer virtuals 0x1461DE0 / 0x1461E20 when states are entered
  and left. Condition type 19 tests them; type 40 checks a blackboard slot; type 92 is a "held for N s" timer;
  type 56 is an edge detector; type 17 combines other results (bit 0 OR, bit 1 negate).
- Test-harness traps found on the way, all from forcing her state index (`goal_check.stand_up`,
  `peer_input.reset_state`), which skips state exits:
  - stale tags (e.g. 0xCD, which blocks the search);
  - a stale search handle at actor +0x153C (the manager then saw "already searching"). `reset_state` now clears
    the handle.
  Also: writing a string over the blackboard hand-item pointer (+0x1458 is a `char*`) crashed the host. The
  "Fists" and trait differences were not needed.

### 2026-10-27 - looting for the joining player: findings so far (in progress)
- Goal chosen by the owner: the full loot screen on her machine (search, loot screen with take/use/close, her own
  backpack), not a simplified version.
- Search spots are not entities. The host's interaction finder `0x1464000(actor)` runs every frame for both
  players (actor update, gated by actor vt+0x130) and writes the interaction kind to blackboard +0x1524 (prompt
  icon 0x406000N): 1 = barricade, 3 = door, 5 = search, 8 = something else seen in cabins. Search spots found
  by sweeping the host player with the kind (`local/search_spots.txt`; a cabin at about 3246,3396 facing 3.14 and
  3276,3406 facing 0). A full hold (`sr_searchDurationNormal` 10 s; `sr_tapWindowDuration` 0.25) runs
  `Interact.Search.Normal.<Ground|LowFlat|...>.Start/Loop/Stop`, then opens the loot screen (journal UI,
  `Item.Journal` state): his backpack on the left, the container's items on the right, Take/Use/Close.
  `sr_debug 1` overlays the node id and the container model's states (base/full/searched/depleted).
- Her host copy gets kind 5 at a search spot, and her use input arrives, but she never enters
  `Interact.Search.Logic.ContextTest`. Forced into it, she drops back to idle at once. Condition type 96
  (0x16FEE50 -> 0x1716740 -> 0x14A5880 -> 0x15325B0) checks traits: record +0 holds 36 trait indices into
  [0x1EE2620] (0x34-byte entries, data id at +0). The host player's record has 0x06090001 (index 492; Ed has it
  too) and 0x060D000A (index 208); her record, created as a random survivor, has neither.
- Adding trait 492 to her record made UL release her player state machine (removed from the extra list by
  0x16366B0 via the release function 0x1636780(rec), callers 0x1461CC1 / 0x1466B6C / 0x14682D7, actor +0x1110 =
  its record). Redirecting her entity to the host player's record in 0x1716740 (`maya_record.share_traits`,
  currently off: `SHARE_TRAITS = False`) dropped it too in one run but not in another; not understood yet.
- Test harness: forcing a state every few ms (`local/search_conds.py`) crashed the host once (entity-query code);
  don't. Teleporting her into small cabins pushes her up to ~2 m; check her host position.
- Next: find why the trait makes UL release her state machine (probably a player/NPC role decision for records
  with that trait) and keep it, then the search result -> loot screen path for a non-local searcher, then the
  client side (loot screen and backpack).

### 2026-10-27 - her door animation on her screen; walking through doors
- No door animation on her screen: a state written into her client state machine neither switches the animation
  graph's "StateMachine" input on (0x1463510) nor starts the state's clips (entry actions skipped). The client stub
  now calls 0x1463510 for door states (0x1463550 for idle). The zombie_sync hook on 0x820BC0 also logs the players'
  clips ("Dude..." entities), and `PeerStates` sends each player's latest `Interact_door_handle_*` clip with the
  state ("...|clipHost|clipPeer"). The client handler starts it on "Dude" / "Dude(1)" 60 ms later. Her screenshot
  mid-`CloseFar` shows her stepping into the doorway, arm out; standing idle she is upright.
- Walking through doors: (1) the relay sent every `Door_N` with a position, but when the pool reuses a spot its
  old entity stays there (two doors on one spot), and a stale one could carry an open state over a closed door.
  `placed_doors` now uses the door system's placed list (entity pointers at 0x1E7E750, count 0x1E7E760). (2)
  Client collision built by a timer after the clip sometimes came from a mid-animation pose (7/9 or 0/9 rays). A
  closing/closed door is now reloaded (default pose = closed) and physicalized at once, then the clip plays.
  `local/door_audit.py` (every placed door: host flags, client state and physics, rays on both machines): 0
  mismatches, also after a pool shuffle. Walk: closed door stops her 0.43 m short; after she opens it she walks
  2.6 m through.
- Test harness: a door use pressed right after a teleport (before the door context and, for the host's player, the
  interaction kind +0x1524 = 3 were set) did nothing on slow frames; `door_use.py` now waits for them.

### 2026-10-27 - test saves separated from Steam; Maya clones; stuck after door use
- **The test copies were saving into the owner's Steam save folder.** The game saves `Class3.0.ulsave`,
  `Sandbox.0.ulsave` and `UserStats*.cfg` into the folder from `ISteamUser::GetUserDataFolder` (vtable slot 6):
  `Steam\userdata\<account>\241540\local`, the same folder the owner's Steam copy uses. `sys_user_folder` /
  `%USER%` only cover profiles. `local/save_redirect.py` replaces that vtable slot in each test-copy process
  (per process, nothing on disk) with a stub returning `local/test-saves/host|client`. `session_join_leave.py`
  installs it right after each process starts, before any game is loaded. The path is only asked for when a game
  is loaded or saved; a menu-only boot has no cached copy. Current save copied there once; the Steam save is no
  longer written (timestamp unchanged after reboots and sessions).
- "Other NPCs look like Maya": every boot `maya_record` created a new Maya record, and the save kept them, so their
  NPC bodies walked around as Mayas. It now reuses a saved Maya record. Extra ones become unused generic
  survivors with their bodies re-skinned. Her own old body (`ULNPC<record>`) is also hidden on the client, and
  setup runs it before the client model sync.
- "Stuck in place after a door": two causes on her machine, both from mirrored door states written into her
  client state machine (their state actions and exits don't run):
  1. The animation graph input "StateMachine" stayed on (actor +0x1114 = 1; on: 0x1463510, off: 0x1463550), so
     the graph kept waiting for the state machine and she could not move at all, even on open ground. The
     client stub now calls 0x1463550 whenever it writes the idle state (`peer_input.reset_state` too). 8 door uses
     in a row: she walks 3.6-3.9 m after each.
  2. ULActor stamina request stacks (+0x12EC cap 4, +0x1300 cap 8) filled up. The assert "SM stamina tank rate
     requests exceeded stack space" (ULActor::ProcessStateMachineBehavior) then quit the client (caught by the
     exit trap). The client stub resets a stack one short of full.
- Host side: her host copy could loop in `...Locked.Loop` (locked door rattle) forever after she released the use
  button (her stand-in input showed it up; her client left at once). `door_sync.end_locked_loop` moves it to
  the matching `...Locked.End` after 0.5 s with the button up.
- Test harness: teleporting her to a `door_pose.txt` pose can embed her in place even with no door involved; a
  stuck walk test there is not a game bug. Test movement on open ground after the door use.

### 2026-10-27 - door gaps closed: host collision, stuck pairing, Marcus turned into Maya
- Host collision for doors she closes out of his camera's view (the gap in the previous entry): the host never
  computes the closing pose, so the door's static physics stayed where the open leaf was. `settle_stalled` now,
  for a stalled clip that ends closed, reloads the door model (default pose = closed) and physicalizes it static
  before the jump-to-end replay; the game's finalizer then sets state and part flags as usual. Four out-of-view
  toggles: closed 9/9 rays on both machines, open 0/9 on both; the host player's in-view use of the same door still
  works. Not fixed by `ENTITY_UPDATE_ALWAYS` + `Activate` + `ForceCharacterUpdate` together either.
- Doors stuck in "clip playing" (0x4) from before the hook (seen on a fresh boot: 0xC) are settled too, using the
  clip of the state they are heading for.
- Stuck door pairing: a player whose state machine left a door state without DoorRelease (action 55: 0x1440960(actor,
  door) + blackboard +0x14E0 = 2) stays in the pair list 0x1E7E754 and never gets a door context again (her
  host copy stuck in Locked.Loop, then reset). `release_stuck` releases players paired but outside Interact
  states for 1.5 s.
- Bug: `maya_record.py` used the re-skin function 0x15472D0, which also re-skins the current player: the host's
  player had been showing Maya's model (both machines) since the record was introduced. It now writes the
  definition index (+0xF0) directly; models restored with `coop_setup.load_model`.
- The Radio Assistance menu that blocked her input yesterday could not be reproduced (held "use" 0.3-2 s, no
  target); unexplained. Test harness: `door_walk.py` now forces focus (ALT tap) and starts 0.15 m above the floor;
  `door_full.py [rounds] [door]` (pool names change between boots; Door_5 was locked in this save).
- Fresh-boot `door_full.py 2 Door_0`: her states mirrored, door state/seq equal, collision equal, walk blocked
  closed (0.49 m short) / through when open.

### 2026-10-26 - doors fully on the joining player's machine: animations, collision, state
- Initial state: door spots keep their state in a flags byte (`[0x1E7E770][spot]`: 0x8 open, 0x4 clip playing,
  0x2 other side, 0x1 locked); placed doors: count 0x1E7E760, spot ids `[0x1E7E758]`, positions `[0x1E7E74C]`;
  door types `[0x1E7E788]` (8 x 0x34: clip names, model at +0x28). Doors placed before the hook get the clip of
  their current state (`door_sync.state_clip`), so every door shows open/closed correctly from the start.
- Stalled doors: a door clip only advances while the host's camera renders the door. A door she used out of his
  view stayed at flag 0x4 with clip time 0 forever, and then no actor could use it again (the door search accepts
  flags & 0xC in {0, 8} only). `door_sync.settle_stalled` replays a clip that hasn't moved for 1.2 s with
  jump-to-end; the game's finalizer 0x143FE20 then sets the state bits and collision (0x143FB50). Not fixed by
  `ENTITY_UPDATE_ALWAYS`, `ForceCharacterUpdate` or `ca_ForceUpdateSkeletons` (cvar block 0x19627F8 +0xD8).
- Collision on her machine: the client handler physicalizes closed doors (`Physicalize(0, PE_STATIC)`, 2 s after a
  normal closing clip / 0.15 s after a jump, so the closed pose is used) and removes physics from open ones.
  Ray probe through the doorway (`local/door_ray.py`): closed 9/9 hits on both machines, open 0/9. Walking with
  real input (`local/door_walk.py --flip`): open -> she walks through (5.4 m past), closed -> stopped 0.47 m short.
- Players' door animations on her screen: the client has no door context, so the host mirrors each player's
  animated `MachineDriven.Interact.Door.*` states (not the logic states) through RMI `ClCoopPeerState` -> cvar
  `coop_peer_state` -> a stub chained in front of the client's pass-1 detour that writes the state index into her
  record or the client's copy of the host's player. Verified: her client went `OpenAway` / `CloseFar` /
  `CloseClose` with the host, and the client's copy of Marcus `OpenStart -> OpenAway -> idle`.
- Latency: the host's per-client send budget (`sv_bandwidth`, default 30000 bit/s) was full of co-op RMIs, so
  door changes reached the client ~3.4 s late; `coop_setup` sets 300000 -> 0.4-0.7 s.
- Test-harness notes: the client's "use" key with no client-side target opened the Radio Assistance menu
  (blocks input; Escape closes it). Teleports to exact floor height can leave her stuck (lift 0.15 m).
- Known gap: on the host, a door she operated out of his camera's view is settled by the jump-to-end replay, but
  its host-side collision does not match the closed pose (rays pass; the host never rendered the pose). Her
  machine is correct. Affects host zombies and the host player at doors he never looked at.

### 2026-10-26 - doors on the joining player's screen (door relay)
- The door system plays door clips through 0x143F0A0 (thiscall door entity; clip name such as
  `door_door_handle_left_open_away` / `..._right_close_close` / `..._close_far`; flag = jump to the last frame,
  used when a spot is set up). Clips are started by id (skeleton anim vt+0x28) with "repeat last key", so the last
  frame is the door's open/closed pose. Door models are animated `.cga` characters (slot 0: `animated_door`,
  `anmated_door_glass`, `animated_door_firedoor`, ...; path at character +0xB8). Lua: the script table's
  `__this` is the IEntity pointer.
- `local/door_sync.py` + new RMI `SinglePlayer.Client:ClCoopDoors` (loose `singleplayer.lua`, both installs):
  the host hooks 0x143F0A0 (ring of entity, flag, clip); `hud_relay.py` sends the placed doors every 0.5-3 s
  (`index,pos,angles,model,clip,flag,seq`, split into parts). The client handler loads the model on the same
  `Door_N`, places it and plays a new clip when the sequence changes (jump-to-end clips at speed 100). Doors the
  host no longer places go back under the map.
- Verified on a fresh boot: the client's door pool matches the host's (9 doors, same positions and angles).
  Marcus opened the restroom door, and it showed open on her screen. She pressed the interact key at it: her
  host copy ran `Interact.Door.Front.CloseClose`, the client handler applied the close clip (seq 1 -> 2), and
  her screenshot shows it closed. `local/door_check.py` prints the host clip/seq and the client's applied seq
  for the door at `door_pose.txt`.
- Not yet: her own door animation on her screen (her client state machine stays idle; the door interaction runs
  on her host copy), and collision for closed doors on the client (the client door has no physics). Doors set up
  before the hook was installed show their default pose until their next clip.
- `coop_setup.py` now runs `maya_record.install` (her survivor record + door-system registration).

### 2026-10-26 - doors are a host-side entity pool; the client has none
- `Door_N` entities (class `ULDoor`, native, empty script table) are a pool that UL's door system assigns to door
  spots near the player: the same name moves between doors (Door_2 was at 3100,3323, later at 3175,3366); unused
  ones sit at 0,0. On the host the assigned ones have one geometry slot and their open/closed angle; on the
  client every `Door_N` is at 0,0 with no slots (no UL world tick there). So her screen shows no door state at
  all, for either player's doors. Mirroring needs a door relay like the zombie one: for each assigned host door,
  position, angle and geometry (slot 0 static object), applied to the client's pool entities.
- Unlocked doors found for tests: Door_5 at (3173,3349) at the time of the test (the host player opened it:
  `Interact.Door.Back.OpenStart -> OpenAway`). Door_1 at (3102,3339) is locked. Pool names change, so look doors
  up by position.

### 2026-10-26 - a real survivor record for her; her host copy uses doors
- Record creation path found: console command `com_addMember` ("Add a new character to your community",
  handler 0x14B6BF0) takes the first free record of the character table ([0x1EA83B8], 100 x 0x178; free =
  flags +0x15C bit 0 clear, handle +0x48 = 0x0FFF8000), initialises it (0x15473C0 random character; with a name
  argument 0x1547580 with a data id), adds it to the community (flags 0x29) and spawns its NPC body `ULNPCnn`.
  The name argument is a `Table.Entry` data-id name, not a character name; character definitions are not in
  that registry, so re-skinning is used instead: 0x15472D0 (thiscall record, definition index).
- `local/maya_record.py`: com_addMember -> re-skin to definition 145 (Maya Torres) -> record +0x58 = her entity id
  -> hide/deactivate/move the spawned body (otherwise it floods the log in her place) -> register her with the
  door system (`interact.register`). Unlike the earlier borrowed "known" record, her player state machine stays
  in UL's extra list, and the "Entity <her id> is not part of the community" flood stops (all three log sites in
  0x1716740/0x17167F0/0x1716880 go to 0; they find records by iterating active records and comparing +0x58).
- Result (fresh boot): at the door pose the host player gets (`door_pose.py`), her host copy now gets the door
  context (blackboard +0x14D8 = 383) and runs `Interact.Door.Front.ContextTest -> OpenStart -> Locked.Start ->
  Locked.Loop` on the host (Door_1 is locked) when she presses the interact key on her machine
  (`local/door_use.py client`). Her own screen does not show it yet: the client runs no door system, so her
  states and the door state must be mirrored from the host.
- In a long session the same setup gave no context: the door routine (0x1442B60, per-actor loop over 0x1E7EBB0)
  visited her entry but her distance test to every door candidate failed. Not explained; it worked again after a
  reboot.
- Tooling: `Proc.count_exec(..., when=(esp_offset, value))` counts only hits where a stack slot (e.g. a loop
  index) matches. Fixed a crash in `count_exec`/`watch_writes`: when the exception handler resumes a thread,
  Windows restores the debug registers from the exception context, which could re-arm a breakpoint after it was
  cleared. The handler would then be removed and the next hit was an unhandled breakpoint exception that killed
  the host (dump 0x80000004 at the breakpoint address). The handler now clears Dr7 itself while disarming.

### 2026-10-26 - host out-of-memory after ~7 h idle: our own leak, fixed
- The host died after sitting idle for about 7 hours with "Memory allocation for 54884 bytes failed" (game log;
  dump save failed for the same reason). Cause: `sodlib.Proc.lua()` / `console()` / `run_code()` allocated a
  buffer in the game per call and never freed it. Each `VirtualAllocEx` reserves at least 64 KB of the 32-bit
  process's address space, and `hud_relay` calls `lua()` every 0.5 s (about 7.5 MB/min). This probably also
  explains the older "long sessions degrade" observations.
- Fix: `Proc.free()` and the per-call buffers are released after the call (kept only on a timeout, when the game
  may still use them). `local/vm_watch.py` (address space per process): after the fix, flat at ~636 allocations
  / 1.3 GB on the host over 5 minutes with the relay running.
- Side note: the host log grows by gigabytes per day from "Entity <her id> is not part of the community"
  (every frame); it goes away once she has a survivor record.

### 2026-10-26 - test-harness side effects in zombie fights (not co-op bugs)
- `goal_check.stand_up()` (forcing her host state machine to idle while she is knocked down) leaves the zombies
  believing she is still down: they cycle `Combat.*.Strike.TargetDown.*` and walk loops without moving, and her
  health does not change. Moving her a few metres frees them (they then wander off). Prefer a fresh group and
  `SetHealth` only (`local/reaction_ab.py`, `local/contact_ab.py`). `stand_up` stays for the fatal-landing case
  only.
- `SetHealth(100)` reads back 90 / 80 after heavy damage: max health drops with injuries (game rule), not a
  sync problem.
- Zombies teleported onto her show `Reaction.Basic` (light hit reactions) for a while; around the host's player
  they never do. In steady state (15 s later) there were 0 reaction samples and `peer_melee` fired 0 times, so it
  is the arrival push of her teleport, not phantom hits. zombie_targets' injected "Hero" contact makes no
  difference (A/B, radius 1.4 m vs 0).
- Fresh-boot baseline (`reaction_ab.py`): the host's player among a group goes 90 -> 1 in 15 s; she goes
  100 -> 74 with him 20 m away. The zombie spawner around the test sites runs dry after ~30 min of tests (reboot).

### 2026-10-26 - far combat works; the "never attack" result was a test artifact
- Fresh boot, `local/far_strikes.py` (her among a group, host's player moved, 4 s settle, 15 s): with him 300 m
  away her health went 100 -> 86 and 100 -> 72; with him 20 m away 100 -> 17 and 94 -> 2. Far fights are weaker
  because fewer zombies stay around her (about half the zombie-samples within 4 m), not because combat is
  blocked. The damage is the game's own: zombie_targets' fallback strike hook fired 0 times in all runs.
- Teleport hitch: for 2-3 s after the host's player jumps 300 m the host streams the new area and runs at about
  8 fps, and each frame advances at most about one normal frame of game time, so every zombie's animation clock
  (UL time-in-state, blackboard +0x1340) runs at about 0.12x. Measure only after a settle delay
  (`local/anim_rate.py`). Animation-state time: actor vt+0x158 -> vt+0x16C -> state +0x28, written each frame
  at 0x81FBD4 from the animation's normalized time.
- The earlier "0 strikes in 20 s" run was in a long session where her host state machine was stuck in
  `Stunned.Basic.Down` and her health read a constant 60/80 even after `SetHealth(100)`. Reboot before judging
  combat.
- Console variables `ca_ForceUpdateSkeletons`, `g_forceFastUpdate` and `es_VisCheckForUpdate` do not change at
  runtime (`System.SetCVar` and the console both leave them at the defaults 0/0/1), so the earlier "not
  caused by" checks with them proved nothing.
- Condition snapshots: the UL flag buffer (0x1F36D60) is shared by all records. Each evaluation writes only the
  first `[0x1F36D58]` flags, and which condition sits at which index depends on the active states. Compare by
  (condition type, data), not by index (`local/zcomb_conds.py`; `cond_snap` now keeps the flag count and
  arguments).

### 2026-10-26 - final-build zombie checks, animation NaN guard, far-combat gap
- `local/goal_check.py` on fresh boots: her kills counted for her (+1, host +0), the host player's weapon kills
  counted for him (+4, hers +0), zombies take him from 90 to 1 in 15 s, and her from 100 to 64 in 9 s with him
  20 m away.
- Test-harness trap: teleports can leave a player in a fatal-landing state (client) / lying downed (host); zombie
  hits then never apply and `SetHealth` reads oddly (she read 80). `goal_check.stand_up()` resets both players'
  state machines to idle and heals them. Earlier soak "80 (from 100)" readings were this, not damage.
- New client exit: assert `!_isnan(val)` in `CAnimationGraphState::SetInput` (0x7C9F60). `crash_guards.
  anim_input_guard` (both machines) replaces non-finite values with 0.
- (Resolved in the newer entry above: a test artifact.) With the host's player 300 m away, zombies targeting her walk up and stop 2-4 m away in Basic.Move/Taunt
  states and never enter Combat (0 strikes in 20 s; with him at 20 m they close to ~1.5 m and hit her). Not the
  physics range (es_MaxPhysDist 2000), skeleton updates (ca_ForceUpdateSkeletons 1), g_visibilityTimeout (already
  0) or g_forceFastUpdate 1. zombie_targets' strike reach (1.8 m) is not the limit either (2.6 m: no strikes).

### 2026-10-26 - repository housekeeping
- Moved the old handoff notes and chat backup from the parent folder into the repo root (excluded from git via
  `.git/info/exclude`), deleted all logs, created this file. `restart_session.sh` now starts `dismiss_gpu.ps1`.

### 2026-10-24 - host crash root cause: minimap renderer races the name table
- `Proc.trap_faults` caught the host crash on the render thread: access violation in the engine's global
  name-table hash map (0x112700E), reached from the minimap renderer 0x1380B00 asking the UI manager for
  "Class3HUDRadar" every frame (0x1380D7F). The table is not thread-safe; the main thread changes it constantly.
  Explains the hangs (self-linked bucket node), heap-corruption exits (0xC0000374) and dump-less crashes.
  A debugger hides it (timing).
- Fix `crash_guards.radar_cache`: element looked up once on the main thread, renderer uses the pointer. Two
  fresh-boot 40-minute soaks clean on both machines (before: crash within about a minute).
- Not the cause: relay rate/size, pooled zombie characters shared by two slots (engine returns early).

### 2026-10-23 - zombie animation on the client, position matching
- Client zombies were in T-pose: only the host runs their UL state machine. Host clips (0x820BC0 by name) are
  relayed and played on the client; entity ids/names differ between machines, so zombies are matched by
  position; client ghosts the host removed are hidden. A NaN-position assert exit on the client led to
  `crash_guards.nan_guard`.
- Joining player once showed a zombie body on her own screen: the old model relay matched by entity index.

### 2026-10-23 - zombies around her at any distance
- Cull at 0x13AD9C6 (175 m from camera) now uses the nearer player; population pass run a second time around
  her (<=300 m from camera); physics range raised (es_MaxPhysDist 2000). Spawn density is per map area.

### 2026-10-23 - test environment
- Laptop sleep/Modern Standby and display-off break or kill test sessions; run `keep_awake.py`.
- Never place 5-byte hooks inside functions (branch targets); count with `Proc.count_exec` instead.
