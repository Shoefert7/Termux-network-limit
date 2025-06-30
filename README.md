# Termux-network-limit
# Termux on Non-Rooted Android: A Practical Test of Network Security Limitations

## Introduction

There's a common belief that Termux, a powerful terminal emulator for Android, can turn your non-rooted smartphone into a full-fledged hacking and network analysis tool. As someone interested in cybersecurity, I decided to put this claim to a practical test to see what's truly possible for low-level network operations without root access.

This repository documents my experiment, the commands used, the results, and my conclusion that **Termux on a non-rooted Android device is severely limited for true network reconnaissance and penetration testing**, often due to Android's built-in security sandbox.

## The Challenge & Hypothesis

**Challenge:** Can Termux on a non-rooted Android phone perform practical network discovery, specifically finding other devices' IP addresses and open ports on a local network, in a way that would be useful for a cybersecurity professional in the field?

**Hypothesis:** Due to Android's increasing security restrictions (especially Android 10+), Termux, as a user-space application, will be fundamentally limited in its ability to access low-level network information (like its own IP via `ip a`) and perform comprehensive network scans (like `nmap -sS` or listening for incoming connections).

## Setup for the Experiment

* **Phone A:** My primary Android phone, non-rooted.
* **Phone B:** A second Android phone, non-rooted.
* **Operating System:** Both phones running modern Android (e.g., Android 10+).
* **Termux:** Installed, fully `updated` and `upgraded` on both phones.
* **Key Packages Installed (on both where needed):** `nmap` (includes `ncat`), `python`, `curl`.
* **Network:** Both phones connected to the **same Wi-Fi network**.
* **Initial Known Information:** Phone A's IP address (`192.168.0.227`) was obtained *outside Termux* (via Android's Wi-Fi settings). Phone B's IP was unknown *within Termux*.

## The Experiment & Results

We attempted several scenarios, progressing from basic self-identification to network discovery and communication.

---

### Test 1: Can Termux get its Own Local IP Address?

This is fundamental for understanding your position on a network. `ip a` (or `ifconfig`) is the standard Linux command.

* **Command (on Phone A in Termux):**
    ```bash
    ip a
    ```
* **Expected & Actual Result:**
    ```
    # (Similar output for ifconfig)
    ~ $ ip a
    Cannot bind netlink socket: Permission denied
    ```
    * **Observation:** Termux, on a non-rooted Android, cannot directly query the kernel for its own comprehensive network interface details, including the Wi-Fi IP address. This is a direct security restriction by Android.

---

### Test 2: Can Termux Discover Other Hosts and their Services via Scanning?

We tried to find Phone B by scanning the local network, assuming Phone B would be running a Python HTTP server.

* **Step 2.1: On Phone B, start a simple HTTP server.**
    ```bash
    python -m http.server 8000
    ```
    *(This command was left running on Phone B)*

