```cpp

// src/main.cpp
#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <glm/glm.hpp> // alternatívne je includnuť len napr #include <algorithm>
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
// tu vykresľujeme štvorec
static const PosColor square[] = {
    {{ -0.35f, -0.35f,  0.5f, 1.0f }, { 1, 1, 0, 1 }},  // žltá
    {{ -0.35f,  0.35f,  0.5f, 1.0f }, { 1, 0, 0, 1 }},  // červená
    {{  0.35f,  0.35f,  0.5f, 1.0f }, { 0, 0, 0, 1 }},  // čierna
    {{  0.35f, -0.35f,  0.5f, 1.0f }, { 0, 1, 0, 1 }}   // zelená
};
// GLFW callbacky (nechány beze změny)
static void error_callback(int error, const char* description) { fputs(description, stderr); }
static void key_callback(GLFWwindow* window, int key, int scancode, int action, int mods) {
    if (key == GLFW_KEY_ESCAPE && action == GLFW_PRESS)
        glfwSetWindowShouldClose(window, GL_TRUE);
    if (key == ARROW_RIGHT && action == GLFW_PRESS) { direction = false; printf("Zmena smeru: doleva\n"); }
    if (key == ARROW_LEFT  && action == GLFW_PRESS) { direction = true;  printf("Zmena smeru: doprava\n"); }
    if (key == ARROW_UP    && action == GLFW_PRESS) { rotationSpeed *= 2;  rotationSpeed = glm::min(rotationSpeed, 2000.0f); }
    if (key == ARROW_DOWN  && action == GLFW_PRESS) { rotationSpeed /= 2;  rotationSpeed = glm::max(rotationSpeed,   1.0f); }
}
static void window_focus_callback(GLFWwindow* window, int focused)   { printf("window_focus_callback\n"); }
static void window_iconify_callback(GLFWwindow* window, int iconified){ printf("window_iconify_callback\n"); }
static void window_size_callback(GLFWwindow* window, int w, int h)   {
    printf("resize %d, %d\n", w, h);
    glViewport(0, 0, w, h);
}
static void cursor_callback(GLFWwindow* window, double x, double y)   { /* printf("cursor_callback\n"); */ }
static void button_callback(GLFWwindow* window, int b, int a, int m)  {
    if (a == GLFW_PRESS) printf("button_callback [%d, %d, %d]\n", b, a, m);
}


static const float points2[] = {
    // x       y     z     - symetrický trojuholník, oddelený od štvorca
    0.7f,   0.9f,  0.0f,  // vrchný vrchol
    0.5f,   0.6f,  0.0f,  // ľavý dolný vrchol
    0.9f,   0.6f,  0.0f   // pravý dolný vrchol
};

// this is the vertex
static const char* vertex_shader_src = R"(
#version 330 core
layout(location = 0) in vec3 vp;
void main() {
    gl_Position = vec4(vp, 1.0);
}
)";

static const char* fragment_shader_src = R"(
#version 330 core
out vec4 fragColor;
void main() {
    fragColor = vec4(0.5, 0.0, 0.5, 1.0);
}
)";


static const char* yellow_fragment_shader_src = R"(
#version 330 core
out vec4 fragColor;
void main() {
    fragColor = vec4(1.0, 1.0, 0.0, 1.0); // žltá farba
}
)";

// tghis s the for the purpose of having the uotime near
// Pomocné funkce pro kompilaci shaderů
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
    // na macOS vyžadujeme forward‐compat
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
    // volitelně vsync
    glfwSwapInterval(1);
    // 4) inicializovat GLEW (musí být po vytvoření OpenGL kontextu!)
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
    // 6) viewport podle framebufferu
    // … předchozí kód zůstává beze změny …
    // 6) viewport
    int width, height;
    glfwGetFramebufferSize(window, &width, &height);
    glViewport(0, 0, width, height);
    // 7) Task 3: vytvoření vertex a fragment shaderu + shader‐programu
    // 7.1 Vytvoření a kompilace vertex shaderu
    GLuint vertexShader = glCreateShader(GL_VERTEX_SHADER);
    glShaderSource(vertexShader, 1, &vertex_shader_src, NULL);
    glCompileShader(vertexShader);
    // (sem můžete přidat kontrolu stavů a logů kompilace)
    // 7.2 Vytvoření a kompilace fragment shaderu
    GLuint fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
    glShaderSource(fragmentShader, 1, &fragment_shader_src, NULL);
    glCompileShader(fragmentShader);


    GLuint yellowShaderProgram = glCreateProgram();
    GLuint yellowVertexShader = compileShader(GL_VERTEX_SHADER, vertex_shader_src);
    GLuint yellowFragmentShader = compileShader(GL_FRAGMENT_SHADER, yellow_fragment_shader_src);
    glAttachShader(yellowShaderProgram, yellowVertexShader);
    glAttachShader(yellowShaderProgram, yellowFragmentShader);
    glLinkProgram(yellowShaderProgram);
    glDeleteShader(yellowVertexShader);
    glDeleteShader(yellowFragmentShader);


    // (a tady rovněž kontrola stavu kompilace)
    // 7.3 Linkování shader‐programu
    GLuint shaderProgram = glCreateProgram();
    glAttachShader(shaderProgram, fragmentShader);
    glAttachShader(shaderProgram, vertexShader);
    glLinkProgram(shaderProgram);
    GLint status = GL_FALSE;
    glGetProgramiv(shaderProgram, GL_LINK_STATUS, &status);
    if (status == GL_FALSE) {
        GLint infoLogLength = 0;
        glGetProgramiv(shaderProgram, GL_INFO_LOG_LENGTH, &infoLogLength);
        GLchar* strInfoLog = new GLchar[infoLogLength + 1];
        glGetProgramInfoLog(shaderProgram, infoLogLength, NULL, strInfoLog);
        fprintf(stderr, "Linker failure: %s\n", strInfoLog);
        delete[] strInfoLog;
    }
    // po úspěšném linku můžeme surové shadery smazat
    glDeleteShader(vertexShader);
    glDeleteShader(fragmentShader);
    // 8) Vytvoříme VBO a VAO podle zadání:
    // 8.1 Buffer objekt (VBO)
    GLuint VBO = 0;
    glGenBuffers(1, &VBO);                             // vyrobíme buffer
    glBindBuffer(GL_ARRAY_BUFFER, VBO);                // bindneme ho na target
    glBufferData(GL_ARRAY_BUFFER,
                 sizeof(points2),                       // velikost dat
                 points2,                               // pointer na data
                 GL_STATIC_DRAW);                     // usage hint
    // 8.2 Vertex Array Object (VAO)
    GLuint VAO = 0;
    glGenVertexArrays(1, &VAO);                        // vyrobíme VAO
    glBindVertexArray(VAO);                            // bindneme VAO
    // řekneme OpenGL, že atribut 0 bude obsahovat 3 floaty na vrchol
    glEnableVertexAttribArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);                // musí být znovu bindnut
    glVertexAttribPointer(
        0,             // location = 0 ve vertex shaderu
        3,             // 3 složky (x,y,z)
        GL_FLOAT,      // datový typ
        GL_FALSE,      // normalized?
        0,             // stride (0 = tightly packed)
        nullptr        // offset v bufferu
    );
    // odbindneme VAO i VBO (čistá state)
    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);

    // … předchozí kód trojúhelníku zůstává …
    // --- nový objekt: čtverec s pozicí + barvou ----------------------
    GLuint VBO2, VAO2;
    glGenBuffers(1, &VBO2);
    glGenVertexArrays(1, &VAO2);
    // naplníme buffer daty z struktury PosColor square[]
    glBindBuffer(GL_ARRAY_BUFFER, VBO2);
    glBufferData(GL_ARRAY_BUFFER,
                 sizeof(square),
                 square,
                 GL_STATIC_DRAW);
    // nakonfigurujeme VAO pro čtverec
    glBindVertexArray(VAO2);
    // atribut 0 = pozice (vec4)
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(
        0,                  // location 0
        4,                  // vec4
        GL_FLOAT,
        GL_FALSE,
        sizeof(PosColor),   // stride = velikost celé struktury
        (void*)0            // offset 0 = začátek struktury
    );
    // atribut 1 = barva (vec4)
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(
        1,                              // location 1
        4,                              // vec4
        GL_FLOAT,
        GL_FALSE,
        sizeof(PosColor),
        (void*)(sizeof(glm::vec4))      // offset = za první vec4 (pos)
    );
    // odpojíme VAO a VBO
    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);

    GLuint tri2VBO, tri2VAO;
    glGenBuffers(1, &tri2VBO);
    glGenVertexArrays(1, &tri2VAO);
    glBindBuffer(GL_ARRAY_BUFFER, tri2VBO);
    glBufferData(GL_ARRAY_BUFFER,
                 sizeof(points2),    // points2[] musí být nadefinováno nad main()
                 points2,
                 GL_STATIC_DRAW);
    glBindVertexArray(tri2VAO);
    glEnableVertexAttribArray(0);        // shodná pozice jako u prvního trojúhelníku
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, nullptr);
    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);

    // 9) hlavní render loop
    // 9) hlavní render loop
    while (!glfwWindowShouldClose(window)) {
        // clear color and depth buffer
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

        // aktivovat program
        glUseProgram(shaderProgram);

        // 1) vykreslíme čtverec
        glBindVertexArray(VAO2);
        glDrawArrays(GL_TRIANGLE_FAN, 0, 4);

        // 2) vykreslíme původní trojúhelník
        glBindVertexArray(VAO);
        glDrawArrays(GL_TRIANGLES, 0, 3);

        // 3) ZDE PŘIDÁTE DOPLŇUJÍCÍ TROJÚHELNÍK
        glUseProgram(yellowShaderProgram);
        glBindVertexArray(tri2VAO);
        glDrawArrays(GL_TRIANGLES, 0, 3);

        // odbindneme
        glBindVertexArray(0);
        glUseProgram(0);

        // události + swap
        glfwPollEvents();
        glfwSwapBuffers(window);
    }
    // cleanup
    glfwDestroyWindow(window);
    glfwTerminate();
    return EXIT_SUCCESS;
}
static GLuint createProgram() {
    GLuint vs = compileShader(GL_VERTEX_SHADER,   vertex_shader_src);
    GLuint fs = compileShader(GL_FRAGMENT_SHADER, fragment_shader_src);
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

```
