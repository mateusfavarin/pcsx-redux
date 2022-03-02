# Lua API

PCSX-Redux features a Lua API that's available through either a direct Lua console, or a Lua editor, both available through the Debug menu.

## Lua engine
The Lua engine that's being used is LuaJIT 2.1.0-beta3. The [Lua 5.1 user manual](https://www.lua.org/manual/5.1/) and [LuaJIT user manual](https://luajit.org/extensions.html) are recommended reads. In particular, the bindings heavily make use of LuaJIT's FFI capabilities, which allows for direct memory access within the emulator's process. This means there is little protection against dramatic crashes the LuaJIT FFI engine can cause into the emulator's process, and the user must pay extra attention while manipulating FFI objects. Despite that, the code tries as much as possible to sandbox what the Lua code does, and will prevent crashes on any recoverable exception, including OpenGL and ImGui exceptions.

## Lua console
All of the messages coming from Lua should display into the Lua console directly. The input text there is a single line execution, so the user can type one-liner Lua statements and get an immediate result.

## Lua editor
The editor allows for more complex, multi-line statements to be written, such as complete functions. The editor will by default auto save its contents on the disc under the filename `pcsx.lua`, which can potentially be a problem if the last statement typed crashed the emulator, as it'll be reloaded on the next startup. It might become necessary to either edit the file externally, or simply delete it to recover from this state.

The auto-execution of the editor permits for rapid development loop, with immediate feedback of what's done.

## API

