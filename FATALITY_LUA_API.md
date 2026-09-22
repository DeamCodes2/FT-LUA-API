# Fatality Legacy Lua API Reference

> Source-reviewed documentation for the Lua API in this Fatality legacy CS:GO source tree.
>
> This reference follows the **runtime implementation** in `lua_new/lua/` rather than old comments where they disagree with the code.

## Quick start

Lua scripts are loaded from:

```text
fatality/scripts/*.lua
```

Libraries used with `require()` are loaded from:

```text
fatality/scripts/lib/*.lua
```

Optional metadata is read from the first few lines:

```lua
--.name Example Script
--.description Example using the current Fatality Lua API
--.author Phillip
```

Callbacks are defined as global Lua functions. There is no `set_event_callback` API:

```lua
local white = render.color(255, 255, 255, 255)

function on_paint()
    render.text(
        render.font_control,
        20, 20,
        "hello from fatality",
        white
    )
end
```

### Current naming

This source uses:

```lua
render.get_screen_size()
render.text(...)
gui.get_config_item(...)
entities.get_entity(...)
engine.is_in_game()
```

It does **not** use Gamesense-style names such as `client.screen_size()` or `renderer.text()`.

---

# Callbacks

## `on_paint()`

Called during the Lua render pass.

Drawing primitives, texture stack operations, clipping and UV stack functions are guarded and must be called from `on_paint()`.

```lua
function on_paint()
    local w, h = render.get_screen_size()

    render.text(
        render.font_control,
        w / 2, h / 2,
        "CENTER",
        render.color("#FFFFFF"),
        render.align_center,
        render.align_center
    )
end
```

## `on_paint_traverse()`

Called from the PaintTraverse hook. No arguments.

## `on_frame_stage_notify(stage, before)`

Called twice around frame-stage processing.

- `stage`: one of the `csgo.frame_*` constants.
- `before`: `true` before the source's stage handling and `false` after it.

```lua
function on_frame_stage_notify(stage, before)
    if stage == csgo.frame_render_start and before then
        -- ...
    end
end
```

## `on_setup_move(cmd, send_packet)`

Called early in CreateMove.

- `cmd`: `csgo.user_cmd`
- `send_packet`: current boolean value

`send_packet` is pushed as a Lua boolean, not a writable reference.

## `on_create_move(cmd, send_packet)`

Called near the end of CreateMove.

- `cmd`: `csgo.user_cmd`
- `send_packet`: current boolean value

Changes made through the command object affect the current command.

```lua
function on_create_move(cmd, send_packet)
    local forward, side = cmd:get_move()
end
```

## `on_run_command(cmd)`

Receives a `csgo.user_cmd`.

In this source the normal RunCommand-hook invocation is commented out. The active invocation happens in the anti-aim processing path, so this is not a generic once-per-command callback.

## `on_level_init()`

Called after level initialization. No arguments.

## `on_input(message, wparam, lparam)`

Receives raw Win32 window-message data.

```lua
function on_input(message, wparam, lparam)
end
```

## `on_game_event(event)`

Called for every game event seen by the listener.

`event` is a `csgo.event`.

```lua
function on_game_event(e)
    if e:get_name() == "player_hurt" then
        print("damage: ", e:get_int("dmg_health"))
    end
end
```

### Event-specific callbacks

The engine also creates callbacks named:

```text
on_<game event name>
```

Examples:

```lua
function on_player_hurt(e)
    print(e:get_int("dmg_health"))
end

function on_bullet_impact(e)
    local x = e:get_float("x")
    local y = e:get_float("y")
    local z = e:get_float("z")
end
```

## `on_shutdown()`

Called when a Lua script is unloaded/shut down.

## `on_config_load(config_name)`

Called after a Fatality config is loaded.

## `on_config_save(config_name)`

Called after a Fatality config is saved.

## `on_console_input(command)`

Called when an unrestricted console command passes through the hook.

## `on_shot_registered(shot)`

Receives a table describing a registered shot.

