# AdGuard Home

AdGuard Home is a network-wide, software-defined DNS proxy and filtering engine designed to intercept internet tracking, advertisements, and malicious infrastructure before they ever reach your local devices. Unlike traditional browser extensions that scrub web elements line-by-line inside an application, AdGuard Home acts as a control checkpoint at the protocol level. It sits between your local network clients and your upstream domain resolution layer (like Unbound), evaluating every single outbound connection request against curated, community-hardened host blocklists.

Because it operates strictly at the DNS level, it strips out advertising and telemetry domains entirely without requiring any software, extensions, or certificates installed on individual endpoints. This extends ad-blocking coverage to traditionally locked-down environments like Smart TVs, IoT systems, mobile applications, and gaming consoles.

AdGuard Home also allows you to break away from rigid network-wide rules. You can group devices by MAC or IP address to apply distinct filtering policies, enforcing aggressive tracking blocks on telemetry-heavy smart home appliances, deploying native parental controls on family devices, and maintaining a lightweight, permissive profile on your primary workstation. It utilizes a robust matching engine that supports complete Adblock Plus (ABP) syntax and Regular Expressions (RegEx). This permits complex, condition-based rule handling and explicit domain rewrites that flat, basic hosts files cannot parse.

It features integrated, hardware-level toggle switches to immediately sever access to major social and media platforms such as TikTok, YouTube, Facebook, or Discord across specific client profiles without requiring manual domain mining.

It replaces obscure, text-based system logs with an administrative interface, offering an instantaneous, searchable ledger of all network DNS traffic. It details exactly which device made a request, the intended destination, and the specific blocklist rule triggered, allowing for instantaneous, one-click exception whitelisting.

## Getting the Repository

* Access the OPNsense CLI and use the following command:

```
fetch -o /usr/local/etc/pkg/repos/mimugmail-single.conf https://www.routerperformance.net/mimugmail-single.conf
```

* Install the Plugin:
    * Go back to the GUI and in `System: Firmware: Status`
    * Click `🔃 Check for updates`.
    * After it is complete, go to `Plugins` and check `🔳 Show community plugins`.
    * Find `os-adguardhome-maxit` and click the `+` icon to install it.

# Configuring AdGuard Home

## DNS Settings

* When it's done installing, refresh the page with `F5` to see it under `⚙️ Services` in the navigation pane on the left. Enable it and set it as the primary DNS. If it hasn't started, start the service on the top-right of this page.

* Open the AdGuard Home GUI by going to http://192.168.1.1:3000 in your web browser.

* Set up the administrator password. Again, I recommend you use a password manager to have a secure, complex password.

* Choose the upstream DNS servers you want AdGuard Home to use. Since we configured Unbound, use `127.0.0.1:5335`. If you need to double check the Unbound settings, do so. Since you followed my guide, it should still be on `Listening port` `5335`.

* AdGuard Home must be bound to `All Interfaces (0.0.0.0)`. Complete the setup wizard to finish the installation process.

* I'll let you configure most things as desired, but I'll mention some things that either need to be done, or things that should be changed if you've followed my guide faithfully. In the Settings menu, under `General settings`, you can leave this at their default values.

* Under DNS settings:
    * Upstream DNS server should still just be `127.0.0.1:5335`. If anything else is here, delete it.
    * Default should be 🔘 Load-balancing, but it doesn't really matter. There's only one upstream listed.
    * You can configure `Fallback DNS servers`, but I don't.
    * You can delete the `Bootstrap DNS servers` list, or leave it. The setting is disabled in AdGuard and it doesn't matter.
    * In `Private reverse DNS servers`, add `127.0.0.1:5335` so it properly gets DHCP information.
    * Ensure the boxes are checked for `☑️ Use private reverse DNS resolvers` and `☑️ Enable reverse resolving of clients' IP addresses`.
    * You can test the upstream server. It should say `Specified DNS servers are working correctly`.
    * For `Rate limit`, you can set it to something reasonable like `20`, or raise it if there's intermittent connectivity problems due to too many domains being requested at once. It's common for scripts in many larger websites to request 15+ domains within a few seconds.
    * Leave the Subnet prefix lengths alone.
    * Ensure `🔳 DNSSEC` is unchecked. Enabling it in both here and Unbound is redundant, wastes system resources, and can break the local DNS resolution pipeline.
    * Ensure `☑️ Disable resolving of IPv6 addresses` is checked, assuming you don't need IPv6. Since my guide is strictly IPv4-only, this forces AdGuard to immediately drop all AAAA queries and remove IPv6 hints from HTTPS responses, preventing client OS connection timeouts and unnecessary protocol overhead.
    * For `Blocking mode`, choose `🔘 Null IP`. This drops blocked tracking and advertising domains instantly at the client level, preventing aggressive app retry loops, log spam, and secondary DNS fallback leaks.
    * For `DNS cache configuration`, uncheck `🔳 Enable cache`. Unbound should be doing the cache storage and serving.
    
* DNS Encryption settings can be left at default values.

* If you want to configure Client settings, go ahead.

## Filters

* In the `Filters` settings menu, go to `DNS blocklists`. The `AdGuard DNS filter` is enabled by default. You can `Add blocklist` by the AdGuard provided `Choose from the list` option, or you can `Add a custom list`. I highly recommend the following:
    * `HaGeZi's Pro++ Blocklist`: HaGeZi lists are awesome for network-wide DNS filtering. The maintainer aggressively cleans the data, ensuring telemetry, aggressive ad trackers, and metrics are dropped without breaking core web functionality, banking portals, or content delivery networks (CDNs). `PRO` is a great baseline; `PRO++` adds tighter tracking restrictions with only an entry-level risk of minor false positives.
    * If you're highly concerned about breakage, `OISD - Full` is probably the best. The maintainer actively tests against popular platforms, streaming apps, and gaming servers to prune any domain that might trigger a support ticket.
    * `HaGeZi's Threat Intelligence Feeds`: This compiles high-confidence, real-time threat intelligence data covering active malware command-and-control (C2) servers, active phishing campaigns, cryptojacking scripts, and immediate internet scams. It operates independently of consumer ad trackers, ensuring high-risk structural blocks without impacting daily usage.
    * `HaGeZi's NSFW`: Unlike massive, uncurated adult lists that accidentally sweep up content delivery networks shared by non-explicit sites, [HaGeZi’s adult filter](https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/nsfw.txt) is strictly maintained. The `Light` variant targets the most prominent global networks and explicit hosting platforms, while the `Normal` variant tightens restrictions across minor networks. Both maintain high accuracy without breaking standard web traffic.

* `DNS rewrites` are valuable for a few reasons. If you host any web server with a domain pointing an A/AAAA record to your public IP, you won't be able to connect to it over LAN. If you rewrite those domains to point directly to the local server IP (e.g., `192.168.1.50`), that resolves that conflict. If you need it, use it.

* `Blocked services` is where you can explicitly block certain applications. This mostly applies to families, but can be useful in workplaces as well.

# Next Steps

🌐 [CrowdSec](./CrowdSec/README.md): Collaborative, behavior-based security engine that analyzes system logs to detect malicious activity and automatically blocks aggressive IP addresses using a globally shared threat intelligence network.

🥷 [Zenarmor](./Zenarmor/README.md): Lightweight, next-generation firewall plugin that provides enterprise-grade deep packet inspection (DPI), application filtering, and advanced web security controls.


