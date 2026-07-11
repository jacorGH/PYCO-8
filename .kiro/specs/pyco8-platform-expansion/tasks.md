# Implementation Plan: PYCO-8 Platform Expansion

## Overview

This plan breaks the PYCO-8 platform expansion into incremental implementation tasks within the single `PYCO-8.html` file. Each task builds on previous work, starting with foundational subsystems (audio, physics, cache) and progressing to gameplay templates, cartridge serialization v2.0, and the stretch-goal P2P multiplayer layer. All JavaScript is added directly into the existing `<script>` block following the established `window.GameEngine` bridge pattern.

## Tasks

- [x] 1. Implement Audio Synthesis Engine
  - [x] 1.1 Create AudioSynth class with Web Audio API lifecycle management
    - Add the `AudioSynth` class inside the `<script>` block with AudioContext creation, master GainNode, suspended-state resume handling, and master volume control
    - Implement `setMasterVolume(vol)` with clamping to [0.0, 1.0]
    - Implement voice pool (8 max simultaneous voices) with oldest-voice eviction
    - Wire `engine.set_master_volume()` through the GameEngine bridge
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 1.9_

  - [x] 1.2 Implement SFX definition and playback
    - Implement `defineSfx(id, config)` to store SFX definitions (waveform, frequency, duration, envelope, pitchSlide, vibrato)
    - Implement `playSfx(id, volume)` with oscillator creation for five waveform types (square, triangle, sawtooth, noise, pulse), ADSR envelope shaping, pitch slide via `linearRampToValueAtTime`, and vibrato via LFO modulator
    - Clamp volume to [0.0, 1.0], silently ignore invalid SFX IDs, allow overlapping instances of same SFX
    - Schedule oscillator disconnect at `duration + 0.01` seconds
    - Wire `engine.play_sfx(id, volume)` through the GameEngine bridge
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10_

  - [ ]* 1.3 Write property tests for Audio Synth
    - **Property 1: No Audio Leak** — Verify oscillator disconnects within duration + 0.01s for any valid SFX duration
    - **Property 2: Volume Bounded** — Verify all gain values remain in [0.0, 1.0] for any combination of master volume, SFX volume, and ADSR parameters
    - **Validates: Requirements 1.2, 1.7, 3.2, 3.3**

  - [x] 1.4 Implement Music Pattern Sequencer
    - Implement `definePattern(patternId, channels, tempo, noteData)` for multi-channel pattern storage (1–4 channels, 1–64 steps)
    - Implement `playPattern(patternId, loop)` with accumulator-based step advancement at `tempo * 4 / 60` steps/sec
    - Implement `stopMusic()` to halt playback and reset step to 0
    - Handle loop wrapping (seamless restart at step 0) and non-loop stop
    - Silently ignore invalid pattern IDs, stop current pattern before starting new one
    - Wire `engine.play_music(pattern_id, loop)`, `engine.stop_music()` through the GameEngine bridge
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9_

  - [ ]* 1.5 Write property tests for Music Sequencer
    - **Property 3: Sequencer Timing Accuracy** — For any tempo T and elapsed time t, steps advanced equals `floor(t * T * 4 / 60)`
    - **Property 4: Idempotent Music Stop** — `stop_music()` from any state produces no errors and leaves step at 0
    - **Property 21: Single Active Pattern** — At most one pattern active at any time after any sequence of `play_music()` calls
    - **Validates: Requirements 2.3, 2.6, 2.2**

- [x] 2. Checkpoint - Ensure audio subsystem works
  - Ensure all tests pass, ask the user if questions arise.

