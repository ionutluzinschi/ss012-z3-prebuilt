# ss012-z3-prebuilt

Prebuilt Z3 build trees for the Lambo Express long-horizon task
`ss012-smt-memory-safety` (Z3 `src/nlsat` RAII modernization). The task's
Docker image downloads the release asset at image-build time instead of
compiling Z3 from scratch, because the Nest/Daytona sandbox-create budget
(~887s) cannot fit two full Z3 builds (verified: killed at step 76/873 of
the Release build).

## Asset

`ss012-z3-prebuilt-v2.tar.zst` (~298 MiB, zstd -19) contains two directories,
extracted at `/app` in the image:

- `build/` — Release `test-z3` + `z3` (~215 MB)
- `build-asan/` — Debug + ASan/UBSan `test-z3`, debug sections stripped
  with `strip -g` (symbol table kept: the grader's `nm` check for
  `__asan_report` / `__ubsan_handle` still passes)

## Provenance / rebuild recipe

Built from a container of `ubuntu:24.04` with
`cmake g++ python3 ninja-build git ca-certificates`
(gcc 13.3.0-6ubuntu2, cmake 3.28.3):

```
git clone https://github.com/Z3Prover/z3 /app
git reset --hard d0d5497f3ea79364c9a3430e5821f59756c70e18
cmake -G Ninja -S . -B build -DCMAKE_BUILD_TYPE=Release -DZ3_BUILD_TEST_EXECUTABLES=ON
cmake --build build --target z3 --parallel $(nproc)
cmake --build build --target test-z3 --parallel $(nproc)
cmake -G Ninja -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug -DZ3_BUILD_TEST_EXECUTABLES=ON \
  -DCMAKE_C_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined" \
  -DCMAKE_SHARED_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build-asan --target test-z3 --parallel $(nproc)
```

Strip debug info BUT keep every output's recorded build mtime: Ninja's
deps log requires exact output mtimes, and with no depfiles on disk it
cannot re-verify a retouched tree (it would schedule a full rebuild).
So strip, then restore mtimes from pre-strip reference copies:

```
find build-asan -name '*.o' -exec strip -g {} +
strip -g build-asan/test-z3
# for each stripped file F with pre-strip reference R: touch -r R F
tar --owner=0 --group=0 --numeric-owner -cf - build build-asan | zstd -19 -o ss012-z3-prebuilt.tar.zst
```

After extraction the image touches everything under /app EXCEPT the two
build trees to 2000-01-01 (sources AND .git -- Z3's CMake tracks
.git/logs/HEAD as a configure input). The build outputs keep their
recorded mtimes untouched: retouching them invalidates Ninja's deps log
(see above) and forces a full rebuild. (v1 of this asset made exactly
that mistake and was deleted; v2 restores the recorded mtimes.)

The Dockerfile pins the download URL and verifies this SHA-256:

```
6f5f308d67def6bf4d53b73a22f4fa197436a2728bb0a3d4e232940076f74325  ss012-z3-prebuilt-v2.tar.zst
```

Tags are immutable: never re-upload different bytes under the same tag.
A rebuild ships under a new tag and the task Dockerfile pins the new URL+hash.
