# Libin Smart Router Firmware: TL-WR845N v4

Custom Padavan firmware for the **TP-Link TL-WR845N v4** (MT7628NN, 4 MB flash, 32 MB RAM), by **Libin**.

- **Smart WAN Watchdog**: the router fixes its own internet drops.
- **Speed Control**: chosen devices get full speed, everyone else gets the limit you set. Set it from its own web page.
- **Dark theme**: the whole web UI, including the icons, in dark colours. No extra size.
- **Live status bar**: WAN IP, connected time, internet state and live download/upload speed on every page.
- **Libin branding**: logo, page title, footer, SSH banner, default Wi-Fi name.

Based on [lucktu/padavan_TW641](https://github.com/lucktu/padavan_TW641) (padavan-ng, Linux 3.4), pinned to commit `90f694d`. GPL v2.

## Version 2 changes

| | v1 | v2 |
|---|---|---|
| Smart WAN Watchdog | ✅ | ✅ |
| Hardware CPU watchdog | ✅ | ✅ |
| Speed Control (VIP / per-device / shared pool) | ❌ | ✅ web page + commands |
| Dark theme (pages, buttons, tables, icons, scrollbars) | ❌ | ✅ |
| Live WAN speed, WAN IP and connected time in the status bar | ❌ | ✅ |
| VIP / LIMIT tags on the Network Map client list | ❌ | ✅ |
| VPN Server / VPN Client menu entries | shown | hidden (not used on this router) |
| Traffic control (`tc`, HTB) | ❌ | ✅ |
| IPSet | ✅ | ❌ removed to make room |
| Branding | Padavan | Libin |
| Default Wi-Fi name | `TL_WR845N-V4_%s` (bug) | `Libin_WiFi` |

Upgrading from v1 keeps your settings (Wi-Fi, PPPoE, password). Take a **Settings Backup** first anyway.

## Smart WAN Watchdog

| Step | When | Action |
|---|---|---|
| 1 | Internet down < 3 min | Nothing. Short blips are ignored. |
| 2 | Down 3 min | Restart the WAN connection, up to 2 tries 90 s apart. |
| 3 | Still down | Reboot the router. |
| 4 | Still down after a reboot | Each next reboot waits longer (+5, +10, +20, max +30 min). |
| — | Stable 10 min | Backoff resets. |

Settings are nvram keys, e.g. `nvram set wd_fail_time=300 && nvram commit`:
`wd_enable`, `wd_fail_time`, `wd_wan_retries`, `wd_wan_wait`, `wd_backoff_base`, `wd_backoff_max`, `wd_stable_time`, `wd_boot_grace`.

## Speed Control

Three kinds of device:

- **VIP**: full speed, and **priority**. When a VIP device needs bandwidth it gets up to 90 % of your line first.
- **Custom**: a device with its own download/upload limit.
- **Others**: everyone else. Two modes:
  - `each`: every device gets its own limit (e.g. 5 Mbps each, however many connect).
  - `pool`: all of them **share** one limit (e.g. 10 Mbps together).

Devices are matched by **MAC address**, so rules keep working when IPs change. New devices are picked up within about 15 s.

### Web page (easiest)

**Advanced Settings → Customization → Speed Control**

1. **Enable Speed Control?** → Yes
2. **Your internet plan**: your real ISP speed, download / upload in Mbps.
3. **All other devices**: choose *Each device gets its own limit* or *All devices share one limit*, and the speed.
4. **Devices**: every connected device is listed with name, IP and MAC. Set its **Rule** to *Others*, *VIP (full speed)* or *Custom limit* (then type its download/upload). Add a device that is not connected yet with **Add device by MAC**.
5. Press **Apply**. Rules take effect within about 15 seconds; no reboot needed.

The device list refreshes every 15 s; press **Refresh** to reload it.

### Commands

The same settings can be changed in **Administration → Console** (or SSH). Speeds are in **Mbps** (`5`, `1.5`) or kbps with `k` (`512k`). `0` = unlimited.

```sh
speedctl list                          # connected devices, MAC, name, current rule
speedctl line 30 30                    # your real ISP plan: down / up  (do this first)
speedctl others each 5 1               # every other device: 5 Mbps down / 1 Mbps up each
speedctl others pool 10 2              # ...or all others share 10 down / 2 up
speedctl vip add aa:bb:cc:dd:ee:ff     # full speed + priority
speedctl vip del aa:bb:cc:dd:ee:ff
speedctl limit aa:bb:cc:dd:ee:ff 2 0.5 # custom: 2 Mbps down / 500 kbps up
speedctl unlimit aa:bb:cc:dd:ee:ff
speedctl on                            # enable
speedctl off                           # disable (everything back to normal)
speedctl status                        # show settings
```

Rules are saved and survive reboots. Changes apply immediately.

### First-time setup example

```sh
speedctl list                              # find your laptop/phone MAC
speedctl line 30 30                        # plan speed
speedctl vip add 11:22:33:44:55:66         # your device
speedctl others each 5 1                   # everyone else
speedctl on
```

### Notes

- Set `speedctl line` to your **real** plan speed. Too high and VIP priority gets weaker; too low and everyone is capped below what you pay for.
- When Speed Control is on, it automatically switches off Software Offload (SFE), which would otherwise skip the limits. Peak routing speed is lower while limits are on; leave Speed Control off if you only want maximum speed.
- Traffic between devices on your own network (e.g. file copy laptop → PC) is never limited, only internet traffic.
- Activity is logged under **Administration → System Log** with the tag `speedctl`.

## Dark theme

The web UI uses a dark theme everywhere: panels, tables, tabs, menus, buttons, inputs, pop-ups and alert boxes, plus the icons.

- **CSS:** only existing colour values in `main.css` and `bootstrap.min.css` were changed. No rules were added, so the files stay the same size (`bootstrap.min.css` +60 bytes, `main.css` −6 bytes).
- **Icons:** the Network Map icon sheet `big-icons.png` has a dark palette and is the same size as before (9181 bytes). Active icons are bright, inactive ones are dim. The small menu and button icons use the white icon sheet.
- **Light bars:** many pages set a light grey background directly on section headers. A few lines in `state.js` darken any such light background automatically, so light text stays readable on every page.
- **Blue stays blue:** title bars and primary buttons keep their blue colour and white text.
- **Browser cache:** the browser keeps CSS and images for about 30 days. After upgrading, press **Ctrl+F5** to see the new theme.

## Status bar

The box at the top of every page has two new rows, refreshed every 3 seconds (paused while the tab is hidden):

- **WAN speed**: current download ↓ and upload ↑ on the internet connection.
- **Internet**: a dot (green = internet OK, amber = limited, red = down), the WAN IP and how long the connection has been up.

The speed comes from the WAN interface counters, so it shows the router's whole internet traffic, not one device. If it stays at 0 during a download, tell me which build you used.

## Network Map

When Speed Control is on, the client list on the Network Map shows a green **VIP** tag or a yellow **LIMIT** tag next to each device that has a Speed Control rule.

## Menu

The **VPN Server** and **VPN Client** menu entries are hidden because this build doesn't include OpenVPN, WireGuard or IPsec. This only hides the menu items; it does not change the firmware size.

## Mobile

The web UI adapts to phones and tablets: the menu stacks above the page, settings rows stack, and wide tables scroll sideways. Desktop layout is unchanged.

The web UI is **English only**; extra language packs (Russian, Chinese and others) are left out to save flash space.

## Build (GitHub Actions)

1. Put this folder in a GitHub repository, keeping `.github/workflows` and `patches`.
2. **Actions → Build Libin Smart Firmware → Run workflow**.
3. After 30–60 min download the **Libin-WR845Nv4-v2** artifact: `Libin_WR845Nv4_v2.bin` + `SHA256SUMS.txt`.

The build fails on purpose if the image is larger than the 3.84 MB firmware partition.

The patch contains binary changes (logo and icon sheet), so apply it with `git apply`, not `patch -p1`.

## Flash

**From Padavan (v1 → v2):** **Administration → Firmware Upgrade**, choose the `.bin`, wait for the reboot. Use a cable, don't power off.

**From TP-Link stock:** System Tools → Firmware Upgrade on `192.168.0.1`. If the stock firmware rejects it, use a CH341A programmer. Always keep a full 4 MB flash backup.

## Defaults (fresh install or after reset)

- Web UI: `http://192.168.0.1`, user `admin`, password `admin`
- Wi-Fi: `Libin_WiFi`, password `1234567890`

**Change both immediately.**

## Disclaimer

Use at your own risk. No warranty. Third-party firmware voids the TP-Link warranty and a failed flash can brick the router. For **TL-WR845N v4 only**; do not flash other models or versions.

The upstream footer notice ("Firmware distribution is prohibited / Non-Commercial Use Only") from the Padavan fork authors is kept unchanged in the web UI. Respect it when sharing builds.

## Files

```
.github/workflows/build.yml                   Build pipeline
patches/0001-libin-smart-firmware.patch All changes to the Padavan source
```
