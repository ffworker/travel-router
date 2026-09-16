# Travel Router Recovery

Use this procedure when the appliance is partially working. Restore the last-known-good behavior without changing unrelated NetBird or home-network configuration.

Exact addresses and identity values are kept in the private `ffworker/infra-configs` instance record. Substitute those recorded values where this runbook uses `<...>`.

## Optional private configuration backup

This public repository contains the reusable recovery procedure, not every device-specific configuration file. The commands below are helper commands for inspection and recovery; they are not a requirement to use this project and they are not a complete backup by themselves.

For a more recoverable device, back up the important non-secret scripts, service definitions, USB/network configuration, and other files needed to rebuild your own appliance. A private Git repository such as `ffworker/infra-configs` is recommended for that device-specific record, but any secure, backed-up location that fits your setup is fine. This backup is optional and is only intended to help if the device fails and its configuration needs to be rewritten.

Do not commit Wi-Fi credentials, NetBird setup keys, peer databases, tokens, private SSH keys, cookies, or other live secrets. Keep those in the appliance's protected storage or an approved secret store, and back them up separately if needed.

The placeholders and helper commands in this guide should be adapted to your own host, addresses, paths, users, and implementation. Do not copy the private instance values from another deployment.


## 1. Confirm the USB gadget layer

On the appliance:

```bash
ip -br link
```

If `usb0` exists, continue. If it does not, stop treating the fault as a NetworkManager problem: recover the hardware-specific USB gadget boot configuration from a system backup first. That configuration was not available in Git during repository extraction.

## 2. Recover the USB NetworkManager profile

Inspect current profiles before changing them:

```bash
nmcli connection show
```

If the deployed USB profile is missing, recreate it with the interface and private address from the instance record:

```bash
sudo nmcli con add type ethernet \
  ifname usb0 \
  con-name travel-usb \
  ipv4.method manual \
  ipv4.addresses <USB_GATEWAY_CIDR> \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv6.method disabled
sudo nmcli con mod travel-usb connection.autoconnect yes
sudo nmcli con up travel-usb
ip -4 addr show usb0
```

## 3. Check client DHCP

Reconnect the USB client and inspect its address and route:

```bash
ip -br addr
ip route
```

The client must receive an address in the instance's USB subnet and use the Travel Router as its gateway. If it receives no lease, identify the existing DHCP owner before replacing it:

```bash
systemctl is-active dnsmasq 2>/dev/null || true
sudo grep -R "<USB_SUBNET_PREFIX>" /etc/dnsmasq* /etc/NetworkManager 2>/dev/null || true
```

The deployed DHCP configuration was not available in Git during repository extraction. Recover it from a system backup rather than selecting a new implementation during an incident.

## 4. Check NetBird

```bash
systemctl is-active netbird
netbird status
netbird networks list
```

Do not delete peer state or re-enroll the appliance as an early troubleshooting step. If the service is down:

```bash
sudo systemctl restart netbird
sudo netbird up
```

If exit-node routes remain while NetBird is disconnected, inspect routes, rules, and logs before changing persistent configuration:

```bash
ip route
ip rule
sudo journalctl -u netbird -n 50 --no-pager
```

## 5. Check forwarding and firewall state

```bash
sysctl net.ipv4.ip_forward
sudo nft list ruleset
sudo iptables -S 2>/dev/null || true
sudo iptables -t nat -S 2>/dev/null || true
```

IPv4 forwarding must be enabled. Do not add duplicate masquerade or forwarding rules blindly. Recover the known-good persistent rules from a system backup if they are missing.

## 6. Check the home path

From the appliance and then from the USB-only client, probe a non-gateway home address recorded in `infra-configs`.

If ordinary home addresses work but the home gateway address does not, test for an upstream address collision:

```bash
ip route get <HOME_GATEWAY_IPV4>
ip route show dev wlan0
```

Keep a valid upstream Wi-Fi gateway route. Reach `pi3-jumper` through its unique NetBird identity when the LAN address collides.

## 7. Check home egress

Compare the public IPv4 reported from `pi3-jumper`, the Travel Router, and the USB-only client:

```bash
curl -4 https://icanhazip.com
```

They must match when the home exit path is active. If the router uses home egress but the client does not, diagnose forwarding, policy routing, and NAT on the Travel Router before changing the centrally managed exit-node definition.

## 8. Verify physical controls

First validate the control relay without pressing K1 or K2. On `pi3-jumper`:

```bash
sudo sshd -t
sudo ss -lntp | grep ':2222'
```

From the Travel Router, use the port, user, identity path, and peer address in
the private instance record:

```bash
ssh -F /dev/null \
  -p <PI3_OPENSSH_PORT> \
  -i <APPLIANCE_IDENTITY_PATH> \
  -o IdentitiesOnly=yes \
  -o BatchMode=yes \
  -o ConnectTimeout=5 \
  <PI3_USER>@<PI3_NETBIRD_IPV4> true
```

Confirm the host firewall and NetBird policy allow TCP/2222 only from the
required appliance identity/group. The sshd drop-in alone does not enforce that
network boundary.

With the display service active, test one action at a time:

- K1 wakes Jellyfin through `pi3-jumper`;
- K2 requires a hold before shutdown;
- K3 restarts local NetBird without replacing configuration;
- K4 requires a hold, renders `SAFE TO UNPLUG`, and powers off.

Allow each e-Paper refresh to finish. Confirm presses made while the panel is busy are discarded rather than replayed.

## 9. Reboot acceptance

After an authorized configuration repair, reboot the appliance and repeat:

```bash
systemctl is-active netbird
systemctl is-active travel-router-display.service
systemctl is-active travel-router-status.service
ip -4 addr show usb0
sysctl net.ipv4.ip_forward
netbird status
netbird networks list
```

From a client using only USB Ethernet, verify:

```bash
ip route
curl -4 https://icanhazip.com
ping -c 3 <KNOWN_HOME_HOST>
```

Recovery passes only when DHCP, the client default route, home-network access, home Internet egress, NetBird, the display, and all four controls survive reboot without manual runtime fixes.

## Bare-metal rebuild boundary

A complete rebuild requires a known-good OS backup containing the deployed Python application, USB gadget boot configuration, DHCP configuration, persistent forwarding/firewall rules, and appliance-local secrets. Those sources were not available in Git during extraction. This repository is sufficient for architecture-guided recovery, but it does not claim a fully automated bare-metal rebuild.
