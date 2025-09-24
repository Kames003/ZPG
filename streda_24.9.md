```cpp

// src/main.cpp
#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <glm/glm.hpp>
#include <glm/vec3.hpp>
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>
#include <stdlib.h>
#include <stdio.h>

#define ARROW_UP    265
#define ARROW_DOWN  264
#define ARROW_RIGHT 262
#define ARROW_LEFT  263

float rotationSpeed = 40.0f;
bool  direction     = false;

struct PosColor {
    glm::vec4 pos;   // x,y,z,w
    glm::vec4 col;   // r,g,b,a
};

// Farebný trojuholník v strede
static const PosColor colorTriangle[] = {
    {{ -0.35f, -0.35f,  0.0f, 1.0f }, { 1, 0, 1, 1 }},  // fialová (ľavý dolný)
    {{  0.35f, -0.35f,  0.0f, 1.0f }, { 0, 0, 1, 1 }},  // modrá (pravý dolný)
    {{  0.0f,   0.35f,  0.0f, 1.0f }, { 0, 1, 0, 1 }}   // zelená (vrchný)
};

// Štvorec na pozícii pôvodného žltého trojuholníka
static const PosColor yellowSquare[] = {
    {{ 0.6f,  0.7f,  0.0f, 1.0f }, { 1, 1, 0, 1 }},  // žltá
    {{ 0.6f,  0.9f,  0.0f, 1.0f }, { 1, 1, 0, 1 }},  // žltá
    {{ 0.8f,  0.9f,  0.0f, 1.0f }, { 1, 1, 0, 1 }},  // žltá
    {{ 0.8f,  0.7f,  0.0f, 1.0f }, { 1, 1, 0, 1 }}   // žltá
};

// GLFW callbacky
static void error_callback(int error, const char* description) {
    fputs(description, stderr);
}

static void key_callback(GLFWwindow* window, int key, int scancode, int action, int mods) {
    if (key == GLFW_KEY_ESCAPE && action == GLFW_PRESS)
        glfwSetWindowShouldClose(window, GL_TRUE);
    if (key == ARROW_RIGHT && action == GLFW_PRESS) {
        direction = false;
        printf("Zmena smeru: doleva\n");
    }
    if (key == ARROW_LEFT  && action == GLFW_PRESS) {
        direction = true;
        printf("Zmena smeru: doprava\n");
    }
    if (key == ARROW_UP    && action == GLFW_PRESS) {
        rotationSpeed *= 2;
        rotationSpeed = glm::min(rotationSpeed, 2000.0f);
    }
    if (key == ARROW_DOWN  && action == GLFW_PRESS) {
        rotationSpeed /= 2;
        rotationSpeed = glm::max(rotationSpeed,   1.0f);
    }
}

static void window_focus_callback(GLFWwindow* window, int focused) {
    printf("window_focus_callback\n");
}

static void window_iconify_callback(GLFWwindow* window, int iconified) {
    printf("window_iconify_callback\n");
}

static void window_size_callback(GLFWwindow* window, int w, int h) {
    printf("resize %d, %d\n", w, h);
    glViewport(0, 0, w, h);
}

static void cursor_callback(GLFWwindow* window, double x, double y) {
    /* printf("cursor_callback\n"); */
}

static void button_callback(GLFWwindow* window, int b, int a, int m) {
    if (a == GLFW_PRESS) printf("button_callback [%d, %d, %d]\n", b, a, m);
}

// Vertex shader
static const char* vertex_shader_src = R"(
#version 330 core
layout(location = 0) in vec3 vp;
void main() {
    gl_Position = vec4(vp, 1.0);
}
)";

// Fragment shader pre fialový trojuholník
static const char* fragment_shader_src = R"(
#version 330 core
out vec4 fragColor;
void main() {
    fragColor = vec4(0.5, 0.0, 0.5, 1.0);
}
)";

// Fragment shader pre žltý trojuholník
static const char* yellow_fragment_shader_src = R"(
#version 330 core
out vec4 fragColor;
void main() {
    fragColor = vec4(1.0, 1.0, 0.0, 1.0); // žltá farba
}
)";

// Vertex shader pre štvorec s farbami
static const char* color_vertex_shader_src = R"(
#version 330 core
layout(location = 0) in vec4 pos;
layout(location = 1) in vec4 color;
out vec4 vertexColor;
void main() {
    gl_Position = pos;
    vertexColor = color;
}
)";

// Fragment shader pre štvorec s farbami
static const char* color_fragment_shader_src = R"(
#version 330 core
in vec4 vertexColor;
out vec4 fragColor;
void main() {
    fragColor = vertexColor;
}
)";

// Pomocné funkcie pre kompilaci shaderov
static GLuint compileShader(GLenum type, const char* src) {
    GLuint shader = glCreateShader(type);
    glShaderSource(shader, 1, &src, nullptr);
    glCompileShader(shader);
    GLint status;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &status);
    if (!status) {
        char log[512];
        glGetShaderInfoLog(shader,512, nullptr, log);
        fprintf(stderr,"Shader compile error: %s\n", log);
    }
    return shader;
}

static GLuint createProgram(const char* vertexSrc, const char* fragmentSrc) {
    GLuint vs = compileShader(GL_VERTEX_SHADER, vertexSrc);
    GLuint fs = compileShader(GL_FRAGMENT_SHADER, fragmentSrc);
    GLuint prog = glCreateProgram();
    glAttachShader(prog, vs);
    glAttachShader(prog, fs);
    glLinkProgram(prog);
    GLint status;
    glGetProgramiv(prog, GL_LINK_STATUS, &status);
    if (!status) {
        char log[512];
        glGetProgramInfoLog(prog,512, nullptr, log);
        fprintf(stderr,"Program link error: %s\n", log);
    }
    glDeleteShader(vs);
    glDeleteShader(fs);
    return prog;
}

int main()
{
    GLFWwindow* window;

    // 1) nastavit error callback a inicializovat GLFW
    glfwSetErrorCallback(error_callback);
    if (!glfwInit()) {
        fprintf(stderr, "ERROR: could not start GLFW\n");
        return EXIT_FAILURE;
    }

    // 2) požadovat OpenGL 3.3 Core Profile
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

    // 3) vytvořit okno a kontext
    window = glfwCreateWindow(800, 600, "ZPG Modern OpenGL 3.3+", NULL, NULL);
    if (!window) {
        fprintf(stderr, "ERROR: could not create GLFW window\n");
        glfwTerminate();
        return EXIT_FAILURE;
    }
    glfwMakeContextCurrent(window);
    glfwSwapInterval(1);

    // 4) inicializovat GLEW
    glewExperimental = GL_TRUE;
    if (glewInit() != GLEW_OK) {
        fprintf(stderr, "ERROR: could not initialize GLEW\n");
        glfwDestroyWindow(window);
        glfwTerminate();
        return EXIT_FAILURE;
    }

    // 5) vypsat informace o verzích
    printf("OpenGL  : %s\n", glGetString(GL_VERSION));
    printf("GLSL    : %s\n", glGetString(GL_SHADING_LANGUAGE_VERSION));
    printf("Vendor  : %s\n", glGetString(GL_VENDOR));
    printf("Renderer: %s\n", glGetString(GL_RENDERER));
    int glfwMaj, glfwMin, glfwRev;
    glfwGetVersion(&glfwMaj, &glfwMin, &glfwRev);
    printf("GLFW    : %d.%d.%d\n", glfwMaj, glfwMin, glfwRev);

    // 6) viewport
    int width, height;
    glfwGetFramebufferSize(window, &width, &height);
    glViewport(0, 0, width, height);

    // Vytvorenie shader programov
    GLuint triangleProgram = createProgram(vertex_shader_src, fragment_shader_src);
    GLuint yellowProgram = createProgram(vertex_shader_src, yellow_fragment_shader_src);
    GLuint colorProgram = createProgram(color_vertex_shader_src, color_fragment_shader_src);

    // === Farebný trojuholník v strede ===
    GLuint colorTriVBO, colorTriVAO;
    glGenBuffers(1, &colorTriVBO);
    glGenVertexArrays(1, &colorTriVAO);

    glBindBuffer(GL_ARRAY_BUFFER, colorTriVBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(colorTriangle), colorTriangle, GL_STATIC_DRAW);

    glBindVertexArray(colorTriVAO);
    // atribut 0 = pozice (vec4)
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, sizeof(PosColor), (void*)0);
    // atribut 1 = barva (vec4)
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, sizeof(PosColor), (void*)(sizeof(glm::vec4)));

    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);

    // === Žltý štvorec na pozícii pôvodného žltého trojuholníka ===
    GLuint yellowSquareVBO, yellowSquareVAO;
    glGenBuffers(1, &yellowSquareVBO);
    glGenVertexArrays(1, &yellowSquareVAO);

    glBindBuffer(GL_ARRAY_BUFFER, yellowSquareVBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(yellowSquare), yellowSquare, GL_STATIC_DRAW);

    glBindVertexArray(yellowSquareVAO);
    // atribut 0 = pozice (vec4)
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, sizeof(PosColor), (void*)0);
    // atribut 1 = barva (vec4)
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, sizeof(PosColor), (void*)(sizeof(glm::vec4)));

    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);

    // Nastaviť callbacky
    glfwSetKeyCallback(window, key_callback);
    glfwSetWindowFocusCallback(window, window_focus_callback);
    glfwSetWindowIconifyCallback(window, window_iconify_callback);
    glfwSetWindowSizeCallback(window, window_size_callback);
    glfwSetCursorPosCallback(window, cursor_callback);
    glfwSetMouseButtonCallback(window, button_callback);

    // Hlavný render loop
    while (!glfwWindowShouldClose(window)) {
        // clear color and depth buffer
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

        // 1) vykreslíme farebný trojuholník v strede
        glUseProgram(colorProgram);
        glBindVertexArray(colorTriVAO);
        glDrawArrays(GL_TRIANGLES, 0, 3);

        // 2) vykreslíme žltý štvorec v pravom hornom rohu
        glUseProgram(colorProgram);
        glBindVertexArray(yellowSquareVAO);
        glDrawArrays(GL_TRIANGLE_FAN, 0, 4);

        // odbindneme
        glBindVertexArray(0);
        glUseProgram(0);

        // události + swap
        glfwPollEvents();
        glfwSwapBuffers(window);
    }

    // cleanup
    glDeleteProgram(triangleProgram);
    glDeleteProgram(yellowProgram);
    glDeleteProgram(colorProgram);
    glDeleteVertexArrays(1, &colorTriVAO);
    glDeleteVertexArrays(1, &yellowSquareVAO);
    glDeleteBuffers(1, &colorTriVBO);
    glDeleteBuffers(1, &yellowSquareVBO);
    
    glfwDestroyWindow(window);
    glfwTerminate();
    return EXIT_SUCCESS;
}
```

