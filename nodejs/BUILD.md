# Building and Testing the Node.js Package

The `opendht` npm package bundles native binaries for supported platforms.
Applications can install it directly as shown in the [README](README.md).
This guide covers building the Node-API addon from an OpenDHT source checkout,
testing changes, and packaging binaries.

## Prerequisites

Install the native dependencies described in the
[OpenDHT build guide](../BUILD.md), plus Node.js 22.9 or later, a C++17 compiler,
and CMake 3.21 or later. Ninja is optional. The bindings use `node-addon-api` and
target Node-API version 8.

Proxy-client support defaults to enabled and requires ASIO, llhttp, RESTinio,
and expected-lite headers.

## Build and Test

From the root of an OpenDHT source checkout:

```sh
cd nodejs
npm ci
npm run build
npm test
npm run typecheck
```

CMake.js builds OpenDHT from the parent source directory in `nodejs/build/native`.
It does not change the repository's existing build directory. Tools, Python,
peer discovery, documentation, and native tests default to disabled in this build.
Native CMake options can be passed through, for example:

```sh
npm run build -- --CDOPENDHT_PEER_DISCOVERY=ON
```

For an existing build, change cached options directly with CMake from the
`nodejs` directory, then rebuild:

```sh
cmake -S . -B build -DOPENDHT_PROXY_CLIENT=ON
npm run build
```

A custom build can disable proxy support with `-DOPENDHT_PROXY_CLIENT=OFF`;
requesting a proxy then throws an explicit unsupported-build error.

The loader checks local builds, then compatible Node-API binaries under
`prebuilds/`. Set `OPENDHT_NODE_BINDING` to an absolute addon path to select an
explicit build.

## Run the Example

From the `nodejs` directory after building:

```sh
npm run dhtnode -- --port 0 --bootstrap bootstrap.jami.net
```

The interactive node supports get, put, signed put, and value subscriptions.
Run `npm run dhtnode -- --help` for networking options.

## Package and Test Binaries

From a successful Release build in the `nodejs` directory:

```sh
npm run package
npm run test:package
```

`bundle` copies the addon into `prebuilds/<platform>-<arch>/`, gathers its runtime
dependencies, rewrites native loader paths, and copies dependency license
notices. macOS needs `dylibbundler` and Apple developer tools; Linux needs
`patchelf` and Debian/Ubuntu dependency copyright files. Windows uses the vcpkg
installed tree (`VCPKG_INSTALLED_DIR`, `VCPKG_DEFAULT_TRIPLET`) and MSVC tools.
`OPENDHT_LIBRARY_PATH` adds native library search directories when bundling.
An explicit addon path may be passed to `npm run bundle -- /absolute/addon.node`.
Bundling replaces only the current platform's generated prebuild directory.

`test:package` creates an npm archive, installs it in a temporary consumer with
install scripts disabled and no dev dependencies, forces prebuild-only loading,
and runs the full runtime suite. Pass an archive file or a directory containing
one archive to test it directly, for example `npm run test:package -- dist`.
Only runtime JS, declarations, the example's TypeScript source, documentation,
and prebuilds are included in the archive. `prepack` compiles the example, so
archive creation needs the development dependencies; archive installation does not.

Review the included dependency licenses and source-distribution obligations
before redistributing binaries.

## CI

The [Node.js package workflow](../.github/workflows/nodejs-package.yml) builds:

| Target | Build baseline |
|--------|----------------|
| Linux x64 | Ubuntu 24.04 / glibc 2.39 (Noble dependency image) |
| macOS ARM64 and x64 | macOS 15 |
| Windows x64 | Windows Server 2022 / MSVC |

CI combines platform prebuilds into one npm archive and verifies that archive on
Node 22, 24, and 26 on each platform, including Ubuntu 24.04 and 26.04. Branch
pushes, pull requests, and manual runs build and test without publishing.
musl/Alpine and other architectures are not currently built. Node-API stability
does not remove OS/libc or native dependency compatibility requirements.
