# nx2elf

Convert Nintendo Switch executable files (NSO/NRO/MOD) to ELFs.

## Downloads

Prebuilt binaries are available from [GitHub Releases](https://github.com/tkgstrator/nx2elf/releases):

| Host platform | Release asset |
| --- | --- |
| Linux x86_64 | `nx2elf-Linux-x86_64.zip` |
| Linux arm64 | `nx2elf-Linux-arm64.zip` |
| macOS x86_64 | `nx2elf-macOS-x86_64.zip` |
| macOS arm64 | `nx2elf-macOS-arm64.zip` |
| Windows x86_64 | `nx2elf-Windows-x86_64.zip` |

Each release also includes `SHA256SUMS`. Download the manifest and the archive for your platform into the same directory, then verify the archive before extracting it.

On Linux:

```sh
archive=nx2elf-Linux-x86_64.zip
grep -F "  $archive" SHA256SUMS > "$archive.sha256" &&
  test "$(wc -l < "$archive.sha256")" -eq 1 &&
  sha256sum -c "$archive.sha256"
```

On macOS:

```sh
archive=nx2elf-macOS-arm64.zip
grep -F "  $archive" SHA256SUMS > "$archive.sha256" &&
  test "$(wc -l < "$archive.sha256")" -eq 1 &&
  shasum -a 256 -c "$archive.sha256"
```

On Windows PowerShell:

```powershell
$archive = "nx2elf-Windows-x86_64.zip"
$expected = (Get-Content .\SHA256SUMS | Where-Object { $_ -match "  $([regex]::Escape($archive))$" }).Split()[0]
$actual = (Get-FileHash $archive -Algorithm SHA256).Hash.ToLowerInvariant()
if ($actual -ne $expected) { throw "SHA-256 mismatch for $archive" }
```

## Building

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

### Building Linux packages with Docker

Docker can build a Linux release package without installing CMake or a compiler on the host. Run the following from the repository root in a POSIX-compatible shell:

```sh
platform=linux/amd64
package=nx2elf-Linux-x86_64

mkdir -p build-docker
docker run --rm \
  --platform "$platform" \
  --env PACKAGE="$package" \
  --mount type=bind,src="$(pwd)",dst=/src,readonly \
  --mount type=bind,src="$(pwd)/build-docker",dst=/out \
  alpine:3.22 sh -lc '
    set -eu
    apk add --no-cache build-base cmake zip
    cmake -S /src -B /tmp/build \
      -DCMAKE_BUILD_TYPE=Release \
      -DCPACK_PACKAGE_FILE_NAME="$PACKAGE"
    cmake --build /tmp/build --config Release --target package
    cp "/tmp/build/$PACKAGE.zip" /out/
  '
```

The archive is written to `build-docker/nx2elf-Linux-x86_64.zip`. To build the Linux arm64 package instead, use:

```sh
platform=linux/arm64
package=nx2elf-Linux-arm64
```

The selected platform must be supported by the Docker installation. Building for a different CPU architecture may use emulation and take longer. These commands create Linux binaries only; use the native CMake commands or the GitHub Actions workflow for macOS and Windows packages.

## Usage

```text
nx2elf <file or directory> [--export-uncompressed <path>] [--export-elf <path>]
```

- `<file or directory>` — path to an NSO/NRO/MOD file, or a directory to batch-convert all files within it.
- `--export-elf <path>` — override the output ELF path (default: `<input>.elf`).
- `--export-uncompressed <path>` — export an uncompressed NSO variant.

## Features

- Supports compressed and uncompressed 32-bit and 64-bit NSO files
- NRO and MOD (raw binary) input formats
- Reconstructs ELF sections: `.dynsym`, `.dynstr`, `.dynamic`, `.rela.dyn`, `.rela.plt`, `.plt`, `.got`, `.got.plt`, `.hash`, `.gnu.hash`, `.init`, `.fini`, `.init_array`, `.fini_array`, `.eh_frame_hdr`, `.eh_frame`, `.note`, `.bss`
- Non-overlapping section headers for clean loading in IDA and other tools
- Batch conversion of directories

## Release process

Pushing a `v*` tag runs the complete five-platform build matrix. GitHub creates the release only after every package succeeds, and publishes the same ZIP artifacts that passed the build together with their checksum manifest.

## Licensing

This repository currently does not declare a license. Do not assume permission to redistribute its source or binaries beyond rights you hold independently.
