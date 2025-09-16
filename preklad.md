# Build & Run Guide (macOS + CMake + Homebrew)

## Command 1 -- Configure

``` bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=/opt/homebrew -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

-   **CMake prečíta `CMakeLists.txt`**, nájde knižnice a pripraví build
    systém do priečinka `build/`\
-   `CMAKE_PREFIX_PATH=/opt/homebrew` → povie CMake, kde hľadať Homebrew
    knižnice a hlavičky\
-   `CMAKE_EXPORT_COMPILE_COMMANDS=ON` → vygeneruje súbor
    `compile_commands.json` (užitočné pre IntelliSense vo VS Code)

------------------------------------------------------------------------

## Command 2 -- Build

``` bash
cmake --build build
```

-   Spustí kompiláciu podľa pravidiel, ktoré vytvoril configure krok\
-   Výstupom je spustiteľný súbor `zpgproj` v priečinku `build/`

------------------------------------------------------------------------

## Command 3 -- Run

``` bash
./build/zpgproj
```

-   Spustí výsledný program (GLFW otvorí okno, v ktorom sa vykresľuje
    OpenGL scéna)

------------------------------------------------------------------------

## Extra -- Ak pridáš ďalšie `.cpp` súbory

Buď ich dopíš priamo do `add_executable(...)`, alebo prepneme na
auto-zbieranie:

``` cmake
file(GLOB_RECURSE SRC_LIST CONFIGURE_DEPENDS src/*.cpp)
add_executable(zpgproj ${SRC_LIST})
```

------------------------------------------------------------------------

## Užitočné príkazy navyše

**Vymazať build priečinok a začať odznova (clean build):**

``` bash
rm -rf build
```

**Skontrolovať cestu Homebrew prefixu:**

``` bash
brew --prefix
```

**Skontrolovať, či sú balíky nainštalované:**

``` bash
brew list glfw
brew list glm
```

**Zobraziť verziu clang kompilátora:**

``` bash
clang++ --version
```
