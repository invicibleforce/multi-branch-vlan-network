# Troubleshooting notes

This file is the long-form version of the troubleshooting log in the README.
It captures the actual diagnostic process — including the dead ends — for
anyone studying the project or hitting similar issues.

---

## Issue 1: All PCs unable to ping anything

### Initial symptom

After completing all configuration on R1, R2, SW1–SW4 and verifying that
every show command looked correct, ping tests from any PC to any other PC
failed with `Request timed out` — even within the same VLAN.

### What I checked first (the dead ends)

1. **`show ip interface brief` on R1** — all sub-interfaces (Gi0/0.10
   through .40) showed `up/up`. Serial 0/3/0 was `up/up`. No problems.

2. **`show vlan brief` on SW1** — all four VLANs present, access ports
   correctly assigned (Fa0/1=VLAN 10, Fa0/2=VLAN 20, etc).

3. **`show interfaces trunk` on SW1** — Gi0/1 and Gi0/2 both trunking
   with VLANs 10,20,30,40 allowed. The trunk to R1 was correctly a trunk
   (this is a common silent failure point).

4. **`show ip route` on R1** — connected routes for all four VLANs plus
   the four static routes to Branch. Nothing missing.

5. **CDP neighbours visible everywhere** — physical layer was fine.

At this point I had verified every device-side configuration. I was
convinced the problem was in routing or trunking somewhere.

### The actual problem

Ran `show ip dhcp binding` on R1:

    R1# show ip dhcp binding
    IP address  Client-ID/Hardware address  Lease expiration  Type

Empty. No bindings. **No PC had ever requested an address.**

That meant the issue wasn't on R1 — it was on the PCs.

Clicked PC1 → Desktop → IP Configuration:

    IP Configuration: [Static]
    IP Address:       (blank)
    Subnet Mask:      (blank)
    Default Gateway:  (blank)

Every PC was in Static mode with no address.

### Fix

On each PC: Desktop → IP Configuration → tick **DHCP**.

Within a second, the PC pulled an address from R1's pool, and pings
started working immediately.

### Root cause

Packet Tracer defaults PCs to Static IP mode, not DHCP. The PCs were
never broadcasting DHCP Discover packets, so the router had no clients
to respond to.

### Lesson learned

**Check the client side before assuming the server side is broken.** An
empty `show ip dhcp binding` table is a definitive signal that no
clients have asked for an address. If no clients are asking, the
problem is upstream of the DHCP server.

My new default troubleshooting order for DHCP issues:

1. Is the client configured for DHCP? (one click in Packet Tracer; one
   `ipconfig /renew` on Windows)
2. Is the client actually broadcasting Discovers? (use Packet Tracer's
   Simulation mode, or Wireshark in real environments)
3. Is the router/server seeing the Discover? (`show ip dhcp binding`,
   or DHCP server logs)
4. If yes, but no address is being offered — then check pools, scope,
   exclusions on the server side.

---

## Issue 2: Serial interface stuck in "down/down" state

### Initial symptom

After applying `ip address 10.0.0.1 255.255.255.252` and `no shutdown`
on R1's Serial 0/3/0, IOS logged:

    %LINK-5-CHANGED: Interface Serial0/3/0, changed state to down

But never followed up with:

    %LINEPROTO-5-UPDOWN: Line protocol on Interface Serial0/3/0,
                          changed state to up

### Diagnosis

`show ip interface brief` confirmed the interface was `down/down` —
not `administratively down`, which would mean `no shutdown` hadn't
taken effect. So the interface was enabled, but the link wasn't coming
up.

Serial interfaces need three things to come up:
- Both ends configured with `no shutdown`
- A clock signal from the DCE end
- The cable physically connected

Ran `show controllers serial 0/3/0`:

    Interface Serial0/3/0
    Hardware is PowerQUICC MPC860
    DCE V.35, clock rate 2000000

R1 was the DCE end (it holds the female end of the cable in Packet
Tracer). The clock rate was at default — but more importantly, R2's
Serial 0/3/0 had not yet been `no shutdown`.

### Fix

1. On R1: applied `clock rate 64000` explicitly (good practice; the
   default works but explicit is better).
2. On R2: ran `no shutdown` on its Serial 0/3/0.

The link came up immediately:

    %LINEPROTO-5-UPDOWN: Line protocol on Interface Serial0/3/0,
                          changed state to up

### Lesson learned

When a serial link won't come up, check both ends before assuming a
configuration error. The LINK message alone doesn't tell you which end
is the problem — you need to look at both routers.

`show controllers serial X/X/X` is the fastest way to tell which end
is DCE. The DTE end never has a clock rate setting.

---

## Issue 3: Concerns about dashed lines between switches

### Initial symptom

After cabling everything, the links between SW1↔SW2 and SW3↔SW4
appeared as dashed lines rather than solid lines. Initial assumption
was that the links were down or broken.

### Diagnosis

The triangle indicators at each end of the dashed links showed green,
meaning the links were actually up.

Looked at Packet Tracer's cable colour conventions:

- **Solid line** = straight-through copper cable
- **Dashed line** = crossover copper cable
- **Solid red lightning** = serial cable
- **Dashed black** = console cable

The dashed appearance was just Packet Tracer showing the cable type
(crossover, used for switch-to-switch links by convention), not
indicating a problem.

### Lesson learned

In Packet Tracer:
- **Line style** (solid vs dashed) → cable type
- **Triangle colour** at each end → link state (green=up, amber=STP
  converging, red=down/admin-down, black=no protocol)

These two visual indicators are independent. A dashed line can be a
perfectly working link; a solid line can be administratively down.
Always read the triangle colour for link status.