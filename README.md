# Libin Smart Router Firmware: TL-WR845N v4

Custom Padavan firmware for the **TP-Link TL-WR845N v4** (MT7628NN, 4 MB flash, 32 MB RAM), by **Libin**.

- **Smart WAN Watchdog**: the router fixes its own internet drops.
- **Device Usage**: live download and upload speed and totals for every device, on its own web page.
- **Dark purple theme**: the whole web UI, including the icons, in dark colours with a purple accent. No extra size.
- **Quick find**: press **Ctrl+K** (or `/`) on any page, or use the "Find a setting" link in the footer, and type a name like `mac` or `wan` to jump to that settings page.
- **Device Usage tools**: click a column header to sort, type in the filter box, click an IP or MAC to copy it.
- **Purple favicon**: a purple Wi-Fi icon in the browser tab.
- **Notices**: a banner on every page when the admin or Wi-Fi password is still the default, or the router clock is wrong.
- **Live extras**: the browser tab title shows the live speed, the status bar has a 2-minute speed graph, and Device Usage can export a CSV file.
- **Wi-Fi QR code**: scan it from the Network Map to join the Wi-Fi.
- **Login page**: a styled login page instead of the browser pop-up, with the old pop-up kept as a fallback.
- **Live status bar**: WAN IP, connected time, internet state and live download/upload speed on every page.
- **Libin branding**: logo, page title, footer, SSH banner, default Wi-Fi name.

Based on [lucktu/padavan_TW641](https://github.com/lucktu/padavan_TW641) (padavan-ng, Linux 3.4), pinned to commit `90f694d`. GPL v2.

## Version 2 changes

| | v1 | v2 |
|---|---|---|
| Smart WAN Watchdog | ✅ | ✅ |
| Hardware CPU watchdog | ✅ | ✅ |
| Device Usage (live per-device download / upload) | ❌ | ✅ web page |
| Dark purple theme (pages, buttons, tables, icons, scrollbars) | ❌ | ✅ |
| Live WAN speed, WAN IP and connected time in the status bar | ❌ | ✅ |
| VPN Server / VPN Client menu entries | shown | hidden (not used on this router) |
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

## Device Usage

**Advanced Settings → Customization → Device Usage** shows the internet traffic of every device on your network, live:

- **Download ↓ and Upload ↑** for each device, updated every 5 seconds, with small bars to compare devices.
- **Total down and total up** since the router started.
- Devices are listed with name, IP and MAC. The busiest device is on top.

How it works: a small background program (`devusage`) reads the router's connection table every few seconds and adds up the bytes each device sent and received. Software Offload (SFE) stays **on**, so routing speed is not reduced. Traffic between your own devices (for example a file copy from laptop to PC) is not counted, only internet traffic.

Notes:

- It starts about 40 seconds after the router boots, and the numbers begin after the second reading.
- Connections that were already open when it started are not counted, so totals start from zero.
- Totals are kept in memory and reset on reboot.
- To turn it off, run `nvram set du_enable=0 && nvram commit` in **Administration → Console**. Use `du_enable=1` to turn it back on.
- **Export CSV** saves the table (name, IP, MAC, speeds, totals) as `device-usage.csv`.
- Cost: it runs every 5 seconds at low priority. If counting takes long (many connections), it automatically waits longer so it uses under about 5 % of the CPU, up to 30 seconds between readings.

## Dark purple theme

The web UI uses a dark theme everywhere: panels, tables, tabs, menus, buttons, inputs, pop-ups and alert boxes, plus the icons.

- **CSS:** only existing colour values in `main.css` and `bootstrap.min.css` were changed. No rules were added, so the files stay the same size (`bootstrap.min.css` +60 bytes, `main.css` −6 bytes).
- **Icons:** the Network Map icon sheet `big-icons.png` has a dark palette and is the same size as before (9181 bytes). Active icons are bright, inactive ones are dim. The small menu and button icons use the white icon sheet.
- **Light bars:** many pages set a light grey background directly on section headers. A few lines in `state.js` darken any such light background automatically, so light text stays readable on every page.
- **Traffic charts:** the Network Traffic charts (Real Time and Last 24 Hours) use dark panels, dark range buttons and scrollbar, and a purple uplink line (downlink stays orange).
- **Modern layout and style:** rounded panels with soft shadows, a purple gradient title bar, a sidebar with a highlighted current page and hover effect, lighter and easier-to-read links, uppercase table headers, row hover, purple focus glow on inputs and buttons, dark scrollbars. The page is centred and up to 1240 px wide on large screens, and stacks as cards on phones. The style is added by `state.js` when the page loads, so the CSS files do not grow.
- **Dark fixes:** ticked rows in the MAC filter, DHCP, firewall, port-forward and similar lists no longer turn light blue, the current menu item is no longer dark teal, and the "someone else is logged in" page is dark.
- **Background:** a faint purple network-node pattern with a soft glow. It is drawn by CSS inside `state.js` (about 1 KB), so there is no image file.
- **Purple accent:** every blue became purple: title bars, buttons, links, tabs, focus glow, alert boxes, the ON toggle switch, the loading spinner, the logo and the blue Network Map icons. White text on them is unchanged. Only the colours changed, so file sizes are the same.
- **Browser cache:** the browser keeps CSS and images for about 30 days. After upgrading, press **Ctrl+F5** to see the new theme.

## Status bar

The box at the top of every page has two new rows, refreshed every 5 seconds (paused while the tab is hidden):

- **WAN speed**: current download ↓ and upload ↑ on the internet connection.
- **Internet**: a dot (green = internet OK, amber = limited, red = down), the WAN IP and how long the connection has been up.

The speed comes from the WAN interface counters, so it shows the router's whole internet traffic, not one device. While a browser tab is in the background, the page stops polling the router, including the stock status refresh. The status bar and the Device Usage page share one small data file, so the router answers only one extra request every 5 seconds. The router's own status refresh was also slowed from 2 to 3 seconds to lower CPU use.

## Wi-Fi QR code

The Network Map shows a small **QR code** under the device icons. Scan it with a phone camera to join your 2.4GHz Wi-Fi without typing the password. Click the QR code to enlarge it. It works for WPA/WPA2 and open networks and includes the hidden-network flag. For other security types (WEP, enterprise) no QR code is shown. The QR code contains the Wi-Fi password, so it is only visible to someone logged in to the router.

## Router name

The router answers to **http://libin.lan** as well as its IP address (host name `libin`, domain `lan`).

## Login page

Opening the router in a browser shows a **login page** (dark purple, with the logo) instead of the browser's pop-up. Type your username and password, and the browser remembers the login for the session, like before.

- **Wrong password:** the page says so and stays open.
- **Fallback:** the link "Use browser login" on the page opens the old pop-up.
- **Not a new password system:** the same admin username and password are checked in the same way. Change the default `admin` password.
- **Logout** works as before and brings you back to the login page.

## Notices

A yellow banner appears at the top of every page when:

- the **admin password** is still `admin`,
- the **Wi-Fi password** is still `1234567890`, or
- the **router clock** differs from your device's clock by more than 10 minutes (logs and time schedules would be wrong).

Click ✕ to hide it for the current browser session. The password check only asks the router "is it the default?" and never shows the password.

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
