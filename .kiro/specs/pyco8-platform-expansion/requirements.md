# Requirements Document

## Introduction

This document defines the requirements for the PYCO-8 Platform Expansion—a major feature set that extends the single-file HTML fantasy console with audio synthesis, collision physics, performance caching, gameplay templates, and a stretch-goal P2P multiplayer layer. All new subsystems integrate through the existing `window.GameEngine` bridge pattern and persist state within the `.pyco8` cartridge JSON format, preserving the mobile-first, single-file architecture.

## Glossary

- **Audio_Synth**: The Web Audio API–based chiptune sound engine responsible for generating waveforms, applying ADSR envelopes, and sequencing multi-channel music patterns.
- **Collision_Engine**: The spatial collision detection subsystem that resolves entity movement against map tiles and other entities using AABB and distance-vector math.
- **Perf_Cache**: The LRU sprite column caching subsystem that stores pre-rendered OffscreenCanvas bitmaps keyed by asset ID, quantized rotation, and scale.
- **Gameplay_Templates**: The engine-level module providing screen transitions, scene management, combat primitives, and procedural generation helpers.
- **P2P_Multiplayer**: The stretch-goal WebRTC data channel subsystem enabling peer-to-peer game state synchronization via room codes.
- **Cartridge**: The `.pyco8` JSON format containing all game data (code, assets, maps, audio definitions, physics metadata, gameplay config).
- **Bridge**: The `window.GameEngine` JavaScript object that exposes engine capabilities to Python scripts running in Brython.
- **Entity**: A registered collision participant with a position, bounding box, solidity flag, and optional event triggers.
- **Trigger_Zone**: A rectangular region that fires an event when an entity enters it.
- **SFX**: A single sound effect definition consisting of waveform type, frequency, duration, ADSR envelope, pitch slide, and vibrato parameters.
- **Pattern**: A multi-channel music sequence defined as note events at a given tempo, played through the tracker sequencer.
- **ADSR_Envelope**: Attack-Decay-Sustain-Release volume shaping applied to oscillator output.
- **LRU**: Least Recently Used eviction policy for the sprite cache.
- **Rotation_Bucket**: A quantized rotation angle used as part of the cache key (e.g., 32 buckets = 11.25° resolution).

## Requirements

### Requirement 1: Sound Effect Playback

**User Story:** As a game developer, I want to trigger chiptune sound effects from my Python script, so that my game has responsive retro audio feedback.

#### Acceptance Criteria

1. WHEN `engine.play_sfx(id, volume)` is called with a valid SFX ID and a volume in the range [0.0, 1.0], THE Audio_Synth SHALL generate and play the defined waveform within one audio frame (~3ms).
2. WHEN an SFX finishes its defined duration, THE Audio_Synth SHALL disconnect the oscillator node from the audio graph within `duration + 0.01` seconds.
3. IF `engine.play_sfx(id)` is called with an invalid SFX ID, THEN THE Audio_Synth SHALL silently ignore the call without raising an error to the Python layer.
4. THE Audio_Synth SHALL support five waveform types: square, triangle, sawtooth, noise, and pulse.
5. WHEN an SFX definition includes a non-zero `pitchSlide` value, THE Audio_Synth SHALL linearly ramp the oscillator frequency over the duration, clamping to the range [20 Hz, 20000 Hz].
6. WHEN an SFX definition includes non-zero `vibratoDepth` (in cents, range [1, 1200]) and `vibratoSpeed` (in Hz, range [0.1, 50.0]), THE Audio_Synth SHALL modulate the oscillator frequency with a low-frequency oscillator at the specified speed in Hz and depth in cents.
7. THE Audio_Synth SHALL shape all SFX output through an ADSR envelope where gain values remain in the range [0.0, 1.0] at all times.
8. IF `engine.play_sfx(id, volume)` is called with a volume value outside [0.0, 1.0], THEN THE Audio_Synth SHALL clamp the volume to the nearest valid bound before playback.
9. THE Audio_Synth SHALL support a maximum of 8 simultaneous SFX voices; WHEN a new SFX is triggered and all voices are active, THE Audio_Synth SHALL stop the oldest playing voice to free a channel for the new sound.
10. IF `engine.play_sfx(id, volume)` is called with a valid SFX ID while the same SFX ID is already playing, THEN THE Audio_Synth SHALL play a new overlapping instance using a separate voice.

