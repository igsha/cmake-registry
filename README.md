# cmake-registry

CMake Registry Manager is a small tool to work with cmake local registry.

# Motivations

E.g., I have a project `bit_iterator` that can represent file as a bit stream.
Another project `bit_parser` uses my `bit_iterator` to parse a file bit by bit.
I need to connect these projects.

The obvious way is to create installable package for `bit_iterator` like deb-package.
But I need to contantly fix both project at any time.
`CMake` is a build system that I always use.
It has the local [registry](https://cmake.org/cmake/help/latest/manual/cmake-packages.7.html#package-registry)
that can be used for my goals.

# Installation

Just put `cmake-registry` executable file (a simple `bash`-script) into `/usr/local/bin` or `~/bin` or whatever you want
(add it\`s location into `PATH` system variable).

# Help

`cmake-registry` has a small help, just type `cmake-registry help`.
It supports getting help by topic: `cmake-registry help install`.

# Registry flow

Let's install `bit_iterator` into cmake registry.
The first approach is to install a `tar.gz` archive into the cache of the registry (`$HOME/.cmake/cache`).
```sh
$ ./cmake-registry install ~/bit_iterator/build/bit_iterator-0.1.0-Linux.tar.gz
Installing /home/igor/bit_iterator/build/bit_iterator-0.1.0-Linux.tar.gz into /home/igor/.cmake/cache
Created path-config for project : /home/igor/.cmake/packages/BitIterator/542d8ff09ae330ea4baaa87b2f1ba4ee
```

This command creates so-called `path-config` that contains the location of `BitIteratorConfig.cmake` file.
The last one is a project configuration file that is used by `cmake` to correctly set include and library paths.
My `BitIteratorConfig.cmake` contains:
```cmake
find_path(BitIterator_INCLUDE_DIRS NAMES bit_iterator.hpp HINTS ${CMAKE_CURRENT_LIST_DIR}/../../include)
```
It sets `INCLUDE_DIRS` relatively to the script directory.

Now my `bit_parser` `CMakeLists.txt` file calls `find_package(BitIterator REQUIRED)` and nothig else.
`cmake` looks into `/home/igor/.cmake/packages/BitIterator/542d8ff09ae330ea4baaa87b2f1ba4ee`, sees
```sh
$ cat ~/.cmake/packages/BitIterator/542d8ff09ae330ea4baaa87b2f1ba4ee 
/home/igor/.cmake/cache/bit_iterator-0.1.0-Linux/share/cmake
$ ls ~/.cmake/cache/bit_iterator-0.1.0-Linux/share/cmake
BitIteratorConfig.cmake  BitIteratorConfigVersion.cmake
```
Then reads `BitIteratorConfig.cmake` and correctly sets `BitIterator_INCLUDE_DIRS` variable.

The second approach is to point `path-config` to `bit_iterator` build tree folder.
After `make package` the build tree folder contains `_CPack_PACKAGES` folder that is the content of `tar.gz` package archive.
To create the link in registry I just need to pass the full path of the `BitIteratorConfig.cmake` file:
```sh
$ ./cmake-registry make ~/bit_iterator/build/_CPack_Packages/Linux/TGZ/bit_iterator-0.1.0-Linux/share/cmake/BitIteratorConfig.cmake
Created path-config for project : /home/igor/.cmake/packages/BitIterator/4c74e74620b1ba554d7d65776b243518
$ cat /home/igor/.cmake/packages/BitIterator/4c74e74620b1ba554d7d65776b243518
/home/igor/bit_iterator/build/_CPack_Packages/Linux/TGZ/bit_iterator-0.1.0-Linux/share/cmake
```

Now I can see all my installations through `cmake-registry list`:
```sh
$ ./cmake-registry list
BitIterator/4c74e74620b1ba554d7d65776b243518: /home/igor/bit_iterator/build/_CPack_Packages/Linux/TGZ/bit_iterator-0.1.0-Linux/share/cmake
BitIterator/542d8ff09ae330ea4baaa87b2f1ba4ee: /home/igor/.cmake/cache/bit_iterator-0.1.0-Linux/share/cmake
```

To remove `path-config` use `remove` command (you need to pass left-hand name returned by `list` command):
```sh
$ ./cmake-registry remove BitIterator/4c74e74620b1ba554d7d65776b243518
Removed path-config for project BitIterator/4c74e74620b1ba554d7d65776b243518
```
The cache packages should be uninstalled:
```sh
$ ./cmake-registry uninstall BitIterator/542d8ff09ae330ea4baaa87b2f1ba4ee
Uninstalled BitIterator/542d8ff09ae330ea4baaa87b2f1ba4ee
Removed path-config for project BitIterator/542d8ff09ae330ea4baaa87b2f1ba4ee
```

# Make path-config for an existing package

Some helpful packages does not have their own `FindPackage.cmake` in cmake repository.
Let me show how to add them into the registry.
For example, `Catch` package.
I need only `CatchConfig.cmake` file in my cache and `path-config`.
The command `create` takes 2 arguments: the name of the packet and location of `CatchConfig.cmake`.
The last one is optional.
If it is omitted the stdin will be read for content of `CatchConfig.cmake`.
```sh
$ echo "find_path(CATCH_INCLUDE_DIRS NAMES catch/catch.hpp)" | ./cmake-registry create Catch
Created path-config for project : /home/igor/.cmake/packages/Catch/9e4a4c3341da669737a34c39e688f51d
$ cat /home/igor/.cmake/packages/Catch/9e4a4c3341da669737a34c39e688f51d
/home/igor/.cmake/cache/Catch
$ cat /home/igor/.cmake/cache/Catch/CatchConfig.cmake
find_path(CATCH_INCLUDE_DIRS NAMES catch/catch.hpp)
```
To have such `CatchConfig.cmake` you should have the path of installed `Catch` in your `CMAKE_PREFIX_PATH` variable.

# Diagnostic

If I manually remove my `Catch` folder from the cache registry, the `list` command will show the absence of the path:
```sh
$ rm -r /home/igor/.cmake/cache/Catch
$ ./cmake-registry list
BitIterator/15cca7d08a511bac1a4af9dd66cfb44f: /home/igor/bit_iterator/build/_CPack_Packages/Linux/TGZ/bit_iterator-0.1.0-Linux/share/cmake
Catch/9e4a4c3341da669737a34c39e688f51d: /home/igor/.cmake/cache/Catch (The given path does not exists!)
```
Now, the message `(The given path does not exists!)` shows that something wrong with `Catch` package.
