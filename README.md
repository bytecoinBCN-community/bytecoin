# Bytecoin v3.5.1 for Ubuntu 24.04 (community maintenance)

**This is an independent community fork. It is not affiliated with the Bytecoin founders or bcndev and does not speak for them.** All code stays under LGPLv3, and the upstream history and authorship are kept. The upstream README follows below.

Upstream `bcndev/bytecoin` last changed on 18 July 2019 (v3.5.1). This branch makes that release build and run correctly on a current system. **Consensus is unchanged**, so it runs on the existing network alongside other nodes.

## What changed

| File | Change |
| --- | --- |
| `src/crypto/jh/jh.c` | **Fixes the JH hash.** The code read byte buffers through `uint64*` (undefined behaviour under strict aliasing). GCC 13 at `-O3` dropped padding writes in `Final()`, so JH was wrong for short inputs. JH is part of CryptoNight finalization, so this breaks consensus when built with a modern compiler. |
| `CMakeLists.txt` | Adds `-fno-strict-aliasing` as a safeguard for the rest of the type-punning crypto code. |
| `src/CryptoNoteConfig.hpp` | Adds live seed nodes in front of the old ones. |
| `src/Core/Config.cpp` | Makes the seed node loop work with lists of different lengths. |
| `crypto.hpp`, `Math.hpp`, `WalletLegacy.hpp`, `DBsqlite3.cpp`, `benchmarks.cpp` | Adds includes that GCC 13 no longer pulls in transitively. |

GCC 13, Boost 1.83 and OpenSSL 3 come from the distribution instead of vendored copies.

## Run with Docker

A ready image is built by GitHub Actions from this branch. The test suites run during every build, and a failed test stops it.

```bash
docker run -d --name bytecoind --restart unless-stopped \
  -p 8080:8080 -p 127.0.0.1:8081:8081 \
  -v bytecoin-data:/var/lib/bytecoin \
  ghcr.io/bytecoinbcn-community/bytecoin:v3.5.1-community.1
```

