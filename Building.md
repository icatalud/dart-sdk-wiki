[Dependencies](#dependencies)

[Getting the source](#source)

[Building](#building)

[Testing](#testing)

[Building the standalone VM only](#standalone)

<a name="dependencies"/>

# Dependencies

## C++11

Dart uses some new C++ features from C++11 (previously called C++0x).  Compilers that support these features
are GCC version 4.8 (4.6 is known not to work) and Microsoft Visual Studio 2015 (available in a free community edition for many users).  The clang compiler for linux, included in the Dart distribution, and the compilers used by Xcode on macOS (previously called OS X) are also known to work.

## Python 2 

Dart SDK current requires Python 2 to build as our build scripts are not compatible with Python 3. The depot tools come with it's own Python 2 version and when installed correctly there shouldn't be any Python version conflicts. If the depot tools Python version is not getting retrieved, check if the depot tools path environment variable is leading over other Python paths (it should be the leading path).

An alternative to not having depot tools Python lead over the previously installed Python is to use a virtual environment.

If you default installation is Python 3 you could workaround this by using `virtualenv`: 

```
$ virtualenv .env -p python2
$ source .env/bin/activate
# Can now run Dart SDK build scripts
```

## Linux

Install build tools:

```bash
sudo apt-get install g++-multilib git python curl
```

Install Chromium's [depot tools](http://dev.chromium.org/developers/how-tos/install-depot-tools):

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PATH:$PWD/depot_tools"
```

## Mac OS X

Install XCode.
> If you encounter the error: `xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance`, run the command: 
> ```
> sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
> ```
> See https://stackoverflow.com/questions/17980759.

Install Chromium's [depot tools](http://dev.chromium.org/developers/how-tos/install-depot-tools):
```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PATH:$PWD/depot_tools"
```

## Windows

Install VisualStudio.

Install Chromium's depot tools following [this](http://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up) or [this](https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md#install) instructions.

**Important:** If you are not a Googler make sure to set `DEPOT_TOOLS_WIN_TOOLCHAIN` system variable to `0` otherwise depot tools will attempt to pull down a Google-internal toolchain instead of using a local installation of Visual Studio.

<a name="source"/>

## Visual Studio (required for Windows)

Only VS 2017 and 2019 are compatible. Steps to get a free version of Visual Studio ready for building the SDK:
- Download [Visual Studio 2019 Community](https://visualstudio.microsoft.com/vs/community/) and install.
- Open Visual Studio and in the menu go to Tools -> Get Tools and Features. Select "Desktop Development with C++" in the Workloads tab (default tab) and install.
- Go to Windows Apps & Features (where you uninstall programs), search for "Windows Software Development Kit" and click modify. Select "Change", continue, select "Debugger" in the checkbox list and proceed.

# Getting the source

You can choose whatever name you want for the directory to check dart out in, here it is called `dart-sdk`.

```bash
mkdir dart-sdk
cd dart-sdk
fetch dart
```

<a name="building"/>

# Building

Build the 64-bit SDK:

```bash
cd dart-sdk/sdk
python ./tools/build.py --mode release --arch x64 create_sdk
```

The output will be in `out/ReleaseX64/dart_sdk` on Linux and Windows, and `xcodebuild/ReleaseX64/dart_sdk` on MacOS.

Build the 32-bit SDK:

```bash
cd dart-sdk/sdk
python ./tools/build.py --mode release --arch ia32 create_sdk
```
The output will be in `out/ReleaseIA32/dart_sdk` on Linux and Windows, or `xcodebuild/ReleaseIA32/dart_sdk` on MacOS.

See also [building for ARM](https://github.com/dart-lang/sdk/wiki/Building-Dart-SDK-for-ARM-processors).

## Tips

By default the build and test scripts select the debug binaries. You can build and test the release version of the VM by specifying `--mode=release` or both debug and release by specifying `--mode=all` on the respective `build.py` and `test.py` command lines.  This can be shortened to `-mrelease` or `-m release`, and the architecture can be specified with `--arch=ia32` or `-a x64`, the default.  Other architectures, like `arm`, `arm64`, and `mips`, are also supported.

We recommend that you use a local file system at least for the output of the builds. The output directory is `out` on linux, `xcodebuild` on Mac OS, and `build` on Windows.  If your code is in some NFS partition, you can link the `out` directory to a local directory:
```bash
$ cd sdk/
$ mkdir -p /usr/local/dart-out/
$ ln -s -f /usr/local/dart-out/ out
```

### Notification when build is done

The environment variable `DART_BUILD_NOTIFICATION_DELAY` controls if `build.py` displays a notification when the build is done. If a build takes longer than `DART_BUILD_NOTIFICATION_DELAY` a notification will be displayed.

A notification is a small transient non-modal window, for now, only supported on Mac and Linux.

## Special note for Windows users using Visual Studio Community Edition:
Your Visual Studio executable command may have a different name from the standard Visual Studio installations. You can specify the name for that executable by passing the additional flag "--executable=$VS\_EXECUTABLE\_NAME" to build.py. The executable name will probably be something like "VSExpress.exe".

## Building on Windows with Visual Studio 2015
Gyp should autodetect the version of Visual Studio you have, and produce solution files in the correct format.
If this is not happening, then set environment variable gyp\_msvs\_version to 2015.

For example, this will produce Visual Studio 2015-compliant solution files:
```bash
set gyp_msvs_version=2015
gclient runhooks
```

## Building revisions from before August 2017

At the end of July 2017, the `gclient` tool's behavior changed.  If you try to build old versions of the SDK from before that time (e.g., for `git bisect`), it might fail.  If you follow a workflow where you check out a commit and then run `gclient sync` you might encounter a message like:

```
sdk (ERROR)
----------------------------------------
[0:00:00] Started.
[0:00:00] ________ unmanaged solution; skipping sdk
----------------------------------------
```

This is because `gclient` has changed the domain-specific language of the `DEPS` files in a way that makes our old `DEPS` files invalid; and because `gclient` will automatically update itself to the latest revisionwhenever it is run.

There is a workaround:

  1. In your `depot_tools` repository, check out the last `gclient` commit that works with our old `DEPS` files:

     ```
     git checkout 5aa5cd76f00e7774f71367f34d9998cfa0034d04
     ```

  2. In your Dart SDK repository, run `gclient` without updating itself:

     ```
     DEPOT_TOOLS_UPDATE=0 gclient sync
     ```

An then build according to the instructions for your platform.

<a name="testing"/>

# Testing

All tests are executed using the `test.py` script under `tools/`.  You may need to build everything once before testing, using `build.py` as described above. E.g.
```bash
$ ./tools/build.py --mode release --arch ia32
```

Now you can run all tests as follows (Safari, Firefox, Chrome, and IE must be installed, if you want to run the tests on them.  Dartium, and drt (content\_shell) are automatically downloaded by the test scripts.):
```bash
$ ./tools/test.py -mrelease --arch=ia32 --compiler=none,dart2js --runtime=vm,d8,drt,chrome,dartium,ff,[safari,ie]
```
Specify the compiler used (optional -- only necessary if you are compiling to JavaScript (required for most browsers), the default is "none") and a runtime (where the code will be run).

You can run a specific test by specifying its full name or a prefix. For instance, the following runs only tests from the core libraries:
```bash
$ ./tools/test.py -mrelease --arch=ia32 --runtime=vm corelib
```
The following runs a single test:
```bash
$ ./tools/test.py -mrelease --arch=ia32 --runtime=vm corelib/ListTest
```

Make sure to run tests using the release VM if you have built the release VM, and for the same architecture as you have built.

See also [Testing Dart2js](https://github.com/dart-lang/sdk/wiki/Testing-Dart2js) for dart2js specific examples.

## Complex examples

Adjust the flags to your needs: appropriate values for `--arch` and `--tasks` will depend on your system.  The below examples are taken from a 12 core x86\_64 system.

Dart analyzer tests example:
```bash
./tools/test.py \
  --compiler dart2analyzer \
  --runtime none \
  --progress color \
  --arch x64 \
  --mode release \
  --report \
  --time \
  --tasks 6 \
  language
```

VM tests example:
```bash
./tools/test.py \
  --compiler none \
  --runtime vm \
  --progress color \
  --arch x64 \
  --mode release \
  --checked \
  --report \
  --time \
  --tasks 6 \
  language
```

Dart2JS example:
```bash
./tools/test.py \
  --compiler dart2js \
  --runtime chrome \
  --progress color \
  --arch x64 \
  --mode release \
  --checked \
  --report \
  --time \
  --tasks 6 \
  language
```

## Troubleshooting and tips for browser tests
To debug a browser test failure, you must start a local http server to serve the test.  This is made easy with a helper script in dart/tools/testing.  The error report from test.py gives the command line to start the server in a message that reads:
```bash
To retest, run:  /usr/local/[...]/dart/tools/testing/bin/linux/dart /usr/local/[...]/dart/tools/testing/dart/http_server.dart ...
```
After starting the server, you can run the browser using the command line starting with `Command[[browser name]]`.  The debugging tools in the browser can be used to set Javascript or Dart breakpoints, and the reload button should rerun the test.
    * **Tip:** In the Sources tab of Chrome's developer tools, a "stop sign" button in the upper right tells the browser to break when an exception is thrown.

Some common problems you might get when running tests for the first time:
  * Some fonts are missing. Chromium tests use `DumpRenderTree`, which needs some sets of fonts. See [LayoutTestsLinux](http://code.google.com/p/chromium/wiki/LayoutTestsLinux) for instructions in how to fix it.
  * No display is set (e.g. running tests from an ssh terminal). `DumpRenderTree` is a nearly headless browser. Even though it doesn't open any GUI, it still must have an X session available. There are several ways to work around this:
    * Use `xvfb`:

      ```bash
      xvfb-run ./tools/test.py --arch=ia32 --compiler=dart2js --runtime=drt ...
      ```

    * Other options include using ssh X tunneling (`ssh -Y`), NX, VNC, setting up `xhost` and exporting the DISPLAY environment variable, or simply running tests locally.

<a name="standalone"/>

# Building the standalone VM only

You can tell the build script to build the runtime targets only with the command:
```bash
$ ./tools/build.py runtime
```
