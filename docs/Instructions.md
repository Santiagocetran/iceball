# Instructions: Integrate Sackit as Git Submodule for Self-Contained Build

## Context
The iceball project currently depends on an external sackit library that must be manually installed in the parent directory (`../sackit`). This creates a poor developer experience and makes the project difficult to build on new systems.

## Current Problem
- Sackit is a music module player library used for `.it` (Impulse Tracker) file playback
- Currently requires manual installation of sackit in parent directory
- Uses outdated `xlibinc/libsackit.a` static library approach
- Creates dependency management issues for new developers

## Goal
Integrate sackit as a Git submodule to make the project self-contained and eliminate external dependency requirements.

## Tasks Required

### 1. Add Sackit as Git Submodule
```bash
git submodule add https://github.com/greasemonkey/sackit.git external/sackit
git submodule update --init --recursive
```

### 2. Update CMakeLists.txt
- Remove the `find_package(sackit REQUIRED)` line (around line 109)
- Add `add_subdirectory(external/sackit)` before the main target definitions
- Set sackit variables manually:
  ```cmake
  set(sackit_LIBRARIES sackit)
  set(sackit_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/external/sackit)
  ```

### 3. Update Makefiles
- **Makefile** (line 23): Change `LIBS_sackit = xlibinc/libsackit.a` to use the submodule path
- **Makefile.mingw** and **Makefile.osx**: Update similar lines
- Update include paths to point to `external/sackit`

### 4. Test the Build
- Ensure both CMake and Makefile builds work
- Verify music playback still functions correctly
- Test on a clean system without external sackit installation

### 5. Update Documentation
- Update `docs/READ_THIS_FIRST.txt` to remove sackit installation instructions
- Update `BUILD_ANALYSIS.md` to reflect the change
- Add submodule initialization instructions to README

## Files to Modify
- `CMakeLists.txt` (lines around 108-109)
- `Makefile` (line 23)
- `Makefile.mingw` and `Makefile.osx` (similar lines)
- `docs/READ_THIS_FIRST.txt`
- `BUILD_ANALYSIS.md`
- `README.md` (add submodule instructions)

## Success Criteria
- Project builds without requiring external sackit installation
- Music playback functionality preserved
- Clean build on fresh system
- Documentation updated to reflect changes

## Notes
- Sackit is public domain, so no licensing concerns
- The library is used in `src/wav.c` and `src/lua_mus.h` for audio playback
- Current API usage should remain compatible

## Implementation Steps

1. **Initialize submodule**:
   ```bash
   git submodule add https://github.com/greasemonkey/sackit.git external/sackit
   git submodule update --init --recursive
   ```

2. **Modify CMakeLists.txt**:
   ```cmake
   # Remove this line:
   # find_package(sackit REQUIRED)
   
   # Add these lines before the main target definitions:
   add_subdirectory(external/sackit)
   set(sackit_LIBRARIES sackit)
   set(sackit_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/external/sackit)
   ```

3. **Update Makefile**:
   ```makefile
   # Change this line:
   # LIBS_sackit = xlibinc/libsackit.a
   # To:
   LIBS_sackit = external/sackit/build/libsackit.a
   ```

4. **Test build**:
   ```bash
   # Test CMake build
   mkdir build && cd build
   cmake ..
   make
   
   # Test Makefile build
   cd ..
   make clean
   make
   ```

5. **Update documentation**:
   - Remove sackit installation instructions from `docs/READ_THIS_FIRST.txt`
   - Add submodule initialization instructions to README
   - Update `BUILD_ANALYSIS.md` to reflect the change

## Verification
After implementation, verify that:
- The project builds without external sackit installation
- Music files (`.it` format) play correctly in the game
- The build works on a fresh system
- All documentation is updated appropriately