### Requirement 2: Music Pattern Sequencer

**User Story:** As a game developer, I want to play multi-channel music patterns, so that my game has looping background music in classic chiptune style.

#### Acceptance Criteria

1. WHEN `engine.play_music(pattern_id, loop)` is called with a valid pattern ID, THE Audio_Synth SHALL begin playback from step 0 at the pattern's defined tempo.
2. WHEN music is already playing and `engine.play_music()` is called, THE Audio_Synth SHALL immediately stop the current pattern before starting the new one with no crossfade or overlap.
3. WHILE a pattern is playing, THE Audio_Synth SHALL advance steps at a rate of `tempo * 4 / 60` steps per second with cumulative timing drift not exceeding 1 millisecond per 60 seconds of playback.
4. WHEN a pattern reaches its final step with looping enabled, THE Audio_Synth SHALL reset the step counter to 0 and continue playback with no silence or gap between the last step and the first step of the next iteration.
5. WHEN a pattern reaches its final step with looping disabled, THE Audio_Synth SHALL stop playback and reset the step counter to 0.
6. WHEN `engine.stop_music()` is called, THE Audio_Synth SHALL stop the sequencer and reset the step counter to 0 regardless of current playback state.
7. THE Audio_Synth SHALL support patterns with 1 to 4 simultaneous channels and 1 to 64 steps per pattern, with all channels advancing to the same step index on each tick.
8. IF `engine.play_music(pattern_id, loop)` is called with a pattern ID that does not exist in the cartridge data, THEN THE Audio_Synth SHALL silently ignore the call without raising an error to the Python layer.
9. WHEN a step in a channel contains no note data (rest step), THE Audio_Synth SHALL produce no sound on that channel for that step while continuing to advance the sequencer.

### Requirement 3: Audio Context Lifecycle

**User Story:** As a game developer, I want audio to work reliably on mobile browsers, so that players hear sound without manual intervention beyond their first interaction.

#### Acceptance Criteria

1. WHEN the first `play_sfx` or `play_music` call occurs and the AudioContext is in a suspended state, THE Audio_Synth SHALL call `AudioContext.resume()` and defer processing the sound request until the resume promise resolves.
2. IF `AudioContext.resume()` fails or does not resolve within 3 seconds, THEN THE Audio_Synth SHALL discard the pending sound request and retry resume on the next audio call.
3. WHEN `engine.set_master_volume(vol)` is called with a value in [0.0, 1.0], THE Audio_Synth SHALL apply the volume scalar to the master GainNode before the next audio frame is rendered.
4. IF `engine.set_master_volume(vol)` is called with a value outside [0.0, 1.0], THEN THE Audio_Synth SHALL clamp the value to the nearest valid bound.
5. WHEN the Audio_Synth is initialized, THE Audio_Synth SHALL set the master volume to 1.0.

### Requirement 4: Collision Movement Resolution

**User Story:** As a game developer, I want my entities to move through the world with automatic collision detection, so that characters cannot walk through walls or solid objects.

#### Acceptance Criteria

1. WHEN `engine.move_entity(entity_id, dx, dy)` is called, THE Collision_Engine SHALL resolve movement along X and Y axes independently to enable wall-sliding behavior.
2. WHEN movement along one axis is blocked by a solid tile, THE Collision_Engine SHALL preserve full movement along the unblocked axis.
3. WHEN movement would place an entity inside a solid tile, THE Collision_Engine SHALL snap the entity position to the nearest tile edge.
4. WHEN movement would place an entity outside the map boundaries, THE Collision_Engine SHALL block the movement as if the boundary were a solid wall.
5. WHEN `engine.move_entity()` is called with an unregistered entity ID, THE Collision_Engine SHALL return `{x:0, y:0, collisions:[], events:[]}` without error.
6. THE Collision_Engine SHALL guarantee that no entity bounding box overlaps any solid tile after `resolveMovement()` completes, regardless of the movement vector magnitude.

### Requirement 5: Entity Registration and Overlap Detection

**User Story:** As a game developer, I want to register game entities and check for overlaps between them, so that I can implement interactions like combat triggers and NPC contact.

