# ss012-z3-prebuilt

Prebuilt Z3 build trees for the Lambo Express long-horizon task
`ss012-smt-memory-safety` (Z3 `src/nlsat` RAII modernization). The task's
Docker image downloads the release asset at image-build time instead of
compiling Z3 from scratch, because the Nest/Daytona sandbox-create budget
(~887s) cannot fit two full Z3 builds (verified: killed at step 76/873 of
the Release build).

## Asset

`ss012-z3-prebuilt.tar.zst` (~298 MiB, zstd -19) contains two directories,
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
find build-asan -name '*.o' -exec strip -g {} +
strip -g build-asan/test-z3
tar --owner=0 --group=0 --numeric-owner -cf - build build-asan | zstd -19 -o ss012-z3-prebuilt.tar.zst
```

After extraction the image runs `find /app/build /app/build-asan -exec touch {} +`
so Ninja sees every build output as newer than the freshly cloned sources
(otherwise the first grade-time build would be a full rebuild).

The Dockerfile pins the download URL and verifies this SHA-256:

```
e4bc1a846270f62ee33f20ad9584ea394bb00071747f6167302875254ba0c540  ss012-z3-prebuilt.tar.zst
```

This tag is immutable: never re-upload different bytes under the same tag.
A rebuild ships under a new tag and the task Dockerfile pins the new URL+hash.
