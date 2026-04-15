# Lscript

Windows-driven serial automation for the LSBB lab flow.

The script now supports three modes:

- `detect`: stop NXP autoboot on `COM20` and confirm the switch `Telesat>>` prompt on `COM21`
- `mac-only`: stop both sides in U-Boot, program NXP and switch MACs, and stop there
- `provision`: run the documented flow from MAC programming through Linux install, switch image install, SONiC setup, and `LSBB_Utils`

Serial logs are saved under `logs/`, and both `COM20` and `COM21` are opened and logged from the start of the run.

## Detect Mode

```powershell
python .\lscript.py --mode detect
```

If the target expects a different autoboot interrupt key:

```powershell
python .\lscript.py --mode detect --boot-stop-key space
python .\lscript.py --mode detect --boot-stop-key ctrl-c
```

To validate only the NXP side:

```powershell
python .\lscript.py --mode detect --skip-switch
```

## Provision Mode

The concrete command sequence was mapped from the manual starting at `BURN MAC ADDRESSES in NXP`.

Required runtime MAC argument:

- `--base-mac`
  Written only to NXP `mac 0`

NXP `mac 1`, `mac 2`, and `mac 3` are always written with these fixed values:

- `mac 1 00:04:9F:08:44:A2`
- `mac 2 00:04:9F:08:44:A3`
- `mac 3 00:04:9F:08:44:A4`

Optional derived MAC arguments:

- `--switch-uboot-mac`
  Default is `base-mac + 1`
- `--switch-onie-mac`
  Default is `base-mac + 1`

Example:

```powershell
python .\lscript.py --mode provision `
  --base-mac 70:B3:D5:97:07:C0
```

Optional image overrides:

```powershell
python .\lscript.py --mode provision `
  --base-mac 70:B3:D5:97:07:C0 `
  --deploy-script deploy-lsbb-1.1.1-20260324.sh `
  --switch-image sonic-marvell-arm64.bin `
  --switch-itb telesat_lsbb-r0.itb
```

If these fields are present in [script_setup.yaml](C:/Users/alexeyt/source/repos/Lscript/script_setup.yaml), the filenames are read from YAML by default:

- `dut.image_file` for the deploy script
- `switch.image_file` for the switch ITB
- `sonic.image_file` for the SONiC image

Timeouts are now separated in YAML:

- `timeouts.uboot_boot_seconds` for the U-Boot capture phase
- `timeouts.emergency_boot_seconds` for the reboot into emergency Linux

`timeouts.first_boot_seconds` is still accepted as a fallback for older configs.

To skip the final `LSBB_Utils` copy and run:

```powershell
python .\lscript.py --mode provision `
  --base-mac 70:B3:D5:97:07:C0 `
  --skip-utils
```

## What Provision Mode Does

- Stops autoboot on the NXP console and enters U-Boot
- Monitors NXP and switch boot in parallel from the start of the run
- Verifies on NXP boot that both CLUs are locked and that `Switch ready` and `FPGA ready` appear before continuing
- Waits for the switch `Telesat>>` U-Boot prompt
- Writes NXP `mac 0` from the runtime argument
- Writes fixed NXP values to `mac 1`, `mac 2`, and `mac 3`
- Burns the switch U-Boot MAC with `setenv ethaddr` using `base-mac + 1`
- Pauses for the physical DUT reset into emergency Linux, waits for the maintenance message, and then waits for the `sh-5.2#` prompt on the NXP terminal
- Configures DUT IP and verifies image-server reachability with one combined emergency command:
  `ifconfig eth0 10.10.10.2 ; ping 10.10.10.1 -c1`
- Continues only after detecting:
  `1 packets transmitted, 1 packets received, 0% packet loss`
- Sends Enter twice after successful emergency ping, then starts SCP
- Pulls the deploy script from the server path built from YAML, for example:
  `scp deploy@10.10.10.1:/home/deploy/images/deploy-lsbb-1.1.1-20260324.sh /tmp`
- Handles SCP first-connect prompts by sending `y` for Dropbear-style host-key confirmation and then sends `server.password` from YAML
- Verifies the deploy script appears in `/tmp`, and only then runs it
- Pauses for the documented reboot steps
- Configures persistent DUT networking with `nmcli`
- Copies switch image files to `/tmp`
- Configures `fm1-mac10` as `192.168.2.1/24`
- After `nmcli con show` on the NXP terminal, switches to COM21 and sends `ping $serverip` from the switch U-Boot prompt
- Starts the local TFTP and HTTP services on the DUT
- Programs the switch install URL and boots the switch image
- After `bootm $onie_loadaddr`, waits on COM21 for `System is ready`
- Sends Enter twice after SONiC is ready, then logs in with `sonic.login` and `sonic.password` from YAML
- Logs into SONiC and runs the documented config commands:
  `sudo sonic-cfggen -w -j /usr/share/sonic/device/arm64-telesat_lsbb-r0/telesat-lsbb/default_config.json`
  `sudo config qos reload`
  `sudo config interface ip add eth0 192.168.2.2/24`
  `sudo config save -y`
- Copies `LSBB_Utils` and runs `run.sh` unless `--skip-utils` is used

## Manual Intervention Points

The procedure still requires an operator for the physical steps from the manual:

- Reset the DUT after MAC programming
- Reboot after the deploy script completes
- Reboot after saving the persistent DUT IP configuration
- Connect the ATE PC Ethernet debug port before the final `LSBB_Utils` run

The script pauses at each of those points and continues when Enter is pressed.

## Known Manual Gaps

The Word manual includes some unclear or placeholder text that was not converted into executable commands:

- `First command`, `Second command`, `Third command`, `Fourth command`
- The garbled lines around the second SCP block
- The upgrade section with `swupdate-client`, which appears to be a separate flow

Those steps will need the exact intended commands before they can be automated safely.

## MAC-Only Mode

To stop both chips in U-Boot, write the NXP MAC set, write the switch `ethaddr`, and stop there:

```powershell
python .\lscript.py --mode mac-only --base-mac 70:B3:D5:97:07:D8
```

This mode does only:

- stop NXP in U-Boot
- stop switch in U-Boot
- write NXP `mac 0` from `--base-mac`
- write fixed NXP values to `mac 1`, `mac 2`, `mac 3`
- write switch `ethaddr` as `base-mac + 1`
- save both sides and exit