- [x] 3. Implement Collision & Physics Engine
  - [x] 3.1 Create CollisionEngine class with entity registration and map collision
    - Add `CollisionEngine` class with entity registry (Map), tile size config, and mapGrid reference
    - Implement `registerEntity(id, x, y, w, h, solid)` with validation (w > 0, h > 0), overwrite on duplicate ID
    - Implement `unregisterEntity(id)` to remove from spatial index
    - Implement `collidesWithMap(box)` checking the OBJECT layer tiles, treating out-of-bounds as solid
    - _Requirements: 5.1, 5.2, 5.7, 5.8, 4.4_

  - [x] 3.2 Implement movement resolution with wall-sliding
    - Implement `resolveMovement(entityId, dx, dy)` with separate X/Y axis resolution
    - Snap to tile edge on collision, preserve movement on unblocked axis
    - Block movement at map boundaries as if solid walls
    - Return `{x, y, collisions, events}` — return `{x:0, y:0, collisions:[], events:[]}` for unregistered entities
    - Wire `engine.move_entity(entity_id, dx, dy)` through the GameEngine bridge
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

  - [ ]* 3.3 Write property tests for Collision Resolution
    - **Property 5: No Solid Tile Penetration** — After resolveMovement, entity bbox never overlaps any solid tile regardless of movement magnitude
    - **Property 6: Wall Slide Preservation** — When only one axis is blocked, full movement along the other axis is preserved
    - **Property 7: Boundary Containment** — After resolution, entity position stays within world boundaries
    - **Validates: Requirements 4.6, 4.1, 4.2, 4.4**

  - [x] 3.4 Implement entity overlap detection
    - Implement `checkOverlap(idA, idB)` with AABB area overlap test (edge-touching returns false)
    - Ensure symmetry: `checkOverlap(a, b) === checkOverlap(b, a)`
    - Return false for unregistered IDs, null/undefined, and same-ID checks
    - Wire `engine.check_overlap(id_a, id_b)` and `engine.register_entity()` / `engine.unregister_entity()` through the GameEngine bridge
    - _Requirements: 5.3, 5.4, 5.5, 5.6_

  - [ ]* 3.5 Write property test for Overlap Symmetry
    - **Property 9: Overlap Symmetry** — For any two entities at any positions, `check_overlap(a, b) === check_overlap(b, a)`
    - **Validates: Requirements 5.4**

  - [x] 3.6 Implement trigger zone system
    - Implement `registerTrigger(id, x, y, w, h, eventType, payload)` with supported types: "door", "chest", "zone", "custom"
    - Track entity-in-zone state to fire events exactly once per entry (not while remaining inside)
    - Re-fire on exit and re-entry
    - Include triggered events in `move_entity()` return `events` array
    - Wire `engine.register_trigger()` through the GameEngine bridge
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ]* 3.7 Write property test for Trigger Determinism
    - **Property 8: Trigger Determinism** — Entering a zone fires the event exactly once per entry; continuous presence does not re-fire; exit+re-enter fires again
    - **Validates: Requirements 6.2, 6.3, 6.4**

- [x] 4. Checkpoint - Ensure collision subsystem works
  - Ensure all tests pass, ask the user if questions arise.

- [x] 5. Implement Performance Cache System
  - [x] 5.1 Create PerfCache class with LRU eviction and rotation quantization
    - Add `PerfCache` class with configurable `MAX_CACHE_ENTRIES` (default 128) and `ROTATION_BUCKETS` (default 32, range 4–360)
    - Implement cache key generation: `${assetId}_${bucket}_${scale.toFixed(1)}`
    - Implement `getCachedSprite(assetId, rotation, scale)` returning OffscreenCanvas or null, updating LRU on hit
    - Implement `cacheSprite(assetId, rotation, scale, canvas)` with LRU eviction when at capacity
    - Track hit, miss, and eviction counts; expose via `getCacheStats()`
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 8.1, 8.2, 8.6_

  - [x] 5.2 Implement cache invalidation and integrate with renderer
    - Implement `invalidateAsset(assetId)` removing all entries for that asset from map + LRU without modifying eviction count
    - Implement `invalidateAll()` clearing all entries and resetting stats to zero
    - Integrate cache into `renderPerspectiveCameraStackedSprite()` — check cache on render, store on miss
    - Hook asset editor save to call `invalidateAsset()` for hot-reload correctness
    - _Requirements: 8.3, 8.4, 8.5_

  - [ ]* 5.3 Write property tests for Cache System
    - **Property 10: Cache Key Determinism** — Same assetId/rotation/scale always produces same key
    - **Property 11: LRU Size Invariant** — Cache size never exceeds MAX_CACHE_ENTRIES after any sequence of inserts
    - **Property 12: LRU Eviction Correctness** — Evicted entry always has the oldest access timestamp
    - **Property 13: Cache Invalidation Safety** — After `invalidateAsset(id)`, no `getCachedSprite(id, ...)` returns data
    - **Validates: Requirements 7.4, 7.5, 8.1, 8.2, 8.3**

- [x] 6. Checkpoint - Ensure cache system works
  - Ensure all tests pass, ask the user if questions arise.

