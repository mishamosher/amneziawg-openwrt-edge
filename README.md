# YAAWG: Yet another AmneziaWG variation for OpenWrt

This project is aimed to update sources of the initial AmneziaWG repository and make them as close to four upstream projects ([luci-proto-wireguard](https://github.com/openwrt/luci/tree/master/protocols/luci-proto-wireguard), [amneziawg-tools](https://github.com/amnezia-vpn/amneziawg-tools/), [amneziawg-linux-kernel-module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module), [amneziawg-go](https://github.com/amnezia-vpn/amneziawg-go)) as possible.

Why? Because it looks like the original repository is abandoned: more than half a year has passed since the last commit, no bugs were fixed and no new protocol versions support were added while four upstream projects receive updates on a regular basis (at least from the community).

The main differences and objectives:
1. `luci-proto-amneziawg` has been aligned in accordance with [luci-proto-wireguard](https://github.com/openwrt/luci/tree/master/protocols/luci-proto-wireguard):
   - AmneziaWG settings tab now have placeholders where available (default values, if chosen, will mimic classic Wireguard protocol).
   - Fixed bug with the QR code generation. Please note that the generated QR code will contain AmneziaWG specific information (remove it manually to make it compatible with the original Wireguard protocol).
   - Added checkboxes to enable/disable peers.
   - Took `luci-proto-wireguard` as the codebase.
   - Added the correct icon for the interface.
   - Added support for ranged H1-H4 parameters (with `-` delimiter, e.x. `123456-123500`).
   - Added support for v2.0 protocol parameters: S3-S4, I1-I5.
3. `amneziawg-tools` has been aligned  in accordance with the upstream repo [amneziawg-tools](https://github.com/amnezia-vpn/amneziawg-tools/):
   - The package is now compiled based on the upstream repo.
   - Fixed bug with non-existent `proto_amneziawg_check_installed` method.
   - Changed temp folders and files to match the protocol name.
   - Refactored scripts a bit to make them look more `amneziish`.
   - Fixed bug with incorrect path when using `amneziawg-go`.
   - Added support for ranged H1-H4 parameters (with `-` delimiter, e.x. `123456-123500`).
   - Added support for v2.0 protocol parameters: S3-S4, I1-I5.
4. `kmod-amneziawg` is now compiled totally based on the upstream [amneziawg-linux-kernel-module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module) repo.
   - Added support for v2.0 protocol parameters: S3-S4, I1-I5.
5. `amneziawg-go` is as an alternative for `kmod-amneziawg`. Please check [this section](#kmod-amneziawg-vs-amneziawg-go) for more information. The Go implementation is also totally based on the upstream project [amneziawg-go](https://github.com/amnezia-vpn/amneziawg-go).

# `kmod-amneziawg` vs `amneziawg-go`
When AmneziaWG authors first introduced v1.5 protocol it was supported only in the Go implementation. Thus the user namespace (Go) implementation was added to the repo in order to support a newer version of the protocol. After that v2.0 protocol support has been added to the kernel module implementation. To keep the backwards compatibility this repo will continue to support both Go and kernel module implementation.
The differences are:
1. `kmod-amneziawg`: requires a less powerful device to run and consumes less space, but it still is in the beta state. Use at your own risk.
2. `amneziawg-go`: requires a more powerful device to run and consumes more space, but provides a user namespace implementation of the protocol.

It is recommended to use `amneziawg-go` implementation because the kernel module is still in the beta state and might not work as expected.
Please choose and install only one implementation. If both implementations have been installed, `kmod-amneziawg` will be used by default.

# Results

Everything seems to work fine. No major problems have been detected or reported so far.

# How to build and use

## Build OpenWRT firmware with AmneziaWG packages included

This repository is primarily intended for compiling packages during the firmware build process. To do this follow the steps:
1. Clone the OpenWrt repo by running `git clone https://github.com/openwrt/openwrt.git` command. You may also choose any tag/commit hash you want by adding `-b {tag/commit hash}` to the command.
2. Add line `src-git awgopenwrt https://github.com/this-username-has-been-taken/amneziawg-openwrt.git` to the `feeds.conf.default` file.
3. Update the package feeds by running `{path to openwrt dir}/scripts/feeds update -a` command.
4. If you are going to build a firmware with the `amneziawg-go` package make sure that Go package version included in the firmware is higher than `1.24.4`. Unfortunately at the moment all OpenWRT versions except `SNAPSHOT` have older versions of the Go package. In order to build the firmware successfully you have to upgrade it:
   4.1. Clone the latest OpenWRT Packages repository somewhere by running `https://github.com/openwrt/packages.git`.
   4.2. Replace `{path to openwrt dir}/feeds/packages/lang/golang` folder with the one from the repository you have just cloned: `{patch to the cloned repository dir}/packages/lang/golang`.
5. Install the packages by running `{path to openwrt dir}/scripts/feeds install -a` command.
6. Configure the firmware (choose target, settings, AmneziaWG packages: `amneziawg-go` or `kmod-amneziawg` + `amneziawg-tools` + `luci-proto-amneziawg` and other packages you need) in the menuconfig by running `make -C {path to openwrt dir} menuconfig` command and save the configuration.
7. Make the defconfig: `make -C {path to openwrt dir} defconfig`.
8. Build the firmware: `make -C openwrt -j$(nproc) V=sc`.
9. After the process is finished your firmware will be located at `{path to openwrt dir}/bin/targets/{your target}/{your subtarget}`. Compiled packages will be located at `{path to openwrt dir}/bin/targets/{your target}/{your subtarget}/packages` (for the kernel module) and `{path to openwrt dir}/bin/packages/{your architecture}/awgopenwrt` (for the other packages).

## Compile AmneziaWG packages without building the firmware

You can compile the packages without building the firmware.

Side note: the compilation process is not hard at all, but requires some attention and basic knowledge.
The process consists of two workflows: the first one (takes up to 2.5 hours) builds an OpwnWRT toolchain and the second one (takes less than 20 minutes) compiles AmneziaWG packages. The second workflow requires the first workflow to be completed once. If the second workflow fails or if you want to change the package list it can be restarted as many times as you want.

General steps:
1. Get the parameters for your router:
     - OpenWRT version: `SNAPSHOT` or release (e.x. `24.10.2`). Can be found on the `Status -> Overview` page (`Firmware Version` parameter value). If you are using the `SNAPSHOT` version please check the section about vermagic below.
     - CPU/package architecture: run `apk info kernel` or `opkg info kernel` (depending on the package manager) in the console and check the `Architecture` value (e.x. `aarch64_cortex-a53`) or consult [OpenWRT router database](https://openwrt.org/toh/start).
     - Target: can be found on the `Status -> Overview` page. The first part (before the slash) of the `Target Platform` value.
     - Subtarget: can be found on the `Status -> Overview` page. You guessed it! The second part (after the slash) of the `Target Platform` value.
3. Make a fork of this repo.
4. Go to Actions (enable them if not available).
5. Choose `Build OpenWrt toolchain cache`, put your router parameters (from step 1) in the `Run workflow` menu and run the workflow.
   5.1. Optionally you can choose another version of the YAWWG by setting the release tag/commit hash in the `AmneziaWG version` field.
   5.2. Do not disable `Update Go` checkbox unless you know what you are doing. `amneziawg-go` package requires Go package version higher than `1.24.4`.
6. It will take ~2-2.5 hours to build the cache. So get some cookies, tea, your favorite book and wait.
7. After the cache has been created, choose `Build AmneziaWG from cache` workflow, put the same parameters in the `Run workflow` menu and run it.
   7.1 You can choose whether to compile the kernel module, Go implementation or both at this step by checking the corresponding checkboxes.
8. It will take ~10-20 minutes to build the binaries. After the process has been finished you can download them in the workflow's artifacts section (bottom of the page).
9. Unpack the archive, and install:
   - Via WebInterface (LuCi):
       - Go to `System -> Software` menu.
       - Press `Upload Package...`
       - Select `kmod-amneziawg` or `amneziawg-go` .ipk/.apk file.
       - Confirm installation.
       - Repeat the steps for `amneziawg-tools` .ipk/.apk file and then `luci-proto-amneziawg` .ipk/.apk file.
   - Via console:
       - Transfer files into the router.
       - Run `apk install {path to the kmod-amneziawg or amneziawg-go .apk}` or `opkg install {path to the kmod-amneziawg or amneziawg-go .ipk}` depending on the package manager.
       - Run `apk install {path to the amneziawg-tools .apk}` or `opkg install {path to the amneziawg-tools .ipk}` depending on the package manager.
       - Run `apk install {path to the luci-proto-amneziawg .apk}` or `opkg install {path to the luci-proto-amneziawg .ipk}` depending on the package manager.
10. Reboot the router or run `/etc/init.d/network restart` command in the console.
11. Congratulations - you now have AmneziaWG installed on your router. Go to `Network -> Interfaces` page, press `Add new interface..` and select `AmneziaWG` as a protocol.
*Sometimes is is required to clean the browser's cache to see the new protocol available in the OpenWRT.

### Vermagic control for `SNAPSHOT` versions

Vermagic value - is a hash calculated for the OpenWRT kernel. When installing a kernel-related package the OpenWRT always checks if `vermagic` parameter of the package equals the same parameter of the kernel. If it is not, then the package won't be installed.
`SNAPSHOT` versions of the firmware are updated on a daily basis thus it is possible that `vermagic` values will be different. You can check your firmware `vermagic` by running `apk info kernel` or `opkg info kernel` (depending on the package manager) command in the console and check hash value after the kernel version in the `Version` field. E.x. if the value equals to `6.6.52~f58afd3748410d3b1baa06a466d6682-r1` then `vermagic` equals to `f58afd3748410d3b1baa06a466d6682`. The compiled package `vermagic` value is available in the `vermagic` file included in the workflow artifacts. If those two do not match then the kernel module implementation cannot be installed.