| Field | Type | Meaning |
|---|---|---|
| `manual` | boolean | Manual/non-aimbot style shot flag |
| `secure` | boolean | First safety threshold |
| `very_secure` | boolean | Full safety threshold |
| `result` | string | `hit`, `resolve`, `spread`, `extrapolation`, `anti-exploit`, `server correction`, or `miss` depending on path |
| `target` | integer | Target entity index, sometimes `-1` |
| `tick` | integer | Shot tick |
| `backtrack` | integer | Backtrack ticks |
| `hitchance` | number | Recorded hitchance |
| `client_hitgroup` | integer | Client-selected hitgroup |
| `client_damage` | integer | Client-estimated damage |
| `server_hitgroup` | integer | Server-reported hitgroup |
| `server_damage` | integer | Server-reported damage |
| `shotpos` | `vec3` | Shot origin |
| `client_hitpos` | `vec3` | Client impact/end position |
| `server_hitpos` | `vec3` | Server hit position or zero vector |
| `client_impacts` | table of `vec3` | Client penetration/impact positions |
| `server_impacts` | table of `vec3` | Recorded server impact positions |

## `on_esp_flags(player_index)`

Expected to return a table of values made with `render.esp_flag()`.

```lua
function on_esp_flags(index)
    local ent = entities.get_entity(index)

    if not ent or not ent:is_alive() then
        return {}
    end

    return {
        render.esp_flag("LUA", render.color(255, 120, 200))
    }
end
```

## `on_draw_model_execute(draw_original, entity_index, model_name)`

Called during model rendering.

- `draw_original`: function that invokes original DrawModelExecute for the current model.
- `entity_index`: current model's entity index.
- `model_name`: model path/name.

```lua
function on_draw_model_execute(draw_original, entity_index, model_name)
    -- apply material override if wanted
    -- draw_original()
end
```

## `on_do_post_screen_space_events()`

The forward is in the discovery list, but this source tree does not contain an invocation for it. Treat it as unwired unless a hook call is added.

---

# `render`

## Colors

```lua
render.color(r, g, b[, a]) -> color
render.color("#RRGGBB") -> color
render.color("#RRGGBBAA") -> color
```

A color is:

```lua
{ r = 255, g = 255, b = 255, a = 255 }
```

Examples:

```lua
local white = render.color(255, 255, 255, 255)
local accent = render.color("#EB055AFF")
```

## Constants

### Alignment

```lua
render.align_left
render.align_top
render.align_center
render.align_right
render.align_bottom
```

### Rounded rectangle flags

```lua
render.top_left
render.top_right
render.bottom_left
render.bottom_right
render.top
render.bottom
render.left
render.right
render.all
```

### Font flags

```lua
render.font_flag_shadow
render.font_flag_outline
```

### Built-in fonts

```lua
render.font_tab
render.font_indicator
render.font_control
render.font_esp
```

### Easing

```lua
render.linear
render.ease_in
render.ease_out
render.ease_in_out
render.elastic_in
render.elastic_out
render.elastic_in_out
render.bounce_in
render.bounce_out
render.bounce_in_out
```

## Fonts

```lua
render.create_font(path, size[, flags[, from[, to]]]) -> font_id
render.create_font_gdi(name, size[, flags[, from[, to]]]) -> font_id
render.create_font_stream(bytes, size[, flags[, from[, to]]]) -> font_id
```

Example:

```lua
local verdana = render.create_font_gdi(
    "Verdana",
    12,
    render.font_flag_shadow
)
```

## Textures

```lua
render.create_texture(path) -> texture_id
render.create_texture_svg(path_or_contents, target_height) -> texture_id
render.create_texture_stream(byte_table) -> texture_id
render.create_texture_bytes(pointer, size) -> texture_id
render.create_texture_rgba(pointer, width, height, stride) -> texture_id
render.get_texture_size(texture_id) -> width, height
```

`create_texture_bytes` and `create_texture_rgba` are native-memory APIs.

## Texture / clip / UV stacks

`on_paint()` only:

```lua
render.push_texture(texture_id)
render.pop_texture()

render.push_clip_rect(x1, y1, x2, y2[, intersect])
render.pop_clip_rect()

render.push_uv(x1, y1, x2, y2)
render.pop_uv()
```

## Drawing

`on_paint()` only:

```lua
render.rect_filled(x1, y1, x2, y2, color)
render.rect(x1, y1, x2, y2, color)
render.rect_filled_rounded(x1, y1, x2, y2, color, rounding[, sides])

render.rect_filled_multicolor(
    x1, y1, x2, y2,
    top_left, top_right, bottom_right, bottom_left
)

render.line(x1, y1, x2, y2, color)
render.line_multicolor(x1, y1, x2, y2, color1, color2)

render.triangle(x1, y1, x2, y2, x3, y3, color)
render.triangle_filled(x1, y1, x2, y2, x3, y3, color)
render.triangle_filled_multicolor(
    x1, y1, x2, y2, x3, y3,
    color1, color2, color3
)

render.circle(
    x, y, radius, color
    [, thickness[, segments[, fill[, rotation]]]]
)

render.circle_filled(
    x, y, radius, color
    [, segments[, fill[, rotation]]]
)

render.text(
    font_id, x, y, text, color
    [, horizontal_align[, vertical_align]]
)
```

