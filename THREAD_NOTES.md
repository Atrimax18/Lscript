# Lscript Thread Notes

## Goal

Create a Windows-driven automation flow to install Linux on a DUT and configure the system using:

- `COM20` for the NXP console
- `COM21` for the Marvell switch console
- Linux PC as image/file server at `10.10.10.1`

## Current Lab Assumptions

- Windows PC is the automation controller.
- Linux PC is reachable at `10.10.10.1`.
- DUT is not assumed reachable at `10.10.10.2` during pre-check.
- Linux PC is connected to DUT over Ethernet.
- Serial baud rate is `115200` for both main ports.

## Agreed Automation Flow

1. Pre-check Linux PC connectivity and serial access.
2. Open and log `COM20` and `COM21`.
3. Stop at U-Boot on the NXP side and configure runtime MAC addresses there.
4. Wait for `Telesat>` on the switch side and configure the switch MAC address.
5. Use the Linux PC as the image source for DUT installation.
6. Install Linux image on the DUT.
7. Boot Linux on the DUT and perform full system configuration.
8. Validate the final system state.

## Configuration Decisions

- Configuration has been migrated from INI to YAML.
- YAML file: `script_setup.yaml`
- MAC addresses should not be stored in YAML.
- MAC addresses will be provided as runtime parameters in the script.

## Next Step

User will provide the full command list for:

- U-Boot commands
- Switch commands at `Telesat>`
- Linux image install steps
- Post-install configuration steps
