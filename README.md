# supermicro-x11sdv-tp8f-fw-flake

A Nix flake that provides the toolbox for flashing the BIOS and BMC/IPMI firmware of the [Supermicro X11SDV-4C-TP8F](https://www.supermicro.com/en/products/motherboard/X11SDV-4C-TP8F) board (the SYS-5019D-4C-FN8TP system) in-band from Linux, built on [Supermicro Update Manager (SUM)](https://www.supermicro.com/en/solutions/management-software/supermicro-update-manager), [ipmitool](https://github.com/ipmitool/ipmitool) and [zsrv/supermicro-product-key](https://github.com/zsrv/supermicro-product-key).

## Outputs

- **`devShells.<system>.default`** — the flashing shell: `sum`, `smc-load-ipmi`, the three `smc-license-*` helpers, `ipmitool`, `pciutils`, `dmidecode`. Its hook exports `SMC_FW`, `SMC_BIOS` and `SMC_BMC` and prints the working order.
- **`packages.<system>.sum`** — Supermicro Update Manager, the in-band flasher for **both** BIOS (`UpdateBios`) and BMC (`UpdateBmc`). `packages.<system>.default` is the same package.
- **`packages.<system>.firmware`** — the X11SDV-TP8F software package with its nested archives unpacked, so the flashable images sit at `$out/bios/BIOS_X11SDV-0986_20240903_2.2_STDsp.bin` and `$out/bmc/BMC_X11AST2500-4101MS_20230802_01.74.13_STDsp.bin`, with the vendor readme at `$out/`.
- **`packages.<system>.product-key-tool`** — the standalone `supermicro-product-key` encoder/decoder, usable on its own.

`formatter` and `checks.formatting` are maintenance outputs.

## Consuming it

Open the shell on the machine you are about to flash:

```bash
nix develop github:jgus-org/supermicro-x11sdv-tp8f-fw-flake
```

Or take an output from a flake input — `inputs.<name>.packages.${system}.firmware`, whatever you name the input.

The SUM and firmware sources are fetched by their pinned versioned URLs, so following this flake's `nixpkgs` into your own changes the host tools you get (`ipmitool`, `pciutils`, `dmidecode`) but not the versions of those sources. The flake sets `config.allowUnfree` in its own package set, so consuming its outputs needs no unfree acceptance of your own.

## Supported hosts

`<system>` covers `x86_64-linux` and `aarch64-linux`. `sum` and `firmware` are **x86_64-linux only**: SUM is a prebuilt x86_64 Linux binary from the vendor and the images are for one x86_64 board. There is nothing here to build for macOS. `product-key-tool` is a portable Go binary and runs anywhere nixpkgs builds Go packages.

## Firmware versions

| Component | Version               | Image                                               |
| --------- | --------------------- | --------------------------------------------------- |
| BIOS      | 2.2 (2024-09-03)      | `BIOS_X11SDV-0986_20240903_2.2_STDsp.bin`           |
| BMC/IPMI  | 01.74.13 (2023-08-02) | `BMC_X11AST2500-4101MS_20230802_01.74.13_STDsp.bin` |
| SUM       | 2.15.0 (2025-11-04)   | the vendor's Linux x86_64 tarball                   |

TP8F firmware is shared across the 4C/8C/12C/16C variants of the board. The NICs (X552 10G, i350 1GbE, X557 PHY) carry their firmware inside the BIOS update — there is no separate NIC flash step. Newer releases appear on the board's [download-center page](https://www.supermicro.com/en/support/resources/downloadcenter/firmware/MBD-X11SDV-4C-TP8F/BIOS).

The exported image paths are named for the pinned bundle, so they change whenever the pinned versions do; resolve them through `$SMC_BIOS` and `$SMC_BMC` rather than hard-coding the file names.

## Runtime requirements

Run it on the target board itself, as root, from a booted Linux — bare metal or a NixOS live installer. It makes no other assumption about the host.

In-band SUM talks to the BMC over `/dev/ipmi0`, the IPMI KCS interface. On a freshly-booted machine — including the NixOS installer — the kernel modules are not loaded yet, and they cannot come from this flake: kernel modules belong to the booted kernel, not to a flake's package set. `smc-load-ipmi` loads them from the running kernel (`ipmi_si`, `ipmi_devintf`) and checks for `/dev/ipmi0`; the standard NixOS installer kernel carries both.

```bash
nix develop github:jgus-org/supermicro-x11sdv-tp8f-fw-flake   # enable flakes on the installer if needed
smc-load-ipmi                                  # loads the modules, then reports on /dev/ipmi0
sudo dmidecode -t baseboard                    # confirm you are on an X11SDV-4C-TP8F
```

Where `sudo` reports that it cannot find `sum`, it is because the tool lives in the Nix store and `sudo`'s path does not carry it — invoke it by resolved path (`sudo $(command -v sum) …`). Do not copy `sum` elsewhere and run that: it resolves its `ExternalData/` payload and BIOS driver relative to its own location.

## Flashing order

Use `tmux`/`screen` and ensure stable power, and update **BMC first, then BIOS**:

```bash
sudo sum -c UpdateBmc  --file "$SMC_BMC"
sudo sum -c UpdateBios --file "$SMC_BIOS" --preserve_setting --reboot
```

Both flashes go over `/dev/ipmi0`; no other service on the host needs to be running. `UpdateBios` takes `--reboot`, so it reboots the board when it finishes.

Without any Linux tooling:

- **BMC** — the BMC web UI's **Maintenance → Firmware Update** (needs network to the BMC).
- **BIOS** — boot the **UEFI shell** from USB and run `flash.nsh` (`afuefi.smc`) from `$SMC_FW/bios`; `$SMC_FW/bmc/2.09/AuUpdate.efi` updates the BMC the same way.

## Supermicro licence keys

In-band SUM (`UpdateBios`, `UpdateBmc`) and the web-UI BMC update need no key. A per-node **SFT-DCMS-SINGLE** or **SFT-OOB-LIC** licence is needed only for OOB SUM over the BMC network, BIOS updates through the web UI, and HTML5 iKVM virtual media.

Three helpers, all in-band over `/dev/ipmi0` (pass `-i <bmc> -u <user> -p <pass>` to `sum` to target a BMC over the network instead):

```bash
smc-license-status                               # QueryProductKey — is a node key already active?
smc-license-activate                             # derive + activate BOTH the OOB and DCMS keys (full unlock)
smc-license-keygen [--sku SFT-DCMS-SINGLE] [MAC] # just print a key (OOB by default), e.g. to apply elsewhere
```

With no arguments `smc-license-activate` reads the BMC's own MAC and activates each key, tolerating ones that are already active. Pass a key to activate only that one.

**There are two different keys and they unlock different things:**

- **OOB key** — `HMAC-SHA1-96(secret, BMC-MAC)`, the classic 24-hex `XXXX-…` string ([reverse-engineered by Peter Kleissner](https://peterkleissner.com/2018-05-27/reverse-engineering-supermicro-ipmi/); the secret is constant across the X9–X11 generation). It is a pure function of the MAC and carries no SKU. It unlocks the SFT-OOB-LIC feature set — OOB BIOS flashing — and **not** HTML5 iKVM virtual media. This is what `smc-license-keygen` prints by default.
- **DCMS key** — the longer 344-character base64 "non-JSON" key. The BMC recomputes its validator from the key's own fields, so on gen 10/11 boards (this X11) it can be minted offline without a Supermicro signing key, via [`zsrv/supermicro-product-key`](https://github.com/zsrv/supermicro-product-key). A `SFT-DCMS-SINGLE` key adds what the OOB key lacks: HTML5 iKVM / virtual media, system lockdown.

So enabling HTML5 virtual media takes the DCMS key, not the OOB key; the no-argument `smc-license-activate` already applies both.

> **Generation matters.** Gen 9 has only the OOB key (no non-JSON key). Gen 10/11 — this board — support the offline-mintable non-JSON DCMS key. **X12+** uses RSA-signed "JSON" keys that cannot be minted this way; there you need a real licence from Supermicro. The DCMS _suite software_ (SSM/SPM central management) is a separate entitlement in every generation.

**Fallback for virtual media without a key:** the Java-based iKVM console attaches an ISO with no DCMS licence (only the HTML5 console enforces it), and an ISO on a CIFS/Samba share mounts without one. Useful if a self-minted key is rejected by your firmware.

See [LICENSE.md](LICENSE.md) for licensing and copyright, and [NOTICE.md](NOTICE.md) for the terms of upstream and distributed artifacts.