### Basic Lua
The [LuaJIT extensions](https://luajit.org/extensions.html) are fully loaded, and can be used globally. Most of the [standard Lua libraries](https://www.lua.org/manual/5.1/manual.html#5) are loaded, and are usable. The `require` keyword however doesn't exist at the moment. As a side-effect of Luv, [Lua-compat-5.3](https://github.com/keplerproject/lua-compat-5.3) is loaded.

### Dear ImGui
A good portion of [ImGui](https://github.com/ocornut/imgui) is bound to the Lua environment, and it's possible for the Lua code to emit arbitrary widgets through ImGui. It is advised to consult the [user manual](https://pthom.github.io/imgui_manual_online/manual/imgui_manual.html) of ImGui in order to properly understand how to make use of it. The list of current bindings can be found [within the source code](https://github.com/grumpycoders/pcsx-redux/blob/main/third_party/imgui_lua_bindings/imgui_iterator.inl). Some usage examples will be provided within the case studies.

### OpenGL
OpenGL is bound directly to the Lua API through FFI bindings, loosely inspired and adapted from [LuaJIT-OpenCL](https://github.com/malkia/luajit-opencl
). Some usage examples can be seen in the CRT-Lottes shader configuration page.

### Luv
For network access and interaction, PCSX-Redux uses libuv internally, and this is exposed to the Lua API through [Luv](https://github.com/luvit/luv)

### Zlib
The Zlib C-API is exposed through [FFI bindings](https://github.com/luapower/zlib).

### PCSX-Redux

#### ImGui interaction
PCSX-Redux will periodically try to call the Lua function `DrawImguiFrame` to allow the Lua code to draw some widgets on screen. The function will be called exactly once per actual UI frame draw, which, when the emulator is running, will correspond to the emulated GPU's vsync. If the function throws an exception however, it will be disabled until recompiled with new code.

#### Memory and registers
The Lua code can access the emulated memory and registers directly through some FFI bindings:

- `PCSX.getMemPtr()` will return a `cdata[uint8_t*]` representing up to 8MB of emulated memory. This can be written to, but careful about the emulated i-cache in case code is being written to.
- `PCSX.getRomPtr()` will return a `cdata[uint8_t*]` representing up to 512kB of the BIOS memory space. This can be written to.
- `PCSX.getScratchPtr()` will return a `cdata[uint8_t*]` representing up to 1kB for the scratchpad memory space.
- `PCSX.getRegisters()` will return a structured cdata representing all the registers present in the CPU:

```c
typedef union {
    struct {
        uint32_t r0, at, v0, v1, a0, a1, a2, a3;
        uint32_t t0, t1, t2, t3, t4, t5, t6, t7;
        uint32_t s0, s1, s2, s3, s4, s5, s6, s7;
        uint32_t t8, t9, k0, k1, gp, sp, s8, ra;
        uint32_t lo, hi;
    } n;
    uint32_t r[34];
} psxGPRRegs;

typedef union {
    uint32_t r[32];
} psxCP0Regs;

typedef union {
    uint32_t r[32];
} psxCP2Data;

typedef union {
    uint32_t r[32];
} psxCP2Ctrl;

typedef struct {
    psxGPRRegs GPR;
    psxCP0Regs CP0;
    psxCP2Data CP2D;
    psxCP2Ctrl CP2C;
    uint32_t pc;
} psxRegisters;
```

#### Execution flow
The Lua code has the following 4 API functions available to it in order to control the execution flow of the emulator:

- `PCSX.pauseEmulator()`
- `PCSX.resumeEmulator()`
- `PCSX.softResetEmulator()`
- `PCSX.hardResetEmulator()`

#### Messages
The globals `print` and `printError` are available, and will display logs in the Lua Console. You can also use `PCSX.log` to display a line in the general Log window. All three functions should behave the way you'd expect from a `print` function in Lua.

#### GUI
You can move the cursor within the assembly window and the first memory view using the following two functions:

- `PCSX.GUI.jumpToPC(pc)`
- `PCSX.GUI.jumpToMemory(address[, width])`

#### Breakpoints
If the debugger is activated, and while using the interpreter, the Lua code can insert powerful breakpoints using the following API:

```lua
PCSX.addBreakpoint(address, type, width, cause, invoker)
```

**Important**: the return value of this function will be an object that represents the breakpoint itself. If this object gets garbage collected, the corresponding breakpoint will be removed. Thus it is important to store it somewhere that won't get garbage collected right away.

The only mandatory argument is `address`, which will by default place an execution breakpoint at the corresponding address. The second argument `type` is an enum which can be represented by one of the 3 following strings: `'Exec'`, `'Read'`, `'Write'`, and will set the breakpoint type accordingly. The third argument `width` is the width of the breakpoint, which indicates how many bytes should intersect from the base address with operations done by the emulated CPU in order to actually trigger the breakpoint. The fourth argument `cause` is a string that will be displayed in the logs about why the breakpoint triggered. It will also be displayed in the Breakpoint Debug UI. And the fifth and final argument `invoker` is a Lua function that will be called whenever the breakpoint is triggered. By default, this will simply call `PCSX.pauseEmulator()`. If the invoker returns `false`, the breakpoint will be permanently removed, permitting temporary breakpoints for example.

The returned object will have a few methods attached to it:

- `:disable()`
- `:enable()`
- `:isEnabled()`
- `:remove()`

A removed breakpoint will no longer have any effect whatsoever, and none of its methods will do anything. Remember it is possible for the user to still manually remove a breakpoint from the UI.

## Case studies

### Spyro: Year of the Dragon

By looking up some of the [gameshark codes](https://www.cheatcc.com/psx/codes/spyroyotd.html) for this game, we can determine the following memory addresses:

- `0x8007582c` is the number of lives.
- `0x80078bbc` is the health of Spyro.
- `0x80075860` is the number of unspent jewels available to the player.
- `0x80075750` is the number of dragons Spyro released so far.

With this, we can build a small UI to visualize and manipulate these values in real time:

```lua
-- Declare a helper function with the following arguments:
--   mem: the ffi object representing the base pointer into the main RAM
--   address: the address of the uint32_t to monitor and mutate
--   name: the label to display in the UI
--   min, max: the minimum and maximum values of the slider
--
-- This function is local as to not pollute the global namespace.
local function doSliderInt(mem, address, name, min, max)
  -- Clamping the address to the actual memory space, essentially
  -- removing the upper bank address using a bitmask. The result
  -- will still be a normal 32-bits value.
  address = bit.band(address, 0x1fffff)
  -- Computing the FFI pointer to the actual uint32_t location.
  -- The result will be a new FFI pointer, directly into the emulator's
  -- memory space, hopefully within normal accessible bounds. The
  -- resulting type will be a cdata[uint8_t*].
  local pointer = mem + address
  -- Casting this pointer to a proper uint32_t pointer.
  pointer = ffi.cast('uint32_t*', pointer)
  -- Reading the value in memory
  local value = pointer[0]
  -- Drawing the ImGui slider
  local changed
  changed, value = imgui.SliderInt(name, value, min, max, '%d')
  -- The ImGui Lua binding will first return a boolean indicating
  -- if the user moved the slider. The second return value will be
  -- the new value of the slider if it changed. Therefore we can
  -- reassign the pointer accordingly.
  if changed then pointer[0] = value end
end

-- Utilizing the DrawImguiFrame periodic function to draw our UI.
-- We are declaring this function global so the emulator can
-- properly call it periodically.
function DrawImguiFrame()
  -- This is typical ImGui paradigm to display a window
  local show = imgui.Begin('Spyro internals', true)
  if not show then imgui.End() return end

  -- Grabbing the pointer to the main RAM, to avoid calling
  -- the function for every pointer we want to change.
  -- Note: it's not a good idea to hold onto this value between
  -- calls to the Lua VM, as the memory can potentially move
  -- within the emulator's memory space.
  local mem = PCSX.getMemPtr()

  -- Now calling our helper function for each of our pointer.
  doSliderInt(mem, 0x8007582c, 'Lives', 0, 9)
  doSliderInt(mem, 0x80078bbc, 'Health', -1, 3)
  doSliderInt(mem, 0x80075860, 'Jewels', 0, 65000)
  doSliderInt(mem, 0x80075750, 'Dragons', 0, 70)

  -- Don't forget to close the ImGui window.
  imgui.End()
end
```

You can see this code in action [in this demo video](https://youtu.be/WeHXTLDy5rs).

### Crash Bandicoot

Using exactly the same as above, we can repeat the same sort of cheats for Crash Bandicoot, using the following Lua code:

```lua
local function crash_Checkbox(mem, address, name, value, original)
    address = bit.band(address, 0x1fffff)
    local pointer = mem + address
    pointer = ffi.cast('uint32_t*', pointer)
    local changed
    local check
    local tempvalue = pointer[0]
    if tempvalue == original then check = false end
    if tempvalue == value then check = true else check = false end
    changed, check = imgui.Checkbox(name, check)
    if check then pointer[0] = value else pointer[0] = original end
end

function DrawImguiFrame()
  local show = imgui.Begin('Crash Bandicoot Mods', true)
  if not show then imgui.End() return end
  local mem = PCSX.getMemPtr()
  crash_Checkbox(mem, 0x80027f9a, 'Neon Crash', 0x2400, 0x100c00)
  crash_Checkbox(mem, 0x8001ed5a, 'Unlimited Time Aku', 0x0003, 0x3403)
  crash_Checkbox(mem, 0x8001dd0c, 'Walk Mid-Air', 0x0000, 0x8e0200c8)
  crash_Checkbox(mem, 0x800618ec, '99 Lives at Map', 0x6300, 0x0200)
  crash_Checkbox(mem, 0x80061949, 'Unlock all Levels', 0x0020, 0x00)
  crash_Checkbox(mem, 0x80019276, 'Disable Draw Level', 0x20212400, 0x20210c00)
  imgui.End()
end
```

### Crash Bandicoot - Using Conditional BreakPoints

This example will showcase using the BreakPoints and Assembly UI, as well as using the Lua console to manipulate breakpoints.

Crash Bandicoot 1 has several modes of execution. These modes tell the game what to do, such as which level to load into, or to load back into the map. These modes are passed to the main game loop routine as an argument. Due to this, manually manipulating memory at the right time with the correct value to can be tricky to ensure the desired result.

The game modes are listed here - https://github.com/wurlyfox/crashutils/blob/da21a40a3e8928762eb58b551a54a6e6f8ed73e9/doc/crash/disasm_guide.txt#L131

In Crash 1, there is a level that was included in the game but cut from the final level selection due to difficulty, 'Stormy Ascent'. This level can be accessed only by manipulating the game mode value that is passed to the main game routine. There is a gameshark code that points us to the memory location and value that needs to be written in order to set the game mode to the Story Ascent level.

- `30011DB0 0022` - This is telling us to write the value 0x0022 at memory location `0x8001db0` 0x0022 is the value of the Stormy Ascent level we want to play.

The issue is that GameShark uses a hook to achieve setting this value at the correct time. We will set up a breakpoint to see where the main game routine is.

Setting the breakpoint can be done through the Breakpoint UI or in the Lua console. There is a link to a video at the bottom of the article showing the entire procedure.

Breakpoints can alternatively be set through the Lua console. In PCSX-Redux top menu, click Debug -> Show Lua Console

We are going to add a breakpoint to pause execution when memory address 0x8001db0 is read. This will show where the main game loop is located in memory.

In the Lua console, paste the following hit enter.

```lua
bp = PCSX.addBreakpoint(0x80011db0, 'Read', 1, 'Find main loop')
```

You should see where the breakpoint was added in the Lua console, as well as in the Breakpoints UI. Note that we need to assign the result of the function to a variable to avoid garbage collection.

Now open Debug -> Show Assembly

Start the emulator with Crash Bandicoot 1 SCUS94900

Right before the BIOS screen ends, the emulator should pause. In the assembly window we can see a yellow arrow pointing to `0x80042068`. We can see this is a `lw` instruction that is reading a value from `0x8001db0`. This is the main game loop reading the game mode value from memory!

Now that we know where the main game loop is located in memory, we can set a conditional breakpoint to properly set the game mode value when the main game routine is executed.

This breakpoint will be triggered when the main game loop at `0x80042068` is executed, and ensure the value at `0x80011db0` is set to `0x0022`

In the Lua console, paste the following and hit enter.

```lua
bp = PCSX.addBreakpoint(0x80042068, 'Exec', 4, 'Stormy Ascent', function() PCSX.getMemPtr()[0x11db0] = 0x22 end)
```

We can now disable/remove our Read breakpoint using the Breakpoints UI, and restart the game. Emulation -> Hard Reset

If the Emulator status shows Idle, click Emulation -> Start

Once the game starts, instead of loading into the main menu, you should load directly into the Stormy Ascent level.

You can see this in action [in this demo video](https://youtu.be/BczviiXUYOY).