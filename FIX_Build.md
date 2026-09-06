# Fix: `nvcc` fails with "exception specification is incompatible" on `rsqrt` / `rsqrtf`

Workaround for the incompatibility between glibc ≥ 2.41 and the CUDA Toolkit headers.

**Verified on:** Ubuntu 26.04, glibc 2.43, CUDA Toolkit 13.1 (V13.1.115), RTX 4060 Ti / 5060 Ti (sm_89).
First hit while running [FreeToken](https://github.com/FlashML-org/FreeToken), but it affects **any**
`nvcc` compilation on this combination, JIT (just-in-time, compiled at runtime) or ahead-of-time.

## Symptom

```text
/usr/include/x86_64-linux-gnu/bits/mathcalls.h(206): error: exception specification is
incompatible with that of previous function "rsqrt"
  (declared at line 629 of /usr/local/cuda-13.1/targets/x86_64-linux/include/crt/math_functions.h)
   extern double rsqrt (double __x) noexcept (true); extern double __rsqrt (double __x) noexcept (true);

/usr/include/x86_64-linux-gnu/bits/mathcalls.h(206): error: exception specification is
incompatible with that of previous function "rsqrtf"
  (declared at line 653 of .../crt/math_functions.h)

2 errors detected in the compilation of "...".
ninja: build stopped: subcommand failed.
```

With FreeToken this surfaces during `Capturing graphs`, at the first model start, as
`RuntimeError: ninja exited with status 2`, followed by
`Backend worker is gone and cannot be restarted`.

## Cause

glibc ≥ 2.41 declares `rsqrt` / `rsqrtf` as `noexcept (true)`; the CUDA headers declare
the same functions with no exception specification. In C++ two declarations of the same
function with different exception specifications are an error, so the compilation aborts.

## Check your environment

```bash
readlink -f /etc/alternatives/cuda   # actual toolkit path
nvcc --version | tail -2
ldd --version | head -1              # glibc version
```

If more than one toolkit is installed, switching to a CUDA release that predates glibc 2.41
is the cleaner fix — patch the header only when no such toolkit is available.

## Fix

Two-line patch to the CUDA header, with backup.

```bash
H="$(readlink -f /etc/alternatives/cuda)/targets/x86_64-linux/include/crt/math_functions.h"
sudo cp --update=none -a "$H" "$H.bak"
sudo sed -i -E \
  -e 's/(__device_builtin__[[:space:]]+double[[:space:]]+rsqrt\(double x\));/\1 noexcept (true);/' \
  -e 's/(__device_builtin__[[:space:]]+float[[:space:]]+rsqrtf\(float x\));/\1 noexcept (true);/' \
  "$H"
grep -n 'rsqrt(double x)\|rsqrtf(float x)' "$H"
```

`grep` must print exactly two lines, both ending in `noexcept (true);`:

```text
629:extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double  rsqrt(double x) noexcept (true);
653:extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ float   rsqrtf(float x) noexcept (true);
```

Anything other than two lines: roll back (see below) and inspect the header manually.

Then clear the JIT cache — the failed build is cached and the error would persist otherwise:

```bash
rm -rf ~/.cache/tvm-ffi     # FreeToken / tvm-ffi
# rm -rf ~/.cache/torch_extensions   # if you also use torch.utils.cpp_extension
```

## Verify

```bash
ft serve --model /path/to/your-model/
```

`Capturing graphs` should complete and the server should stay up on `http://127.0.0.1:1919`.

## Rollback

```bash
sudo cp -a "$H.bak" "$H" && rm -rf ~/.cache/tvm-ffi
```

## Caveats

- The patch must be re-applied after every CUDA Toolkit upgrade or reinstall.
- The header path depends on the installed toolkit version — always derive it from
  `readlink -f /etc/alternatives/cuda` rather than hardcoding `cuda-13.1`.
- On CUDA ≥ 13.2, check first whether NVIDIA has added its own guard: if `grep` finds no
  match for the pattern, the workaround is no longer needed.
- This only silences a declaration mismatch; it changes no code generation and no numerical
  behaviour.