- [x] 7. Implement Gameplay Logic & Templates
  - [x] 7.1 Implement screen transition system
    - Add transition rendering (fade, wipe_left, wipe_right, iris_close, iris_open) drawn over game canvas
    - Implement `startTransition(type, duration, callbackName)` with duration validation [0.1, 5.0]
    - Implement `isTransitioning()` returning true while active
    - Invoke named Python callback on completion; ignore if callback doesn't exist
    - Ignore calls with unsupported type or while transition already active
    - Continue game update loop during transitions
    - Wire `engine.transition()` and `engine.is_transitioning()` through the GameEngine bridge
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8_

  - [x] 7.2 Implement scene/screen management
    - Implement `registerScreen(id, enterFn, updateFn, exitFn)` storing screen definitions, overwriting on duplicate ID
    - Implement `switchScreen(id, transitionType)` with exit callback → transition → enter callback → update loop routing
    - Implement `getCurrentScreen()` returning active screen ID or null
    - Silently ignore switch to unregistered screen ID
    - Wire `engine.register_screen()`, `engine.switch_screen()`, `engine.get_current_screen()` through the GameEngine bridge
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6_

  - [x] 7.3 Implement combat system primitives
    - Implement `createCombatEntity(id, hp, attack, defense)` with validation (hp [1, 99999], attack [0, 99999], defense [0, 99999]), overwrite on duplicate ID
    - Implement `calculateDamage(attackerId, defenderId)` returning `max(1, attack - defense)`, return 0 for unregistered IDs
    - Implement `applyDamage(entityId, amount)` clamping HP at 0, return 0 for unregistered IDs
    - Implement `getCombatEntity(id)` returning {hp, attack, defense}
    - Wire all through the GameEngine bridge
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5_

  - [ ]* 7.4 Write property test for Damage Calculation
    - **Property 18: Damage Calculation Bounds** — For any attack/defense stats, damage = max(1, attack - defense) and HP after apply_damage is always >= 0
    - **Validates: Requirements 12.2, 12.3**

  - [x] 7.5 Implement procedural generation helpers
    - Implement `generateDungeon(width, height, seed)` producing a deterministic 2D grid of "wall"/"floor"/"corridor" cells for dimensions 5–128
    - Use seeded PRNG for deterministic output (same seed = identical grid)
    - Implement `spawnEntitiesInRegion(templateId, count, region)` placing entities at floor cells within region, derived from dungeon seed
    - Return empty grid for invalid dimensions or non-numeric seed; cap spawn at available floor cells
    - Wire through the GameEngine bridge
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5_

  - [ ]* 7.6 Write property tests for Procedural Generation
    - **Property 19: Dungeon Generation Determinism** — Same seed/width/height always produces identical grid
    - **Property 20: Entity Spawn Containment** — All spawned entities have positions within the specified region boundaries
    - **Validates: Requirements 13.2, 13.3**

  - [x] 7.7 Implement timer and delayed event system
    - Implement `setTimer(id, duration, callbackName, repeat)` with duration range [0.016, 3600], max 64 concurrent timers
    - Fire callback within one frame (~16ms) of elapsed duration; accumulate no drift on repeating timers
    - Implement `clearTimer(id)` cancelling the timer; silently ignore non-existent IDs
    - Overwrite existing timer on duplicate ID; skip invocation if callback doesn't exist
    - Integrate timer update into the main game loop
    - Wire `engine.set_timer()` and `engine.clear_timer()` through the GameEngine bridge
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5, 14.6, 14.7_

- [x] 8. Checkpoint - Ensure gameplay templates work
  - Ensure all tests pass, ask the user if questions arise.

- [x] 9. Implement Cartridge Serialization v2.0
  - [x] 9.1 Extend cartridge format with audio, physics, and gameplay sections
    - Modify `assembleCartridgeBundle()` to include `audio: {sfx, patterns}`, `physics: {entityDefs, triggers}`, and `gameplay: {screens, combatEntities, timers}` sections
    - Update `cartVersion` to "2.0.0"
    - Implement backward-compatible `unpackCartridgeBundle()` that defaults missing sections to empty structures when loading v1.2 carts
    - _Requirements: 11.1, 11.2_

  - [x] 9.2 Implement cartridge validation logic
    - Validate SFX IDs are unique non-negative integers, waveforms are valid, envelope values in [0.0, 5.0], frequencies in [20, 20000] Hz, durations in [0.01, 10.0]s, tempos in [30, 300] BPM, channels 1–4
    - On load: default invalid sections to empty structures while loading other sections independently
    - On save: reject save and report which section/field failed validation
    - _Requirements: 11.3, 11.5, 11.6_

  - [ ]* 9.3 Write property tests for Cartridge Serialization
    - **Property 14: Cartridge Round-Trip Fidelity** — `deserialize(serialize(cart))` produces equivalent value for any valid cartridge
    - **Property 15: Backward Compatibility** — Any v1.2 cart (without audio/physics/gameplay keys) loads without error
    - **Property 16: Subsystem Isolation** — Corrupted section in one subsystem does not prevent other sections from loading
    - **Property 17: Cartridge Validation Correctness** — Validator accepts valid data and rejects data violating constraints
    - **Validates: Requirements 11.4, 11.2, 11.5, 11.3**

