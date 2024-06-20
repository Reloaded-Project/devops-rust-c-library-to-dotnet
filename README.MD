<div align="center">
  <a href="https://github.com/Reloaded-Project/devops-rust-c-library-to-dotnet" target="_blank">
    <img src="assets/reloaded-logo.png" alt="reloaded Logo" width="100"/>
  </a>

  <h1 align="center">Reloaded Deploy Rust C Library to .NET Wrapper Action</h1>

  <p>
    <a href="LICENSE">
      <img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License">
    </a>
  </p>
</div>

The `devops-rust-c-library-to-dotnet` GitHub Action builds and packages a .NET library that
acts as a wrapper for a Rust library with C exports.

It works by taking the artifact outputs of [devops-rust-lightweight-binary@v1][rust-lightweight-library]
action (usually named `C-Library-*`), and extracting the native (`.dll`, `.so`, `.dylib`) binaries
from them.

These are then placed in appropriate `runtime` folders in the .NET library project, which is then
built and packed into a NuGet package.

## An Example

Suppose you run the following step in your GitHub workflow:

```yaml
test-library-build:
  strategy:
    matrix:
      include:
        - os: ubuntu-latest
          target: x86_64-unknown-linux-gnu
          use-pgo: true
          use-cross: false
        - os: ubuntu-latest
          target: i686-unknown-linux-gnu
          use-pgo: true
          use-cross: false
        - os: ubuntu-latest
          target: aarch64-unknown-linux-gnu
          use-pgo: true # x64 host to aarch64 simulated guest via cross
          use-cross: true
        - os: ubuntu-latest
          target: armv7-unknown-linux-gnueabihf
          use-pgo: true # x64 host to armv7 simulated guest via cross
          use-cross: true
        - os: windows-latest
          target: x86_64-pc-windows-msvc
          use-pgo: true
          use-cross: false
        - os: windows-latest
          target: i686-pc-windows-msvc
          use-pgo: true
          use-cross: false
        - os: windows-latest
          target: aarch64-pc-windows-msvc
          use-pgo: false # no virtualization support (proprietary OS)
          use-cross: false
        - os: macos-13 # x86
          target: x86_64-apple-darwin
          use-pgo: true
          use-cross: false
        - os: macos-14 # M1
          target: aarch64-apple-darwin
          use-pgo: true
          use-cross: false
  runs-on: ${{ matrix.os }}
  steps:
    - name: Checkout Test Repository
      uses: actions/checkout@v4
      with:
        repository: Sewer56/prs-rs
        ref: d08599ed5473616f57d57a0966939e1a5dbda9b4
    - name: Test Library Build
      uses: Reloaded-Project/devops-rust-lightweight-binary@v1-test
      with:
        crate-name: prs-rs
        target: ${{ matrix.target }}
        use-pgo: ${{ matrix.use-pgo }}
        use-cross: ${{ matrix.use-cross }}
        features: "c-exports"
        build-library: true
        upload-artifacts: true
```

This will produce the following artifacts: 

- `C-Library-aarch64-apple-darwin-c-exports.zip`
- `C-Library-aarch64-pc-windows-msvc-c-exports.zip`
- `C-Library-aarch64-unknown-linux-gnu-c-exports.zip`
- `C-Library-armv7-unknown-linux-gnueabihf-c-exports.zip`
- `C-Library-i686-unknown-linux-gnu-c-exports.zip`
- `C-Library-i686-pc-windows-msvc-c-exports.zip`
- `C-Library-x86_64-apple-darwin-c-exports.zip`
- `C-Library-x86_64-pc-windows-msvc-c-exports.zip`
- `C-Library-x86_64-unknown-linux-gnu-c-exports.zip`


Running this action with the following configuration:

```yaml
- name: Build and Package .NET Library
  uses: Reloaded-Project/devops-rust-c-library-to-dotnet@v1
  with:
    csharp-project-path: bindings/csharp
    install-dotnet: true
```

Will perform the following.

1. Install .NET, if needed.
2. Download artifacts and insert them into appropriate subdirectories of `bindings/csharp`

```
bindings/csharp/
├── MyLibrary.csproj
├── runtimes/
│   ├── linux-x64/
│   │   └── native/
│   │       └── libmylibrary.so
│   ├── win-x64/
│   │   └── native/
│   │       └── mylibrary.dll
│   └── osx-x64/
│       └── native/
│           └── libmylibrary.dylib
└── // Other C# files and directories
```

3. Build & Pack the Project into a NuGet package.
4. Upload the NuGet package as an artifact, with name specified in `nuget-artifact-name`.

## Setting up a new Repository