#### Acceptance Criteria

1. WHEN `engine.register_entity(id, x, y, w, h, solid)` is called with w > 0 and h > 0, THE Collision_Engine SHALL add the entity to the spatial index for collision participation starting the next frame.
2. WHEN an entity is registered with an ID that already exists, THE Collision_Engine SHALL overwrite the previous registration with the new parameters.
3. WHEN `engine.check_overlap(id_a, id_b)` is called with two registered entities, THE Collision_Engine SHALL return true if and only if their axis-aligned bounding boxes share any area (edge-touching without area overlap returns false).
4. THE Collision_Engine SHALL ensure overlap detection is symmetric: `check_overlap(a, b)` produces the same result as `check_overlap(b, a)` for all entity pairs.
5. WHEN `engine.check_overlap()` is called with an entity ID that is not currently registered (including null or undefined), THE Collision_Engine SHALL return false without raising an error.
6. WHEN `engine.check_overlap(id, id)` is called with the same entity ID for both arguments, THE Collision_Engine SHALL return false.
7. IF `engine.register_entity()` is called with w <= 0 or h <= 0, THEN THE Collision_Engine SHALL ignore the call and not add the entity to the spatial index.
8. WHEN `engine.unregister_entity(id)` is called with a registered entity ID, THE Collision_Engine SHALL remove the entity from the spatial index so that subsequent overlap checks involving that ID return false.

### Requirement 6: Trigger Zone Events

**User Story:** As a game developer, I want to define rectangular trigger zones that fire events when an entity enters them, so that I can implement doors, chests, and area transitions.

#### Acceptance Criteria

1. WHEN `engine.register_trigger(id, x, y, w, h, event_type, payload)` is called, THE Collision_Engine SHALL register a rectangular zone that fires when any entity overlaps it.
2. WHEN an entity enters a trigger zone, THE Collision_Engine SHALL include the trigger event in the `events` array returned by `engine.move_entity()` exactly once per entry.
3. WHILE an entity remains inside a trigger zone without exiting, THE Collision_Engine SHALL NOT re-fire the trigger event.
4. WHEN an entity exits a trigger zone and re-enters it, THE Collision_Engine SHALL fire the trigger event again.
5. THE Collision_Engine SHALL support trigger event types of "door", "chest", "zone", and "custom" with an arbitrary JSON-serializable payload.

### Requirement 7: Sprite Cache Lookup and Storage

**User Story:** As a game developer, I want the engine to cache pre-rendered sprite columns, so that my game maintains smooth framerates on mobile even with dense sprite-stacked maps.

#### Acceptance Criteria

1. WHEN a sprite is rendered at a given asset ID, rotation, and scale, THE Perf_Cache SHALL store the result as an OffscreenCanvas bitmap keyed by asset ID, quantized rotation bucket, and quantized scale.
2. WHEN a cached sprite is requested for an asset/rotation/scale combination that exists in the cache, THE Perf_Cache SHALL return the stored bitmap and mark it as most-recently-used.
3. WHEN a cached sprite is requested for a combination not in the cache, THE Perf_Cache SHALL return null without error.
4. THE Perf_Cache SHALL quantize rotation angles (provided in radians) into a configurable number of discrete buckets ranging from 4 to 360 (default 32), mapping each input angle to the nearest bucket so that angles within the same bucket share cache entries.
5. THE Perf_Cache SHALL produce cache keys deterministically: identical asset ID, quantized rotation bucket, and scale rounded to 1 decimal place always produce the same key, and different combinations of those three components always produce different keys.

### Requirement 8: Cache Eviction and Invalidation

**User Story:** As a game developer, I want the cache to manage its own memory usage, so that my game does not consume unbounded memory on constrained mobile devices.

#### Acceptance Criteria

