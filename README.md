# What is OPNsense?

OPNsense is an open-source, FreeBSD-based firewall and routing platform that provides features like traffic shaping, VPN support, and intrusion detection/prevention. It is designed for home, small business, and enterprise networks, offering a user-friendly web interface and a wide range of functionalities.

# Pre-Requisites

If you know little of networking, this whole guide may be a bit over your head. However, I'm going to do my best to aid you. I'm only covering IPv4 and not IPv6 as I haven't been forced to study up on configuring IPv6.

* You want to configure this router on an isolated device, as in do not connect to the existing local network. If you do, you'll cause conflicts and break internet connectivity to your whole infrastructure. You can use an ethernet cable from the device you're installing OPNsense on to another device to use the GUI. When it's ready, disconnect the current router and then use the OPNsense router. If you have two Ethernet ports in your device, plug in your secondary device into the second port. OPNsense assumes the numerically lowest port will be WAN.

* If your hardware you will be installing OPNsense on does not allow you to have more than one Ethernet port (lots of Intel NUCs and other mini-PCs), you have to do the "Router-on-a-Stick" method. That means you *need* to use a VLAN-capable switch. USB adapters will not work because they are not reliable. Ask me how I know. Because a single 1 Gbps network card on the firewall must handle both incoming internet traffic and outgoing local traffic simultaneously over the exact same physical wire, the total bandwidth of the port is split down the middle in a worst case scenario. Full-duplex contention will cap your real-world throughput to roughly half of the physical link capacity. Put simply: the single 1 Gbps port causes the bottleneck when sharing simultaneous ingress/egress traffic at 1 Gbps.

# Getting OPNsense