* **Step 2.2: On Phone A, perform an Nmap scan for port 8000 on the local subnet (inferred from Phone A's known IP).**
    ```bash
    nmap -sT -p 8000 192.168.0.0/24
    ```
* **Actual Result (from Phone A):**
    ```
    Starting Nmap 7.97 ( [https://nmap.org](https://nmap.org) ) at 2025-06-30 01:35 -0400
    Stats: 0:00:12 elapsed; 0 hosts completed (0 up), 256 undergoing Ping Scan
    Ping Scan Timing: About 99.90% done; ETC: 01:36 (0:00:00 remaining)
    Nmap scan report for 192.168.0.1
    Host is up (0.057s latency).

    PORT     STATE  SERVICE
    8000/tcp closed http-alt

    Nmap scan report for 192.168.0.75
    Host is up (0.62s latency).

    PORT     STATE  SERVICE
    8000/tcp closed http-alt

    Nmap scan report for 192.168.0.227
    Host is up (0.0018s latency).

    PORT     STATE  SERVICE
    8000/tcp closed http-alt

    Nmap scan report for 192.168.0.232
    Host is up (0.066s latency).

    PORT     STATE  SERVICE
    8000/tcp closed http-alt

    Nmap scan report for 192.168.0.242
    Host is up (0.061s latency).

    PORT     STATE  SERVICE
    8000/tcp closed http-alt

    Nmap done: 256 IP addresses (5 hosts up) scanned in 60.89 seconds
    ```
* **Observation & Analysis:**
    * While Nmap successfully found active hosts (`.1`, `.75`, `.227`, etc.), **port 8000 was reported as `closed` on *all* of them.**
    * This is the critical finding: Even though Phone B was running a Python server listening on port 8000 *within Termux*, **Android's system-level security blocked incoming connections from reaching that service from the network.** This prevents external scanning tools from detecting the service.
    * Thus, Phone B's IP could **not** be identified through this method, as its distinguishing feature (open port 8000) was masked by Android's sandbox.

---

### Test 3: Can Termux Identify Another Phone via Outgoing Connections (Beaconing)?

Since direct scanning failed, we tried a workaround: making Phone B "beacon" its presence to Phone A. This relies on Termux's ability to make *outgoing* connections.

* **Step 3.1: On Phone A, set up a listener.**
    ```bash
    ncat -lvp 9999
    ```
    *(This command was left running on Phone A)*

* **Step 3.2: On Phone B, send a unique message to Phone A's IP and listener port.**
    ```bash
    echo "Hello from Phone B! My (outgoing) IP is unknown to me in Termux. My timestamp is $(date +%H:%M:%S)." | ncat 192.168.0.227 9999
    ```
    *(Note: Initial attempts resulted in "Ncat refused" because the listener on Phone A was not active - a common human error! Once corrected, it worked.)*

* **Actual Result (from Phone A's `ncat` listener):**
    ```
    Ncat: Version 7.97 ( [https://nmap.org/ncat](https://nmap.org/ncat) )
    Ncat: Listening on 0.0.0.0:9999
    Ncat: Connection from 192.168.0.250:46168.   <-- SUCCESS! THIS IS PHONE B's IP!
    Hello from Phone B! My (outgoing) IP is unknown to me in Termux. My timestamp is 01:54:44.
    Ncat: Awaiting connection...
    ```
* **Observation & Analysis:**
    * **Success!** Phone B's IP address was identified as `192.168.0.250`.
    * This confirms that **outgoing connections initiated from Termux are generally allowed and work.**
    * However, this is a **workaround**, not a standard reconnaissance technique. It requires the target (Phone B) to actively initiate communication to a known listener, which is impractical for discovering unknown hosts in a real "field" scenario.

---

## Conclusion: Termux on Non-Rooted Android is *Not* What I Need

This practical test confirms my initial skepticism. While Termux provides a fantastic Linux command-line environment and is excellent for scripting, SSH into remote servers, and making *outgoing* network requests, its capabilities for **low-level network reconnaissance, discovery, and penetration testing on a non-rooted Android device are severely limited.**

**My key findings:**

* **No Self-IP Discovery:** You cannot reliably get your own Wi-Fi IP address directly from Termux.
* **Incoming Connections Blocked:** Android's security sandbox prevents external connections from reaching services you start within Termux (like a Python web server), making devices "invisible" to typical scans.
* **Network Discovery is Crippled:** Comprehensive port scanning (e.g., `nmap -sS` for stealthy SYN scans, OS detection) is impossible due to lack of raw socket access/privileges. Host discovery relies on unreliable methods.
* **Workarounds are Impractical:** While "beaconing" (outgoing connections) can identify an IP, it's not a viable method for general network discovery in an unknown environment.

**Therefore, for anyone looking to use their phone for serious "cat and mouse" cybersecurity work in the field, a non-rooted Android device running Termux is insufficient.**

**The actual tools needed for such work require:**

1.  **A rooted Android device (often with a specialized distribution like Kali NetHunter).**
2.  **A dedicated laptop/computer running a full Linux distribution (like Kali Linux).**

This experiment solidified my understanding and proved that relying solely on Termux on a non-rooted phone for these types of tasks is not a practical solution.

---