```makefile

cmake_minimum_required(VERSION 3.29)
project(untitled)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Get the Homebrew prefix to locate installed packages
execute_process(
        COMMAND brew --prefix
        OUTPUT_VARIABLE HOMEBREW_PREFIX
        OUTPUT_STRIP_TRAILING_WHITESPACE
)
message(STATUS "Homebrew prefix: ${HOMEBREW_PREFIX}")

# --- GLFW (headers + lib) -----------------------------------------------------
# hľadaj najprv include
find_path(GLFW_INCLUDE_DIR
        NAMES GLFW/glfw3.h
        HINTS ${HOMEBREW_PREFIX}/include
)
if (GLFW_INCLUDE_DIR)
    message(STATUS "Found GLFW include directory: ${GLFW_INCLUDE_DIR}")
else()
    message(FATAL_ERROR "Could not find GLFW include directory")
endif()

# hľadaj knižnicu – skúsi "glfw" aj "glfw3"
find_library(GLFW_LIB
        NAMES glfw glfw3
        HINTS ${HOMEBREW_PREFIX}/lib
)
if (GLFW_LIB)
    message(STATUS "Found GLFW library: ${GLFW_LIB}")
else()
    message(FATAL_ERROR "Could not find GLFW library (tried names 'glfw','glfw3').")
endif()


# --- GLM (header-only) --------------------------------------------------------
find_path(GLM_INCLUDE_DIR
        NAMES glm/glm.hpp
        HINTS ${HOMEBREW_PREFIX}/include
)
if (GLM_INCLUDE_DIR)
    message(STATUS "Found GLM include directory: ${GLM_INCLUDE_DIR}")
else()
    message(FATAL_ERROR "Could not find GLM include directory")
endif()


# --- GLEW (voliteľné, zatiaľ zakomentované) -----------------------------------
# Ak budete chcieť GLEW, odkomentujte celý blok nižšie:
find_path(GLEW_INCLUDE_DIR
     NAMES GL/glew.h
     HINTS ${HOMEBREW_PREFIX}/include
 )
 if (GLEW_INCLUDE_DIR)
     message(STATUS "Found GLEW include directory: ${GLEW_INCLUDE_DIR}")
 else()
     message(FATAL_ERROR "Could not find GLEW include directory")
 endif()

 find_library(GLEW_LIB
     NAMES GLEW glew glew32
     HINTS ${HOMEBREW_PREFIX}/lib
 )
 if (GLEW_LIB)
     message(STATUS "Found GLEW library: ${GLEW_LIB}")
 else()
     message(FATAL_ERROR "Could not find GLEW library")
 endif()


# --- SOIL / ASSIMP (ponechané, ale vypnuté) ----------------------------------
# file(GLOB SOIL_SOURCES
#   ${CMAKE_SOURCE_DIR}/lib/soil/src/*.c
# )
# find_path(ASSIMP_INCLUDE_DIR assimp/Importer.hpp HINTS ${HOMEBREW_PREFIX}/include)
# find_library(ASSIMP_LIB NAMES assimp HINTS ${HOMEBREW_PREFIX}/lib)
# include_directories(${CMAKE_SOURCE_DIR}/lib/soil/include)
# add_library(SOIL STATIC ${SOIL_SOURCES})


# --- ZDROJÁKY -----------------------------------------------------------------
# PÔVODNÝ rozsiahly zoznam súborov – ponechaný na neskôr, zatiaľ to isté čo bolo:
# add_executable(zpgproj
#     src/main.cpp
#     src/App.cpp
#     …
# )
# Teraz minimalisticky iba main.cpp:
add_executable(untitled src/main.cpp)

# --- INCLUDE DIRECTORIES ------------------------------------------------------
target_include_directories(untitled PRIVATE
        ${GLFW_INCLUDE_DIR}
        ${GLM_INCLUDE_DIR}
        ${GLEW_INCLUDE_DIR}   # odkomentujte, ak používate GLEW
        src
)

# --- LINKING ------------------------------------------------------------------
target_link_libraries(untitled PRIVATE
        ${GLFW_LIB}
        ${GLEW_LIB}           # odkomentujte, ak používate GLEW
        # SOIL
        # ${ASSIMP_LIB}
        # macOS frameworks
        "-framework Cocoa"
        "-framework IOKit"
        "-framework CoreVideo"
        "-framework OpenGL"
        "-framework CoreFoundation"
        "-lobjc"
)

# Voliteľne: potlačí deprecated warny fixed‐function OpenGL na macOS
target_compile_definitions(untitled PRIVATE GL_SILENCE_DEPRECATION)


```
