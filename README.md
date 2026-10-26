<div align="center">

# 🔑 EIGRP Authentication: "Plain" vs MD5 & the Key-Chain Pitfall

### Does EIGRP Support Plain-Text Authentication? · Key Chains · Why Authentication Silently Stays OFF

![Cisco](https://img.shields.io/badge/Cisco-IOS%20Routers-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Protocol](https://img.shields.io/badge/Protocol-EIGRP%20AS%201-blue?style=for-the-badge)
![Auth](https://img.shields.io/badge/Auth%20Mode-MD5%20only-critical?style=for-the-badge)
![Focus](https://img.shields.io/badge/Focus-Troubleshooting-orange?style=for-the-badge)
![Routers](https://img.shields.io/badge/Routers-3-yellow?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA%20%2F%20CCNP-red?style=for-the-badge)

**Key finding: EIGRP has no plain-text mode, and a key chain alone does not turn authentication on.**

</div>

---

## 📖 Table of Contents

[Overview](#-project-overview) · [Lab Snapshot](#-lab-snapshot) · [Timeline](#-what-happens-in-this-lab-timeline) · [Topology](#️-network-topology) · [Configuration](#️-configuration-as-typed-in-the-lab) · [Stage-by-Stage Evidence](#-evidence--stage-by-stage) · [Findings](#-findings-what-the-outputs-reveal) · [The Fix](#-the-fix-working-md5-authentication) · [Protocol Comparison](#-authentication-options-by-protocol) · [Troubleshooting](#-troubleshooting-guide) · [Advanced](#-advanced-topics) · [Takeaways](#-key-takeaways) · [Challenges](#-challenges)

---

## 📌 Project Overview

Routing protocols like RIPv2 and OSPF offer a **plain-text password** option. So I set out to configure **"plain" authentication on EIGRP** and explored the options directly in the IOS CLI.

The CLI answered the first question right away:

```
R1(config-if)# ip authentication mode eigrp 1 ?
  md5  Keyed message digest          ◄── 🔥 the ONLY option
```

> 🎯 **EIGRP has no plain-text authentication.** Classic-mode EIGRP supports **MD5 only**.

The lab then built a **key chain** and applied it to the interface — and the running-config revealed two subtle problems that would leave authentication **switched off** without any error message. Finding and understanding those is the real value of this project.

---

## 📸 Lab Snapshot

| Item | Value |
|:-----|:------|
| 🌐 Routing protocol | **EIGRP AS 1** |
| 🖧 Routers | Router1 · Router2 · Router3 (linear topology) |
| 🔗 Links | 1.1.1.0/24 (R1–R2, Fa0/0) · 2.1.1.0/24 (R2–R3, Fa1/0) |
| 🗝️ Key chain created | `khan` · key `1` · key-string `cisco` |
| 🔌 Interface command used | `ip authentication key-chain eigrp 1 cisco` on **Fa0/0** (R1 and R2) |
| 🔍 Tools | IOS CLI help (`?`), `show running-config interface f0/0` |
| 🧪 Outcome | CLI shows MD5 is the only mode; config analysis exposes two gaps |

---

## 🎬 What Happens in This Lab (Timeline)

```
 ① KEY CHAIN        ② CLI EXPLORATION        ③ APPLY TO INTERFACE      ④ VERIFY            ⑤ ANALYZE & FIX
 ───────────        ─────────────────        ────────────────────      ────────────        ───────────────
 key chain khan     ip authentication ?      ip authentication         show run int f0/0   spot the 2 gaps:
 key 1              → key-chain / mode       key-chain eigrp 1 cisco   → no "mode md5"     • name mismatch
 key-string cisco   mode eigrp 1 ?                                     → name = "cisco"    • no md5 mode
                    → md5 ONLY                                                              then correct it
```

| Stage | What was done | Key evidence |
|:-----:|:--------------|:-------------|
| ① | Created key chain `khan` (after deleting a first attempt named `1`) | `key chain khan / key 1 / key-string cisco` |
| ② | Explored authentication options with `?` | `md5  Keyed message digest` is the only mode |
| ③ | Applied the key chain on Fa0/0 of **R1 and R2** | `ip authentication key-chain eigrp 1 cisco` |
| ④ | Verified the interface configuration | No `ip authentication mode eigrp 1 md5` line |
| ⑤ | Analyzed the result | Two gaps found — corrected below |

---

## 🎯 Objectives

- ✅ Find out **which authentication modes EIGRP supports** using the CLI
- ✅ Create a **key chain** with a key ID and key string
- ✅ Apply a key chain to an **interface** for EIGRP AS 1
- ✅ **Verify** the configuration and read it critically
- ✅ Understand **why a key chain alone does not authenticate**
- ✅ Avoid the **key-chain name vs key-string** confusion
- ✅ Compare **EIGRP, RIPv2 and OSPF** authentication options

---

## 🗺️ Network Topology

```
        Router1                          Router2                         Router3
   ┌────────────────┐               ┌────────────────┐              ┌──────────────┐
   │ key chain khan │ fa0/0   fa0/0 │ key chain khan │ fa1/0  fa1/0 │              │
   │ key 1          ├───────────────┤ key 1          ├──────────────┤  (no auth)   │
   │ key-string     │   1.1.1.0/24  │ key-string     │  2.1.1.0/24  │              │
   │   cisco        │               │   cisco        │              │              │
   └────────────────┘               └────────────────┘              └──────────────┘
    Fa0/0: ip authentication         Fa0/0: ip authentication
           key-chain eigrp 1 cisco          key-chain eigrp 1 cisco
```

> 📷 Add your screenshot: `![Topology](images/topology.png)`

### 📋 Addressing (from the outputs)

| Device | Interface | IP Address |
|:------:|:---------:|:----------:|
| **R1** | Fa0/0 | 1.1.1.1/24 |
| **R2** | Fa0/0 | 1.1.1.2/24 |
| **R2 / R3** | Fa1/0 | 2.1.1.x/24 *(R2–R3 link)* |

---

## ⚙️ Configuration (As Typed in the Lab)

### 🔹 Key chain (R1, and the same on R2)

```bash
key chain khan
 key 1
  key-string cisco
```

### 🔹 Interface command (R1 and R2)

```bash
interface FastEthernet0/0
 ip authentication key-chain eigrp 1 cisco
```

<details>
<summary><b>📜 Click to see the full CLI session on R1 (including the first attempt)</b></summary>

```
R1(config-router)# do wr
Building configuration...
[OK]
*Oct 26 12:38:15.583: %DUAL-5-NBRCHANGE: EIGRP-IPv4 1: Neighbor 1.1.1.2 (FastEthernet0/0) is up: new adjacency

R1(config)# key chain 1                ◄── first attempt: chain named "1"
R1(config-keychain)# ex
R1(config)# no key chain 1             ◄── removed
R1(config)# key chain khan             ◄── recreated as "khan"
R1(config-keychain)# key 1
R1(config-keychain-key)# key-string cisco
R1(config-keychain-key)# ex
R1(config-keychain)# ex
R1(config)# interface f0/0
R1(config-if)# ip authentication mode eigrp 1 ?
  md5  Keyed message digest
R1(config-if)# ip authentication key-chain eigrp 1 ?
  WORD  name of key-chain
R1(config-if)# ip authentication key-chain eigrp 1 cisco
```

</details>

---

## ✅ Evidence — Stage by Stage

### 🟡 Stage 1: Create the Key Chain

```
R1(config)# key chain khan
R1(config-keychain)# key 1
R1(config-keychain-key)# key-string cisco
R1(config-keychain-key)# exit
R1(config-keychain)# exit
```

```
R1# show running-config | section key chain
key chain khan                         ◄── 🏷️ chain NAME  = khan
 key 1                                 ◄── 🔢 key ID      = 1
  key-string cisco                     ◄── 🔑 key STRING  = cisco  (the shared secret)
```

---

### 🟢 Stage 2: What Does the CLI Offer? (Exploring with `?`)

```
R1(config-if)# ip authentication ?
  key-chain  key-chain
  mode       mode

R1(config-if)# ip authentication mode ?
  eigrp  Enhanced Interior Gateway Routing Protocol (EIGRP)

R1(config-if)# ip authentication mode eigrp 1 ?
  md5  Keyed message digest                      ◄── 🔥 ONLY MD5 — there is no "text" / plain option

R1(config-if)# ip authentication key-chain eigrp ?
  <1-65535>  AS number

R1(config-if)# ip authentication key-chain eigrp 1 ?
  WORD  name of key-chain                        ◄── expects the key chain NAME
```

🧠 **What this shows**

| CLI menu | Meaning |
|:---------|:--------|
| `ip authentication mode eigrp 1 ?` → **md5** | EIGRP offers **only MD5** — no plain text |
| `ip authentication key-chain eigrp 1 ?` → **WORD** | The command wants the **name of a key chain** |

---

### 🔵 Stage 3: Apply to the Interface (R1 and R2)

```
R2(config)# interface f0/0
R2(config-if)# ip authentication key-chain eigrp 1 cisco
R2(config-if)# do wr
Building configuration...
[OK]
```

---

### 🔴 Stage 4: Verify the Interface Configuration (R1) 🔎

```
R1# show running-config interface f0/0
interface FastEthernet0/0
 ip address 1.1.1.1 255.255.255.0
 ip authentication key-chain eigrp 1 cisco          ◄── ⚠️ references a chain named "cisco"
 duplex full                                        ◄── ⚠️ no "ip authentication mode eigrp 1 md5" line
end
```

📸 **Running-config excerpt** (global section):

```
key chain khan                                      ◄── the real chain is named "khan"
 key 1
  key-string cisco
```

---

## 🔍 Findings: What the Outputs Reveal

| # | Finding | Evidence | Impact |
|:-:|:--------|:---------|:-------|
| 1️⃣ | **EIGRP has no plain-text mode** | `mode eigrp 1 ?` lists only `md5` | "Plain authentication" cannot exist in classic EIGRP |
| 2️⃣ | **Key-chain name mismatch** | Interface uses `cisco`, chain is named `khan` | The interface points to a chain that **doesn't exist** — `cisco` is the *key-string*, not the chain name |
| 3️⃣ | **`mode md5` was never enabled** | No `ip authentication mode eigrp 1 md5` in the interface config | Without it EIGRP **does not authenticate at all** |
| 4️⃣ | **R2–R3 link has no authentication** | Router3 has no key chain; Fa1/0 has no commands | That link is unauthenticated by design here |

> ⚠️ **Why it's dangerous:** IOS accepts the interface command silently and the neighbor **stays up** — no error, no warning. The network *looks* secured but isn't.

### 🧠 Authentication needs BOTH pieces

```
          ┌───────────────────────────────┐        ┌────────────────────────────────┐
          │ ip authentication mode        │   +    │ ip authentication key-chain    │
          │ eigrp 1 md5                   │        │ eigrp 1 <EXISTING-CHAIN-NAME>  │
          │ → TURNS authentication ON     │        │ → says WHICH key to use        │
          └───────────────────────────────┘        └────────────────────────────────┘
                                =  working EIGRP MD5 authentication
```

| Configured | Authentication active? |
|:-----------|:----------------------:|
| Key chain only | ❌ No |
| `key-chain eigrp 1 <name>` on interface **without** `mode md5` | ❌ **No** *(this lab as captured)* |
| `mode eigrp 1 md5` **+** `key-chain eigrp 1 <correct name>` on **both** sides | ✅ Yes |

---

## 🛠️ The Fix: Working MD5 Authentication

Run on **R1 and R2** (interface Fa0/0):

```bash
interface FastEthernet0/0
 ip authentication mode eigrp 1 md5
 ip authentication key-chain eigrp 1 khan        ◄── use the chain NAME (khan), not the key-string
 exit
do write memory
```

> If you want the R2–R3 link protected too, create the same key chain on **Router3** and apply both commands on **R2 Fa1/0** and **R3 Fa1/0**.

### 🔬 What you should see (expected behavior)

| Step | Event | Expected log message |
|:----:|:------|:---------------------|
| 1 | MD5 enabled on **R1 only** | `Neighbor 1.1.1.2 (FastEthernet0/0) is down: authentication mode changed` ❌ |
| 2 | Same config applied on **R2** | `Neighbor 1.1.1.1 (FastEthernet0/0) is up: new adjacency` ✅ |

### 📷 Capture these after the fix

```bash
show running-config interface f0/0     # now shows BOTH mode md5 and key-chain khan
show ip eigrp neighbors                # adjacency Up on Fa0/0
show key chain                         # key chain khan, key 1
ping 1.1.1.2                           # connectivity after securing
```

---

## 🧾 Evidence Summary

| # | Screenshot | What it proves |
|:-:|:-----------|:---------------|
| 1 | R1 key chain creation | Chain `khan`, key 1, key-string `cisco` (first attempt `1` removed) |
| 2 | R1 CLI help | Only `md5` exists; key-chain command expects a **name** |
| 3 | R2 interface command | Same `key-chain eigrp 1 cisco` applied on Fa0/0 |
| 4 | Running-config excerpt | Key chain is named **khan** |
| 5 | R1 `show running-config interface f0/0` | No `mode md5` line; name `cisco` ≠ `khan` |
| 6 | Topology | Router1/Router2 annotated; Router3 has no authentication |

---

## 🔐 Authentication Options by Protocol

| Protocol | Plain text | MD5 | Stronger (SHA) |
|:---------|:----------:|:---:|:--------------:|
| **RIPv2** | ✅ | ✅ | — |
| **OSPFv2** | ✅ | ✅ | ✅ (HMAC-SHA via key chain on newer IOS) |
| **EIGRP (classic)** | ❌ **None** | ✅ **Only option** | — |
| **EIGRP (named mode)** | ❌ | ✅ | ✅ **HMAC-SHA-256** |

---

## 🧰 Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|:--------|:-------------|:----|
| Neighbor stays up after "configuring authentication" | `mode md5` missing — authentication never turned on | Add `ip authentication mode eigrp 1 md5` |
| Neighbor drops after enabling MD5 on one side | Other side not configured yet | Configure the matching mode and key chain on the neighbor |
| Neighbor never recovers after both sides configured | Key-chain **name** wrong, or key ID / key-string differ | Reference the real chain name; match key ID and string |
| `key-chain` command accepted but nothing happens | IOS doesn't validate that the chain exists | `show key chain` to confirm the name |
| Confusing `cisco` and `khan` | Key-string mixed up with the chain name | Chain **name** = label · **key-string** = secret |
| Works on one link only | Authentication is **per interface** | Configure each interface that needs it |

---

## 🚀 Advanced Topics

### 🔄 Key rotation with lifetimes

```bash
key chain khan
 key 1
  key-string cisco
  accept-lifetime 00:00:00 Jan 1 2026 00:00:00 Jan 1 2027
  send-lifetime   00:00:00 Jan 1 2026 00:00:00 Jan 1 2027
 key 2
  key-string NewStr0ngKey!
  accept-lifetime 00:00:00 Dec 1 2026 infinite
  send-lifetime   00:00:00 Jan 1 2027 infinite
```

### 🔒 Hardening ideas

- Use a **long, complex key string** — `cisco` is lab-only
- Give the key chain a **meaningful, different name** from the secret to avoid mixing them up
- `passive-interface` on links that should never form neighbors
- On newer IOS, use **EIGRP named mode with HMAC-SHA-256**

---

## 🎓 Key Takeaways

1. 🔸 **EIGRP has no plain-text authentication** — the CLI offers only `md5`
2. 🔸 Authentication needs **two commands**: `mode eigrp 1 md5` **and** `key-chain eigrp 1 <name>`
3. 🔸 A key chain on an interface **without `mode md5` does nothing**
4. 🔸 The interface command wants the **key-chain name**, not the key-string
5. 🔸 IOS **accepts a non-existent chain name silently** — always verify with `show running-config`
6. 🔸 `show key chain` and `show running-config interface` are your proof tools
7. 🔸 Authentication is **per interface** — protect every link that matters
8. 🔸 A neighbor that **stays up** after "enabling authentication" is a red flag, not a success

---

## 🧪 Challenges

1. Apply the fix on R1 only — what log message appears, and why?
2. Rename the key chain on R2 (e.g., `secure`) and reference it — does the adjacency still form?
3. Use a different key-string on R2 — what happens and which command shows it?
4. Extend authentication to the **R2–R3** link
5. Add a second key with lifetimes and test a smooth key rollover

---

## 📁 Repository Structure

```
📦 EIGRP-Plain-vs-MD5-Authentication
 ┣ 📜 README.md
 ┣ 📂 images
 ┃ ┣ 🖼️ topology.png
 ┃ ┣ 🖼️ 01-r1-key-chain.png
 ┃ ┣ 🖼️ 02-cli-help-md5-only.png
 ┃ ┣ 🖼️ 03-r2-interface-command.png
 ┃ ┣ 🖼️ 04-running-config-key-chain.png
 ┃ ┣ 🖼️ 05-r1-interface-no-mode-md5.png
 ┃ ┗ 🖼️ 06-after-fix-neighbors.png
 ┗ 📂 lab-file
   ┗ 📦 EIGRP-Authentication (project file)
```

---

## 🔗 EIGRP Lab Series

| # | Lab |
|:-:|:----|
| 1 | EIGRP Successor – Single Path |
| 2 | EIGRP Successor – 2 Paths |
| 3 | EIGRP Unequal-Cost Load Balancing & Variance |
| 4 | EIGRP MD5 Authentication |
| 5 | EIGRP Metric Calculation |
| 6 | EIGRP Redistribution |
| 7 | EIGRP Route Summarization |
| 8 | **EIGRP Plain vs MD5 Authentication — Key-Chain Pitfall** ◄── *you are here* |

`EIGRP` · `Authentication` · `Key Chain` · `MD5` · `Troubleshooting` · `Routing Security` · `IOS CLI`

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-Certified Network & NOC Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/maazkhanms)

---

<div align="center">

⭐ **If this lab helped you, please star the repository!** ⭐

*Part of my ongoing networking lab series — routing, switching, security & more.*

</div>