A robust [guide to installing](https://docs.opnsense.org/manual/install.html) is already listed in their documentation.

You can download what you need from [OPNsense's website](https://opnsense.org/download/).

I highly encourage you to [verify the authenticity/integrity](https://docs.opnsense.org/manual/install.html#download-and-verification).

# Installation

When you boot the OPNsense image (DVD, VGA, Serial formats), the firewall will be in the live environment with and without the use of OPNsense Importer. You can interact with the environment via console, GUI (HTTPS), or SSH, but for this guide, I'm just explaining it over console.

* To identify your disk to install on, we need to run a command before we install. The default username is `root` and the default password is `opnsense`. Press `8` for `8) Shell`, then `Enter` and run:

```
geom disk list
```

* This outputs a detailed block for every detected drive. Look for these specific keys:

    * `Geom name`: This matches the identifier you will see in the installer (e.g., `nda0` for NVMe, `ada0` for SATA, `da0` for USB).

    * `Media size`: Shows the capacity (e.g., 250 GB). Match this against the known size of your internal drive.

    * `Descr`: Shows the manufacturer string and model name (e.g., Crucial CT500P3SSD8 or SanDisk Ultra). If you see "Kingston DataTraveler" or "JetFlash," that is most likely your USB installer stick.
   
* Once you identified the correct storage, memorize or write the `geom` identifier.

* To use the installer, either run the command `opnsense-installer` here, or sign in with the `installer` user. To do that, press `Ctrl` + `D` to go back and press `0` to log out. The password is `opnsense`. The installation process involves the following steps:

    1: `Keymap selection:` The default configuration should be fine for pretty much everyone using this guide.
    
    2: `Install (UFS|ZFS):` Choose UFS or ZFS filesystem. I *highly* encourage ZFS. It creates backups with little downsides and is very reliable. It does require at least a few gigabytes of storage, worst case scenario.
    
    3: `Partitioning (ZFS):` Choose a device type. The default option (stripe) is acceptable when using a single disk.
    
    4: `Disk Selection (ZFS):` Select the Storage device. e.g. `da0` or `nvd0`.
    
    5: `Last Chance:` Select Yes to continue with partitioning and to format the disk. However, doing so will destroy the contents of the disk.
    
    6: `Continue with recommended swap (UFS):` Yes is usually fine here unless the install target is very small (< 16GB)
    
    7: `Select Root Password:` Change and confirm the new root password. If you want to use a password manager, wait to do this until you have access to it. The more complex the password, the better.
    
    8: `Select Complete Install:` Exits the installer and reboots the machine. The system is now installed and ready for initial configuration.

* After rebooting into the system (not your USB), the system will prompt you for the interface assignment. If you ignore this, then the default settings will be applied. Installation ends with the login prompt. At this point, you can access the GUI. I recommend you configure things from the GUI from this point unless notated otherwise. The configured IP to access the GUI is `192.168.1.1`. The default username is `root` and the default password is `opnsense`.

If your device has two ethernet ports: Cool, too easy. Interface assignment and configuration should be super easy. If your device only has one: Well this is fun! /s

<details>
<summary>One Ethernet Port</summary>

The GUI is a little wonky. Unless something has changed since version `26.1.5`, do it the way I'm guiding you.

* Use the CLI and sign in as `root`. Use `1) Assign interfaces`. The configuration engine will initialize and scan your hardware.

* The wizard will first list your available physical interface names (on Intel NUCs, this is typically an interface name like em0 or igb0). Write this identifier down. This will be the parent device. The wizard will prompt you with the following sequence:

    1: `Do you want to configure LAGGs now?` Type `n` and hit `Enter`.

    2: `Do you want to configure VLANs now?` Type `y` and hit `Enter`.

    3: `Enter the parent interface name for the new VLAN:` Type your physical interface name (e.g., `em0`) and hit Enter.

    4: `Enter the VLAN tag (1-4094):` Create your LAN tag (e.g., `20`) and hit Enter.

    5: `Enter a description for this VLAN (optional):` Type `LAN` and hit `Enter`.

    6: `Enter the parent interface name for the new VLAN:` Type your physical interface name again (e.g., `em0`) and hit Enter.

    7: `Enter the VLAN tag (1-4094):` Create your ISP/WAN tag (e.g., `10`) and hit `Enter`.

    8: `Enter a description for this VLAN (optional):` Type `WAN` and hit `Enter`.

    9: `Enter the parent interface name for the new VLAN:` Assuming you don't have any other core system trunking to establish right now, leave this line completely blank and hit `Enter` to finish the VLAN build phase.

    10: `Enter the WAN interface name or 'a' for auto-detect:` Type your WAN virtual identifier exactly: `em0_vlan10` and hit `Enter`.

    11: `Enter the LAN interface name or 'a' for auto-detect:` Type your LAN virtual identifier exactly: `em0_vlan20` and hit `Enter`.

    12: `Enter the Optional interface 1 name:` Leave this blank and hit `Enter` unless you want to bind additional isolated tags, like for an IoT network.

The screen will output a summary block detailing the pending network transformation:

```
The interfaces will be assigned as follows:

WAN  -> em0_vlan10
LAN  -> em0_vlan20

Do you want to proceed? [y/n]:
```

Type `y` and hit `Enter`. The console will cycle, tear down the legacy untagged routing tables, spin up the new virtual ones, and reload the core firewall rules.

* Once the assignments return you to the main console home layout, you must explicitly bind a static IP network to your new virtual LAN interface so your browser can find it later:

    1: Type `2` and hit `Enter` to select `2) Set interface IP address`.

    2: Select the number corresponding to your LAN interface.

    3: `Configure IPv4 address via DHCP?` Type `n` and hit `Enter`.

    4: `Enter the new LAN IPv4 address:` Type a gateway address (e.g., `192.168.1.1`) and hit `Enter`.

    5: `Enter the new LAN IPv4 subnet bit count:` Type `24` and hit `Enter`. This just means it creates the subnet for `192.168.1.1` - `192.168.1.255`

    6: For WAN upstream settings, hit Enter to skip/leave blank.

    7: `Do you want to enable the DHCP server on LAN?` Type `y` if you want the device to hand out IPs locally over tag 20 immediately, then set your pool ranges when prompted.

* If you haven't configured your `802.1Q VLAN` capable switch to pass VLAN tags, you just need to do access the device's GUI by typing in its IP address, then configure as referenced in the table below. This is assuming it's a 5-port switch.

| VLAN ID | VLAN NAME | Member Ports | Tagged Ports | Untagged Ports |
| ------- | --------- | ------------ | ------------ | -------------- |
| 1 | Default | 1 | None | 1 |
| 10 | N/A | 1-2 | 1 | 2 |
| 20 | N/A | 1, 3-5 | 1 | 3-5 |

*Port 1 is the Trunk Port:* It is a tagged member of both VLAN 10 and VLAN 20, meaning this is the physical cable that plugs directly into your single-port OPNsense NUC appliance to pass trunked traffic.

*Port 2 is the WAN Access Port:* It is an untagged member of VLAN 10. Anything you plug into Port 2 will instantly drop straight into the OPNsense WAN subnet.

*Ports 3, 4, and 5 are the LAN Ports:* They are untagged members of VLAN 20 (with 4 and 5 also sharing space on the default management network VLAN 1). This is where the local devices will plug in.

In your `802.1Q PVID` settings, leave or configure Port 1 as `1`, Port 2 as `10`, and the rest as `20`.

Save the settings, plug the OPNsense device into port 1, and any LAN device into 3 or greater.

* Ensure in the GUI that you have something listed like below within the basic configuration of `Interfaces: [LAN]`:

```
Enable:        ☑️ Enable Interface

Identifier:    lan

Device:        vlan20

Description:   LAN
```

* Ensure in the GUI that you have something listed like below within the basic configuration of `Interfaces: [WAN]`:

```
Enable:        ☑️ Enable Interface

Identifier:    wan

Device:        vlan10

Description:   WAN
```

* If something seems off, go to `Interfaces: Assignments` and change the settings as listed:

| Interface | Identifier| Device |
| --------- | --------- | -------- |
| [LAN] | lan | vlan20 VLAN_LAN (Parent: em0, Tag: 20) |
| [WAN] | wan | vlan10 VLAN_WAN (Parent: em0, Tag: 10) |

</details>

<details>
<summary>Two Ethernet Ports</summary>

Yay. This is very simple.

OPNsense requires at least one interface to be assigned to the LAN role so that an administrator can access the management plane. It will scan your motherboard, find the first two active network controllers (e.g., `em0` and `em1`), and assign them to roles. By default, it assigns the lower-numbered hardware adapter (e.g., `em0`) to WAN and the second hardware adapter (e.g., `em1`) to LAN.

To summarize, you shouldn't have to do anything. But worst case scenario, go to the `Interfaces: Assignments` settings in the GUI and change the device bound to the specific interfaces to match what you want to do.

</details>

# DHCP

I highly recommend you set up static IPs for any devices that need it. I'll give an example below.

* Go to `Services: Dnsmasq DNS & DHCP: General` and select and apply the following settings:

| Option | Value |
| :---  | :--- |
| Enable | ☑️ |
| Interface | LAN |
| Listen port | 53053
| DNSSEC | 🔳 |
| No hosts lookup | 🔳 |
| Query DNS servers sequentially | 🔳 |
| Require domain | 🔳 |
| Do not forward to system defined DNS servers | 🔳 |
| DHCP FQDN | ☑️ |
| DHCP default domain | internal |
| DHCP authoritative | 🔳 |
| DHCP reply delay | |
| DHCP register firewall rules | ☑️ |
| Router advertisements | ☑️ |
| Disable HA sync | 🔳 |

The remaining options can be left alone.

* Go to `Services: Dnsmasq DNS & DHCP: Hosts` and click the orange `+` button to create an entry. Clicking on the orange ℹ️ will give you more information, but I will be generalizing for our use case.

    * *Host:* Enter your device name here, one that would be recognizable. If you've configured hostnames for a device using Linux, this should be similar. Example: `homeserver`.

    * *Domain:* Since my example are for LAN devices, use something like `internal`.

    * *Local:* Since we used `internal` above, select this.

    * *IP addresses:* This is where you can assign it a static IP address. Example: `192.168.1.50`.

    * *Alias records:* This gives it essentially another hostname. You shouldn't need this.

    * *CNAME records:* This gives it essentially another hostname when not assigning an IP. You shouldn't need this.

    * *Client identifier:* You shouldn't need this.

    * *Hardware addresses:* This is where you will input a device's network MAC address.

    * *Lease time:* Leave this as default.

# Firewall Aliases

This is to make things easier while creating rules, for example, if you have multiple devices that need an exclusion rather than a single device. You can also reference single devices as well, and I'll include an example for one.

* Go to `Firewall: Aliases` and click the orange `+` button to create an alias.

    * *Name:* Something recognizable, like `homeserver`.
    
    * *Type:* Host(s)
    
    * *Content:* This is where you'd put the previously configured static IP.
    
* For a group of devices, create a single alias.

# Firewall

I'm going to give you some basic firewall rules. I'm aware that there are automatically generated rules, however, this will ensure you will have high security unless you explicitly delete your created rule. *If you want to exempt a specific device or devices from the rule, you would select `Source Invert` and specify the source(s), which would be our previously created alias.* I'll include an example only on the first rule and explain it.

OPNsense processes rules in order, which really matters because of match detection.

## NAT (Port Forwarding)

We want to force all DNS traffic to use OPNsense's DNS instead of bypassing it, right? All devices should use your secure and private DNS configuration. Some devices, like phones, smart TVs, and IoT devices hardcode their DNS and will bypass your configuration unless you do this. In this example, I invert the sources to exclude a specific alias from this force-redirection rule; `homeserver` in this case.

* Go to `Firewall: NAT: Destination NAT` and click the orange `+` button to create a rule. Expand all the tabs so we can see all options.

| Option | Value |
| :---  | :--- |
| Disabled | 🔳 |
| Categories | Nothing selected |
| Description | Force DNS to OPNsense (except homeserver) |
| Interface | LAN |
| Version | IPv4 |
| Protocol | TCP/UDP |
| Invert Source | ☑️ |
| Source Address | homeserver |
| Source Port | any |
| Invert Destination | ☑️ |
| Destination Address | LAN address |
| Destination Port | DOMAIN (53) |
| Redirect Target IP | LAN address |
| Redirect Target Port | DOMAIN (53)
| Log | ☑️ |
| Firewall rule | Register rule |

<details>
<summary>Let me explain what exactly the invert and redirect settings mean.</summary>

| Option | Value | Operational Logic |
| :---  | :--- | :--- |
| Invert Source | ☑️ | Apply this rule to any device except homeserver.
| Source Address| homeserver | Specifies the excluded alias. |
| Invert Destination | ☑️ | Apply this rule to any destination except the firewall itself. |
| Destination Address | LAN address | Specifies the local gateway IP to be excluded from redirection. |
| Redirect Target IP | Loopback network (or 127.0.0.1) | The internal destination where the hijacked packets are sent. |

* Scenario 1: A smart TV tries to bypass your network and queries a rogue DNS (8.8.8.8)

    1: The packet enters the LAN interface with a destination of `8.8.8.8`.
    2: The firewall checks the rule: Is `8.8.8.8` *NOT* the LAN address (`192.168.1.1`)?
    3: Result: `True`. The destination is inverted, so it matches the rule.
    4: The NAT rule intercepts the packet, rewrites the destination to the `Loopback network` (`127.0.0.1`), and forces it to AdGuard Home/Unbound.

* Scenario 2: A standard client cleanly queries the OPNsense Gateway IP (192.168.1.1)

    1: The packet enters the `LAN` interface with a destination of `192.168.1.1`.
    2: The firewall checks the rule: Is `192.168.1.1` NOT the LAN address (`192.168.1.1`)?
    3: Result: `False`. The packet is destined for the `LAN address`, failing the inversion match.
    4: The firewall skips the NAT rule, allowing the packet to pass directly to your local DNS listener without altering the data headers. This is also the correct, desired behavior.

    What happens if you uncheck `Invert Destination`? The rule would only trigger if a client explicitly targeted 192.168.1.1. It would completely ignore rogue traffic bound for 8.8.8.8, allowing devices to bypass your local DNS entirely.

</details>

## Optional: VLAN-Specific Network Rule

### Rule: Tag LAN Packets with NO_WAN_EGRESS

We didn't go over configuring a VLAN interface for IoT devices or something else, but figured I'd include this just in case. This rule is only applicable if, for example, you have IoT devices you don't communicating outside the LAN. We're creating a standard pass rule on the LAN (but it can be applied to others you might configure like an IoT VLAN or a local server VLAN). If a device on that network tries to talk, we configure OPNsense to stamp that packet with a local tag: `NO_WAN_EGRESS`, and prevent it from talking outside the LAN. It references an alias we didn't create, `Isolated_Devices`, that would cover IoT devices.

If you had a smart switch or even a regular switch hooked up to a smart switch, you'd give the Port itself a unique `801.1Q PVID` (like 50), and make Port 1 a tagged member (if using Router-on-a-Stick method as configured earlier in the guide) and the VLAN 50 ports untagged.

| Option | Value |
| :---  | :--- |
| Action | Pass |
| Disabled | 🔳 Disable this rule |
| Quick | ☑️ Apply the action immediately on match |
| Interface | VLAN_IoT |
| Direction | in |
| TCP/IP Version | IPv4 |
| Protocol | any |
| Source / Invert | 🔳 Use this option to invert the sense of the match. |
| Source | Isolated_Devices |
| Destination / Invert | 🔳  Use this option to invert the sense of the match. |
| Destination | any |
| Destination port range | from / to: `any` |
| Log | 🔳 Log packets that are handled by this rule |
| Category | |
| Description | Allow isolated devices local access but tag for WAN block |

Advanced options:

| Option | Value |
| :---  | :--- |
| Set local tag | `NO_WAN_EGRESS` |

## LAN Rules

* Go to `Firewall: Rules: LAN` and click the orange `+` button to create rules.

### Rule: Allow LAN to Any

| Option | Value |
| :---  | :--- |
| Action | Pass |
| Disabled | 🔳 Disable this rule |
| Quick | ☑️ Apply the action immediately on match |
| Interface | LAN |
| Direction | in |
| TCP/IP Version | IPv4 |
| Protocol | any |
| Source / Invert | 🔳 Use this option to invert the sense of the match. |
| Source | LAN network |
| Destination / Invert | 🔳  Use this option to invert the sense of the match. |
| Destination | any |
| Destination port range | from / to: `any` |
| Log | 🔳 Log packets that are handled by this rule |
| Category | |
| Description | Default rule, allow LAN to any |

## Floating Rules

### Optional Rule: Stop Tagged LAN Packets from Egressing

This rule is used to enforce tight security boundaries, like keeping isolated IoT devices, local media servers, or secure subnets completely locked inside the local network, even if someone accidentally changes a rule later. Because this floating rule sits at the very edge of your network (the WAN interface), we make a bulletproof catch-all. No matter what happens on the internal interfaces, and no matter what other rules are written later, if a packet carrying that tag tries to sneak out to the open internet, this floating rule kills it instantly.

This means we need to create a specific rule to help the floating rule work. But we already did that with `Rule: Tag LAN Packets` in LAN.

Why use a Floating rule with a `Local Tag` instead of a normal rule? It eliminates rule duplication. We write the blocking logic once on the WAN interface, rather than repeating blocks on every single VLAN tab. And if you accidentally add a `Pass All` rule on a local interface down the road while troubleshooting, that local interface rule might override a standard block rule. However, because this floating rule uses the `Quick` option at the WAN egress point, it acts as the absolute final authority. If the packet is tagged, it dies at the border, saving you from accidental internet leaks.

| Option | Value |
| :---  | :--- |
| Action | Block |
| Disabled | 🔳 Disable this rule |
| Quick | ☑️ Apply the action immediately on match |
| Interface | WAN |
| Direction | out |
| TCP/IP Version | IPv4 |
| Protocol | any |
| Source / Invert | 🔳 Use this option to invert the sense of the match. |
| Source | any |
| Destination / Invert | 🔳  Use this option to invert the sense of the match. |
| Destination | any |
| Destination port range | from / to: `any` |
| Log | 🔳 Log packets that are handled by this rule |
| Category | |
| Description |  |

For Advanced Features:

| Option | Value |
| :---  | :--- |
| allow options | 🔳 |
| reply-to | default |
| Set priority | All packets: Keep current priority and Low Delay/TCP ACK: Use main priority |
| Match priority | Any priority
| Match TOS / DSCP | Any |
| Set local tag | |
| Match local tag | `NO_WAN_EGRESS`

The rest of the options are default values or unset.

# Unbound Configuration

You can likely leave this as default. However, adding AdGuard Home is highly beneficial for DNS blocking, whether for stopping ads or protecting against certain domains from loading. I recommend you follow the [AdGuard Home guide](./AdGuard/README.md) after reading the rest of this guide. As the overhead is minimal and it isn't complex to get running, I'm going to continue this section assuming you will be using AdGuard Home.

* Go to `Services: Unbound DNS: General`: 

| Option | Value |
| :---  | :--- |
| Enable Unbound | ☑️ |
| Listen Port | `5335` |
| Network Interfaces | All (recommended) |
| Enable DNSSEC Support | ☑️ |
| Enable DNS64 Support | 🔳 |
| DNS64 Prefix | |
| Enable AAAA-only mode | 🔳 |
| Register ISC DHCP4 Leases | ☑️ |
| DHCP Domain Override | |
| Register DHCP Static Mappings | ☑️ |
| Do not register IPv6 Link-Local addresses | 🔳 |
| Do not register system A/AAAA records | 🔳 |
| TXT Comment Support | 🔳 |
| Flush DNS Cache during reload | 🔳 |
| Local Zone Type | transparent |

Apply the settings.

* Go to `Services: Unbound DNS: Advanced`. The first seven checkboxes are for security and privacy. The rest are mostly for performance. You should figure out what you're able to do based on your hardware by researching it. As there is a very broad range of hardware that can run OPNsense, I won't cover that. But here are the settings to check except one:

| Option | Value |
| :---  | :--- |
| Hide Identity | ☑️ |
| Hide Version | ☑️ |
| Prefetch DNS Key Support | ☑️ |
| Harden DNSSEC Data | ☑️ |
| Harden Below NXDOMAIN | ☑️ |
| Aggressive NSEC | ☑️ |
| Strict QNAME Minimisation | 🔳 |

* Go to `Services: Unbound DNS: Access Lists` and ensure you have LAN in there. If not, click the orange `+` button to create one.

| Option | Value |
| :---  | :--- |
| Enabled | ☑️ |
| Access List Name | LAN |
| Action | Allow |
| Networks | 192.168.1.0/24 |
| Description | |

* Go to `Services: Unbound DNS: Query Forwarding`. This can be redundant with our `Register ISC DHCP4 Leases` and `Register DHCP Static Mappings` settings checked, but creating an entry here ensures Unbound knows about our DHCP clients. If you find that the logs for Unbound (or AdGuard Home) do not list clients in the query logs, then click the orange `+` button to create one:

| Option | Value |
| :---  | :--- |
| Domain | 1.168.192.in-addr.arpa |
| Server IP | 127.0.0.1 |
| Server Port | 53053 |
| Forward first | 🔳 |
| Description | |

* Go to `Services: Unbound DNS: DNS over TLS`. This is where the magic happens to prevent your ISP from knowing what domains you are requesting. There are a few options, but I'll show the configuration for Quad9 and Cloudflare. I recommend one (or two of Quad9's) or the other. Click the orange `+` button to create entries:

| Option | Value |
| :---  | :--- |
| Enabled | ☑️ |
| Domain | |
| Server IP | 9.9.9.9 |
| Server Port | 853 |
| Forward first | 🔳 |
| Verify CN | dns.quad9.net |
| Description | |

| Option | Value |
| :---  | :--- |
| Enabled | ☑️ |
| Domain | |
| Server IP | 149.112.112.112 |
| Server Port | 853 |
| Forward first | 🔳 |
| Verify CN | dns.quad9.net |
| Description | |

| Option | Value |
| :---  | :--- |
| Enabled | ☑️ |
| Domain | |
| Server IP | 1.1.1.1 |
| Server Port | 853 |
| Forward first | 🔳 |
| Verify CN | cloudflare-dns.com |
| Description | |

Make sure you pick either Quad9 or Cloudflare, not both. Unless you want to. I don't care.

---------

# Updates

When updating an OPNsense stack running complex plugins always create a ZFS safety checkpoint.

Do NOT use the GUI `Check for updates` button. The GUI triggers a global, concurrent update that can cause dependency, API, and database migration loops.

## Pre-Update Safeguard

Because OPNsense integrates `bectl` into its backend framework, you can manage snapshots via the CLI or right from the Web GUI dashboard under `System: Snapshots`.

<details>
<summary>From GUI</summary>

* Go to `System: Snapshots` and click the orange `+` button to create a ZFS backup (snapshot).

* The name defaults to a time format. You can change that or add a note. The name can be `Pre_Change_Backup`.

* Click Save. Easy and done.

</details>


<details>
<summary>From CLI</summary>

* Whenever you are about to modify a major system configuration, install an experimental plugin, or adjust core networking routing logic, run this terminal command:

```
bectl create Pre_Change_Backup
```

This freezes your current operating OS dataset. Because ZFS uses a redirect-on-write model, this operation is instantaneous and takes up exactly 0 bytes of space initially. It only consumes storage as files are modified or updated over time.

* To see your existing recovery points:

```
bectl list
```

The output would look something like this:

```
BE                Active Mountpoint Space Created
Pre_Change_Backup -      -          8K    2026-05-18 17:18
default           NR     /          1.85G 2026-03-17 14:54
```

`N` means active Now.
`R` means active upon Reboot.

* For a security perimeter firewall, a weekly rotation is ideal. You do not want a daily cron snapshot script running indefinitely, as a massive influx of automated snapshots can clutter your ZFS storage pool metadata over time.

<details>

## Scheduled ZFS Backups

You can use the built-in OPNsense cron manager to automate a rolling backup script. Skip if you don't care.

<details>

<summary>Creating automated ZFS backups.</summary>

Here's a quick cheat sheet for VIM, which I don't like (yet?).

| What you want to delete | The Keystroke |
| :---  | :--- |
| A single character | `x` |
| A whole word | `dw` |
| An entire line | `dd` |
| Everything from cursor to end of line | `d$` |
| Delete everything inside the file | `ggdG` |

Change something quickly: If you want to delete a word and immediately start typing a replacement, use `cw` (Change Word). It deletes the word and drops you straight into standard typing mode.

If you accidentally butchered the file while fighting the keys, Hit `Esc` a few times. Type `:q!` and hit `Enter`. This forces Vim to quit immediately without saving any of the changes you just made, letting you open the file fresh. If you did make edits you want to keep, press `Esc`, type `:wq` and hit `Enter` to save and crash out.

* First, we need to save the rolling backup logic into a script file on the firewall. In the root shell, create the script file:

```
vi /usr/local/bin/zfs_rollup.sh
```

Paste the following rotation script into the file:

```
#!/bin/sh
# Prune the oldest historical environment to control storage footprint
bectl destroy -F Weekly_Two_Weeks_Ago >/dev/null 2>&1

# Cycle the existing backup backward in rotation
bectl rename Weekly_Last_Week Weekly_Two_Weeks_Ago >/dev/null 2>&1

# Freeze a fresh, current checkpoint of the operational engine
bectl create Weekly_Last_Week
```

<details>
<summary>What exactly is this script?</summary>

The script (obviously) creates a ZFS backup every week and handles its own cleanup completely through a rolling rotation. It will never run away with your disk space or pile up an infinite number of old snapshots.

Here is exactly how the script controls its footprint every time it runs:

[Execution Step]                     [Storage State]
1. bectl destroy Weekly_Two_Weeks_Ago --> Frees up space from the oldest backup
2. bectl rename Weekly_Last_Week ...  --> Rolls last week's backup into the old slot
3. bectl create Weekly_Last_Week      --> Freezes a fresh snapshot of your system

Because of that order of operations, the script maintains a strict maximum of two historical snapshots at any given time (`Weekly_Last_Week` and `Weekly_Two_Weeks_Ago`).

The first two commands end with `>/dev/null 2>&1`. This is a silent safety valve for cron.

The very first time this cron job runs, `Weekly_Two_Weeks_Ago` and `Weekly_Last_Week` won't exist yet. Normally, trying to delete or rename a file that doesn't exist causes the system to scream an error message. That syntax silences the empty errors so the script can smoothly cruise right past them and create the very first fresh backup. By the third week, the rotation engine is fully primed, and it will cleanly drop the oldest snapshot before taking a new one—keeping the ZFS pool tidy without you ever needing to intervene.

</details>

Save and exit by pressing `Shift` + `Colon`, then `wq` (`:wq`), then make the script executable so the system is allowed to run it:

```
chmod +x /usr/local/bin/zfs_rollup.sh
```

* To make this script visible to the GUI interface, create a new template action file:

```
vi /usr/local/opnsense/service/conf/actions.d/actions_zfsbackup.conf
```

Paste this configuration inside. This maps a GUI name and description to your backend script:

```
[create]
command:/usr/local/bin/zfs_rollup.sh
parameters:
type:script
message:Creating Weekly ZFS Boot Environment Backup
description:ZFS Weekly Rolling Backup
```

Save and exit (`:wq`).

* Tell the OPNsense configuration daemon to reload its actions array and read your new file:

```
service configd restart
```

* Now that `configd` knows about the command, it will appear directly in your browser.

1: Log into your OPNsense Web GUI. Navigate to `System` ➔ `Settings` ➔ `Cron`.

2: Click the `+` (`Add`) button in the bottom right corner of the table.

3: Set the Description to something recognizable like: `Weekly Rolling ZFS Snapshot`.

4: Click the Command dropdown menu and select `ZFS Weekly Rolling Backup`.

5: Configure the schedule for a quiet time (for example, every Sunday morning at 2:00 AM):
    * Minutes: `0`

    * Hours: `2`

    * Days: `*`

    * Months: `*`

    * Days of week: `0` (`0` is Sunday in cron format)

6: Click Save.

* The automated backup task is now fully integrated into the native OPNsense ecosystem. On Sunday at 2:00 AM, the system will run the script, create `Weekly_Last_Week`, and you can monitor its execution or spacing anytime by running `bectl list` from the terminal.

</details>

## Updating Everything Safely

* Update OPNsense:

When updating an OPNsense stack running deep packet inspection or automated security agents (like Zenarmor and CrowdSec), a global concurrent update via the Web GUI can cause the backend configuration daemon (`configd`) to deadlock, resulting in infinite interface loops or corrupted service states. To prevent this, we decouple the heavy processing daemons first, create a ZFS safety checkpoint, and execute the upgrade sequence safely from the system terminal using official framework tools.

* Manually stop CrowdSec and Zenarmor before initiating the update engine:

```
pluginctl -s sensei stop
pluginctl -s crowdsec stop
```

* The standardized, production-safe way to upgrade an complex OPNsense cluster is from the command line. Invoke the framework's native wrapper script:

```
configctl firmware update
```

This command connects to the core OPNsense repository, maps out all dependencies in the correct structural order, runs native configuration migrations, updates the base OS, and upgrades all plugins cleanly without relying on the browser interface.

* Once the console engine finishes processing the updates, execute a clean system restart to initialize the new kernel space and cleanly bring up your security infrastructure:

```
shutdown -r now
```

## If It is Broke:

* List your available environments to confirm the name of your target snapshot:

```
bectl list
```

* Activate your known-good backup snapshot (we made `Pre_Change_Backup`):

```
bectl activate Pre_Change_Backup
```

* Restart:

```
shutdown -r now
```

When it reboots, it mounts `Pre_Change_Backup` as the primary root filesystem (`/`). It will function exactly as it did the moment you took the snapshot.

## If It is REALLY Broke

If an update corrupts the kernel or boot partition so severely that the machine encounters a kernel panic or boot loop, you can bypass the entire operating system loading phase using the native FreeBSD bootloader menu.

1: Hook up a local monitor/console or use your remote management port on the device.

2: Turn the machine on and wait for the classic OPNsense / FreeBSD Boot Menu (the screen with the large ascii-art logo and a countdown timer).  

3: Press the `Spacebar` to halt the countdown clock.

4: Press `8` on your keyboard to open the Boot Environments submenu.

5: Press `2` repeatedly to cycle through the list of available snapshots until your target backup string is highlighted (e.g., `zfs:zroot/ROOT/Pre_Change_Backup`).

6: Press `1` to go back to the main engine menu.

7: Hit `Enter` to boot into Multi-User mode.

The firewall will bypass the broken default system image completely and boot directly off your chosen frozen snapshot. Once you are securely back into the operational system interface, log back into the terminal and run `bectl activate default` (or whatever naming convention your healthy environment uses) to make that snapshot permanent moving forward.
