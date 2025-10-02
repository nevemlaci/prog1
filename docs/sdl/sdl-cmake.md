# SDL2 telepítése CMake build system környezetben

Kezdő `CMakeLists.txt`:
```cmake
cmake_minimum_required(VERSION 3.31)
project(prog1_sdl2 C)

set(CMAKE_C_STANDARD 11)

add_executable(prog1_sdl2 main.c)
```

## SDL2 csomagok letöltése

#### MinGW fordító esetén

- <https://github.com/libsdl-org/SDL/releases/download/release-2.32.10/SDL2-devel-2.32.10-mingw.zip>
- <https://github.com/libsdl-org/SDL_image/releases/download/release-2.8.8/SDL2_image-devel-2.8.8-mingw.zip>
- <https://github.com/libsdl-org/SDL_ttf/releases/download/release-2.24.0/SDL2_ttf-devel-2.24.0-mingw.zip>

#### MSVC (Visual Studio fordító) esetén

- <https://github.com/libsdl-org/SDL/releases/download/release-2.32.10/SDL2-devel-2.32.10-VC.zip>
- <https://github.com/libsdl-org/SDL_image/releases/download/release-2.8.8/SDL2_image-devel-2.8.8-VC.zip>
- <https://github.com/libsdl-org/SDL_ttf/releases/download/release-2.24.0/SDL2_ttf-devel-2.24.0-VC.zip>

#### Unix alapú rendszerek

Unix alapú rendszereken a telepítést legegyszerűbben a rendszer package manageréből lehet végrehajtani.

APT:
```bash
sudo apt update
sudo apt install libsdl2-dev libsdl2-image-dev libsdl2-ttf-dev
```

Pacman:
```bash
sudo pacman -S sdl2 sdl2_image sdl2_ttf
```


## Kicsomagolás, mappastruktúra

!!! note 
    Csak Windows operációs rendszeren szükséges

Készíts a projekt gyökérmappájában egy `external_dependencies` nevű almappát! 
Ebbe az almappába csomagold ki a három letöltött tömörített állományt!

A mappastruktúra valahogy így kell kinézzen:
```
CMakeLists.txt
external_dependencies
|-SDL2-verzió
|-SDL2_image-verzió
|-SDL2_TTF-verzió
...
```

## PATH CMake változó beállítása

!!! note 
    Csak Windows operációs rendszeren szükséges

A CMake a `CMAKE_PREFIX_PATH` változóban kapott könyvtárakban keresi először a függőségeket.

Adjuk hozzá ehhez a listához az előbb létrehozott könyvtárat! Ehhez a `CMakeLists.txt` fájlhoz, valahol a `projekt` deklaráció után,
hozzá kell adni egy sort:

```cmake
list(APPEND CMAKE_PREFIX_PATH external_dependencies)
```

## SDL megkeresése

Az SDL beépített CMake támogatással érkezik. A CMake `find_package` függvényét fogjuk használni.

Adjuk hozzá ezt a három sort a `CMakeLists.txt`-hez, a `list(append) ...` sor **után**!

```cmake
find_package(SDL2 REQUIRED)
find_package(SDL2_image REQUIRED)
find_package(SDL2_ttf REQUIRED)
```

## SDL_GFX könyvtár hozzáadása

Az SDL_GFX könyvtárat a többi SDL modullal ellentétben forráskódként a legegyszerűbb a projekthez adni.

A <https://github.com/nevemlaci/SDL2_gfx> weboldalon a *Code* gombra kattintva ZIP-ként töltsd le a fájlokat!
Ezután az `external_dependencies` könyvtárba csomagold ki és nevezd át `SDL2_GFX`-re a kicsomagolt könyvtárat!

A következő sort add hozzá a `CMakeLists.txt`-hez:
```cmake
add_subdirectory(external_dependencies/SDL2_GFX)
```

!!! note `add_subdirectory`
    Ez a függvény bevon egy alkönyvtárat a buildbe. Ha belenézel az `SDL2_GFX` könyvtárban, ott is találsz egy
    `CMakeLists.txt` fájlt, ami a függvénykönyvtár CMake targetjeit definiálja. 

## SDL könyvtár csatolása

Ahhoz, hogy az SDL könyvtárat használni tudjuk, csatolni (linkelni) kell azt az alkalmazásunkhoz. 
Szerencsére CMake -ben ez is egy sor.

`CMakeLists.txt`:
```cmake
cmake_minimum_required(VERSION 3.31)
project(prog1_sdl2 C)
set(CMAKE_C_STANDARD 11)

list(APPEND CMAKE_PREFIX_PATH external_dependencies)

find_package(SDL2 REQUIRED)
find_package(SDL2_image REQUIRED)
find_package(SDL2_ttf REQUIRED)

add_subdirectory(external_dependencies/SDL2_GFX)


add_executable(prog1_sdl2 main.c)

target_link_libraries(
        prog1_sdl2
        PRIVATE SDL2::SDL2main
        PRIVATE SDL2::SDL2
        PRIVATE SDL2_image::SDL2_image
        PRIVATE SDL2_ttf::SDL2_ttf
        PRIVATE sdl2_gfx_static
)
```

## Windows és DLL probléma

Windowson szükség van a DLL-ek executable mellé másolására. Ezen problámát oldjuk most meg.

Készíts a projekt gyökérmappájába egy `cmake` nevű könyvtárat, majd hozz benne létre egy `copy_dlls.cmake` 
nevű fájlt!

A fájl tartalma legyen az alábbi:
```cmake
function(copy_depend_dlls target_name)
    if(WIN32)
        add_custom_command(TARGET ${target_name} POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy -t $<TARGET_FILE_DIR:${target_name}> $<TARGET_RUNTIME_DLLS:${target_name}>
                COMMAND_EXPAND_LISTS
        )
    endif (WIN32)
endfunction()
```

A `CMakeLists.txt` file végére (az SDL2 csatolása után) pedig az alábbi két sort illeszd be:
```cmake
include(cmake/copy_dlls.cmake)
copy_depend_dlls(prog1_sdl2)
```

!!! warning
    A `copy_depend_dlls` függvény az executable target nevét várja el (az első paraméter, amit az `add_executable` függvény kap).
    Ha ez nem egyezik a példában használttal, akkor természetesen a saját nevet add meg!

## SDL2 Bootstrap

Az útmutatóban elkészített projekt 
erről a linkről: <https://github.com/nevemlaci/Prog1_SDL2> letölthető.

A projekt egy bootstrap Python scriptet tartalmaz, amely letölti a szükséges függőségeket.

A használat elég egyszerű, a megfelelő script file-t kell futtatni:

* `mingw.bat`: MinGW toolchain esetén
* `msvc.bat`: MSVC fordító esetén
* `unix.sh`: Unix (Linux/MacOS) esetén

!!! note Python
    A script futtatásához, ha még nincs telepítve, a Python3 telepítése szükséges.
    Ez Windowson a `python3` parancs kiadása után a Microsoft Storeból lehetséges.
    Unix alapú rendszereken a `python3` csomagot kell telepíteni a package managerből.

!!! note Futtatás Unix rendszereken
    Mielőtt a script futtatása engedélyezve lenne, ki kell adni a következő parancsot:
    ```bash
    chmod a+x unix.bash
    ```