## JIT

```
./tools/build.py -mdebug,release runtime
./tools/test.py -mdebug,release
./tools/test.py -mdebug,release --checked
```

## [AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)

```
export CC="third_party/clang/linux/bin/clang -fsanitize=address -fPIC"
export CXX="third_party/clang/linux/bin/clang++ -fsanitize=address -fPIC"
export C_INCLUDE_PATH="third_party/clang/linux/lib/clang/3.4/include/"
export CPLUS_INCLUDE_PATH="third_party/clang/linux/lib/clang/3.4/include/"
export ASAN_OPTIONS="handle_segv=0:detect_stack_use_after_return=1"
gclient runhooks
./tools/build.py -mrelease runtime
./tools/test.py -mrelease
```

## [ThreadSanitizer](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual)

```
export CC="third_party/clang/linux/bin/clang -fsanitize=thread"
export CXX="third_party/clang/linux/bin/clang++ -fsanitize=thread"
export C_INCLUDE_PATH="third_party/clang/linux/lib/clang/3.4/include/"
export CPLUS_INCLUDE_PATH="third_party/clang/linux/lib/clang/3.4/include/"
gclient runhooks
./tools/build.py -mrelease runtime
./tools/test.py -mrelease
```

## Noopt

```
./tools/build.py -mdebug,release runtime
./tools/test.py -mdebug,release --noopt
```

## Precompilation

```
./tools/build.py -mrelease runtime
./tools/test.py -mrelease -cprecompiler -rdart_precompiled
```

## App snapshots

```
./tools/build.py -mproduct runtime
./tools/test.py -mproduct -cdart2app -rdart_product
```