1. THE Perf_Cache SHALL enforce a maximum cache size of `MAX_CACHE_ENTRIES` entries (default 128) at all times, such that no insertion causes the total entry count to exceed this limit.
2. WHEN the cache reaches capacity and a new entry must be stored, THE Perf_Cache SHALL evict the least-recently-used entry, increment the eviction count, and then insert the new entry.
3. WHEN `invalidateAsset(assetId)` is called with an asset ID that has one or more cache entries, THE Perf_Cache SHALL remove all cache entries for that asset from both the cache map and the LRU tracking structure without modifying the eviction count.
4. IF `invalidateAsset(assetId)` is called with an asset ID that has no entries in the cache, THEN THE Perf_Cache SHALL return without error and without modifying cache state or statistics.
5. WHEN `invalidateAll()` is called, THE Perf_Cache SHALL remove all entries from the cache map and LRU tracking structure and reset hit, miss, and eviction counts to zero.
6. THE Perf_Cache SHALL track hit, miss, and eviction counts and expose them via `getCacheStats()`, which returns an object containing at minimum the current hit count, miss count, eviction count, current cache size, and maximum cache size.

### Requirement 9: Screen Transitions

**User Story:** As a game developer, I want smooth screen-to-screen transition effects, so that scene changes feel polished like classic console games.

#### Acceptance Criteria

1. WHEN `engine.transition(type, duration, callback_name)` is called with a valid type and a duration between 0.1 and 5.0 seconds, THE Gameplay_Templates SHALL render the specified transition animation over the game canvas for the given duration.
2. WHILE a transition is active, THE Gameplay_Templates SHALL cause `engine.is_transitioning()` to return true.
3. WHEN a transition completes, THE Gameplay_Templates SHALL invoke the named Python callback function and set `is_transitioning()` to false.
4. THE Gameplay_Templates SHALL support transition types: "fade", "wipe_left", "wipe_right", "iris_close", and "iris_open".
5. WHILE a transition is active, THE Gameplay_Templates SHALL continue running the game update loop so that background logic can still execute.
6. IF `engine.transition()` is called with a type not in the supported set, THEN THE Gameplay_Templates SHALL ignore the call and not start a transition.
7. IF `engine.transition()` is called while a transition is already active, THEN THE Gameplay_Templates SHALL ignore the call and allow the current transition to complete.
8. IF a transition completes and the named callback function does not exist in the Python scope, THEN THE Gameplay_Templates SHALL set `is_transitioning()` to false without raising an error.

### Requirement 10: Scene Management

**User Story:** As a game developer, I want to register and switch between named game screens with lifecycle callbacks, so that I can organize my game into distinct states (title, gameplay, combat, pause).

#### Acceptance Criteria

1. WHEN `engine.register_screen(id, enter_fn, update_fn, exit_fn)` is called, THE Gameplay_Templates SHALL store the screen definition for later activation, overwriting any previously registered screen with the same ID.
2. WHEN `engine.switch_screen(id, transition_type)` is called and a current screen is active, THE Gameplay_Templates SHALL invoke the current screen's exit callback, perform the specified transition, then invoke the new screen's enter callback and begin calling the new screen's update_fn on each frame.
3. WHEN `engine.switch_screen(id, transition_type)` is called and no screen is currently active, THE Gameplay_Templates SHALL skip the exit callback, perform the specified transition, then invoke the new screen's enter callback and begin calling the new screen's update_fn on each frame.
4. WHEN `engine.get_current_screen()` is called, THE Gameplay_Templates SHALL return the ID of the currently active screen, or null if no screen has been activated.
5. IF `engine.switch_screen(id, transition_type)` is called with an unregistered screen ID, THEN THE Gameplay_Templates SHALL ignore the call without error and maintain the current screen state.
6. WHILE a screen is active, THE Gameplay_Templates SHALL invoke that screen's update_fn once per game loop frame until a screen switch or exit occurs.

### Requirement 11: Cartridge Serialization

**User Story:** As a game developer, I want all new subsystem data (audio, physics, gameplay) to persist in my `.pyco8` cartridge file, so that saving and loading preserves my complete game state.

#### Acceptance Criteria