This action was made to be used in conjunction with the excellent [Cysharp/csbindgen][csbindgen]
binding generator by Yoshifumi Kawai (neuecc).

Typical usage is as follows, [prs-rs][prs-rs] will be used as an example.

1. Set up `csbindgen` to generate C# bindings for your Rust library.

```rust
// build.rs in Rust project
fn main() {
    csbindgen::Builder::default()
        .input_extern_file("src/exports.rs")
        .csharp_dll_name("prs_rs")
        .csharp_class_accessibility("public")
        .csharp_namespace("prs_rs.Net.Sys")
        .generate_csharp_file("bindings/csharp/NativeMethods.g.cs")
        .unwrap();
}
```

```csharp
// Ensure DLL is loaded when requested.
class Init
{
    [ModuleInitializer]
    internal static void RegisterImportResolver()
    {
        // .NET Core 3.X and above only!!
        NativeLibrary.SetDllImportResolver(typeof(NativeMethods).Assembly, NativeMethods.DllImportResolver);
    }
}

/// <summary/>
public static unsafe partial class NativeMethods
{
    // https://docs.microsoft.com/en-us/dotnet/standard/native-interop/cross-platform
    // Library path will search
    // win => __DllName, __DllName.dll
    // linux, osx => __DllName.so, __DllName.dylib
    internal static IntPtr DllImportResolver(string libraryName, Assembly assembly, DllImportSearchPath? searchPath)
    {
        if (libraryName == __DllName)
        {
            var dllName = __DllName;
            var path = "runtimes/";
            var extension = "";

            if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
            {
                path += "win-";
                extension = ".dll";
            }
            else if (RuntimeInformation.IsOSPlatform(OSPlatform.OSX))
            {
                path += "osx-";
                extension = ".dylib";
                dllName = "lib" + dllName;
            }
            else
            {
                path += "linux-";
                extension = ".so";
                dllName = "lib" + dllName;
            }

            if (RuntimeInformation.ProcessArchitecture == Architecture.X86)
            {
                path += "x86";
            }
            else if (RuntimeInformation.ProcessArchitecture == Architecture.X64)
            {
                path += "x64";
            }
            else if (RuntimeInformation.ProcessArchitecture == Architecture.Arm)
            {
                path += "arm";
            }
            else if (RuntimeInformation.ProcessArchitecture == Architecture.Arm64)
            {
                path += "arm64";
            }

            path += "/native/" + dllName + extension;
            try
            {
                return NativeLibrary.Load(Path.Combine(AppContext.BaseDirectory, path), assembly, searchPath);
            }
            catch (DllNotFoundException)
            {
                return NativeLibrary.Load(Path.Combine(AppContext.BaseDirectory, dllName + extension));
            }
        }

        return IntPtr.Zero;
    }
}
```

For more information, read the [full documentation for csbindgen][csbindgen].

2. Use the [Example Above](#an-example) to set up a workflow.

## Typical Workflow

Once set up, usage is very easy. In fact, it's automatic.

1. You add 1 or more C exports to your Rust library.
2. When you next build, `csbindgen` will automatically update `NativeMethods.g.cs`.

You're done. Everything else is automated.

## Inputs

| Input                 | Required | Default           | Description                                                         |
| --------------------- | -------- | ----------------- | ------------------------------------------------------------------- |
| `artifact-prefix`     | No       | `C-Library-`      | Prefix for the artifact names to download                           |
| `csharp-project-path` | No       | `bindings/csharp` | Path to the C# project directory                                    |
| `install-dotnet`      | No       | `false`           | If true, installs the .NET SDK with specified version               |
| `dotnet-version`      | No       | `8.0.x`           | Version of the .NET SDK to install                                  |
| `nuget-artifact-name` | No       | `NuGet-Package`   | Name for the NuGet artifact                                         |

## Supported Targets

This action automatically injects the following target platforms:

- Windows (x86_64, i686, aarch64)
- Linux (x86_64, i686, aarch64, armv7)
- macOS (x86_64, aarch64)
- Android (x86_64, aarch64, armv7, i686)
- iOS (aarch64, x86_64)

Basically everything upstream .NET CoreCLR supports, and some hypothetical platforms.

## Output

The action generates a NuGet package containing the .NET librarys.
The NuGet package is uploaded as an artifact with the name specified by the
`nuget-artifact-name` input.

## Contributing

Contributions are welcome! If you encounter any issues or have suggestions for improvements, please
open an issue or submit a pull request in this repository.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

[rust-lightweight-library]: https://github.com/Reloaded-Project/devops-rust-lightweight-binary
[csbindgen]: https://github.com/Cysharp/csbindgen
[prs-rs]: https://github.com/Sewer56/prs-rs