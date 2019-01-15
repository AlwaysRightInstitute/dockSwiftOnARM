<h2>macOS -> RasPi Cross Compilation Toolchain
  <img src="http://zeezide.com/img/rpi-swift.svg?2"
       align="right" width="180" height="180" />
</h2>

![Swift3](https://img.shields.io/badge/swift-3-blue.svg)
![tuxOS](https://img.shields.io/badge/os-Xenial-green.svg?style=flat)
![ARM](https://img.shields.io/badge/cpu-ARM-red.svg?style=flat)

End of April
[Johannes Weiß](https://github.com/weissi)
added
[custom toolchain support](https://github.com/apple/swift-package-manager/pull/1098)
to 
[Swift Package Manager](https://github.com/apple/swift-package-manager).
Johannes also provided 
[a script](https://github.com/apple/swift-package-manager/blob/master/Utilities/build_ubuntu_cross_compilation_toolchain)
which shows how to build an Ubuntu toolchain for x86-64.
So what we did is take that script and make it produce a Swift 3.1.1 cross 
compiler toolchain for the Raspberry Pi (armhf) Ubuntu Xenial port.

What this is good for?
You can build Raspberry Pi Swift binaries on a Mac. Like this:
```
mkdir helloworld && cd helloworld
swift package init --type=executable
swift build --destination /tmp/cross-toolchain/rpi-ubuntu-xenial-destination.json
file .build/debug/helloworld
.build/debug/helloworld: ELF 32-bit LSB executable, ARM, 
                         EABI5 version 1 (SYSV), dynamically linked, 
                         interpreter /lib/ld-linux-armhf.so.3, 
                         for GNU/Linux 3.2.0, not stripped
```

(We also have a toolchain kit which does the reverse, compile macOS Swift 
 binaries on a Raspberry Pi: [macos](macos/README.md))
 

## UPDATE 2019-01-15

Swift 4.2 Raspi builds for arm64v8 are now available, and 5.0dev builds too.
Also
[PR-1546](https://github.com/apple/swift-package-manager/pull/1546)
got merged, not sure in which version though.

TODO: Check whether we can get a 4.2.1 X working.

 
## UPDATE 2018-05-09

The tested and working version is still Swift 3.1 w/ 32-bit Raspi Ubuntu.
However, Swift 4.1 builds start popping up, so we are working on

1. Swift 4.1 32-bit Raspi - branch [swift-4.1-release](https://github.com/AlwaysRightInstitute/swift-mac2arm-x-compile-toolchain/tree/swift-4.1-release)
2. Swift 4.1 64-bit Raspi - branch [swift-4.1-arm64v8](https://github.com/AlwaysRightInstitute/swift-mac2arm-x-compile-toolchain/tree/swift-4.1-arm64v8)

toolchains.

This was blocked because SPM 4.1 b0rked support for custom target triples
([PR-1546](https://github.com/apple/swift-package-manager/pull/1546)).
And it still is, even Swift 4.2 won't ship this.


## Building the ARM toolchain

What we are going to do is build a Swift 3.1.1 cross compilation toolchain
for ARM Ubuntu Xenial.
Note that we are going to use the Swift 4 package manager, but the Swift 3.1
compiler (because Swift 4 on Raspi is not in shape yet).

Requirements:
- Xcode 9 or later (http://developer.apple.com/)
- a Raspi 3 w/ Ubuntu Xenial

Recommended:
- [Docker for Mac](https://docs.docker.com/docker-for-mac/install/)

### Build Toolchain using Script

First download our script and make it executable:
[build_rpi_ubuntu_cross_compilation_toolchain](https://raw.githubusercontent.com/AlwaysRightInstitute/swift-mac2arm-x-compile-toolchain/master/build_rpi_ubuntu_cross_compilation_toolchain),
e.g. like:

```
pushd /tmp
curl https://raw.githubusercontent.com/AlwaysRightInstitute/swift-mac2arm-x-compile-toolchain/master/build_rpi_ubuntu_cross_compilation_toolchain \
  | sed "s/$(printf '\r')\$//" \
  > build_rpi_ubuntu_cross_compilation_toolchain
chmod +x build_rpi_ubuntu_cross_compilation_toolchain
```

Next step is to download Swift 3.1.1 tarballs. 
We need the macOS pkg for the host compiler and a Raspberry Pi tarball for the
Swift runtime. We use the one provided by Florian Friedrich for the latter:

```
pushd /tmp
curl -o /tmp/swift-3.1.1-armv7l-ubuntu16.04.tar.gz https://cloud.sersoft.de/nextcloud/index.php/s/0qty8wJxlgfVCcx/download
curl -o /tmp/swift-3.1.1-RELEASE-osx.pkg https://swift.org/builds/swift-3.1.1-release/xcode/swift-3.1.1-RELEASE/swift-3.1.1-RELEASE-osx.pkg
```
Those are a little heavy (~400 MB), so grab a 🍺 or 🍻.
Once they are available, build the actual toolchain using the script
(it will take a minute or two to build through the dependencies):

```
pushd /tmp
./build_rpi_ubuntu_cross_compilation_toolchain \
  /tmp/ \
  /tmp/swift-3.1.1-RELEASE-osx.pkg \
  /tmp/swift-3.1.1-armv7l-ubuntu16.04.tar.gz
```

If everything worked fine, it'll end like that:
```
OK, your cross compilation toolchain for Raspi Ubuntu Xenial is now ready to be used
 - SDK: /tmp/cross-toolchain/rpi-ubuntu-xenial.sdk
 - toolchain: /tmp/cross-toolchain/swift.xctoolchain
 - SwiftPM destination.json: /tmp/cross-toolchain/rpi-ubuntu-xenial-destination.json
```

### Use the Toolchain

Lets create a simple `helloworld` tool first:

```
mkdir helloworld && cd helloworld
swift package init --type=executable
swift build --destination /tmp/cross-toolchain/rpi-ubuntu-xenial-destination.json
```

Which gives:
```
Compile Swift Module 'helloworld' (1 sources)
Linking ./.build/debug/helloworld
```

Check whether it actually produced an ARM binary:
```
file .build/debug/helloworld
.build/debug/helloworld: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, not stripped
```

Excellent! It worked. Now either copy your binary to a Raspi or test it using
QEmu as described below.

If you are not using our 
[Docker image](https://hub.docker.com/r/helje5/rpi-swift/),
you may need to setup the `LD_LIBRARY_PATH`, so that the dynamic linker finds
the Swift runtime. E.g. like that:

```
sudo cat > /etc/ld.so.conf.d/swift.conf <<EOF
/usr/lib/swift/linux
/usr/lib/swift/clang/lib/linux
/usr/lib/swift/pm
EOF
sudo ldconfig
```

If that didn't work, you'll see an error like:

    ./helloworld: error while loading shared libraries: libswiftCore.so: \
      cannot open shared object file: No such file or directory


## Testing builds using Docker on macOS

Docker for Mac comes with QEmu support enabled, meaning that you can run
simple ARM binaries without an actual Raspberry Pi.

```
docker run --rm --tty -i -v "$PWD/.build/debug/:/home/swift" \
  helje5/rpi-swift:3.1.1 \
  ./helloworld
```

This works for simple builds, more complex stuff does not run in QEmu. Use
a proper Pi for that :-)


## No README w/o 🐄🐄🐄

Lets build something very useful, an ASCII cow generator.
The snapshot's `swift package init` produces a Swift 4 setup by default.
We want to use 3.1, so we do the setup manually:

```
mkdir vaca && cd vaca
cat > Package.swift <<EOF
import PackageDescription

let package = Package(
  name: "vaca",
  dependencies: [
    .Package(url: "https://github.com/AlwaysRightInstitute/cows.git",
             majorVersion: 1, minor: 0)
  ]
)
EOF

cat > main.swift <<EOF
import cows
print(vaca())
EOF
```

Then build the thing:

```
swift build \
  --destination /tmp/cross-toolchain/rpi-ubuntu-xenial-destination.json
Fetching https://github.com/AlwaysRightInstitute/cows.git
Cloning https://github.com/AlwaysRightInstitute/cows.git
Resolving https://github.com/AlwaysRightInstitute/cows.git at 1.0.1
Compile Swift Module 'cows' (3 sources)
Compile Swift Module 'vaca' (1 sources)
Linking ./.build/debug/vaca
```

And you get the most awesome Swift tool:

```
docker run --rm --tty -i -v "$PWD/.build/debug/:/home/swift" \
           helje5/rpi-swift:3.1.1 ./vaca
   (___)
   (o o)
  __\_/__
 //^^*^^\
 /   *   \
/ |  *  | \
\ |=====| /
 "|_____|"
   | | |
   | | |
   |_|_|
    ^ ^
   COWNT
```

Wanna have Server Side Cows on the Pi? Try this:
[mod_swift](http://mod-swift.org/raspberrypi/).
Having the cows on your Raspi is not enough?
Get: [CodeCows](https://itunes.apple.com/de/app/codecows/id1176112058) 
for Xcode and macOS,
and [ASCII Cows](https://itunes.apple.com/de/app/ascii-cows/id1176152684)
for iOS.


## Notes of interest

- `#if swift(>=4.0)` will be true. Seems like a bug in Swift that the version
  is attached to the driver, not to the active language.
  In other words: `#if swift` version checks are useless.
  Same BTW for `#if os` checks in the Package.swift. Those do not account for
  cross compilers.
- SwiftPM reuses the `.build` directory even if you call it w/ 
  different destinations. So make sure you clean before building for a
  different architecture.
- Ubuntu system headers and such for the toolchain are directly pulled
  from the Debian packages (which are retrieved from the Ubuntu repository)
- The cross compiler is just a regular clang/swiftc provided as part of
  a macOS toolchain. Yes, clang/swift are always setup as cross compilers
  and can produce binaries for all supported targets! (didn't know that)
- To trace filesystem calls on macOS you can use `fs_usage`, e.g.:
  `sudo fs_usage -w -f pathname swift` (I only knew `strace` ;-)
- `swift build --static-swift-stdlib` does not seem to work. Presumably because
  the Swift 3 host compiler does not support the necessary flags.

### Who

Brought to you by
[The Always Right Institute](http://www.alwaysrightinstitute.com)
and
[ZeeZide](http://zeezide.de).
We like 
[feedback](https://twitter.com/ar_institute), 
GitHub stars, 
cool [contract work](http://zeezide.com/en/services/services.html),
presumably any form of praise you can think of.
We don't like people who are wrong.

<table width="100%" border="0">
  <tr>
    <td align="center" width="20%">
      <a href="http://apacheexpress.io"
        ><img src="http://zeezide.com/img/ApexIcon128.png" width="64" height="64" /></a>
    	<br />
    	ApacheExpress
    </td>
    <td align="center" width="20%">
      <a href="http://mod-swift.org"
        ><img src="http://zeezide.com/img/mod_swift-128x128.png" width="64" height="64" /></a>
    	<br />
    	mod_swift
    </td>
    <td align="center" width="20%">
      <a href="http://zeeql.io"
        ><img src="http://zeezide.com/img/ZeeQLIconQL128.png" width="64" height="64" /></a>
      <br />
      ZeeQL
    </td>
    <td align="center" width="20%">
      <a href="http://noze.io"
        ><img src="https://pbs.twimg.com/profile_images/725354235056017409/poiNAOlB_400x400.jpg" width="64" height="64" /></a>
      <br />
      Noze.io
    </td>
    <td align="center" width="20%">
      <a href="https://github.com/ZeeZide/UXKit"
        ><img src="http://zeezide.com/img/UXKitIcon1024.png" width="64" height="64" /></a>
      <br />
      UXKit
    </td>
  </tr>
</table>

There is the [swift-arm](https://slackpass.io/swift-arm) Slack channel
if you have questions about running Swift on ARM/Raspberry Pi.
