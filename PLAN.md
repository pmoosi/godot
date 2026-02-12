# 2D PointLight `render_first` Plan

## Scope
Add a boolean `render_first` flag so point lights marked true are processed before normal point lights in 2D canvas lighting.

Constraints:
- Prioritization is applied only when building final `canvas_lights` chain (second pass).
- First-pass global light cull remains unchanged.
- No sorting.
- No shadow-ordering changes.
- No directional-light behavior changes.

## Required Data Flow
1. `Light2D` property setter writes node state and calls:
   - `RS::canvas_light_set_render_first(light_rid, value)`.
2. RenderingServer forwards that call to renderer implementation.
3. Renderer writes flag into `RendererCanvasRender::Light::render_first`.
4. `RendererViewport::_draw_viewport()` reads that flag in second pass and builds:
   - preferred chain,
   - normal chain,
   - final chain = preferred + normal.

If any step is missing, feature does not work.

## Implementation Checklist

1. Scene API
- File: `scene/2d/light_2d.h`
  - add `bool render_first = false;`
  - add `set_render_first(bool)`, `is_render_first() const`.
- File: `scene/2d/light_2d.cpp`
  - implement setter/getter.
  - setter must call `RS::canvas_light_set_render_first(...)`.
  - bind methods in `_bind_methods()`.
  - add `ADD_PROPERTY` entry (`render_first`).
  - optional: push default to RS in ctor after `canvas_light_create()`.

2. RenderingServer bridge
- File: `servers/rendering/rendering_server.h`
  - add virtual `canvas_light_set_render_first(RID, bool)`.
- File: `servers/rendering/rendering_server_default.h`
  - add forward macro `FUNC2(canvas_light_set_render_first, RID, bool)`.
- File: `servers/rendering/rendering_server.cpp`
  - bind method in `_bind_methods()`.

3. Renderer-side light state
- File: `servers/rendering/renderer_canvas_render.h`
  - add `bool render_first` to `RendererCanvasRender::Light`.
  - default initialize to `false`.
- File: `servers/rendering/renderer_canvas_cull.h`
  - declare `canvas_light_set_render_first(RID, bool)`.
- File: `servers/rendering/renderer_canvas_cull.cpp`
  - implement setter: lookup RID in `canvas_light_owner`, assign `clight->render_first`.

4. Final chain prioritization (selected location)
- File: `servers/rendering/renderer_viewport.cpp`
  - in second pass where `canvas_lights` is built from `lights`:
    - keep current layer-range filter.
    - split into two transient lists:
      - `canvas_lights_pref` if `ptr->render_first`,
      - `canvas_lights_norm` otherwise.
    - concatenate preferred before normal.
  - do not change first-pass `lights` build.
  - do not change shadow list logic.

5. Docs
- File: `doc/classes/Light2D.xml`
  - add member docs for `render_first`.
  - state:
    - affects point-light processing order only,
    - relative order among multiple preferred lights is not guaranteed.

## Complexity / Performance
- Per frame:
  - first pass unchanged.
  - second pass adds one branch and two head pointers; still linear.
  - no sort, no extra standalone traversal.
- Mutation time:
  - O(1) property update path.

## Verification
1. Compile:
- Ensure RS interface additions compile (no virtual/forwarding mismatches).

2. Behavior:
- Scene with overlapping non-commutative point lights (`Subtract`/`Mix`).
- With one light `render_first=true`, verify visual order is deterministic.
- Verify layer filtering still works (priority only where eligible).
- Verify >15-light case: preferred lights are packed before normal ones.

3. Regression:
- Directional lights unchanged.
- Shadows unchanged.
- No crashes with interpolation on/off.

4. Perf sanity:
- Profile `Cull 2D Lights`.
- Confirm no sort hotspot and only small constant-factor overhead in second pass.

## Non-goals
- No general priority sorting.
- No guaranteed total order for all lights.
- No shadow-ordering policy changes.
