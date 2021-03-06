# GLFM
Write OpenGL ES 2.0 code in C/C++ without writing platform-specific code.

GLFM is an OpenGL ES 2.0 layer for mobile devices and the web. GLFM supplies an OpenGL ES context and input events. It is largely inspired by [GLFW](http://www.glfw.org/).

GLFM is written in C and runs on iOS 8, Android 2.3.3, and WebGL 1.0 (via [Emscripten](https://github.com/kripken/emscripten)).

## Features
* OpenGL ES 2.0, 3.0, and 3.1 display setup.
* Retina / high-DPI support.
* Touch and keyboard events.
* Events for application state and context loss.

## Non-goals
GLFM is limited in scope, and isn't designed to provide everything needed for an app. For example, GLFM doesn't provide (and will never provide) the following:

* No image loading.
* No text rendering.
* No audio.
* No menus, UI toolkit, or scene graph.
* No integration with other mobile features like web views, maps, or game scores.

Instead, GLFM can be used with other cross-platform libraries that provide what an app needs.

## Example
This example initializes the display in <code>glfmMain()</code> and draws a triangle in <code>onFrame()</code>. A more detailed example is available [here](example/src/main.c).

```C
#include "glfm.h"
#include <string.h>

static GLint program = 0;
static GLuint vertexBuffer = 0;

static void onFrame(GLFMDisplay *display, const double frameTime);
static void onSurfaceCreated(GLFMDisplay *display, const int width, const int height);
static void onSurfaceDestroyed(GLFMDisplay *display);

void glfmMain(GLFMDisplay *display) {
    glfmSetDisplayConfig(display,
                         GLFMRenderingAPIOpenGLES2,
                         GLFMColorFormatRGBA8888,
                         GLFMDepthFormatNone,
                         GLFMStencilFormatNone,
                         GLFMMultisampleNone,
                         GLFMUserInterfaceChromeFullscreen);
    glfmSetSurfaceCreatedFunc(display, onSurfaceCreated);
    glfmSetSurfaceResizedFunc(display, onSurfaceCreated);
    glfmSetSurfaceDestroyedFunc(display, onSurfaceDestroyed);
    glfmSetMainLoopFunc(display, onFrame);
}

static void onSurfaceCreated(GLFMDisplay *display, const int width, const int height) {
    glViewport(0, 0, width, height);
}

static void onSurfaceDestroyed(GLFMDisplay *display) {
    // When the surface is destroyed, all existing GL resources are no longer valid.
    program = 0;
    vertexBuffer = 0;
}

static GLuint compileShader(const GLenum type, const GLchar *shaderString) {
    const GLint shaderLength = (GLint)strlen(shaderString);
    GLuint shader = glCreateShader(type);
    glShaderSource(shader, 1, &shaderString, &shaderLength);
    glCompileShader(shader);
    return shader;
}

static void onFrame(GLFMDisplay *display, const double frameTime) {
    if (program == 0) {
        const GLchar *vertexShader =
            "attribute highp vec4 position;\n"
            "void main() {\n"
            "   gl_Position = position;\n"
            "}";

        const GLchar *fragmentShader =
            "void main() {\n"
            "  gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);\n"
            "}";

        program = glCreateProgram();
        GLuint vertShader = compileShader(GL_VERTEX_SHADER, vertexShader);
        GLuint fragShader = compileShader(GL_FRAGMENT_SHADER, fragmentShader);

        glAttachShader(program, vertShader);
        glAttachShader(program, fragShader);

        glLinkProgram(program);

        glDeleteShader(vertShader);
        glDeleteShader(fragShader);
    }
    if (vertexBuffer == 0) {
        const GLfloat vertices[] = {
             0.0,  0.5, 0.0,
            -0.5, -0.5, 0.0,
             0.5, -0.5, 0.0,
        };
        glGenBuffers(1, &vertexBuffer);
        glBindBuffer(GL_ARRAY_BUFFER, vertexBuffer);
        glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
    }

    glClearColor(0.4f, 0.0f, 0.6f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    glUseProgram(program);
    glBindBuffer(GL_ARRAY_BUFFER, vertexBuffer);

    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, 0);
    glDrawArrays(GL_TRIANGLES, 0, 3);
}
```
## API
See [glfm.h](include/glfm.h)

## Build requirements
* iOS: Xcode 8
* Android: Android Studio 2.2, SDK 25, NDK Bundle 13.1.3345770
* WebGL: Emscripten 1.35.0

## Create a new GLFM project
Use the `new_project.py` command-line script to automatically create a new project setup for iOS, Android, and Emscripten.

The script will ask a few questions and output a new project. After creation, you can edit the `main.c` file.

## Use GLFM in an existing project

1. Remove the project's existing <code>void main()</code> function, if any.
2. Add the GLFM source files (in `include` and `src`).
3. Include a <code>void glfmMain(GLFMDisplay *display)</code> function in a C/C++ file.

## Future ideas
* Accelerometer and gyroscope input.
* Gamepad / MFi controller input.

## Caveats
* OpenGL ES 3.1 support in Android is untested, and iOS currently has no OpenGL ES 3.1 support.
* GLFM is not thread-safe. All GLFM functions must be called on the main thread (that is, from `glfmMain` or from the callback functions).
* Key input on iOS is not ideal. Using the keyboard (on an iOS device via Bluetooth keyboard or on the simulator via a Mac's keyboard), only a few keys are detected (arrows, enter, space, escape). Also, only key press events can be detected, but not key repeat or key release events.
* Orientation lock probably doesn't work on HTML5.

## Questions
**Why is the entry point <code>glfmMain()</code> and not <code>main()</code>?**

Otherwise, it wouldn't work on iOS. To initialize the Objective-C environment, the <code>main()</code> function must create an autorelease pool and call the <code>UIApplicationMain()</code> function, which *never returns*. On iOS, GLFM doesn't call <code>glfmMain()</code> until after the <code>UIApplicationDelegate</code> and <code>UIViewController</code> are initialized.

**Why is GLFM event-driven? Why does GLFM take over the main loop?**

Otherwise, it wouldn't work on iOS (see above) or on HTML5, which is event-driven.

**What are the `glfmAsset*` functions for? Doesn't `stdio` work?**

The `stdio` functions work fine on iOS and Emscripten, but Android's assets are stored in a compressed file (the APK) that can only be accessed via a proprietary API. The `glfmAsset*` functions use NDK's proprietary assets API on Android, and `stdio` on iOS and Emscripten.

## License
[ZLIB](http://en.wikipedia.org/wiki/Zlib_License)