## Measurements

```lua
render.get_screen_size() -> width, height
render.get_text_size(font_id, text) -> width, height
```

## ESP flags

```lua
render.esp_flag(text, color) -> render.esp_flag
```

## Animators

Float:

```lua
local anim = render.create_animator_float(initial, duration[, interpolation])

anim:direct(to)
anim:direct(from, to)
anim:get_value() -> number
```

Color:

```lua
local anim = render.create_animator_color(
    initial_color,
    duration
    [, interpolation[, interpolate_hue]]
)

anim:direct(to_color)
anim:direct(from_color, to_color)
anim:get_value() -> color
```

---

# `gui`

## GUI paths

Paths are case-insensitive and separated with `>`.

A lookup path normally includes the control:

```text
TAB>SUBTAB>CHILD>CONTROL
```

Some areas include a nested subtab:

```text
TAB>SUBTAB>SUBSUBTAB>CHILD>CONTROL
```

When adding a new control, the path points to the target container/child and does not include the new control's name.

## Add controls

```lua
gui.add_checkbox(name, path) -> cfg.value
gui.add_slider(name, path, min, max, step) -> cfg.value
gui.add_combo(name, path, {"One", "Two"}) -> cfg.value
gui.add_multi_combo(name, path, {"One", "Two"}) -> cfg.value, cfg.value, ...
gui.add_button(name, path, callback)
gui.add_textbox(name, path) -> cfg.value
gui.add_listbox(name, path, items_to_show, search_bar, items) -> cfg.value
```

The actual slider argument order is:

```lua
gui.add_slider(name, path, min, max, step)
```

## Subcontrols

```lua
gui.add_keybind(path)
gui.add_colorpicker(path, alpha_bar[, default_color]) -> cfg.value
```

## Existing controls

```lua
gui.get_config_item(path) -> cfg.value
gui.get_keybind(path) -> key_code, mode

gui.get_listbox_items(path) -> table
gui.set_listbox_items(path, items)

gui.get_combo_items(path) -> table
gui.set_combo_items(path, items)

gui.set_visible(path, visible)
```

The current implementation uses **path first, bool second** for `set_visible`.

## Menu

```lua
gui.is_menu_open() -> boolean
gui.get_menu_rect() -> x1, y1, x2, y2
```

## Keybind modes

```lua
gui.always_on   -- 0
gui.hold        -- 1
gui.toggle      -- 2
gui.force_off   -- 3
```

---

# `cfg.value`

```lua
value:get_int() -> integer
value:get_bool() -> boolean
value:get_float() -> number
value:get_color() -> color
value:get_string() -> string

value:set_int(integer)
value:set_bool(boolean)
value:set_float(number)
value:set_color(color)
value:set_string(string)
```

---

# `input`

```lua
input.is_key_down(vk_code) -> boolean
input.get_cursor_pos() -> x, y
```

Windows virtual-key codes are used.

---

# `engine`

```lua
engine.is_in_game() -> boolean
engine.exec(command)

engine.get_local_player() -> entity_index
engine.get_view_angles() -> pitch, yaw, roll
engine.set_view_angles(pitch, yaw)

engine.get_player_for_user_id(user_id) -> entity_index
engine.get_player_info(entity_index) -> table
```

Player info:

```lua
{
    name = "...",
    user_id = 1,
    steam_id = "...",
    steam_id64 = "...",
    steam_id64_low = 0,
    steam_id64_high = 0
}
```

---

# `entities`

```lua
entities[index] -> csgo.entity | nil

entities.get_entity(index) -> csgo.entity | nil
entities.get_entity_from_handle(handle) -> csgo.entity | nil

entities.for_each(callback)
entities.for_each_z(callback)
entities.for_each_player(callback)
entities.for_each_player_z(callback)
```

---

# `csgo.entity`