1. WHEN a cartridge is saved, THE Cartridge SHALL include `audio`, `physics`, and `gameplay` sections alongside existing `code`, `assets`, `palette`, and `map` sections.
2. WHEN a v1.2 cartridge (without audio/physics/gameplay keys) is loaded, THE Cartridge SHALL default the `audio` section to `{sfx: [], patterns: []}`, the `physics` section to `{entityDefs: [], triggers: []}`, and the `gameplay` section to `{screens: [], combatEntities: [], timers: []}`, and load successfully.
3. WHEN a cartridge is loaded or saved, THE Cartridge SHALL validate that all SFX IDs are unique non-negative integers, waveforms are one of the five valid types (square, triangle, sawtooth, noise, pulse), envelope values (attack, decay, sustain, release) are each in [0.0, 5.0] seconds, frequencies are in [20, 20000] Hz, durations are in [0.01, 10.0] seconds, tempos are in [30, 300] BPM, and channel counts are 1–4.
4. THE Cartridge SHALL produce structurally equivalent JSON (identical keys and values regardless of key order) when serialized, deserialized, and re-serialized for all valid cartridge fields.
5. IF a cartridge section contains data that fails validation rules or is not a valid JSON structure for that section, THEN THE Cartridge SHALL load all other sections independently, defaulting the invalid section to its empty default structure.
6. IF validation fails during a save operation, THEN THE Cartridge SHALL reject the save and report which section and field failed validation.

### Requirement 12: Combat System Primitives

**User Story:** As a game developer, I want access to basic RPG combat math primitives, so that I can quickly prototype turn-based or action combat without writing boilerplate from scratch.

#### Acceptance Criteria

1. WHEN `engine.create_combat_entity(id, hp, attack, defense)` is called with numeric stats where hp is in [1, 99999], attack is in [0, 99999], and defense is in [0, 99999], THE Gameplay_Templates SHALL register a combat entity with the specified stats, overwriting any existing entity with the same ID.
2. WHEN `engine.calculate_damage(attacker_id, defender_id)` is called with two registered entity IDs, THE Gameplay_Templates SHALL return a numeric damage value equal to the attacker's attack stat minus the defender's defense stat, floored at a minimum of 1.
3. WHEN `engine.apply_damage(entity_id, amount)` is called with a registered entity ID and a non-negative numeric amount, THE Gameplay_Templates SHALL reduce the entity's HP by the specified amount, clamping the resulting HP at a minimum of 0.
4. IF `engine.calculate_damage()` or `engine.apply_damage()` is called with an entity ID that is not registered, THEN THE Gameplay_Templates SHALL return 0 without raising an error to the Python layer.
5. WHEN `engine.get_combat_entity(id)` is called with a registered entity ID, THE Gameplay_Templates SHALL return an object containing the entity's current hp, attack, and defense stats.

### Requirement 13: Procedural Generation Helpers

**User Story:** As a game developer, I want seeded procedural dungeon generation, so that I can create varied level layouts without hand-designing every room.

#### Acceptance Criteria

1. WHEN `engine.generate_dungeon(width, height, seed)` is called with valid dimensions in the range 5–128 tiles for both width and height, THE Gameplay_Templates SHALL return a 2D grid array where each cell is marked as "wall", "floor", or "corridor", generated deterministically from the specified seed.
2. WHEN the same seed, width, and height are provided, THE Gameplay_Templates SHALL produce an identical dungeon layout every time, with byte-level equality of the returned grid structure.
3. WHEN `engine.spawn_entities_in_region(template_id, count, region)` is called with a region defined as `{x, y, w, h}` in tile coordinates and a count between 1 and 256, THE Gameplay_Templates SHALL distribute entities at floor cells within the specified region using placement derived from the most recently generated dungeon's seed, producing identical positions for identical inputs.
4. IF `engine.generate_dungeon()` is called with width or height outside the range 5–128 or with a non-numeric seed, THEN THE Gameplay_Templates SHALL return an empty grid and not alter any prior dungeon state.
5. IF `engine.spawn_entities_in_region()` is called with a count that exceeds the number of available floor cells in the specified region, THEN THE Gameplay_Templates SHALL place entities only in available floor cells and return the actual number placed.

### Requirement 14: Timer and Delayed Event System

**User Story:** As a game developer, I want engine-managed timers and delayed callbacks, so that I can schedule game events without manual frame-counting.

#### Acceptance Criteria

