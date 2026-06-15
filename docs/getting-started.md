# Getting Started with CMMC in Docker

This guide uses one Docker image and one persistent Docker container for building CMMC and running the LLVM, RISC-V, MIPS, and ARM test flows.

Run all commands from the repository root:

```bash
cd /home/djp/cmmc
```

The build directory used here is `build-docker-getting-started/`. It does not conflict with `build-docker-llvm/`.

## 1. Build the all-in-one Docker image

```bash
docker build -t cmmc-getting-started - <<'EOF'
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates build-essential cmake ninja-build pkg-config \
      flex bison clang clang-format clangd-15 llvm-15-dev zlib1g-dev libgtest-dev \
      python3 python3-pip python3-yaml git git-lfs \
      vim spim openjdk-11-jre csmith gdb-multiarch \
      libglib2.0-dev libpixman-1-dev \
      gcc-12-arm-linux-gnueabihf g++-12-arm-linux-gnueabihf \
      gcc-12-riscv64-linux-gnu g++-12-riscv64-linux-gnu \
      gcc-10-mipsel-linux-gnu g++-10-mipsel-linux-gnu \
 && update-alternatives --install /usr/bin/clangd clangd /usr/bin/clangd-15 100 \
 && python3 -m pip install --no-cache-dir jinja2 tqdm \
 && ln -sf /usr/bin/riscv64-linux-gnu-gcc-12 /usr/bin/riscv64-linux-gnu-gcc-11 \
 && rm -rf /var/lib/apt/lists/*
RUN git clone --depth 1 https://github.com/dtcxzyw/qemu /qemu \
 && cd /qemu \
 && ./configure --target-list=riscv64-linux-user,mipsel-linux-user,arm-linux-user --enable-plugins \
 && make -j"$(nproc)"
ENV QEMU_PATH=/qemu/build
WORKDIR /workspace
EOF
```

## 2. Create and start one persistent container

```bash
docker rm -f cmmc-getting-started 2>/dev/null || true
docker create --name cmmc-getting-started -v "$PWD":/workspace -w /workspace cmmc-getting-started tail -f /dev/null
docker start cmmc-getting-started
```

All remaining commands run inside this same container with `docker exec`.

## 3. Verify toolchain paths

```bash
docker exec cmmc-getting-started bash -lc '
set -e
command -v cmake
command -v ninja
command -v git-lfs
command -v riscv64-linux-gnu-gcc-11
command -v mipsel-linux-gnu-gcc-10
command -v arm-linux-gnueabihf-gcc-12
ls -l /qemu/build/qemu-riscv64
ls -l /qemu/build/qemu-mipsel
ls -l /qemu/build/qemu-arm
ls -l /qemu/build/tests/plugin/libinsn_clock.so
printf "QEMU_PATH=%s\n" "$QEMU_PATH"
'
```

## 4. Hydrate Git LFS test inputs

Some SysY `.in` files are stored with Git LFS. Hydrate them before running the tests.

```bash
docker exec cmmc-getting-started bash -lc '
git config --global --add safe.directory /workspace
git lfs install --local
git lfs pull
'
```

## 5. Configure and build CMMC

```bash
docker exec cmmc-getting-started bash -lc '
cmake -S . -B build-docker-getting-started -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMMC_LLVM_SUPPORT=ON \
  -DCMAKE_C_COMPILER=gcc \
  -DCMAKE_CXX_COMPILER=g++ &&
cmake --build build-docker-getting-started --target cmmc -j
'
```

## 6. Verify the compiler binary

```bash
docker exec cmmc-getting-started bash -lc '
build-docker-getting-started/bin/cmmc --version
'
```

## 7. Run CMMC interactively in the persistent container

Open a shell inside the already-running container:

```bash
docker exec -it cmmc-getting-started bash
```

From that shell, run CMMC directly:

```bash
build-docker-getting-started/bin/cmmc --help
build-docker-getting-started/bin/cmmc tests/SysY2022/functional/00_main.sy --target=llvm -o /tmp/00_main.ll
sed -n '1,20p' /tmp/00_main.ll
```

Leave the container shell:

```bash
exit
```

The container keeps running after you leave the shell, so you can re-enter it later with:

```bash
docker exec -it cmmc-getting-started bash
```

## 8. Run LLVM-only tests

```bash
docker exec cmmc-getting-started bash -lc '
python3 tests/test_driver.py build-docker-getting-started/bin/cmmc tests llvm
'
```

Expected result for this checkout:

```text
Passed 222 Failed 0 Total 222
```

## 9. Run RISC-V QEMU tests

```bash
docker exec cmmc-getting-started bash -lc '
python3 tests/test_driver.py build-docker-getting-started/bin/cmmc tests qemu,riscv
'
```

Expected result for this checkout:

```text
Passed 222 Failed 0 Total 222
```

## 10. Run ARM QEMU tests

```bash
docker exec cmmc-getting-started bash -lc '
python3 tests/test_driver.py build-docker-getting-started/bin/cmmc tests qemu,arm
'
```

Expected result for this checkout:

```text
Passed 222 Failed 0 Total 222
```

## 11. Run MIPS QEMU tests

The current checkout has known MIPS backend failures. The command below treats the known result as expected and exits successfully only if the test driver reports the current expected summary.

```bash
docker exec cmmc-getting-started bash -lc '
set -o pipefail
mkdir -p build-docker-getting-started/test-logs
set +e
python3 tests/test_driver.py build-docker-getting-started/bin/cmmc tests qemu,mips | tee build-docker-getting-started/test-logs/qemu-mips.log
status=${PIPESTATUS[0]}
set -e
test "$status" -ne 0
grep -q "Passed 212 Failed 10 Total 222" build-docker-getting-started/test-logs/qemu-mips.log
'
```

Expected result for this checkout:

```text
Passed 212 Failed 10 Total 222
```

Known MIPS failures observed in this checkout:

```text
tests/SysY2022/hidden_functional/23_json.sy
tests/SysY2022/hidden_functional/37_dct.sy
tests/SysY2022/performance/dead-code-elimination-1.sy
tests/SysY2022/performance/dead-code-elimination-2.sy
tests/SysY2022/performance/dead-code-elimination-3.sy
tests/SysY2022/performance/fft0.sy
tests/SysY2022/performance/fft1.sy
tests/SysY2022/performance/fft2.sy
tests/SysY2022/performance/instruction-combining-1.sy
tests/SysY2022/performance/instruction-combining-2.sy
```

## 12. Stop or remove the persistent container

Stop it when you want to keep the container but pause it:

```bash
docker stop cmmc-getting-started
```

Start it again later:

```bash
docker start cmmc-getting-started
```

Remove it when you no longer need it:

```bash
docker rm -f cmmc-getting-started
```

The repository files and `build-docker-getting-started/` are bind-mounted from the host, so removing the container does not delete the build directory.
