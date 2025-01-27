# Sokol

[![Build Status](https://github.com/floooh/sokol/workflows/build_and_test/badge.svg)](https://github.com/floooh/sokol/actions)

Simple
[STB-style](https://github.com/nothings/stb/blob/master/docs/stb_howto.txt)
cross-platform libraries for C and C++, written in C.

[See what's new](#updates) (**30-Sep-2020** UWP support for sokol_audio.h)

[Live Samples](https://floooh.github.io/sokol-html5/index.html) via WASM.

Cross-platform libraries:

- **sokol\_gfx.h**: 3D-API wrapper (GL + Metal + D3D11)
- **sokol\_app.h**: app framework wrapper (entry + window + 3D-context + input)
- **sokol\_time.h**: time measurement
- **sokol\_audio.h**: minimal buffer-streaming audio playback
- **sokol\_fetch.h**: asynchronous data streaming from HTTP and local filesystem
- **sokol\_args.h**: unified cmdline/URL arg parser for web and native apps

Utility libraries:

- **sokol\_imgui.h**: sokol_gfx.h rendering backend for [Dear ImGui](https://github.com/ocornut/imgui)
- **sokol\_gl.h**: OpenGL 1.x style immediate-mode rendering API on top of sokol_gfx.h
- **sokol\_fontstash.h**: sokol_gl.h rendering backend for [fontstash](https://github.com/memononen/fontstash)
- **sokol\_gfx\_imgui.h**: debug-inspection UI for sokol_gfx.h (implemented with Dear ImGui)
- **sokol\_debugtext.h**: a simple text renderer using vintage home computer fonts
- **sokol\_memtrack.h**: easily track memory allocations in sokol headers

WebAssembly is a 'first-class citizen', one important motivation for the
Sokol headers is to provide a collection of cross-platform APIs with a
minimal footprint on the web platform while still being useful.

All headers are standalone and can be used independently from each other.

Sample code is in a separate repo: https://github.com/floooh/sokol-samples

Command line tools: https://github.com/floooh/sokol-tools

Tiny 8-bit emulators: https://floooh.github.io/tiny8bit/

### Why C:

- easier integration with other languages
- easier integration into other projects
- adds only minimal size overhead to executables

A blog post with more background info: [A Tour of sokol_gfx.h](http://floooh.github.io/2017/07/29/sokol-gfx-tour.html)

# sokol_gfx.h:

- simple, modern wrapper around GLES2/WebGL, GLES3/WebGL2, GL3.3, D3D11 and Metal
- buffers, images, shaders, pipeline-state-objects and render-passes
- does *not* handle window creation or 3D API context initialization
- does *not* provide shader dialect cross-translation (**BUT** there's now an 'official' shader-cross-compiler solution which
seamlessly integrates with sokol_gfx.h and IDEs: [see here for details](https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md)

A triangle in C99 with GLFW and FlextGL:

```c
#define GLFW_INCLUDE_NONE
#include "GLFW/glfw3.h"
#include "flextgl/flextGL.h"
#define SOKOL_IMPL
#define SOKOL_GLCORE33
#include "sokol_gfx.h"

int main() {

    /* create window and GL context via GLFW */
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    GLFWwindow* w = glfwCreateWindow(640, 480, "Sokol Triangle GLFW", 0, 0);
    glfwMakeContextCurrent(w);
    glfwSwapInterval(1);
    flextInit();

    /* setup sokol_gfx */
    sg_setup(&(sg_desc){0});

    /* a vertex buffer */
    const float vertices[] = {
        // positions            // colors
         0.0f,  0.5f, 0.5f,     1.0f, 0.0f, 0.0f, 1.0f,
         0.5f, -0.5f, 0.5f,     0.0f, 1.0f, 0.0f, 1.0f,
        -0.5f, -0.5f, 0.5f,     0.0f, 0.0f, 1.0f, 1.0f
    };
    sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
        .size = sizeof(vertices),
        .content = vertices,
    });

    /* a shader */
    sg_shader shd = sg_make_shader(&(sg_shader_desc){
        .vs.source =
            "#version 330\n"
            "layout(location=0) in vec4 position;\n"
            "layout(location=1) in vec4 color0;\n"
            "out vec4 color;\n"
            "void main() {\n"
            "  gl_Position = position;\n"
            "  color = color0;\n"
            "}\n",
        .fs.source =
            "#version 330\n"
            "in vec4 color;\n"
            "out vec4 frag_color;\n"
            "void main() {\n"
            "  frag_color = color;\n"
            "}\n"
    });

    /* a pipeline state object (default render states are fine for triangle) */
    sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
        .shader = shd,
        .layout = {
            .attrs = {
                [0].format=SG_VERTEXFORMAT_FLOAT3,
                [1].format=SG_VERTEXFORMAT_FLOAT4
            }
        }
    });

    /* resource bindings */
    sg_bindings binds = {
        .vertex_buffers[0] = vbuf
    };

    /* default pass action (clear to grey) */
    sg_pass_action pass_action = {0};

    /* draw loop */
    while (!glfwWindowShouldClose(w)) {
        int cur_width, cur_height;
        glfwGetFramebufferSize(w, &cur_width, &cur_height);
        sg_begin_default_pass(&pass_action, cur_width, cur_height);
        sg_apply_pipeline(pip);
        sg_apply_bindings(&binds);
        sg_draw(0, 3, 1);
        sg_end_pass();
        sg_commit();
        glfwSwapBuffers(w);
        glfwPollEvents();
    }

    /* cleanup */
    sg_shutdown();
    glfwTerminate();
    return 0;
}
```

# sokol_app.h

A minimal cross-platform application-wrapper library:

- unified application entry
- single window or canvas for 3D rendering
- 3D context initialization
- event-based keyboard, mouse and touch input
- supported platforms: Win32, MacOS, Linux (X11), iOS, WASM, Android, UWP
- supported 3D-APIs: GL3.3 (GLX/WGL), Metal, D3D11, GLES2/WebGL, GLES3/WebGL2

A simple clear-loop sample using sokol_app.h and sokol_gfx.h (does not include
separate sokol.c/.m implementation file which is necessary
to split the Objective-C code from the C code of the sample):

```cpp
#include "sokol_app.h"
#include "sokol_gfx.h"
#include "sokol_glue.h"

sg_pass_action pass_action;

void init(void) {
    sg_setup(&(sg_desc){
        .context = sapp_sgcontext()
    });
    pass_action = (sg_pass_action) {
        .colors[0] = { .action=SG_ACTION_CLEAR, .val={1.0f, 0.0f, 0.0f, 1.0f} }
    };
}

void frame(void) {
    float g = pass_action.colors[0].val[1] + 0.01f;
    pass_action.colors[0].val[1] = (g > 1.0f) ? 0.0f : g;
    sg_begin_default_pass(&pass_action, sapp_width(), sapp_height());
    sg_end_pass();
    sg_commit();
}

void cleanup(void) {
    sg_shutdown();
}

sapp_desc sokol_main(int argc, char* argv[]) {
    return (sapp_desc){
        .init_cb = init,
        .frame_cb = frame,
        .cleanup_cb = cleanup,
        .width = 400,
        .height = 300,
        .window_title = "Clear (sokol app)",
    };
}
```

# sokol_audio.h

A minimal audio-streaming API:

- you provide a mono- or stereo-stream of 32-bit float samples which sokol_audio.h forwards into platform-specific backends
- two ways to provide the data:
    1. directly fill backend audio buffer from your callback function running in the audio thread
    2. alternatively push small packets of audio data from your main loop,
    or a separate thread created by you
- platform backends:
    - Windows: WASAPI
    - macOS/iOS: CoreAudio
    - Linux: ALSA
    - emscripten: WebAudio + ScriptProcessorNode (doesn't use the emscripten-provided OpenAL or SDL Audio wrappers)

A simple mono square-wave generator using the callback model:

```cpp
// the sample callback, running in audio thread
static void stream_cb(float* buffer, int num_frames, int num_channels) {
    assert(1 == num_channels);
    static uint32_t count = 0;
    for (int i = 0; i < num_frames; i++) {
        buffer[i] = (count++ & (1<<3)) ? 0.5f : -0.5f;
    }
}

int main() {
    // init sokol-audio with default params
    saudio_setup(&(saudio_desc){
        .stream_cb = stream_cb
    });

    // run main loop
    ...

    // shutdown sokol-audio
    saudio_shutdown();
    return 0;
```

The same code using the push-model

```cpp
#define BUF_SIZE (32)
int main() {
    // init sokol-audio with default params, no callback
    saudio_setup(&(saudio_desc){0});
    assert(saudio_channels() == 1);

    // a small intermediate buffer so we don't need to push
    // individual samples, which would be quite inefficient
    float buf[BUF_SIZE];
    int buf_pos = 0;
    uint32_t count = 0;

    // push samples from main loop
    bool done = false;
    while (!done) {
        // generate and push audio samples...
        int num_frames = saudio_expect();
        for (int i = 0; i < num_frames; i++) {
            // simple square wave generator
            buf[buf_pos++] = (count++ & (1<<3)) ? 0.5f : -0.5f;
            if (buf_pos == BUF_SIZE) {
                buf_pos = 0;
                saudio_push(buf, BUF_SIZE);
            }
        }
        // handle other per-frame stuff...
        ...
    }

    // shutdown sokol-audio
    saudio_shutdown();
    return 0;
}
```

# sokol_fetch.h

Load entire files, or stream data asynchronously over HTTP (emscripten/wasm)
or the local filesystem (all native platforms).

Simple C99 example loading a file into a static buffer:

```c
#include "sokol_fetch.h"

static void response_callback(const sfetch_response*);

#define MAX_FILE_SIZE (1024*1024)
static uint8_t buffer[MAX_FILE_SIZE];

// application init
static void init(void) {
    ...
    // setup sokol-fetch with default config:
    sfetch_setup(&(sfetch_desc_t){0});

    // start loading a file into a statically allocated buffer:
    sfetch_send(&(sfetch_request_t){
        .path = "hello_world.txt",
        .callback = response_callback
        .buffer_ptr = buffer,
        .buffer_size = sizeof(buffer)
    });
}

// per frame...
static void frame(void) {
    ...
    // need to call sfetch_dowork() once per frame to 'turn the gears':
    sfetch_dowork();
    ...
}

// the response callback is where the interesting stuff happens:
static void response_callback(const sfetch_response_t* response) {
    if (response->fetched) {
        // data has been loaded into the provided buffer, do something
        // with the data...
        const void* data = response->buffer_ptr;
        uint64_t data_size = response->fetched_size;
    }
    // the finished flag is set both on success and failure
    if (response->failed) {
        // oops, something went wrong
        switch (response->error_code) {
            SFETCH_ERROR_FILE_NOT_FOUND: ...
            SFETCH_ERROR_BUFFER_TOO_SMALL: ...
            ...
        }
    }
}

// application shutdown
static void shutdown(void) {
    ...
    sfetch_shutdown();
    ...
}
```

# sokol_time.h:

Simple cross-platform time measurement:

```c
#include "sokol_time.h"
...
/* initialize sokol_time */
stm_setup();

/* take start timestamp */
uint64_t start = stm_now();

...some code to measure...

/* compute elapsed time */
uint64_t elapsed = stm_since(start);

/* convert to time units */
double seconds = stm_sec(elapsed);
double milliseconds = stm_ms(elapsed);
double microseconds = stm_us(elapsed);
double nanoseconds = stm_ns(elapsed);

/* difference between 2 time stamps */
uint64_t start = stm_now();
...
uint64_t end = stm_now();
uint64_t elapsed = stm_diff(end, start);

/* compute a 'lap time' (e.g. for fps) */
uint64_t last_time = 0;
while (!done) {
    ...render something...
    double frame_time_ms = stm_ms(stm_laptime(&last_time));
}
```

# sokol_args.h

Unified argument parsing for web and native apps. Uses argc/argv on native
platforms and the URL query string on the web.

Example URL with one arg:

https://floooh.github.io/tiny8bit/kc85.html?type=kc85_4

The same as command line app:

> kc85 type=kc85_4

Parsed like this:

```c
#include "sokol_args.h"

int main(int argc, char* argv[]) {
    sargs_setup(&(sargs_desc){ .argc=argc, .argv=argv });
    if (sargs_exists("type")) {
        if (sargs_equals("type", "kc85_4")) {
            // start as KC85/4
        }
        else if (sargs_equals("type", "kc85_3")) {
            // start as KC85/3
        }
        else {
            // start as KC85/2
        }
    }
    sargs_shutdown();
    return 0;
}
```

See the sokol_args.h header for a more complete documentation, and the [Tiny
Emulators](https://floooh.github.io/tiny8bit/) for more interesting usage examples.

# Updates

- **30-Sep-2020**:
    sokol_audio.h now works on UWP, thanks again to Alberto Fustinoni
    (@albertofustinoni) for the PR!

- **26-Sep-2020**:
    sokol_app.h gained a new function sapp_set_window_title() to change
    the window title on Windows, macOS and Linux. Many thanks to
    @medvednikov for the initial PR!

- **23-Sep-2020**:
    sokol_app.h now has initial UWP support using the C++/WinRT set of APIs.
    Currently this requires "bleeding edge" tools: A recent VS2019 version,
    and a very recent Windows SDK version (at least version 10.0.19041.0).
    Furthermore the sokol_app.h implementation must be compiled as C++17
    (this is a requirement of the C++/WinRT headers). Note that the Win32
    backend will remain the primary and recommended backend on Windows. The UWP
    backend should only be used when the Win32 backend is not an option.
    The [sokol-samples](https://github.com/floooh/sokol-samples) project
    has two new build configs ```sapp-uwp-vstudio-debug``` and
    ```sapp-uwp-vstudio-release``` to build the sokol-app samples for UWP.

    Many thanks to Alberto Fustinoni (@albertofustinoni) for providing
    the initial PR!

    (also NOTE: UWP-related fixes in other sokol headers will follow)

- **22-Sep-2020**:
    A small fix in sokol_app.h's Win32 backend: when a mouse button is pressed,
    mouse input is now 'captured' by calling SetCapture(), and when the last
    mouse button is released, ReleaseCapture() is called. This also provides
    mouse events outside the window area as long as a mouse button is pressed,
    which is useful for windowed UI applicactions (this is not the same as the
    more 'rigorous' and explicit pointer-lock feature which is more useful for
    camera-controls)

- **31-Aug-2020**:
    Internal change: The D3D11/DXGI backend code in sokol_gfx.h and sokol_app.h
    now use the D3D11 and DXGI C++-APIs when the implementation is compiled as
    C++, and the C-APIs when the implementation is compiled as C (before, the C
    API was also used when the implementation is compiled as C++). The new
    behaviour is useful when another header *must* use the D3D11/DXGI C++ APIs
    but should be included in the same compilation unit as sokol_gfx.h an
    sokol_app.h (for example see this PR:
    https://github.com/floooh/sokol/pull/351).

- **24-Aug-2020**:
    The backend-specific callback functions that are provided to sokol_gfx.h
    in the ```sg_setup()``` initialization call now have alternative
    versions which accept a userdata-pointer argument. The userdata-free functions
    still exist, so no changes are required for existing code.

- **02-Aug-2020**:
    - sokol_app.h now has a mouse-lock feature (aka pointer-lock) via two
      new functions ```void sapp_lock_mouse(bool lock)``` and ```bool sapp_mouse_locked(void)```.
      For documentation, please search for 'MOUSE LOCK' in sokol_app.h.
      The sokol-app samples [events-sapp](https://floooh.github.io/sokol-html5/events-sapp.html)
      and [cgltf-sapp](https://floooh.github.io/sokol-html5/cgltf-sapp.html) have been
      updated to demonstrate the feature.
    - sokol_app.h Linux: mouse pointer visibility (via ```void sapp_show_mouse(bool show)```)
      has been implemented for Linux/X11
    - sokol_app.h WASM: mouse wheel scroll deltas are now 'normalized' between
      the different scroll modes (pixels, lines, pages). See this issue:
      https://github.com/floooh/sokol/issues/339. Many thanks to @bqqbarbhg for
      investigating the issue and providing a solution!
    - sokol_app.h now has [better documentation](https://github.com/floooh/sokol/blob/89a3bb8da0a2df843d6cc60a270ddc69f9aa69d6/sokol_app.h#L70)
      what system libraries must be linked on the various platforms (and on Linux two additional libraries must be
      linked now: Xcursor and Xi)

- **22-Jul-2020**: **PLEASE NOTE** cmake 3.18 breaks some of sokol samples when
  compiling with the Visual Studio toolchain because some C files now actually
  compile as C++ for some reason (see:
  https://twitter.com/FlohOfWoe/status/1285996526117040128).  Until this is
  fixed, or I have come up with a workaround, please use an older cmake version
  to build the sokol samples with the Visual Studio compiler.

  (Update: I have added a workaround to fips: https://github.com/floooh/fips/commit/89997b8ebdca6fc9455a5cfe6145eecaa017df49
  which fixes the issue at least for fips projects)

- **14-Jul-2020**:
    - sapp_mouse_shown() has been implemented for macOS (thanks to @slmjkdbtl for
      providing the initial PR!)
    - On macOS, the lower-level functions CGDisplayShowCursor and CGDisplayHideCursor
      are now used instead of the NSCursor class. This is in preparation for the
      'pointer lock' feature which will also use CGDisplay* functions.
    - Calling ```sapp_show_mouse(bool visible)``` no longer 'stacks' (e.g. there's
      no 'hidden counter' underneath anymore, instead calling ```sapp_show_mouse(true)```
      will always show the cursor and ```sapp_show_mouse(false)``` will always
      hide it. This is a different behaviour than the underlying Win32 and
      macOS functions ShowCursor() and CGDisplaShow/HideCursor()
    - The mouse show/hide behaviour can now be tested in the ```events-sapp``` sample
      (so far this only works on Windows and macOS).

- **13-Jul-2020**:
    - On macOS and iOS, sokol_app.h and sokol_gfx.h can now be compiled with
      ARC (Automatic Reference Counting) **disabled** (previously ARC had to be
      enabled).
    - Compiling with ARC enabled is still supported but with a little caveat:
      if you're compiling sokol_app.h or sokol_gfx.h in ObjC mode (not ObjC++
      mode) *AND* ARC is enabled, then the Xcode version must be more recent
      than before (the language feature ```__has_feature(objc_arc_fields)```
      must be supported, which I think has been added in Xcode 10.2, I couldn't
      find this mentioned in any Xcode release notes though). Compiling with
      ARC disabled should also work on older Xcode versions though.
    - Various internal code cleanup things:
        - sokol_app.h had the same 'structural cleanup' as sokol_gfx.h in
          January, all internal state (including ObjC id's) has been merged into
          a single big state structure. Backend specific struct declarations
          have been moved closer together in the header, and
          backend-specific structures and functions have been named more
          consistently for better 'searchability'
        - The 'mini GL' loader in the sokol_app.h Win32+WGL backend has been
          rewritten to use X-Macros (less redundant lines of code)
        - All macOS and iOS code has been revised and cleaned up
        - On macOS a workaround for a (what looks like) post-Catalina
          NSOpenGLView issue has been added: if the sokol_app.h window doesn't
          fit on screen (and was thus 'clamped' by Cocoa) *AND* the
          content-size was not set to native Retina resolution, the initial
          content size was reported as if it was in Retina resolution. This
          caused an empty screen to be rendered in the imgui-sapp demo. The
          workaround is to hook into the NSOpenGLView reshape event at which
          point the reported content size is correct.
        - On macOS and iOS, the various 'view delegate' objects have been
          removed, and rendering happens instead in the subclasses of MTKView,
          GLKView and NSOpenGLView.
        - On macOS and iOS, there's now proper cleanup code in the
          applicationWillTerminate callback (although note that on iOS this
          function isn't guaranteed to be called, because an application can
          also simply be killed by the operating system.

- **22-Jun-2020**: The X11/GLX backend in sokol_app.h now has (soft-)fullscreen
support, bringing the feature on par with Windows and macOS. Many thanks to
@medvednikov for the PR!

- **20-Jun-2020**: Some work to better support older DX10-level GPUs in the
sokol_gfx.h D3D11 backend:
    - sg_make_shader() now by default compiles HLSL shader code as shader model 4.0
      (previously shader model 5.0 which caused problems with some older
      Intel GPUs still in use, see this issue: https://github.com/floooh/sokol/issues/179)
    - A new string item ```const char* d3d11_target``` in ```sg_shader_stage_desc``` now allows
      to pass in the D3D shader model for compiling shaders. This defaults to
      "vs_4_0" for the vertex shader stage and "ps_4_0" for the fragment shader stage.
      The minimal DX shader model for use with the sokol_gfx.h D3D11 backend is
      shader model 4.0, because that's the first shader model supporting
      constant buffers.
    - The *sokol-shdc* shader compiler tool has a new output option ```hlsl4```
      to generate HLSL4 source code and shader model 4.0 byte code.
    - All embedded D3D shader byte code in the sokol utility headers has been
      changed from shader model 5.0 to 4.0

    If you are using sokol_gfx.h with sokol-shdc, please update both at the same time
    to avoid compilation errors caused by the new ```sg_shader_stage_desc.d3d11_target```
    item. The sg_shader_desc initialization code in sokol-shdc has now been made more
    robust to prevent similar problems in the future.

- **14-Jun-2020**: I have added a very simple utility header ```sokol_memtrack.h```
which allows to track memory allocations in sokol headers (number and overall
size of allocations) by overriding the macros SOKOL_MALLOC, SOKOL_CALLOC and
SOKOL_FREE. Simply include ```sokol_memtrack.h``` before the other sokol
header implementation includes to enable memory tracking in those headers
(but remember that the sokol_memtrack.h implementation must only be included
once in the whole project, so this only works when all other sokol header
implementations are included in the same compilation unit).

- **06-Jun-2020**: Some optimizations in the sokol_gfx.h GL backend to avoid
  redundant GL calls in two areas: in the sg_begin_pass() calls when not
  clearing the color- and depth-stencil-attachments, and in sg_apply_bindings()
  when binding textures.  Everything should behave exactly as before, but if
  you notice any problems in those areas, please file a bug. Many thanks to
  @edubart for the PRs!

- **01-Jun-2020**: sokol_app.h now allows to toggle to and from fullscreen
programmatically and to query the current fullscreen state via 2 new
functions: ```sapp_toggle_fullscreen()``` and ```sapp_is_fullscreen()```.
Currently this is only implemented for Windows and macOS (not Linux).
Thanks to @mattiasljungstrom for getting the feature started and providing
the Win32 implementation!

- **28-May-2020**: a small quality-of-life improvement for C++ coders: when the
sokol headers are included into C++, all public API functions which take a
pointer to a struct now have a C++ overload which instead takes a const-ref.
This allows to move the struct initialization right into the function call
just like in C99. For instance, in C99 one can write:
    ```c
    sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
        .size = sizeof(vertices),
        .type = SG_BUFFERTYPE_VERTEXBUFFER,
        .content = vertices
    });
    ```
    In C++ it isn't possible to take the address of an 'adhoc-initialized'
    struct like this, but with the new reference-wrapper functions (and C++20
    designated initialization) this should work now:
    ```cpp
    sg_buffer buf = sg_make_buffer({
        .size = sizeof(vertices),
        .type = SG_BUFFERTYPE_VERTEXBUFFER,
        .content = vertices
    });
    ```
    Many thanks to @garettbass for providing the PR!


- **27-May-2020**: a new utility header [sokol_debugtext.h](https://github.com/floooh/sokol/blob/master/util/sokol_debugtext.h)
for rendering simple ASCII text using vintage home computer fonts via sokol_gfx.h

- **13-May-2020**: a new function in sokol_time.h to round a measured frame time
to common display refresh rates: ```stm_round_to_common_refresh_rate()```.
See the header documentation for the motivation behind this function.

- **02-May-2020**: sokol_app.h: the 'programmatic quit' behaviour on the
web-platform is now more in line with other platforms: calling
```sapp_quit()``` will invoke the cleanup callback function, perform
platform-specific cleanup (like unregistering JS event handlers), and finally
exit the frame loop. In typical scenarios this isn't very useful (because
usually the user will simply close the tab, which doesn't allow to run
cleanup code), but it's useful for situations where the same
code needs to run repeatedly on a web page. Many thanks to @caiiiycuk
for providing the PR!

- **30-Apr-2020**: experimental WebGPU backend and a minor breaking change:
    - sokol_gfx.h: a new WebGPU backend, expect frequent breakage for a while
      because the WebGPU API is still in flux
    - a new header sokol_glue.h, with interop helper functions when specific combinations
      of sokol headers are used together
    - changes in the way sokol_gfx.h is initialized via a new layout of the
      sg_desc structure
    - sokol_gfx.h: a new ```sg_sampler_type``` enum which is required for
      shader creation to tell the WebGPU backend about the sampler data types
      (float, signed int, or unsigned int) used in the shader
    - sokol_app.h: a handful new functions to query default framebuffer attributes (color- and
      depth-buffer pixel formats, and MSAA sample count)
    - sokol_app.h: WebGPU device and swapchain initialization (currently only
      in the emscripten code path)
    - [sokol-shdc](https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md) has
      been updated with WebGPU support (currently outputs SPIRV bytecode), and to output the new
      ```sg_sampler_type``` enum in ```sg_shader_image_desc```
    - [sokol-samples](https://github.com/floooh/sokol-samples/) has a new set of
      backend-specific WebGPU samples, and the other samples have been updated
      for the new sokol-gfx initialization
    - ```pre-webgpu``` tags have been added to the [sokol](https://github.com/floooh/sokol/releases/tag/pre-webgpu), [sokol-samples](https://github.com/floooh/sokol-samples/releases/tag/pre-webgpu), [sokol-tools](https://github.com/floooh/sokol-tools/releases/tag/pre-webgpu)
      and [sokol-tools-bin](https://github.com/floooh/sokol-tools-bin/releases/tag/pre-webgpu) github repositories (in case you need to continue working with
      the older versions)
    - please see this [blog post](https://floooh.github.io/2020/04/26/sokol-spring-2020-update.html)
      for more details

- **05-Apr-2020**: A bugfix in sokol_gl.h, the (fairly recent) optimization for
    merging draw calls contained a bug that could be triggered in an "empty"
    sgl_begin/sgl_end pair (with no vertices recorded inbetween). This could
    lead to the following draw call being rendered with the wrong uniform data.

- **30-Jan-2020**: Some cleanup in sokol_gfx.h in the backend implementation code,
    internal data structures and documentation comments. The public
    API hasn't changed, so the change should be completely invisible
    from the outside.

- **02-Dec-2019**: Initial clipboard support in sokol_app.h for Windows, macOS
    and HTML5. This allows to read and write UTF-8 encoded strings from and
    to the target platform's shared clipboard.

    A 'real-world' example usage is in the [Visual6502 Remix project](https://github.com/floooh/v6502r).

    Unfortunately clipboard support on the HTML5 platform comes with a lot of
    platform-specific caveats which can't be solved in sokol_app.h alone
    because of the restrictions the web platform puts on clipboard access and
    different behaviours and support levels of the various HTML5 clipboard
    APIs. I'm not really happy with the current HTML5 clipboard
    implementation. It sorta works, but it sure ain't pretty :)

    Maybe the situation will improve in a few years when all browsers agree
    on and support the new [permission-based clipboard
    API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard_API).

    For documentation of the clipboard feature, search for CLIPBOARD SUPPORT
    in sokol_app.h

- **08-Sep-2019**: sokol_gfx.h now supports clamp-to-border texture sampling:
    - the enum ```sg_wrap``` has a new member ```SG_WRAP_CLAMP_TO_BORDER```
    - there's a new enum ```sg_border_color```
    - the struct ```sg_image_desc``` has a new member ```sg_border_color border_color```
    - new feature flag in ```sg_features```: ```image_clamp_to_border```

  Note the following caveats:

    - clamp-to-border is only supported on a subset of platforms, support can
    be checked at runtime via ```sg_query_features().image_clamp_to_border```
    (D3D11, desktop-GL and macOS-Metal support clamp-to-border,
    all other platforms don't)
    - there are three hardwired border colors: transparent-black,
    opaque-black and opaque-white (modern 3D APIs have moved away from
    a freely programmable border color)
    - if clamp-to-border is not supported, sampling will fall back to
    clamp-to-edge without a validation warning

  Many thanks to @martincohen for suggesting the feature and providing the initial
D3D11 implementation!

- **31-Aug-2019**: The header **sokol_gfx_cimgui.h** has been merged into
[**sokol_gfx_imgui.h**](https://github.com/floooh/sokol/blob/master/util/sokol_gfx_imgui.h).
Same idea as merging sokol_cimgui.h into sokol_imgui.h, the implementation
is now "bilingual", and can either be included into a C++ file or into a C file.
When included into a C++ file, the Dear ImGui C++ API will be called directly,
otherwise the C API bindings via cimgui.h

- **28-Aug-2019**: The header **sokol_cimgui.h** has been merged into
[**sokol_imgui.h**](https://github.com/floooh/sokol/blob/master/util/sokol_imgui.h).
The sokol_cimgui.h header had been created to implement Dear ImGui UIs from
pure C applications, instead of having to fall back to C++ just for the UI
code. However, there was a lot of code duplication between sokol_imgui.h and
sokol_cimgui.h, so that it made more sense to merge the two headers. The C vs
C++ code path will be selected automatically: When the implementation of
sokol_imgui.h is included into a C++ source file, the Dear ImGui C++ API will
be used. Otherwise, when the implementation is included into a C source file,
the C API via cimgui.h

- **27-Aug-2019**: [**sokol_audio.h**](https://github.com/floooh/sokol/blob/master/sokol_audio.h)
  now has an OpenSLES backend for Android. Many thanks to Sepehr Taghdisian (@septag)
  for the PR!

- **26-Aug-2019**: new utility header for text rendering, and fixes in sokol_gl.h:
    - a new utility header [**sokol_fontstash.h**](https://github.com/floooh/sokol/blob/master/util/sokol_fontstash.h)
      which implements a renderer for [fontstash.h](https://github.com/memononen/fontstash)
      on top of sokol_gl.h
    - **sokol_gl.h** updates:
        - Optimization: If no relevant state between two begin/end pairs has
        changed, draw commands will be merged into a single sokol-gfx draw
        call. This is especially useful for text- and sprite-rendering (previously,
        each begin/end pair would always result in one draw call).
        - Bugfix: When calling sgl_disable_texture() the previously active
        texture would still remain active which could lead to rendering
        artefacts. This has been fixed.
        - Feature: It's now possible to provide a custom shader in the
        'desc' argument of *sgl_make_pipeline()*, as long as the shader
        is "compatible" with sokol_gl.h, see the sokol_fontstash.h
        header for an example. This feature isn't "advertised" in the
        sokol_gl.h documentation because it's a bit brittle (for instance
        if sokol_gl.h updates uniform block structures, custom shaders
        would break), but it may still come in handy in some situations.

- **20-Aug-2019**: sokol_gfx.h has a couple new query functions to inspect the
  default values of resource-creation desc structures:

    ```c
    sg_buffer_desc sg_query_buffer_defaults(const sg_buffer_desc* desc);
    sg_image_desc sg_query_image_defaults(const sg_image_desc* desc);
    sg_shader_desc sg_query_shader_defaults(const sg_shader_desc* desc);
    sg_pipeline_desc sg_query_pipeline_defaults(const sg_pipeline_desc* desc);
    sg_pass_desc sg_query_pass_defaults(const sg_pass_desc* desc);
    ```
  These functions take a pointer to a resource creation desc struct that
  may contain zero-initialized values (to indicate default values) and
  return a new struct where the zero-init values have been replaced with
  concrete values. This is useful to inspect the actual creation attributes
  of a resource.

- **18-Aug-2019**:
    - Pixelformat and runtime capabilities modernization in sokol_gfx.h (breaking changes):
        - The list of pixel formats supported in sokol_gfx.h has been modernized,
          many new formats are available, and some formats have been removed. The
          supported pixel formats are now identical with what WebGPU provides,
          minus the SRGB formats (if SRGB conversion is needed it should be done
          in the pixel shader)
        - The pixel format list is now more "orthogonal":
            - one, two or four color components (R, RG, RGBA)
            - 8-, 16- or 32-bit component width
            - unsigned-normalized (no postfix), signed-normalized (SN postfix),
              unsigned-integer (UI postfix) and signed-integer (SI postfix)
              and float (F postfix) component types.
            - special pixel formats BGRA8 (default render target format on
              Metal and D3D11), RGB10A2 and RG11B10F
            - DXT compressed formats replaced with BC1 to BC7 (BC1 to BC3
              are identical to the old DXT pixel formats)
            - packed 16-bit formats (like RGBA4) have been removed
            - packed 24-bit formats (RGB8) have been removed
        - Use the new function ```sg_query_pixelformat()``` to get detailed
          runtime capability information about a pixelformat (for instance
          whether it is supported at all, can be used as render target etc...).
        - Use the new function ```sg_query_limits()``` to query "numeric limits"
          like maximum texture dimensions for different texture types.
        - The enumeration ```sg_feature``` and the function ```sg_query_feature()```
          has been replaced with the new function ```sg_query_features()```, which
          returns a struct ```sg_features``` (this contains a bool for each
          optional feature).
        - The default pixelformat for render target images and pipeline objects
          now depends on the backend:
            - for GL backends, the default pixelformat stays the same: RGBA8
            - for the Metal and D3D11 backends, the default pixelformat for
            render target images is now BGRA8 (the reason is because
            MTKView's pixelformat was always BGRA8 but this was "hidden"
            through an internal hack, and a BGRA swapchain is more efficient
            than RGBA in D3D11/DXGI)
        - Because of the above RGBA/BGRA change, you may see pixelformat validation
          errors in existing code if the code assumes that a render target image is
          always created with a default pixelformat of RGBA8.
    - Changes in sokol_app.h:
        - The D3D11 backend now creates the DXGI swapchain with BGRA8 pixelformat
          (previously: RGBA8), this allows more efficient presentation in some
          situations since no format-conversion-blit needs to happen.

- **18-Jul-2019**:
    - sokol_fetch.h has been fixed and can be used again :)

- **11-Jul-2019**:
    - Don't use sokol_fetch.h for now, the current version assumes that
      it is possible to obtain the content size of a file from the
      HTTP server without downloading the entire file first. Turns out
      that's not possible with vanilla HTTP when the web server serves
      files compressed (in that case the Content-Length is the _compressed_
      size, yet JS/WASM only has access to the uncompressed data).
      Long story short, I need to go back to the drawing board :)

- **06-Jul-2019**:
    - new header [sokol_fetch.h](https://github.com/floooh/sokol/blob/master/sokol_fetch.h) for asynchronously loading data.
        - make sure to carefully read the embedded documentation
        for making the best use of the header
        - two new samples: [simple PNG file loadng with stb_image.h](https://floooh.github.io/sokol-html5/loadpng-sapp.html) and  [MPEG1 streaming with pl_mpeg.h](https://floooh.github.io/sokol-html5/plmpeg-sapp.html)
    - sokol_gfx.h: increased SG_MAX_SHADERSTAGE_BUFFERS configuration
    constant from 4 to 8.

- **10-Jun-2019**: sokol_app.h now has proper "application quit handling":
    - a pending quit can be intercepted, for instance to show a "Really Quit?" dialog box
    - application code can now initiate a "soft quit" (interceptable) or
      "hard quit" (not interceptable)
    - on the web platform, the standard "Leave Site?" dialog box implemented
      by browsers can be shown when the user leaves the site
    - Android and iOS currently don't have any of those features (since the
      operating system may decide to terminate mobile applications at any time
      anyway, if similar features are added they will most likely have
      similar limitations as the web platform)
  For details, search for 'APPLICATION QUIT' in the sokol_app.h documentation
  header: https://github.com/floooh/sokol/blob/master/sokol_app.h

  The [imgui-highdpi-sapp](https://github.com/floooh/sokol-samples/tree/master/sapp)
  contains sample code for all new quit-related features.

- **08-Jun-2019**: some new stuff in sokol_app.h:
    - the ```sapp_event``` struct has a new field ```bool key_repeat```
    which is true when a keyboard event is a key-repeat (for the
    event types ```SAPP_EVENTTYPE_KEY_DOWN``` and ```SAPP_EVENTTYPE_CHAR```).
    Many thanks to [Scott Lembcke](https://github.com/slembcke) for
    the pull request!
    - a new function to poll the internal frame counter:
    ```uint64_t sapp_frame_count(void)```, previously the frame counter
    was only available via ```sapp_event```.
    - also check out the new [event-inspector sample](https://floooh.github.io/sokol-html5/wasm/events-sapp.html)

- **04-Jun-2019**: All sokol headers now recognize a config-define ```SOKOL_DLL```
  if sokol should be compiled into a DLL (when used with ```SOKOL_IMPL```)
  or used as a DLL. On Windows, this will prepend the public function declarations
  with ```__declspec(dllexport)``` or ```__declspec(dllimport)```.

- **31-May-2019**: if you're working with emscripten and fips, please note the
  following changes:

    https://github.com/floooh/fips#public-service-announcements

- **27-May-2019**: some D3D11 updates:
    - The shader-cross-compiler can now generate D3D bytecode when
    running on Windows, see the [sokol-shdc docs](https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md) for more
details.
    - sokol_gfx.h no longer needs to be compiled with a
    SOKOL_D3D11_SHADER_COMPILER define to enable shader compilation in the
    D3D11 backend. Instead, the D3D shader compiler DLL (d3dcompiler_47.dll)
    will be loaded on-demand when the first HLSL shader needs to be compiled.
    If an application only uses D3D shader byte code, the compiler DLL won't
    be loaded into the process.

- **24-May-2019** The shader-cross-compiler can now generate Metal byte code
for macOS or iOS when the build is running on macOS. This is enabled
automatically with the fips-integration files in [sokol-tools-bin](https://github.com/floooh/sokol-tools-bin),
see the [sokol-shdc docs](https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md) for more
details.

- **16-May-2019** two new utility headers: *sokol_cimgui.h* and *sokol_gfx_cimgui.h*,
those are the same as their counterparts sokol_imgui.h and sokol_gfx_imgui.h, but
use the [cimgui](https://github.com/cimgui/cimgui) C-API for Dear ImGui. This
is useful if you don't want to - or cannot - use C++ for creating Dear ImGui UIs.

    Many thanks to @prime31 for contributing those!

    sokol_cimgui.h [is used
here](https://floooh.github.io/sokol-html5/wasm/cimgui-sapp.html), and
sokol_gfx_cimgui.h is used for the [debugging UI
here](https://floooh.github.io/sokol-html5/wasm/sgl-microui-sapp-ui.html)

- **15-May-2019** there's now an optional shader-cross-compiler solution for
sokol_gfx.h: [see here for details](https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md).
This is "V1.0" with two notable features missing:

    - an include-file feature for GLSL shaders
    - compilation to Metal- and D3D-bytecode (currently
      only source-code generation is supported)

    The [sokol-app samples](https://floooh.github.io/sokol-html5/) have been
    ported to the new shader-cross-compilation, follow the ```src``` and
    ```glsl``` links on the specific sample webpages to see the C- and GLSL-
    source-code.

- **02-May-2019** sokol_gfx.h has a new function ```sg_query_backend()```, this
will return an enum ```sg_backend``` identifying the backend sokol-gfx is
currently running on, which is one of the following values:

    - SG_BACKEND_GLCORE33
    - SG_BACKEND_GLES2
    - SG_BACKEND_GLES3
    - SG_BACKEND_D3D11
    - SG_BACKEND_METAL_MACOS
    - SG_BACKEND_METAL_IOS

    When compiled with SOKOL_GLES3, sg_query_backend() may return SG_BACKEND_GLES2
    when the runtime platform doesn't support GLES3/WebGL2 and had to fallback
    to GLES2/WebGL2.

    When compiled with SOKOL_METAL, sg_query_backend() will return SG_BACKEND_METAL_MACOS
    when the compile target is macOS, and SG_BACKEND_METAL_IOS when the target is iOS.

- **26-Apr-2019** Small but breaking change in **sokol_gfx.h** how the vertex
layout definition in sg_pipeline_desc works:

    Vertex component names and semantics (needed by the GLES2 and D3D11 backends) have moved from ```sg_pipeline_desc``` into ```sg_shader_desc```.

    This may seem like a rather pointless small detail to change, especially
    for breaking existing code, but the whole thing will make a bit more
    sense when the new shader-cross-compiler will be integrated which I'm
    currently working on (here: https://github.com/floooh/sokol-tools).

    While working on getting reflection data out of the shaders (e.g. what
    uniform blocks and textures the shader uses), it occurred to me that
    vertex-attribute-names and -semantics are actually part of the reflection
    info and belong to the shader, not to the vertex layout in the pipeline
    object (which only describes how the incoming vertex data maps to
    vertex-component **slots**. Instead of (optionally) mapping this
    association through a name, the pipeline's vertex layout is now always
    strictly defined in terms of numeric 'bind slots' for **all** sokol_gfx.h
    backends. For 3D APIs where the vertex component slot isn't explicitly
    defined in the shader language (GLES2/WebGL, D3D11, and optionally
    GLES3/GL), the shader merely offers a lookup table how vertex-layout
    slot-indices map to names/semantics (and the underlying 3D API than maps
    those names back to slot indices, which shows that Metal and GL made the
    right choice defining the slots right in the shader).

    Here's how the code changes (taken from the triangle-sapp.c sample):

    **OLD**:
    ```c
    /* create a shader */
    sg_shader shd = sg_make_shader(&(sg_shader_desc){
        .vs.source = vs_src,
        .fs.source = fs_src,
    });

    /* create a pipeline object (default render states are fine for triangle) */
    pip = sg_make_pipeline(&(sg_pipeline_desc){
        /* if the vertex layout doesn't have gaps, don't need to provide strides and offsets */
        .shader = shd,
        .layout = {
            .attrs = {
                [0] = { .name="position", .sem_name="POS", .format=SG_VERTEXFORMAT_FLOAT3 },
                [1] = { .name="color0", .sem_name="COLOR", .format=SG_VERTEXFORMAT_FLOAT4 }
            }
        },
    });
    ```

    **NEW**:
    ```c
        /* create a shader */
        sg_shader shd = sg_make_shader(&(sg_shader_desc){
            .attrs = {
                [0] = { .name="position", .sem_name="POS" },
                [1] = { .name="color0", .sem_name="COLOR" }
            },
            .vs.source = vs_src,
            .fs.source = fs_src,
        });

        /* create a pipeline object (default render states are fine for triangle) */
        pip = sg_make_pipeline(&(sg_pipeline_desc){
            /* if the vertex layout doesn't have gaps, don't need to provide strides and offsets */
            .shader = shd,
            .layout = {
                .attrs = {
                    [0].format=SG_VERTEXFORMAT_FLOAT3,
                    [1].format=SG_VERTEXFORMAT_FLOAT4
                }
            },
        });
    ```

    ```sg_shader_desc``` has a new embedded struct ```attrs``` which
    contains a vertex attribute _name_ (for GLES2/WebGL) and
    _sem_name/sem_index_ (for D3D11). For the Metal backend this struct is
    ignored completely, and for GLES3/GL it is optional, and not required
    when the vertex shader inputs are annotated with ```layout(location=N)```.

    The remaining attribute description members in ```sg_pipeline_desc``` are:
    - **.format**: the format of input vertex data (this can be different
          from the vertex shader's inputs when data is extended during
          vertex fetch (e.g. input can be vec3 while the vertex shader
          expects vec4)
    - **.offset**: optional offset of the vertex component data (not needed
          when the input vertex has no gaps between the components)
    - **.buffer**: the vertex buffer bind slot if the vertex data is coming
          from different buffers

    Also check out the various samples:

    - for GLSL (explicit slots via ```layout(location=N)```): https://github.com/floooh/sokol-samples/tree/master/glfw
    - for D3D11 (semantic names/indices): https://github.com/floooh/sokol-samples/tree/master/d3d11
    - for GLES2: (vertex attribute names): https://github.com/floooh/sokol-samples/tree/master/html5
    - for Metal: (explicit slots): https://github.com/floooh/sokol-samples/tree/master/metal
    - ...and all of the above combined: https://github.com/floooh/sokol-samples/tree/master/sapp

- **19-Apr-2019** I have replaced the rather inflexible render-state handling
in **sokol_gl.h** with a *pipeline stack* (like the GL matrix stack, but with
pipeline-state-objects), along with a couple of other minor API tweaks.

    These are the new pipeline-stack functions:
    ```c
    sgl_pipeline sgl_make_pipeline(const sg_pipeline_desc* desc);
    void sgl_destroy_pipeline(sgl_pipeline pip);
    void sgl_default_pipeline(void);
    void sgl_load_pipeline(sgl_pipeline pip);
    void sgl_push_pipeline(void);
    void sgl_pop_pipeline(void);
    ```

    A pipeline object is created just like in sokol_gfx.h, but without shaders,
    vertex layout, pixel formats, primitive-type and sample count (these details
    are filled in by the ```sgl_make_pipeline()``` wrapper function. For instance
    to create a pipeline object for additive transparency:

    ```c
    sgl_pipeline additive_pip = sgl_make_pipeline(&(sg_pipeline_desc){
        .blend = {
            .enabled = true,
            .src_factor_rgb = SG_BLENDFACTOR_ONE,
            .dst_factor_rgb = SG_BLENDFACTOR_ONE
        }
    });
    ```

    And to render with this, simply call ```sgl_load_pipeline()```:

    ```c
    sgl_load_pipeline(additive_pip);
    sgl_begin_triangles();
    ...
    sgl_end();
    ```

    Or to preserve and restore the previously active pipeline state:

    ```c
    sgl_push_pipeline();
    sgl_load_pipeline(additive_pip);
    sgl_begin_quads();
    ...
    sgl_end();
    sgl_pop_pipeline();
    ```

    You can also load the 'default pipeline' explicitly on the top of the
    pipeline stack with ```sgl_default_pipeline()```.

    The other API change is:

    - ```sgl_state_texture(bool b)``` has been replaced with ```sgl_enable_texture()```
      and ```sgl_disable_texture()```

    The code samples have been updated accordingly:

    - [sgl-sapp.c](https://github.com/floooh/sokol-samples/blob/master/sapp/sgl-sapp.c)
    - [sgl-lines-sapp.c](https://github.com/floooh/sokol-samples/blob/master/sapp/sgl-lines-sapp.c)
    - [sgl-microui-sapp.c](https://github.com/floooh/sokol-samples/blob/master/sapp/sgl-microui-sapp.c)

- **01-Apr-2019** (not an April Fool's joke): There's a new **sokol_gl.h**
util header which implements an 'OpenGL-1.x-in-spirit' rendering API on top
of sokol_gfx.h (vertex specification via begin/end, and a matrix stack). This is
only a small subset of OpenGL 1.x, mainly intended for debug-visualization or
simple tasks like 2D UI rendering. As always, sample code is in the
[sokol-samples](https://github.com/floooh/sokol-samples) project.

- **15-Mar-2019**: various Dear ImGui related changes:
    - there's a new utility header sokol_imgui.h with a simple drop-in
      renderer for Dear ImGui on top of sokol_gfx.h and sokol_app.h
      (sokol_app.h is optional, and only used for input handling)
    - the sokol_gfx_imgui.h debug inspection header no longer
      depends on internal data structures and functions of sokol_gfx.h, as such
      it is now a normal *utility header* and has been moved to the *utils*
      directory
    - the implementation macro for sokol_gfx_imgui.h has been changed
      from SOKOL_IMPL to SOKOL_GFX_IMGUI_IMPL (so when you suddenly get
      unresolved linker errors, that's the reason)
    - all headers now have two preprocessor defines for the declaration
      and implementation (for instance in sokol_gfx.h: SOKOL_GFX_INCLUDED
      and SOKOL_GFX_IMPL_INCLUDED) these are checked in the utility-headers
      to provide useful error message when dependent headers are missing

- **05-Mar-2019**: sokol_gfx.h now has a 'trace hook' API, and I have started
implementing optional debug-inspection-UI headers on top of Dear ImGui:
    - sokol_gfx.h has a new function *sg_install_trace_hooks()*, this allows
      you to install a callback function for each public sokol_gfx.h function
      (and a couple of error callbacks). For more details, search for "TRACE HOOKS"
      in sokol_gfx.h
    - when creating sokol_gfx.h resources, you can now set a 'debug label'
      in the desc structure, this is ignored by sokol_gfx.h itself, but is
      useful for debuggers or profilers hooking in via the new trace hooks
    - likewise, two new functions *sg_push_debug_group()* and *sg_pop_debug_group()*
      can be used to group related drawing functions under a name, this
      is also ignored by sokol_gfx.h itself and only useful when hooking
      into the API calls
    - I have started a new 'subproject' in the 'imgui' directory, this will
      contain a slowly growing set of optional debug-inspection-UI headers
      which allow to peek under the hood of the Sokol headers. The UIs are
      implemented with [Dear ImGui](https://github.com/ocornut/imgui). Again,
      see the README in the 'imgui' directory and the headers in there
      for details, and check out the live demos on the [Sokol Sample Webpage](https://floooh.github.io/sokol-html5/)
      (click on the little UI buttons in the top right corner of each thumbnail)

- **21-Feb-2019**: sokol_app.h and sokol_audio.h now have an alternative
set of callbacks with user_data arguments. This is useful if you don't
want or cannot store your own application state in global variables.
See the header documentation in sokol_app.h and sokol_audio.h for details,
and check out the samples *sapp/noentry-sapp.c* and *sapp/modplay-sapp.c*
in https://github.com/floooh/sokol-samples

- **19-Feb-2019**: sokol_app.h now has an alternative mode where it doesn't
"hijack" the platform's main() function. Search for SOKOL_NO_ENTRY in
sokol_app.h for details and documentation.

- **26-Jan-2019**: sokol_app.h now has an Android backend contributed by
  [Gustav Olsson](https://github.com/gustavolsson)!
  See the [sokol-samples readme](https://github.com/floooh/sokol-samples/blob/master/README.md)
  for build instructions.

- **21-Jan-2019**: sokol_gfx.h - pool-slot-generation-counters and a dummy backend:
    - Resource pool slots now have a generation-counter for the resource-id
      unique-tag, instead of a single counter for the whole pool. This allows
      to create many more unique handles.
    - sokol_gfx.h now has a dummy backend, activated by defining SOKOL_DUMMY_BACKEND
      (instead of SOKOL_METAL, SOKOL_D3D11, ...), this allows to write
      'headless' tests (and there's now a sokol-gfx-test in the sokol-samples
      repository which mainly tests the resource pool system)

- **12-Jan-2019**: sokol_gfx.h - setting the pipeline state and resource
bindings now happens in separate calls, specifically:
    - *sg_apply_draw_state()* has been replaced with *sg_apply_pipeline()* and
    *sg_apply_bindings()*
    - the *sg_draw_state* struct has been replaced with *sg_bindings*
    - *sg_apply_uniform_block()* has been renamed to *sg_apply_uniforms()*

All existing code will still work. See [this blog
post](https://floooh.github.io/2019/01/12/sokol-apply-pipeline.html) for
details.

- **29-Oct-2018**:
    - sokol_gfx.h has a new function **sg_append_buffer()** which allows to
    append new data to a buffer multiple times per frame and interleave this
    with draw calls. This basically implements the
    D3D11_MAP_WRITE_NO_OVERWRITE update strategy for buffer objects. For
    example usage, see the updated Dear ImGui samples in the [sokol_samples
    repo](https://github.com/floooh/sokol-samples)
    - the GL state cache in sokol_gfx.h handles buffers bindings in a
    more robust way, previously it might have happened that the
    buffer binding gets confused when creating buffers or updating
    buffer contents in the render loop

- **17-Oct-2018**: sokol_args.h added

- **26-Sep-2018**: sokol_audio.h ready for prime time :)

- **11-May-2018**: sokol_gfx.h now autodetects iOS vs MacOS in the Metal
backend during compilation using the standard define TARGET_OS_IPHONE defined
in the TargetConditionals.h system header, please replace the old
backend-selection defines SOKOL_METAL_MACOS and SOKOL_METAL_IOS with
**SOKOL_METAL**

- **20-Apr-2018**: 3 new context-switching functions have been added
to make it possible to use sokol together with applications that
use multiple GL contexts. On D3D11 and Metal, the functions are currently
empty. See the new section ```WORKING WITH CONTEXTS``` in the sokol_gfx.h
header documentation, and the new sample [multiwindow-glfw](https://github.com/floooh/sokol-samples/blob/master/glfw/multiwindow-glfw.c)

- **31-Jan-2018**: The vertex layout declaration in sg\_pipeline\_desc had
some fairly subtle flaws and has been changed to work like Metal or Vulkan.
The gist is that the vertex-buffer-layout properties (vertex stride,
vertex-step-rate and -step-function for instancing) is now defined in a
separate array from the vertex attributes. This removes some brittle backend
code which tries to guess the right vertex attribute slot if no attribute
names are given, and which was wrong for shader-code-generation pipelines
which reorder the vertex attributes (I stumbled over this when porting the
Oryol Gfx module over to sokol-gfx). Some code samples:

```c
// a complete vertex layout declaration with a single input buffer
// with two vertex attributes
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = {
        .buffers = {
            [0] = {
                .stride = 20,
                .step_func = SG_VERTEXSTEP_PER_VERTEX,
                .step_rate = 1
            }
        },
        .attrs = {
            [0] = {
                .name = "pos",
                .offset = 0,
                .format = SG_VERTEXFORMAT_FLOAT3,
                .buffer_index = 0
            },
            [1] = {
                .name = "uv",
                .offset = 12,
                .format = SG_VERTEXFORMAT_FLOAT2,
                .buffer_index = 0
            }
        }
    },
    ...
});

// if the vertex layout has no gaps, we can get rid of the strides and offsets:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = {
        .buffers = {
            [0] = {
                .step_func = SG_VERTEXSTEP_PER_VERTEX,
                .step_rate=1
            }
        },
        .attrs = {
            [0] = {
                .name = "pos",
                .format = SG_VERTEXFORMAT_FLOAT3,
                .buffer_index = 0
            },
            [1] = {
                .name = "uv",
                .format = SG_VERTEXFORMAT_FLOAT2,
                .buffer_index = 0
            }
        }
    },
    ...
});

// we can also get rid of the other default-values, which leaves buffers[0]
// as all-defaults, so it can disappear completely:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = {
        .attrs = {
            [0] = { .name = "pos", .format = SG_VERTEXFORMAT_FLOAT3 },
            [1] = { .name = "uv", .format = SG_VERTEXFORMAT_FLOAT2 }
        }
    },
    ...
});

// and finally on GL3.3 and Metal and we don't need the attribute names
// (on D3D11, a semantic name and index must be provided though)
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = {
        .attrs = {
            [0] = { .format = SG_VERTEXFORMAT_FLOAT3 },
            [1] = { .format = SG_VERTEXFORMAT_FLOAT2 }
        }
    },
    ...
});
```

Please check the sample code in https://github.com/floooh/sokol-samples for
more examples!

Enjoy!