- [x] 10. Checkpoint - Ensure cartridge serialization works
  - Ensure all tests pass, ask the user if questions arise.

- [x] 11. Implement P2P Multiplayer (Stretch Goal)
  - [x] 11.1 Implement WebRTC connection management
    - Add `P2PMultiplayer` class with connection state machine ("disconnected", "connecting", "hosting", "joined")
    - Implement `hostSession(roomCode)` creating RTCPeerConnection and registering with signaling server
    - Implement `joinSession(roomCode)` connecting to host via signaling server
    - Validate room codes (4–8 alphanumeric chars); invoke error callback on invalid codes
    - Implement `disconnect()` closing all peer connections and notifying peers
    - Handle connection timeout (10s) with error callback; support up to 4 simultaneous peers
    - Implement `getConnectionState()` returning current state string
    - Detect unexpected peer disconnection and invoke `on_peer_disconnect` callback within 5s
    - Wire `engine.host_session()`, `engine.join_session()`, `engine.disconnect()`, `engine.get_connection_state()` through the GameEngine bridge
    - _Requirements: 15.1, 15.2, 15.3, 15.4, 15.5, 15.6, 15.7, 15.8_

  - [x] 11.2 Implement P2P state synchronization
    - Implement `broadcastState(entityId, stateObj)` sending serialized state (≤1024 bytes) to all peers
    - Implement `onPeerState(callback)` for receiving peer state updates
    - Implement `sendMessage(peerId, type, data)` for targeted peer messaging
    - Rate-limit outgoing messages to 30/sec per peer, dropping excess
    - Silently discard messages to disconnected peers or when no callback registered
    - Invoke error callback when broadcast exceeds 1024 byte limit
    - Wire `engine.broadcast_state()`, `engine.send_message()` through the GameEngine bridge
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.5, 16.6, 16.7_

- [x] 12. Final Integration and API Documentation
  - [x] 12.1 Update API guide panel with new engine functions
    - Add all new Python API entries to the docked API panel in the CODE tab: `play_sfx`, `play_music`, `stop_music`, `set_master_volume`, `move_entity`, `register_entity`, `unregister_entity`, `check_overlap`, `register_trigger`, `transition`, `is_transitioning`, `register_screen`, `switch_screen`, `get_current_screen`, `create_combat_entity`, `calculate_damage`, `apply_damage`, `get_combat_entity`, `generate_dungeon`, `spawn_entities_in_region`, `set_timer`, `clear_timer`
    - Add multiplayer API entries if stretch goal is included: `host_session`, `join_session`, `disconnect`, `broadcast_state`, `send_message`, `get_connection_state`
    - _Requirements: 1.1, 2.1, 3.3, 4.1, 5.1, 6.1, 9.1, 10.1, 12.1, 13.1, 14.1, 15.1, 16.1_

  - [x] 12.2 Update default game script to demonstrate new subsystems
    - Modify the default code in `#code-editor` textarea to showcase audio, collision, transitions, and combat primitives working together
    - Include entity registration, trigger zones, SFX playback, and a screen transition example
    - _Requirements: 1.1, 4.1, 6.1, 9.1_

- [ ] 13. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation between major subsystems
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- All code is implemented within the single `PYCO-8.html` file's `<script>` block
- The P2P Multiplayer section (task 11) is a stretch goal and can be deferred
- fast-check is the recommended property-based testing library (per design document)

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["1.3", "1.4", "3.1"] },
    { "id": 3, "tasks": ["1.5", "3.2"] },
    { "id": 4, "tasks": ["3.3", "3.4", "3.6"] },
    { "id": 5, "tasks": ["3.5", "3.7", "5.1"] },
    { "id": 6, "tasks": ["5.2"] },
    { "id": 7, "tasks": ["5.3", "7.1", "7.3", "7.5", "7.7"] },
    { "id": 8, "tasks": ["7.2", "7.4", "7.6"] },
    { "id": 9, "tasks": ["9.1"] },
    { "id": 10, "tasks": ["9.2"] },
    { "id": 11, "tasks": ["9.3", "11.1"] },
    { "id": 12, "tasks": ["11.2"] },
    { "id": 13, "tasks": ["12.1", "12.2"] }
  ]
}
```