```lua
ent:get_index() -> integer
ent:is_valid() -> boolean
ent:is_alive() -> boolean
ent:is_dormant() -> boolean
ent:is_player() -> boolean
ent:is_enemy() -> boolean
ent:get_class() -> string

ent:get_prop(netvar[, index]) -> value(s)
ent:set_prop(netvar, index, value...)

ent:get_hitbox_position(hitbox) -> x, y, z
ent:get_eye_position() -> x, y, z
ent:get_player_info() -> table
ent:get_move_type() -> integer
ent:get_bbox() -> x1, y1, x2, y2
ent:get_weapon() -> csgo.entity | nil
ent:get_esp_alpha() -> number
```

Examples:

```lua
local hp = ent:get_prop("m_iHealth")
local scoped = ent:get_prop("m_bIsScoped")
local x, y, z = ent:get_prop("m_vecOrigin")
```

64-bit integer netvars are returned as strings.

### `set_prop` warning

The current source's vector/vector2 `set_prop` implementation writes supplied components through the same indexed element. Scalar bool/int/float writes are the safer paths unless `entity.cpp` is fixed.

---

# `csgo.user_cmd`

```lua
cmd:get_command_number() -> integer

cmd:get_view_angles() -> pitch, yaw, roll
cmd:set_view_angles(pitch, yaw, roll)

cmd:get_move() -> forward_move, side_move
cmd:set_move(forward_move, side_move)

cmd:get_buttons() -> integer
cmd:set_buttons(button_mask)
```

---

# `csgo.event`

```lua
event:get_name() -> string
event:get_bool(key) -> boolean
event:get_int(key) -> integer
event:get_float(key) -> number
event:get_string(key) -> string
```

String/int/float event keys can also be read through the event object's index:

```lua
function on_player_hurt(e)
    print(e.dmg_health)
end
```

---

# `cvar`

```lua
local cv = cvar["name"]

cv:get_int() -> integer
cv:get_float() -> number
cv:get_string() -> string

cv:set_int(value)
cv:set_float(value)
cv:set_string(value)
```

Some cheat/development/risky cvars require Allow insecure; blocked cvars remain unavailable.

---

# `global_vars`

```lua
global_vars.realtime
global_vars.framecount
global_vars.curtime
global_vars.frametime
global_vars.tickcount
global_vars.interval_per_tick
```

---

# `game_rules`

```lua
game_rules.is_valve_server
game_rules.is_freeze_period
```

---

# `info`

## `info.fatality`

```lua
info.fatality.username
info.fatality.lag_ticks
info.fatality.to_lag
info.fatality.can_fastfire
info.fatality.in_fakeduck
info.fatality.in_slowwalk
info.fatality.allow_insecure
info.fatality.desync
info.fatality.shot_command
info.fatality.run_antiaim
```

## `info.server`

While in game:

```lua
info.server.map_name
info.server.address
info.server.max_players
```

---

# `csgo` constants

Frame stages:

```lua
csgo.frame_undefined
csgo.frame_start
csgo.frame_net_update_start
csgo.frame_net_update_postdataupdate_start
csgo.frame_net_update_postdataupdate_end
csgo.frame_net_update_end
csgo.frame_render_start
csgo.frame_render_end
```

Command buttons:

```lua
csgo.in_attack
csgo.in_jump
csgo.in_duck
csgo.in_forward
csgo.in_back
csgo.in_use
csgo.in_left
csgo.in_move_left
csgo.in_right
csgo.in_move_right
csgo.in_attack2
csgo.in_score
```

---

# `math.vec3` / `vec3`

```lua
local v = math.vec3(10, 20, 30)
local zero = math.vec3()
```

Fields:

```lua
v.x
v.y
v.z
```

They are writable.

Methods:

```lua
v:length() -> number
v:length2d() -> number
v:dist(other) -> number
v:dist2d(other) -> number
v:to2d() -> vec3
v:dot(other) -> number
v:cross(other) -> vec3
v:normalize() -> vec3
v:calc_angle(other) -> vec3
v:unpack() -> x, y, z
```

Operators accept another `vec3` or a number:

```lua
a + b
a - b
a * b
a / b
```

Helpers:

```lua
math.vector_angles(forward) -> vec3
math.angle_vectors(angle) -> forward, right, up
```

---

# `utils`

## Random / bit flags

```lua
utils.random_int(min, max) -> integer
utils.random_float(min, max) -> number
utils.flags(number, ...) -> integer
```

`utils.flags` bitwise-ORs all numeric arguments.

## Timers