- Digest of `v3.5.1-community.1`: `sha256:319623040856cd86f162b4539106aa5c08fe9a51f8610dd593a760973486d045`. To run exactly this build, use `ghcr.io/bytecoinbcn-community/bytecoin@sha256:319623040856cd86f162b4539106aa5c08fe9a51f8610dd593a760973486d045`.
- GitHub publishes a build provenance attestation that ties the image to the commit and the workflow run. The [workflow](.github/workflows/docker.yml) and its [runs](https://github.com/bytecoinBCN-community/bytecoin/actions) are public.
- `8080` is P2P. Open it to help the network.
- `8081` is RPC. Keep it on localhost.
- The image contains `bytecoind`, `walletd` and `minerd`, and runs as an unprivileged user.
- On a hard disk, the first sync and a start after an unclean shutdown can take hours. An SSD helps.

### Build it yourself

The build expects `bytecoin` and `lmdb` side by side. `lmdb` is pinned to the commit this branch was built and tested with:

```bash
git clone -b ubuntu-24.04 https://github.com/bytecoinBCN-community/bytecoin.git
git clone https://github.com/bcndev/lmdb.git
git -C lmdb checkout f334f6a4a5c00b873f5721b8265d102c6b6b3667
docker build --build-arg JOBS=2 -f bytecoin/docker/Dockerfile -t bytecoin:3.5.1-noble .
```

Then run it with the command above, using `bytecoin:3.5.1-noble` as the image. These commands use the classic builder, which Ubuntu's `docker.io` package has. Do not set `DOCKER_BUILDKIT=1` there unless you install `docker-buildx`.

## What has been tested

- `tests --crypto --bip32 --hash --json --wallet --db` passes (22 crypto suites, 0 failures).
- `cn_fast_hash` and `cn_slow_hash` reference values match an Ubuntu 18.04 build bit for bit.
- PoW of real mainnet v4 blocks validates.
- A node built from this code has synced the full chain to the tip and has produced blocks accepted by the network.
- A clean build with `docker/Dockerfile` from fresh clones of this branch and the pinned `lmdb` passes the same test suites (November 9, 2026).
- The published image `v3.5.1-community.1` pulls without a login, connects to the network and starts syncing (November 9, 2026). A full sync from it has not been run yet.

## What has not been done

- **No security audit yet.** Do not treat this branch as audited.
- `--blockchain` and `--wallet-state` test suites have not been run.
- Only Linux x86-64 has been tested.

## Security

Please do not open public issues for vulnerabilities. Report them privately through GitHub's private vulnerability reporting on this repository.

---

# Bytecoin

[![Build Status](https://dev.azure.com/bcndev/bytecoin/_apis/build/status/bytecoin-daemons?branchName=releases/3.5.1)](https://dev.azure.com/bcndev/bytecoin/_build/latest?definitionId=1&branchName=releases/3.5.1)

## About

Welcome to the repository of Bytecoin. Here you will find source code, instructions, wiki resources, and integration tutorials.

Contents
* Building on Linux 64-bit
* Building on Mac OSX
* Building on Windows
* Building on other platforms

## Building on Linux 64-bit

All commands below work on Ubuntu 18.*, other distributions may need different command set.

### Building with standard options

Create directory `bcndev` somewhere and go there:
```
$> mkdir bcndev
$> cd bcndev
```

To go futher you have to have a number of packages and utilities. You need at least gcc 5.4.

* `build-essential` package:
    ```
    $bcndev> sudo apt-get install build-essential
    ```

* `libudev`:
    ```
    sudo apt-get install libudev-dev
    ```
    
* CMake (3.0 or newer):
    ```
    $bcndev> sudo apt-get install cmake
    $bcndev> cmake --version
    ```
    If version is too old, follow instructions on [the official site](https://cmake.org/download/).

* Boost (1.65 or newer):
    We use boost as a header-only library via find_boost package. So, if your system has boost installed and set up, it will be used automatically.

    Note - there is a bug in `boost::asio` 1.66 that affects `bytecoind`. Please use either version 1.65 or 1.67+.
    ```
    $bcndev> sudo apt-get install libboost-dev
    ```
    If the latest boost installed is too old (e.g. for Ubuntu 16.*), then you need to download and unpack boost into the `bcndev/boost` folder.

    ```
    $bcndev> wget -c 'https://dl.bintray.com/boostorg/release/1.69.0/source/boost_1_69_0.tar.gz'
    $bcndev> tar -xzf ./boost_1_69_0.tar.gz
    $bcndev> rm ./boost_1_69_0.tar.gz
    $bcndev> mv ./boost_1_69_0/ ./boost/
    ```

* OpenSSL (1.1.1 or newer):
    Install OpenSSL to `bcndev/openssl` folder. (In below commands use switch `linux-x86_64-clang` instead of `linux-x86_64` if using clang.)
    ```
    $bcndev> git clone --single-branch --branch OpenSSL_1_1_1b --depth 1 https://github.com/openssl/openssl.git
    $bcndev> cd openssl
    $bcndev/openssl> ./Configure linux-x86_64 no-shared
    $bcndev/openssl> make -j8
    $bcndev/openssl> cd ..
    ```

* LMDB
    Source files are referenced via relative paths, so you do not need to separately build it:
    Please note, we use LMDB only when building 64-bit daemons. For 32-bit daemons SQLite is used instead.

    Difference to official LMDB repository is lifted 2GB database limit if built by MSVC (even of 64-bit machine).
    ```
    $bcndev> git clone https://github.com/bcndev/lmdb.git

    ```

Git-clone (or git-pull) Bytecoin source code in that folder:
```
$bcndev> git clone https://github.com/bcndev/bytecoin.git
```

Create build directory inside bytecoin, go there and run CMake and Make:
```
$bcndev> mkdir -p bytecoin/build
$bcndev> cd bytecoin/build
$bcndev/bytecoin/build> cmake ..
$bcndev/bytecoin/build> make -j8
```

Check built binaries by running them from `../bin` folder
```
$bcndev/bytecoin/build> ../bin/bytecoind -v
```

## Building on Mac OSX

### Building with standard options (10.11 El Capitan or newer)

You need command-line tools. Either get XCode from an App Store or run 'xcode-select --install' in terminal and follow instructions. First of all, you need [Homebrew](https://brew.sh).

Then open terminal and install CMake and Boost:

* `brew install cmake`
* `brew install boost`

Create directory `bcndev` somewhere and go there:
```
$~/Downloads> mkdir bcndev
$~/Downloads> cd bcndev
```

Git-clone (or git-pull) Bytecoin source code in that folder:
```
$bcndev> git clone https://github.com/bcndev/bytecoin.git
```

Put LMDB source code in `bcndev` folder (source files are referenced via relative paths, so you do not need to separately build it):
```
$~/Downloads/bcndev> git clone https://github.com/bcndev/lmdb.git
```

Install OpenSSL to `bcndev/openssl` folder:
```
$~/Downloads/bcndev> git clone --single-branch --branch OpenSSL_1_1_1b --depth 1 https://github.com/openssl/openssl.git
$~/Downloads/bcndev> cd openssl
$~/Downloads/bcndev/openssl> ./Configure darwin64-x86_64-cc no-shared -mmacosx-version-min=10.11
$~/Downloads/bcndev/openssl> make -j8
$~/Downloads/bcndev/openssl> cd ..
```

Create build directory inside bytecoin, go there and run CMake and Make:
```
$~/Downloads/bcndev> mkdir bytecoin/build
$~/Downloads/bcndev> cd bytecoin/build
$~/Downloads/bcndev/bytecoin/build> cmake ..
$~/Downloads/bcndev/bytecoin/build> make -j8
```

Check built binaries by running them from `../bin` folder:
```
$bcndev/bytecoin/build> ../bin/bytecoind -v
```

## Building on Windows

You need Microsoft Visual Studio Community 2017. [Download](https://www.visualstudio.com/vs/) and install it selecting `C++`, `git`, `cmake integration` packages.
Run `Visual Studio x64 command prompt` from start menu.

Create directory `bcndev` somewhere:
```
$C:\> mkdir bcndev
$C:\> cd bcndev
```

Boost (1.65 or newer):
    We use boost as a header-only library via find_boost package. So, if your system has boost installed and set up, it will be used automatically. If not, you need to download and unpack boost into bcndev/boost folder.

Git-clone (or git-pull) Bytecoin source code in that folder:
```
$C:\bcndev> git clone https://github.com/bcndev/bytecoin.git
```

Put LMDB in the same folder (source files are referenced via relative paths, so you do not need to separately build it):
```
$C:\bcndev> git clone https://github.com/bcndev/lmdb.git
```

Download amalgamated [SQLite 3](https://www.sqlite.org/download.html) and unpack it into the same folder (source files are referenced via relative paths, so you do not need to separately build it).

You need to build openssl, first install ActivePerl (select "add to PATH" option, then restart console):
```
$C:\bcndev> git clone --single-branch --branch OpenSSL_1_1_1b --depth 1 https://github.com/openssl/openssl.git
$C:\bcndev> cd openssl
$C:\bcndev\openssl> perl Configure VC-WIN64A no-shared no-asm
$C:\bcndev\openssl> nmake
$C:\bcndev\openssl> cd ..
```
If you want to build 32-bit binaries, you will also need 32-bit build of openssl in separate folder (configuring openssl changes header files, so there is no way to have both 32-bit and 64-bit versions in the same folder):
```
$C:\bcndev> git clone --single-branch --branch OpenSSL_1_1_1b --depth 1 https://github.com/openssl/openssl.git openssl32
$C:\bcndev> cd openssl32
$C:\bcndev\openssl> perl Configure VC-WIN32 no-shared no-asm
$C:\bcndev\openssl> nmake
$C:\bcndev\openssl> cd ..
```

Now launch Visual Studio, in File menu select `Open Folder`, select `C:\bcndev\bytecoin` folder.
Wait until CMake finishes running and `Build` appears in main menu.
Select `x64-Debug` or `x64-Release` from standard toolbar, and then `Build/Build Solution` from the main menu.

## Building with options

You can build daemons that use SQLite istead of LMDB on any platform by providing options to CMake.
You may need to clean 'build' folder, if you built with default options before, due to cmake aggressive caching.

```
$bytecoin/build> cmake -DUSE_SQLITE=1 ..
$bytecoin/build> time make -j8
```

## Building on 32-bit x86 platforms, iOS, Android and other ARM platforms

Bytecoin works on 32-bit systems if SQLite is used instead of LMDB (we've experienced lots of problems building and running with lmdb in 32-bit compatibility mode, especially on iOS).

We build official x86 32-bit version for Windows only, because there is zero demand for 32-bit version for Linux or Mac.

Building source code for iOS, Android, Raspberry PI, etc is possible (we have experimental `bytecoind` and `walletd` running on ARM64 iPhone) but requires major skills on your part. __TBD__

## Building on Big-Endian platforms

Currently bytecoin does not work out of the box on any Big-Endian platform, due to some endianess-dependent code. This may be fixed in the future. If you wish to run on Big-Endian platform, please contact us.

## Building with parameters

If you want to use tools like `clang-tidy`, run `cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..` instead of `cmake ..`
