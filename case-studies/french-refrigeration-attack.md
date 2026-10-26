# Case study 1: French supermarket refrigeration controllers (Oct 2026)

**Written:** 2026-10-19 (expanded 2026-10-20 for portfolio)  
**Alert:** [CERTFR-2026-ALE-012](https://www.cert.cyber.gouv.fr/alerte/CERTFR-2026-ALE-012/)  
**Status:** Case 1 done (2026-10-20). **Updated 2026-10-26:** ANSSI confirmed the entry path (internet exposure plus weak passwords); Purdue mapping added. Figures below are cited; gaps are marked.

---

## What happened

Over the nights of 11–13 October 2026, attackers remotely took over internet-exposed supermarket refrigeration **system managers** (supervisory controllers) in France. They stretched **defrost** cycles from about **1 hour to 8 hours** in some freezers and cold rooms ([Le Monde, via TIME](https://time.com/article/2026/10/13/france-cyber-attack-cybersecurity-refrigeration-supermarket-food-security/)), so case heaters ran for hours and alarms that normally mute during defrost stayed quiet. Staff found spoiled food, water, and in some places fire/smoke the next morning.

**Products named by CERT-FR:** Danfoss AK-SM / AK-SC, Copeland xweb, Dixell xweb.

**Scale (claimed, not a forensic count):** Economy minister Roland Lescure cited ~**700** of ~10,000 French supermarkets ([AP News](https://apnews.com/article/france-cyberattack-supermarkets-refrigeration-systems-78a813a8efd466b360655339f96083dc)). Chains named in press include Intermarché / Les Mousquetaires, Coopérative U, Carrefour, Monoprix, E.Leclerc. Losses reported as **millions of euros** ([SecurityWeek, 13 Oct](https://www.securityweek.com/refrigeration-controller-hack-spoils-food-across-european-supermarkets/)). No injuries or deaths reported. No national food shortage. No ransom demand and no public claim of responsibility.

---

## Simple architecture (close enough)

```text
Internet
   → remote path (public IP, no VPN; a contractor path is reported as the usual pattern but not confirmed)
      → system manager web UI (Danfoss StoreView / XWEB)
         → case controllers (one per display case / cold room)
            → heaters, valves, fans, sensors
```

The system manager does **not** run the temperature loop itself. Case controllers do that. The system manager sends setpoints and defrost schedules down, coordinates packs, and handles alarms/logs. Compromising the supervisor was enough to change every case at once.

---

## Point of entry

The **system manager web login**, reachable from the public internet.

How attackers likely found it: internet scan engines (Shodan / Censys-style) that index pages by HTTP title. A StoreView login page is easy to fingerprint. This is defensive reconnaissance of the same attack surface, not a claim about the attackers' exact tooling.

**Exposure counts (dated):**

| Source | Figure | When |
| --- | --- | --- |
| Censys (via SecurityWeek / related reporting) | nearly **3,000** StoreView panels online (~**1,800+** in France) | around **11 Oct 2026** ([SecurityWeek follow-up / Censys report context](https://www.securityweek.com/supermarket-refrigeration-attack-widens-to-copeland-dixell-controllers/); [Censys blog](https://censys.com/blog/frostbitten-in-france-a-danfoss-storeview-internet-exposure-report/); [Cybersecurity Dive](https://www.cybersecuritydive.com/news/thousands-refrigerator-devices-exposed-french-cyberattack/833117/)) |
| Own Shodan check (learning exercise) | **2,453** results for France on one title query | **20 Oct 2026 UTC** |

Shodan counts **results**, not stores. Geolocation often reflects ISP hubs, not the real store address. No IPs or hostnames are listed here on purpose.

---

## How access was gained (confirmed by ANSSI, 21 Oct 2026)

**Update 2026-10-26:** On 21 Oct 2026, ANSSI's director told the French Senate that attackers got in through refrigeration controllers left directly on the internet, with unprotected admin access and weak passwords. He rejected reports that the Claroty-disclosed flaws were used ([Public Sénat](https://www.publicsenat.fr/actualites/politique/des-defauts-de-securisation-elementaire-et-des-mots-de-passe-faibles-ont-permis-la-cyberattaque-contre-les-frigos-de-supermarches)). The working assumption below is now confirmed.

**Confirmed:** management interfaces should not have been on the public internet ([CERT-FR](https://www.cert.cyber.gouv.fr/alerte/CERTFR-2026-ALE-012/)). Many units were **end-of-life** or on outdated firmware: Danfoss told [Cooling Post](https://www.coolingpost.com/world-news/danfoss-moves-to-mitigate-cyberattack/) most affected units were legacy AK-SM 800 / AK-SC 355 (no more security updates) or current AK-SM 800A on outdated firmware.

**Original working assumption (now confirmed):** weak / default credentials on the exposed login got the attacker into the defrost settings dashboard. Once logged in, no malware was needed; they changed normal settings and the system manager pushed them to the cases.

**Ruled out by ANSSI:** the Claroty-disclosed flaws. Researchers also discussed a date-based support password (e.g. CVE-2025-41450 on AK-SM 800A before R4.2); ANSSI described the cause as weak passwords and did not name a CVE.

---

## What was missing

1. **No VPN / no firewall** in front of the controller (public login page = perimeter).
2. **Unsupported / outdated gear** that cannot be patched (CERT-FR: factory reset alone is not enough if you put an EOL unit back on the internet).
3. **Weak authentication** (weak passwords — confirmed by ANSSI, 21 Oct 2026).
4. **No independent last line for the physical process.** Temperature alarms ride the same controller. Supermarket refrigeration typically has no SIS. At most, hardwired thermal cutouts on heaters; fires in some stores suggest those were missing, failed, or not reported in detail.

---

## Purdue model mapping (added 2026-10-26)

```text
[ LEVEL 5: Public Internet ]
  │
  │ (Direct web request: skipping Levels 4 & 3.5 via exposed public IP)
  └──────────────────────────────────────────────────────┐
                                                         │
[ LEVEL 4: Business IT ] ─── [ BYPASSED BY ARCHITECTURE ] │
                                                         │
[ LEVEL 3.5: Industrial DMZ ] ─── [ MISSING LAYER ]      │
                                                         │
  ┌──────────────────────────────────────────────────────┘
  ▼
[ LEVEL 2 & 3: Supervisory & Site Operations Control ]
  ├── Component: System manager panel / web dashboard
  │     (e.g., Danfoss, Copeland xweb, Dixell xweb) [2]
  └── Vulnerability: Weak/default admin credentials
        exposed to the internet [1]
  │
  ▼ (Valid commands, accepted as legitimate by the local control plane)
[ LEVEL 1: Local Control ]
  ├── Component: Refrigeration case controllers
  └── Exploit reality: NOT hacked or compromised; controllers
        executed native logic exactly as instructed
  │
  ▼ (Malicious defrost settings)
[ LEVEL 0: Physical Process ]
  ├── Components: Compressors, freezers, defrost heaters,
  │     temperature sensors
  └── Physical impact: Defrost heaters forced into continuous
        operation. Temperatures up to about 50°C (122°F) [2],
        food spoilage at about 700 supermarkets [4], and
        fires at several [3].
```

**Takeaway:** a Level 2 device placed directly on Level 5 skips every defensive layer in between. No malware was needed; the controllers did exactly what the supervisor told them.

**References for this diagram**

- [1] ANSSI Senate testimony: [Public Sénat](https://www.publicsenat.fr/actualites/politique/des-defauts-de-securisation-elementaire-et-des-mots-de-passe-faibles-ont-permis-la-cyberattaque-contre-les-frigos-de-supermarches)
- [2] Brands and temperature: [Cybersecurity Dive](https://www.cybersecuritydive.com/news/refrigeration-france-cyber/832858/)
- [3] Fires: [NBC News / Reuters](https://www.nbcnews.com/world/europe/hackers-heat-french-supermarket-fridges-sparking-fires-spoiling-food-rcna603151)
- [4] About 700 stores: [AP News](https://apnews.com/article/france-cyberattack-supermarkets-refrigeration-systems-78a813a8efd466b360655339f96083dc)

---

## Impact (safety vs availability)

| Dimension | What is known |
| --- | --- |
| **Safety** | Some fires reported; no injuries/deaths publicly reported. Spoiled food was discarded (food-poisoning risk mitigated by discard, not by process integrity). |
| **Availability** | Cold chain lost in hundreds of stores for hours; controllers later pulled offline. Public inconvenience (empty cold aisles), not a government-service halt. |
| **Ransom** | None reported. |

---

## What would have stopped it (ranked)

As OT lead, in order of leverage for *this* attack path:

1. **Remove public internet exposure** — VPN (or private network) only for contractor remote access. CERT-FR's first instruction. **This alone would have blocked most of the euro loss**, because the login page would never have been findable from the open internet.
2. **Firewall / source allowlist** as emergency stopgap, then replace with real VPN. **Reduces risk** if exposure slips; not a substitute for (1).
3. **Replace EOL controllers**; keep supported units on current firmware; change defaults; Strict security mode where available. **Reduces risk** after someone reaches the login; does **not** help if the box is still exposed with a default password.
4. **Unique strong passwords** and no shared "code of the day" support login on the open internet.
5. **Alert on configuration changes** — especially defrost schedule / duration and setpoint changes across a store fleet overnight. Nobody caught hundreds of stores flipping from ~1h to ~8h defrost in real time. This is the monitoring must-do: detect the change before the food cooks. (Roadmap item: Malcolm plus a vendor trial for OT monitoring.)
6. **Independent monitoring** — out-of-band temperature sensors / alarms that are *not* silenced by the same HMI during extended defrost.
7. **Hardwired thermal cutouts** on defrost heaters — last line for fire risk, not for food spoilage.

(5)–(7) limit blast radius after login. Detection (5) is what was missing between settings changed and morning spoil.

---

## Open gaps (honest)

- Entry path: **confirmed by ANSSI (21 Oct 2026)** as internet exposure plus weak passwords; Claroty flaws ruled out. No specific CVE named.
- Attribution: **unknown**.
- Audited nationwide loss total: **unknown** (press says millions of euros).
- Contractor vs retailer ownership of each modem/IP: **reported as the usual pattern; not forensically mapped store-by-store in this note**.

---

## Sources

- [CERTFR-2026-ALE-012](https://www.cert.cyber.gouv.fr/alerte/CERTFR-2026-ALE-012/)
- [SecurityWeek — refrigeration controller hack (13 Oct 2026)](https://www.securityweek.com/refrigeration-controller-hack-spoils-food-across-european-supermarkets/)
- [SecurityWeek — widens to Copeland / Dixell (15 Oct 2026)](https://www.securityweek.com/supermarket-refrigeration-attack-widens-to-copeland-dixell-controllers/)
- [Censys — Frostbitten in France](https://censys.com/blog/frostbitten-in-france-a-danfoss-storeview-internet-exposure-report/)
- [Cybersecurity Dive — thousands still exposed](https://www.cybersecuritydive.com/news/thousands-refrigerator-devices-exposed-french-cyberattack/833117/)
- [CISA ICSA-25-140-03 (Danfoss)](https://www.cisa.gov/news-events/ics-advisories/icsa-25-140-03)
- [AP News — Lescure ~700 stores](https://apnews.com/article/france-cyberattack-supermarkets-refrigeration-systems-78a813a8efd466b360655339f96083dc)
- [TIME — Le Monde on 1h→8h defrost](https://time.com/article/2026/10/13/france-cyber-attack-cybersecurity-refrigeration-supermarket-food-security/)
- [Cooling Post — Danfoss mitigation / EOL statement](https://www.coolingpost.com/world-news/danfoss-moves-to-mitigate-cyberattack/)
- [Public Sénat — ANSSI Senate testimony (21 Oct 2026)](https://www.publicsenat.fr/actualites/politique/des-defauts-de-securisation-elementaire-et-des-mots-de-passe-faibles-ont-permis-la-cyberattaque-contre-les-frigos-de-supermarches)
- [Cybersecurity Dive — flaws disclosed months before attacks](https://www.cybersecuritydive.com/news/refrigeration-france-cyber/832858/)
- [NBC News / Reuters — fires and spoiled food](https://www.nbcnews.com/world/europe/hackers-heat-french-supermarket-fridges-sparking-fires-spoiling-food-rcna603151)

---

## Related log

Session notes and quiz takeaways: [logs/2026-10-19.md](../logs/2026-10-19.md)
