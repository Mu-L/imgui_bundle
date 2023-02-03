# FAQ

## FAQ: high-DPI with ImGui

A high-DPI screen could have a `physical pixel` resolution of 3840x2160, but on this screen the OS will display widgets with a scaling factor of 200%, so that widgets do not look too small on it.

Special care must be taken in order to correctly handle screen with high-DPI within ImGui. The solutions will differ greatly depending on the platform.

This document tries to be a complement to [the related ImGui FAQ article](https://github.com/ocornut/imgui/blob/master/docs/FAQ.md#q-how-should-i-handle-dpi-in-my-application)

::: note
This article deals with standard ImGui applications. `HelloImGui` and `Dear ImGui Bundle` provides a [dedicated additional API ](https://github.com/pthom/hello_imgui/blob/master/src/hello_imgui/dpi_aware.h).
:::

### macOS

On macOS, our screen's physical pixel resolution of 3840x2160 will be hidden by the OS. For example, the OS will answer 1920x1080 when querying the screen resolution (this corresponds to `screen coordinates`, which may differ from the physical pixels).

However, each window has an internal frame buffer whose size is in physical pixels. This is just hidden from the user in most occasions. When using ImGui, this info is stored inside `ImGui::GetDrawData()→FramebufferScale`.

On macOS, the visible windows and widgets sizes will appear consistent when switching from a high-DPI screen (for example a Retina screen) to a standard screen.

However, font rendering may appear poor and blurry if no action is taken. A simple solution is to query the main screen \"backing scale factor\", and to use it when loading fonts, like so:

``` cpp
// Query macOS for the scaling factor (by using NSScreen api)
float windowDevicePixelRatio = NSScreen.mainScreen.backingScaleFactor;

// [...]

// When loading fonts use this factor:
ImGui::GetIO().Fonts->AddFontFromFileTTF(fontFileName, fontSize * windowDevicePixelRatio);

// [...]

// Set ImGui::GetIO().FontGlobalScale
ImGui::GetIO().FontGlobalScale = 1.f / windowDevicePixelRatio;
```

### Emscripten

With emscripten (i.e. inside a browser) the situation is close to macOS: widgets\' sizes are relative to the current scaling factor of the screen, **and** to the current zoom level of the given page.

As a result, widgets\' sizes will appear consistent but font rendering may be blurry if no action is taken. A simple solution is to query the browser's devicePixel ratio:

``` cpp
// Query the browser for the scaling factor (this depends on the screen scaling factor and on the browser zoom level)
double windowDevicePixelRatio = (float)EM_ASM_DOUBLE( { return window.devicePixelRatio; });

// [...]

// When loading fonts use this factor:
ImGui::GetIO().Fonts->AddFontFromFileTTF(fontFileName, fontSize * windowDevicePixelRatio);

// [...]

// Set ImGui::GetIO().FontGlobalScale
ImGui::GetIO().FontGlobalScale = 1.f / windowDevicePixelRatio;
```

### Windows

Under windows, two cases are possible:

1.  If the application is not [\"DPI aware\"](https://learn.microsoft.com/en-us/windows/win32/hidpi/setting-the-default-dpi-awareness-for-a-process) the application will not be informed about the physical screen resolution. Windows and widgets sizes will be consistent between monitors; however fonts will appear blurry, and nothing can be done about it (there is no access to an underlying buffer with a bigger resolution).

2.  If the application is \"DPI aware\", the application will be informed about the physical size of the screen (i.e. in this case, screen coordinates correspond to physical pixels). As a result, windows fonts, and widgets may appear too small, because their sizes need to be multiplied by the scaling factor. Widgets positioning is also impacted. Also, when moving a window from a high-DPI screen to a standard screen, its apparent size may double suddenly.

::: note
Windows applications are **not** DPI aware by default. *Nothing can be done to improve the rendering of non DPI aware applications*. The rest of this FAQ deals with DPI aware applications.
:::

Several issues can be encountered when running a DPI-Aware application with ImGui. For example, with a scaling factor of 200%, and if no action is taken:

-   The application window initial size will be twice too small

-   Widgets\' sizes will be twice too small. Widgets\' position will be also affected

-   Fonts will be displayed twice too small

-   Dear ImGui's styling will be too small (margins, roundings, etc.)

-   The application window size will visually double when moving the window to a standard screen

### Linux

The situation on linux is close to Windows\' DPI-aware applications. Applications are informed about the physical number of pixels, and they need to scale according to the screen dpi factor.

### Windows: how to make the application DPI aware

See [the ImGui related FAQ](https://github.com/ocornut/imgui/blob/master/docs/FAQ.md#q-how-should-i-handle-dpi-in-my-application)

::: note
the function `ImGui_ImplWin32_EnableDpiAwareness` inside [imgui_impl_win32.cpp](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_win32.cpp) is possibly broken. HelloImGui provides a corrected version: [win32_dpi_awareness.h](https://github.com/pthom/hello_imgui/blob/master/src/hello_imgui/internal/backend_impls/backend_window_helper/win32_dpi_awareness.h) and [win32_dpi_awareness.cpp](https://github.com/pthom/hello_imgui/blob/master/src/hello_imgui/internal/backend_impls/backend_window_helper/win32_dpi_awareness.cpp)
:::

### Windows & Linux: how to correctly size and position the widgets

It is almost always a bad idea to use fixed sizes. This will lead to portability issues, especially on high-DPI screens.

Always pre-multiply your positions and sizes by `ImGui::GetFontSize()`!

In order to make this simpler, the `EmVec2` function below can greatly reduce the friction: whenever you need to use ImVec2 for positioning or sizing, use `EmVec2` instead!

`EmVec2()` returns a size in multiples of the font height. It is somewhat comparable to the [em CSS Unit](https://lyty.dev/css/css-unit.html).

``` cpp
ImVec2 EmVec2(float x, float y)
{
    IM_ASSERT(GImGui != NULL);
    float k = ImGui::GetFontSize();
    return ImVec2(k * x, k * y);
}
```

### Windows & Linux: How to load fonts at the correct size

You need to query the application monitor current Dpi scale. To achieve this, you can call [DpiForWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getdpiforwindow) on windows.

Some backends may make this easier, e.g. GLFW:

``` cpp
float DpiWindowSizeFactor(GLFWwindow * window)
{
    float xscale, yscale;
    glfwGetWindowContentScale((GLFWwindow *) window, &xscale, &yscale);
    return xscale; // xscale and yscale will likely be equal
}
```

Once you known the screen Dpi scale, you can use this when loading fonts:

``` cpp
float fontLoadingFactor = DpiWindowSizeFactor(...);
ImGui::GetIO().Fonts->AddFontFromFileTTF(fontFileName, fontSize * fontLoadingFactor);
```

### Windows & Linux: how to adapt Dear ImGui's styling scale

``` cppp
float dpiScale = DpiWindowSizeFactor(...);
ImGui::GetStyle().ScaleAllSizes(dpiScale);
```

### Windows & Linux: how to have a consistent initial window size between monitors

As mentioned before, multiply your window size by DpiWindowSizeFactor.

### Windows & Linux: adapting windows and font size when application is moved between monitors

This is a bit more difficult: see [ImGui related FAQ](https://github.com/ocornut/imgui/blob/master/docs/FAQ.md#q-how-should-i-handle-dpi-in-my-application)