```lua
utils.new_timer(delay_seconds, callback) -> utils.timer
utils.run_delayed(delay_seconds, callback)
```

Timer:

```lua
timer:start()
timer:stop()
timer:run_once()
timer:set_delay(seconds)
timer:is_active() -> boolean
```

Timers are serviced at `FRAME_START`.

## Position / time

```lua
utils.world_to_screen(x, y, z) -> screen_x, screen_y | no values
utils.get_rtt() -> number
utils.get_time() -> table
utils.get_unix_time() -> number
```

`utils.get_time()` returns:

```lua
{
    sec = 0,
    min = 0,
    hour = 0,
    month_day = 1,
    month = 1,
    year = 2026,
    week_day = 1,
    year_day = 1
}
```

Current-source quirk: when not in game, `utils.get_rtt()` pushes `0` internally but returns zero Lua values.

## Weapon information

```lua
utils.get_weapon_info(item_definition_index) -> table
```

Fields:

```text
console_name
max_clip1
max_clip2
world_model
view_model
weapon_type
weapon_price
kill_reward
cycle_time
is_full_auto
damage
range
range_modifier
throw_velocity
has_silencer
max_player_speed
max_player_speed_alt
zoom_fov1
zoom_fov2
zoom_levels
```

## Tracing

```lua
utils.trace(from_vec3, to_vec3, skip_entity_index) -> trace
utils.trace_bullet(item_definition_index, from_vec3, to_vec3) -> damage, trace
```

Trace fields:

```text
endpos
fraction
allsolid
startsolid
fractionleftsolid
plane_normal
plane_dist
contents
disp_flags
surface_name
surface_props
surface_flags
ent_index
hitbox
hitgroup
```

Damage helper:

```lua
utils.scale_damage(
    damage,
    item_definition_index,
    hit_group,
    armor,
    heavy_armor,
    helmet
) -> integer
```

## JSON

```lua
utils.json_encode(table) -> string
utils.json_decode(json) -> value/table
```

## Misc

```lua
utils.set_clan_tag(tag)
utils.print_console(text[, color])
utils.print_dev_console(text)
utils.error_print(text)

utils.aes256_encrypt(key, data) -> binary_string
utils.aes256_decrypt(key, data) -> binary_string
utils.base64_encode(data) -> string
utils.base64_decode(data) -> string

utils.load_file(relative_path) -> string
```

## Unsafe / low-level

Requires Allow insecure:

```lua
utils.find_interface(module, interface_name) -> address | nil
utils.find_pattern(module, pattern[, offset]) -> address
```

Aliases:

```lua
client.create_interface == utils.find_interface
client.find_signature == utils.find_pattern
```

## HTTP

Requires Allow insecure:

```lua
utils.http_get(url, headers, callback)
utils.http_post(url, headers, body, callback)
```

The callback receives the response body as a string.

---

# `database`

Safe per-script style storage:

```lua
database.save(filename, string_or_table)
database.load(filename) -> string_or_table
```

Stored under:

```text
fatality/database/
```

Only a simple filename is accepted; absolute/path traversal forms are rejected.

---

# `fs`

```lua
fs.read(path) -> string
fs.read_stream(path) -> {byte, ...}

fs.write(path, string)
fs.write_stream(path, {byte, ...})

fs.remove(path)

fs.exists(path) -> boolean
fs.is_file(path) -> boolean
fs.is_dir(path) -> boolean
fs.create_dir(path)
```

`read`, `read_stream`, `write`, `write_stream`, `remove`, and `create_dir` require Allow insecure.

---

# `zip`

```lua
zip.create() -> zip
zip.open(path) -> zip
zip.open_stream({byte, ...}) -> zip
```

Object:

```lua
archive:read(path) -> string
archive:read_stream(path) -> {byte, ...}

archive:write(path, data)
archive:write_stream(path, {byte, ...})

archive:get_files() -> table
archive:exists(path) -> boolean

archive:save(path)
archive:extract(path, destination)
archive:extract_all(destination)
```

Unsafe filesystem operations require Allow insecure.

`get_files()` entries contain:

```lua
{
    filename = "...",
    date_time = {
        year = 0,
        month = 0,
        day = 0,
        hours = 0,
        minutes = 0,
        seconds = 0
    },
    comment = "...",
    compress_size = 0,
    file_size = 0
}
```

---

# `panorama`

```lua
panorama.eval(javascript[, panel_id]) -> Lua value
```

Requires Allow insecure.

