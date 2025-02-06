This branch is only for building openwrt for the Fritzbox 7490 with WiFi support
# Quickstart
Build the Lantiq image

```
# Download and update the sources
git clone https://github.com/Zappception/openwrt
cd openwrt
git pull
 
# Select a specific code revision
git branch -a
git tag
git checkout v24.10.0-fritz.box.7490
 
# Update the feeds
./scripts/feeds update -a
./scripts/feeds install -a
 
# Configure the firmware image
make menuconfig

# Select following:
# Target System (Lantiq)
# Subtarget (XRX200)
# Target Profile (AVM FRITZ!Box 7490 Micron NAND) or Target Profile (AVM FRITZ!Box 7490 Other NAND)
# See: https://openwrt.org/toh/avm/fritz.box.7490#installation

# Select LuCI --> Collections ---> luci
# Select whatever you need

# Build the firmware image
make -j$(nproc) defconfig download clean world

# Save .config
cp .config .config-lantiq
```

Flash the Lantiq image

SSH into the Lantiq image and copy `/lib/firmware/ath9k-eeprom-ahb-18100000.wmac.bin` and `/lib/firmware/ath10k/cal-pci-0000:00:00.0.bin` to your PC

Copy ath9k eeprom and ath10k caldata to the placeholders at `target/linux/ath79/generic/base-files/lib/firmware/ath9k-eeprom-ahb-18100000.wmac.bin`
and `target/linux/ath79/generic/base-files/lib/firmware/ath10k/cal-pci-0000:00:00.0.bin`

Build the ath79 WASP Image

```
# Configure the firmware image
make menuconfig

# Select following:
# Target System (Atheros ATH79)
# Subtarget (Generic)
# Target Profile (AVM FRITZ!Box 3490/5490/7490 WASP (Wireless Assist))

# Image configuration ---> Use preinit IP configuration as default LAN IP
# Image configuration ---> Preinit configuration options ---> Preinit configuration options
# ---> (192.168.1.2) IP address for preinit network messages
# ---> (255.255.255.0) Netmask for preinit network messages
# ---> (192.168.1.255) Broadcast address for preinit network messages
# Make sure to set IP address for preinit network messages to something other than 192.168.1.1. Or lantiq and ath79 will have both the same IP!!!

# Select LuCI --> Collections ---> luci

# Deselect Network ---> odhcpd-ipv6only
# No need for multiple DHCP Servers, the WASP will only bridge WiFi to LAN and the Lantiq will handle DHCP

# Select whatever you need

# Build the firmware image
make -j$(nproc) defconfig download clean world

# Save .config
cp .config .config-wasp
```

Obtain the renesas USB FW and ath_tgt_fw1.fw from the AVM stock firmware file. See: https://github.com/openwrt/openwrt/pull/5075#issuecomment-1036819539

Copy xhcifw.mem to the placeholder at `target/linux/lantiq/xrx200/base-files/lib/firmware/renesas_usb_fw.mem`

Copy ath_tgt_fw1.fw to the placeholder at `target/linux/lantiq/xrx200/base-files/lib/firmware/netboot.fw`

Copy `bin/targets/ath79/generic/openwrt-ath79-generic-avm_fritzx490-wasp-initramfs-kernel.bin` to the placeholder at `target/linux/lantiq/xrx200/base-files/lib/firmware/wasp-image.bin`

Rebuild the Lantiq image

```
# Copy saved .config
cp .config-lantiq .config

# Build the firmware image
make -j$(nproc) defconfig download clean world
```
Reflash the Lantiq image

Configure your Local Startup on the Lantiq to boot the WASP image and configure the WiFi. The WASP image is only running in RAM, because the SoC has no flash.
Change, add and remove the uci commands inside the ssh block to your liking.
The avm_wasp driver usually gets loaded before the lan-wasp link is up, so the driver has to be restarted
    
```
ok=0
while true; do
  if ip link show lan-wasp | grep -q "state UP"; then
    ok=$(($ok+1))
  else
    ok=0
  fi

  if [ $ok -gt 10 ]; then
    break
  fi

  sleep 1
done

if lsmod | grep -q avm_wasp ; then
  rmmod avm_wasp
fi
sleep 1
modprobe avm_wasp

while true; do
  if ping -c1 -W1 192.168.1.2 2>&1 >/dev/null; then
    ok=$(($ok+1))
    sleep 1
  else
    ok=0
  fi

  if [ $ok -ge 10 ]; then
    break
  fi
done

ssh -o StrictHostKeyChecking=no root@192.168.1.2 "
  uci set system.cfg01e48a.hostname='AVM-FRITZ-Box-x490-WASP'
  uci set system.cfg01e48a.description='AVM-FRITZ-Box-x490-WASP'
  uci set uhttpd.main.redirect_https='1'
  uci del dhcp.lan.ra
  uci del dhcp.lan.ra_slaac
  uci del dhcp.lan.ra_flags
  uci del dhcp.lan.dhcpv6
  uci del dhcp.lan.domain
  uci set dhcp.lan.ignore='1'
  uci set wireless.radio1.htmode='HT40'
  uci set wireless.radio1.channel='auto'
  uci set wireless.radio1.country='DE'
  uci set wireless.radio1.cell_density='0'
  uci set wireless.default_radio1.encryption='sae-mixed'
  uci set wireless.default_radio1.key='Supersecurepassword'
  uci set wireless.default_radio1.ocv='0'
  uci del wireless.radio1.disabled
  uci set wireless.radio0.channel='auto'
  uci set wireless.radio0.country='DE'
  uci set wireless.radio0.cell_density='0'
  uci set wireless.default_radio0.encryption='sae-mixed'
  uci set wireless.default_radio0.key='Supersecurepassword'
  uci set wireless.default_radio0.ocv='0'
  uci del wireless.radio0.disabled
  uci commit
  reload_config
"
```

