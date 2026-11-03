# Troubleshooting Notes

Problems found while building and testing the network, how each was diagnosed, and what it taught. Each entry follows the same pattern: symptom → diagnosis → fix → lesson.

---

## 1. A router port wouldn't accept a copper cable

- **Symptom:** When cabling HQ-R1 to HQ-CORE, port Gi0/0/2 didn't appear in the port list, even with a copper cable selected.
- **Diagnosis:** Port labels showed Gi0/0/0 and Gi0/0/1 in use and Gi0/0/2 free. A free port that won't accept copper is an empty SFP slot.
- **Fix:** Powered the router off, installed a GLC-T (copper SFP) transceiver, powered it on, and cabled Gi0/0/2 ↔ Gi1/0/1.
- **Lesson:** Many router and switch ports are SFP slots; the transceiver decides copper or fiber.

## 2. Server unreachable from another VLAN

- **Symptom:** HQ-PC1 could reach its gateway and the Guest VLAN, but not HQ-SRV (10.1.30.10).
- **Diagnosis:** The successful pings ruled out the PC, its access switch, the trunk and inter-VLAN routing, leaving the server side. `ipconfig` on the server showed 0.0.0.0: no address assigned.
- **Fix:** Set the static address 10.1.30.10/24, gateway 10.1.30.1. Confirmed with `show mac address-table` on HQ-ACC2 that the server's MAC was on Fa0/21 in VLAN 30.
- **Lesson:** Use what works to eliminate possibilities; confirm Layer 2 placement with the MAC address table.

## 3. "I can reach my gateway but nothing beyond it"

- **Symptom:** BR-PC1 pinged its gateway (10.2.10.1) but not BR-GUEST1 in another VLAN.
- **Diagnosis:** BR-GUEST1 could ping its own gateway, so the router and the trunk were fine. `ipconfig` on BR-PC1 showed default gateway 0.0.0.0.
- **Fix:** Set the default gateway to 10.2.10.1. The ping then succeeded with TTL 127.
- **Lesson:** A host reaches its own subnet without a gateway; the gateway only matters for traffic leaving the subnet.

## 4. OSPF neighbor in 2WAY instead of FULL

- **Symptom:** Right after enabling OSPF, HQ-R1 showed HQ-CORE in 2WAY/DROTHER, and BR-R1 was missing the HQ VLAN routes.
- **Diagnosis:** A branch-to-HQ ping had already succeeded, which needs a full adjacency. The output had been captured during OSPF's startup wait (DR/BDR election).
- **Fix:** No config change. Re-ran the commands: both neighbors FULL, all 4 HQ routes present.
- **Lesson:** Routing protocols need time to converge; take a second reading before changing anything.

## 5. Empty NAT translation table

- **Symptom:** PCs reached the internet server, but `show ip nat translations` was empty.
- **Diagnosis:** The ISP has no route back to 10.x addresses, so replies could only return if NAT had translated them. ICMP translations expire after about a minute.
- **Fix:** Used `show ip nat statistics`, which keeps running counters: 16 translations created, 14 hits.
- **Lesson:** Live tables show the present; counters show history. Choose the command that fits the question.

## 6. STP root bridge: right result, wrong reason

- **Symptom:** `show spanning-tree vlan 10` said HQ-CORE was the root, as intended.
- **Diagnosis:** Its priority was 32778, the default (32768 + VLAN 10). With all switches tied on priority, HQ-CORE had only won because it had the lowest MAC address. Any new switch with a lower MAC could have taken over.
- **Fix:** `spanning-tree vlan 10,20,30,99 root primary`. Priority became 24586, so HQ-CORE is root by design.
- **Lesson:** Check *why* something works, not just *that* it works.

## 7. Configuration audit caught a banner error

- **Symptom:** None visible in normal operation.
- **Diagnosis:** Checking the exported configs found the ISP router's login banner read "Disconnect now if you are authorized", missing "not". Earlier checks had sampled devices rather than covering all of them.
- **Fix:** Re-entered the banner, saved, re-exported the config.
- **Lesson:** Sampling is quick but misses things; audit every device when it's cheap to do so.
