## Cursor Cloud specific instructions

Native x86_64 Linux build of the CarPlay Communication Plug-in (`source/`) and vendored mDNSResponder. `binaries/` are Raspberry Pi 3 (armv7) libraries and do not run here. There is no package manager, no unit-test target, and no separate lint target. The Makefiles already pass `-Wall -Wextra` (`-Werror` is commented out). `.github/workflows/c-cpp.yml` expects `./configure` and `make check`, which this tree does not have.

Build mDNS first (`mDNSResponder/mDNSPosix/ReadMe.txt`):

```bash
make -C mDNSResponder/mDNSPosix os=linux
```

Then the CarPlay debug build. Pass `MDNSROOT` as the absolute path to this repo's `mDNSResponder`. The Accessory SDK sub-make sets `MDNSROOT` relative to `source/AccessorySDK`, so without the override it looks for `source/mDNSResponder` and fails on `dns_sd.h`.

```bash
make -C source/PlatformPOSIX os=linux linux=1 debug=1 stub=1 MDNSROOT="$PWD/mDNSResponder"
```

`stub=1` is the default and builds the audio, screen, and converter DLLs from the stub sources. Outputs are `mDNSResponder/mDNSPosix/build/prod/` (`mdnsd`, `libdns_sd.so`), `mDNSResponder/Clients/build/dns-sd`, and `source/build/Debug-linux/` (`airplayutil` and the shared libraries). The CarPlay Makefile copies headers with `rsync`. Ubuntu 24.04 no longer ships `<sys/sysctl.h>`; this environment provides a declaration-only compatibility header at `/usr/local/include/sys/sysctl.h`. Linux code paths do not call `sysctl()`.

`mdnsd` must be running before AirPlay or `dns-sd`. It binds UDP 5353, creates `/var/run/mdnsd`, and drops privileges, so start it with sudo. `-debug` stays in the foreground. `setsockopt - SO_RECV_ANYIF: Protocol not available` is expected on Linux and is not fatal.

```bash
sudo mDNSResponder/mDNSPosix/build/prod/mdnsd -debug
export LD_LIBRARY_PATH="$PWD/source/build/Debug-linux:$PWD/mDNSResponder/mDNSPosix/build/prod"
```

`dlopen` of `libAudioStream.so`, `libScreenStream.so`, and `libAudioConverter.so` uses that library path. `airplayutil help` is the CLI. `airplayutil mfi` needs `/dev/i2c-1` and the MFi auth IC. Without the IC, `AirPlayReceiverServerCreate` logs `MFi auth copy certificate failed: ENOENT` and the receiver still starts, advertises `_airplay._tcp` on TCP 7000, and reports `srcvers=320.17`.

`source/Examples/AppleCarPlay_AppStub.c` is not a Make target. Its `_RunUI` returns immediately, so a process built from that file exits and tears the server down. A receiver that stays up calls `AirPlayReceiverServerCreate`, `AirPlayReceiverServerStart`, then `CFRunLoopRun`. `make` of the Accessory SDK `all` target fails because `aktest` references `aktest.c`, `CFLiteNotifications.c`, and `HTTPTest.c`, which are not in the tree. The CarPlay Makefile builds only the `CoreUtils` target.