All changes in the LuCI of the WASP Image (192.168.1.2) are not saved to flash. So you have to change the Local Startup on the Lantiq to make these changes survive a reboot


Thanks to [kestrel1974](https://github.com/kestrel1974), [jschwartzenberg](https://github.com/jschwartzenberg) and [timocapa](https://github.com/timocapa) for their work

![OpenWrt logo](include/logo.png)

OpenWrt Project is a Linux operating system targeting embedded devices. Instead
of trying to create a single, static firmware, OpenWrt provides a fully
writable filesystem with package management. This frees you from the
application selection and configuration provided by the vendor and allows you
to customize the device through the use of packages to suit any application.
For developers, OpenWrt is the framework to build an application without having
to build a complete firmware around it; for users this means the ability for
full customization, to use the device in ways never envisioned.

Sunshine!

## Download

Built firmware images are available for many architectures and come with a
package selection to be used as WiFi home router. To quickly find a factory
image usable to migrate from a vendor stock firmware to OpenWrt, try the
*Firmware Selector*.

* [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)

If your device is supported, please follow the **Info** link to see install
instructions or consult the support resources listed below.

## 

An advanced user may require additional or specific package. (Toolchain, SDK, ...) For everything else than simple firmware download, try the wiki download page:

* [OpenWrt Wiki Download](https://openwrt.org/downloads)

## Development

To build your own firmware you need a GNU/Linux, BSD or macOS system (case
sensitive filesystem required). Cygwin is unsupported because of the lack of a
case sensitive file system.

### Requirements

You need the following tools to compile OpenWrt, the package names vary between
distributions. A complete list with distribution specific packages is found in
the [Build System Setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)
documentation.

```
binutils bzip2 diff find flex gawk gcc-6+ getopt grep install libc-dev libz-dev
make4.1+ perl python3.7+ rsync subversion unzip which
```

### Quickstart

1. Run `./scripts/feeds update -a` to obtain all the latest package definitions
   defined in feeds.conf / feeds.conf.default

2. Run `./scripts/feeds install -a` to install symlinks for all obtained
   packages into package/feeds/

3. Run `make menuconfig` to select your preferred configuration for the
   toolchain, target system & firmware packages.

4. Run `make` to build your firmware. This will download all sources, build the
   cross-compile toolchain and then cross-compile the GNU/Linux kernel & all chosen
   applications for your target system.

### Related Repositories

The main repository uses multiple sub-repositories to manage packages of
different categories. All packages are installed via the OpenWrt package
manager called `opkg`. If you're looking to develop the web interface or port
packages to OpenWrt, please find the fitting repository below.

* [LuCI Web Interface](https://github.com/openwrt/luci): Modern and modular
  interface to control the device via a web browser.

* [OpenWrt Packages](https://github.com/openwrt/packages): Community repository
  of ported packages.

* [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically
  focused on (mesh) routing.

* [OpenWrt Video](https://github.com/openwrt/video): Packages specifically
  focused on display servers and clients (Xorg and Wayland).

## Support Information

For a list of supported devices see the [OpenWrt Hardware Database](https://openwrt.org/supported_devices)

### Documentation

* [Quick Start Guide](https://openwrt.org/docs/guide-quick-start/start)
* [User Guide](https://openwrt.org/docs/guide-user/start)
* [Developer Documentation](https://openwrt.org/docs/guide-developer/start)
* [Technical Reference](https://openwrt.org/docs/techref/start)

### Support Community

* [Forum](https://forum.openwrt.org): For usage, projects, discussions and hardware advise.
* [Support Chat](https://webchat.oftc.net/#openwrt): Channel `#openwrt` on **oftc.net**.

### Developer Community

* [Bug Reports](https://bugs.openwrt.org): Report bugs in OpenWrt
* [Dev Mailing List](https://lists.openwrt.org/mailman/listinfo/openwrt-devel): Send patches
* [Dev Chat](https://webchat.oftc.net/#openwrt-devel): Channel `#openwrt-devel` on **oftc.net**.

## License

OpenWrt is licensed under GPL-2.0