If no panel ID is supplied, the source searches for `CSGOJsRegistration`.

JavaScript numbers, booleans, strings, arrays and plain objects can be converted back to Lua values/tables. JS functions are not converted.

---

# `mat`

```lua
mat.create(name, shader_type, keyvalues_text) -> csgo.material
mat.find(name, texture_group) -> csgo.material
mat.for_each_material(callback)
mat.override_material(material_or_nil)
```

Material:

```lua
material:modulate(color)
material:set_flag(flag, state)
material:get_flag(flag) -> boolean
material:find_var(name) -> csgo.material_var
material:get_name() -> string
material:get_group() -> string
```

Material variables can also be accessed by indexing the material:

```lua
local envmap = material["$envmap"]
```

Material variable:

```lua
var:get_float() -> number
var:set_float(value)

var:get_int() -> integer
var:set_int(value)

var:get_string() -> string
var:set_string(value)

var:get_vector() -> number, ...
var:set_vector(x, y[, z[, w]])
```

Material flag constants:

```lua
mat.var_debug
mat.var_no_debug_override
mat.var_no_draw
mat.var_use_in_fillrate_mode
mat.var_vertexcolor
mat.var_vertexalpha
mat.var_selfillum
mat.var_additive
mat.var_alphatest
mat.var_znearer
mat.var_model
mat.var_flat
mat.var_nocull
mat.var_nofog
mat.var_ignorez
mat.var_decal
mat.var_envmapsphere
mat.var_envmapcameraspace
mat.var_basealphaenvmapmask
mat.var_translucent
mat.var_normalmapalphaenvmapmask
mat.var_needs_software_skinning
mat.var_opaquetexture
mat.var_envmapmode
mat.var_suppress_decals
mat.var_halflambert
mat.var_wireframe
mat.var_allowalphatocoverage
mat.var_alpha_modified_by_proxy
mat.var_vertexfog
```

---

# Global functions

## `print(...)`

Custom Fatality print.

Strings, numbers and booleans are concatenated.

## `require(name)`

Loads:

```text
fatality/scripts/lib/<name>.lua
```

The library should return one value.

## `loadfile(path)`

Custom file loader. Requires Allow insecure.

---

# Sandbox / Allow insecure

The environment removes/restricts various normal Lua/LuaJIT features.

Always removed/restricted include:

```text
getfenv
gcinfo
collectgarbage
newproxy
coroutine
setfenv
_G
ffi.C
ffi.load
ffi.gc
ffi.fill
string.dump
```

When Allow insecure is disabled, the source additionally removes or restricts:

```text
pcall
xpcall
load
loadstring
dofile
rawget
rawset
rawequal
```

and blocks API functions such as HTTP, general filesystem access, Panorama eval, interface lookup, and pattern scanning.

---

# Complete minimal example

```lua
--.name API Example
--.description Basic Fatality API example
--.author Phillip

local white = render.color("#FFFFFFFF")
local accent = render.color(235, 5, 90, 255)

function on_paint()
    local w, h = render.get_screen_size()

    render.rect_filled(20, 20, 210, 70, render.color(20, 20, 24, 210))
    render.rect_filled(20, 20, 210, 22, accent)

    render.text(
        render.font_control,
        28, 34,
        "Fatality Lua API",
        white
    )

    render.text(
        render.font_esp,
        28, 51,
        string.format("%dx%d", w, h),
        white
    )
end

function on_player_hurt(e)
    local victim_userid = e:get_int("userid")
    local victim_index = engine.get_player_for_user_id(victim_userid)
    local info = engine.get_player_info(victim_index)

    if info then
        print("hurt ", info.name, " for ", e:get_int("dmg_health"))
    end
end

function on_shutdown()
    print("script unloaded")
end
```

---

# Current-source quirks

1. `gui.set_visible` is `gui.set_visible(path, bool)`, despite an outdated header comment showing the reverse.
2. `gui.add_slider` is `gui.add_slider(name, path, min, max, step)`.
3. `ent:get_hitbox_position()` and `ent:get_eye_position()` return three numbers, not a `vec3`.
4. `utils.get_time()` returns a table, not a string.
5. `utils.get_rtt()` returns no Lua values while not in game because of its current return-count implementation.
6. `entity:set_prop()` vector writes should be treated as buggy in this source.
7. `on_do_post_screen_space_events` is discovered but not invoked anywhere in this tree.
8. Render primitives are guarded and error outside `on_paint()`.