1. WHEN `engine.set_timer(id, duration, callback_name, repeat)` is called with a duration between 0.016 and 3600 seconds, THE Gameplay_Templates SHALL invoke the named callback within one frame (~16ms) after the specified duration elapses.
2. WHEN a repeating timer fires, THE Gameplay_Templates SHALL restart the timer using the original duration value automatically until `engine.clear_timer(id)` is called, accumulating no drift across repetitions.
3. WHEN `engine.clear_timer(id)` is called, THE Gameplay_Templates SHALL cancel the timer and prevent future invocations of its callback.
4. IF `engine.clear_timer(id)` is called with a non-existent timer ID, THEN THE Gameplay_Templates SHALL silently ignore the call without error.
5. IF `engine.set_timer(id, duration, callback_name, repeat)` is called with an ID that already exists, THEN THE Gameplay_Templates SHALL cancel the existing timer and replace it with the new definition.
6. IF a timer fires and the named callback does not exist or is not callable, THEN THE Gameplay_Templates SHALL skip the invocation without error and, if repeating, continue scheduling subsequent firings.
7. THE Gameplay_Templates SHALL support a maximum of 64 concurrent active timers and silently reject `set_timer` calls that would exceed this limit.

### Requirement 15: P2P Multiplayer Connection (Stretch Goal)

**User Story:** As a game developer, I want to host and join peer-to-peer game sessions using room codes, so that players can share game worlds without a dedicated server.

#### Acceptance Criteria

1. WHEN `engine.host_session(room_code)` is called with a valid room code, THE P2P_Multiplayer SHALL create a WebRTC peer connection, register with the signaling server using the given room code, and invoke the registered success callback once the session is available for peers to join.
2. WHEN `engine.join_session(room_code)` is called with a room code matching an active host, THE P2P_Multiplayer SHALL connect to the host via the signaling server and invoke the registered success callback once the peer connection is established.
3. WHEN a peer connection is established, THE P2P_Multiplayer SHALL provide a bidirectional data channel using unreliable (UDP-like) delivery with a maximum message size of 16 KB per message and support for up to 4 simultaneous peers per session.
4. IF a peer connection attempt fails or times out after 10 seconds, THEN THE P2P_Multiplayer SHALL invoke the registered error callback with the failure reason and allow the game to continue in single-player mode.
5. WHEN `engine.disconnect()` is called, THE P2P_Multiplayer SHALL close all peer connections, notify connected peers of the disconnection, and set the connection state to "disconnected".
6. IF `engine.host_session(room_code)` or `engine.join_session(room_code)` is called with a room code that is not a 4 to 8 character alphanumeric string, THEN THE P2P_Multiplayer SHALL invoke the error callback with a validation failure reason without attempting a connection.
7. WHEN `engine.get_connection_state()` is called, THE P2P_Multiplayer SHALL return the current state as one of: "disconnected", "connecting", "hosting", or "joined".
8. IF a connected peer disconnects unexpectedly, THEN THE P2P_Multiplayer SHALL invoke the registered `on_peer_disconnect` callback with the disconnected peer's ID within 5 seconds of connection loss.

### Requirement 16: P2P State Synchronization (Stretch Goal)

**User Story:** As a game developer, I want to broadcast and receive game state between peers, so that multiplayer interactions reflect each player's actions in near-real-time.

#### Acceptance Criteria

1. WHEN `engine.broadcast_state(entity_id, state_obj)` is called and the serialized state is 1024 bytes or fewer, THE P2P_Multiplayer SHALL send the serialized state to all connected peers.
2. WHEN a peer state message is received and an `on_peer_state` callback is registered, THE P2P_Multiplayer SHALL invoke the callback with the sender's entity ID and deserialized state object.
3. WHEN `engine.send_message(peer_id, type, data)` is called with a valid connected peer ID, THE P2P_Multiplayer SHALL deliver the message to the specified peer only.
4. THE P2P_Multiplayer SHALL rate-limit outgoing messages to a maximum of 30 messages per second per peer, dropping excess messages without queuing.
5. IF `engine.send_message(peer_id, type, data)` is called with a peer ID that is not currently connected, THEN THE P2P_Multiplayer SHALL silently discard the message without error.
6. IF a peer state message is received and no `on_peer_state` callback is registered, THEN THE P2P_Multiplayer SHALL silently discard the message without error.
7. IF `engine.broadcast_state(entity_id, state_obj)` is called and the serialized state exceeds 1024 bytes, THEN THE P2P_Multiplayer SHALL discard the message and invoke the error callback indicating the payload exceeded the size limit.
