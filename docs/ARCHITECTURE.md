# Deployed Architecture

## Network design

The appliance exposes a USB Ethernet interface to one attached client and uses Wi-Fi only as upstream transport. NetBird provides the encrypted path to `pi3-jumper`, which owns home-network routing and home Internet egress.

```text
client
  -> USB Ethernet / DHCP
  -> travel-router usb0
  -> travel-router wlan0
  -> NetBird overlay
  -> pi3-jumper
      -> home network
      -> home Internet egress
```

The deployed client LAN is a dedicated private subnet. Exact instance addresses, overlay identities, and policy group names are recorded in the private `infra-configs` instance file.

An upstream Wi-Fi network can use the same gateway address as the home gateway. In that case, preserve the upstream gateway route on `wlan0`; use the unique NetBird identity for direct communication with `pi3-jumper`. Do not delete a valid WAN route to force a colliding home address through the overlay.

## Display and input

The physical UI uses the Waveshare 2.7-inch V1 e-Paper backend and a systemd-managed Python display process. A separate status service performs slow diagnostics and atomically writes a non-secret cache under `/run`; the display reads that cache rather than blocking GPIO handling on network probes.

```text
travel-router-status.service
  -> /run/travel-router/status.json
  -> travel-router-display.service
  -> /usr/bin/python3 -m travel_router.display_app
  -> /opt/travel-router/travel_router/display_app.py
  -> Waveshare e-Paper HAT
```

The deployed backend is `waveshare_epd.epd2in7`. Its vendor library is loaded
from `/opt/travel-router/vendor/e-Paper/RaspberryPi_JetsonNano/python/lib`.
Do not import the hardware module from a second process while the display
service owns SPI/GPIO resources.

| Key | BCM GPIO | Deployed action |
| --- | ---: | --- |
| K1 | 5 | wake Jellyfin |
| K2 | 6 | hold to shut down Jellyfin |
| K3 | 13 | restart local NetBird |
| K4 | 19 | hold to power off the appliance |

The deployed display behavior is:

1. accept one button action while idle;
2. update logical state;
3. run explicit slow actions through the action worker;
4. perform one full e-Paper refresh;
5. discard any button presses received while busy;
6. arm input again only after refresh completes.

K2 and K4 require holds to reduce accidental shutdowns. Do not add an input queue or buffered replay.

## Control relay

Wake and shutdown actions are relayed through `pi3-jumper`, because that host is on the Jellyfin LAN.

```text
Travel Router
  -> OpenSSH key authentication over NetBird, TCP/2222
  -> pi3-jumper allowlisted helper
      -> wake-on-LAN or bounded Jellyfin shutdown
```

NetBird's built-in SSH service intercepts peer-address TCP/22 and requires interactive identity authorization. Hardware buttons therefore use the real OpenSSH daemon on TCP/2222 with batch mode and an appliance-local key. The corresponding sshd drop-in is under `config/pi3-jumper/`.

The private key remains only on the appliance. Policy must allow only the required Travel Router identity/group to reach `pi3-jumper` on TCP/2222. The drop-in opens the additional listener but does not replace host firewall, key-only authentication, or NetBird policy. Keep those controls explicit.

## Ownership boundaries

This repository owns reusable appliance behavior, architecture, recovery procedure, and safe configuration artifacts. The private `infra-configs` repository owns the deployed instance name, addresses, dependencies, and canonical repository pointer. NetBird management-plane state and secrets remain outside Git.

The deployed Python application and complete bare-metal bootstrap sources were not available during repository extraction. Their documented service and path contracts are retained, but missing source is not replaced with speculative automation.
