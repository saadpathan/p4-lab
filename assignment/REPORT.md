# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4077**

## Lab wrap-up answers

1. Two changes that can be made without recompiling are adding or modifying table
   entries and changing action parameters such as output ports. Changes that need a
   new program include changing header definitions or parser states, and adding or
   changing actions or control logic.

2. The packet reached h3 because its tunnel `dst_id` was 3, and s2's tunnel table
   mapped exit 3 to port 3. It still carried the tunnel header because the
   deparser emitted the valid `myTunnel` header.

3. Tunnel forwarding uses `myTunnel_forward`, which does not decrement the IPv4
   TTL, so the packet arrived with TTL 64. Normal IPv4 forwarding decrements the
   TTL at each switch, which is why the ping replies had TTL 62.

4. The compiler caught syntax errors such as the extra closing brace. It did not
   catch the missing tunnel-header emission: that was valid P4 but produced a
   malformed packet. This shows that P4 programs require packet and switch-log
   testing in addition to compilation.

## 1. Analysis: the loop

I dumped s2's `MyIngress.myTunnel_exact` table and changed the entry for
`dst_id` 3 (handle `0x2`) from output port 3 to port 1. I then sent
`10.77.0.3 "loop" --tunnel 3` from h1. The counts in
`assignment/evidence/loop-before.txt` increased from s1=315 and s2=313 to
s1=804 and s2=801 after five seconds, confirming that the packet circulated
between the switches.

The tunnel path uses `myTunnel_forward`, which does not decrement IPv4 TTL,
so IPv4 TTL does not stop this tunnel loop. Although `ipv4_forward` decrements
TTL, that action is only used by `ipv4_lpm`, not by the tunnel table. Therefore
the packet keeps looping until the network is stopped.

## 2. Design: the hop limit

I added an 8-bit `hop_limit` field after `dst_id` in the tunnel header.
Eight bits allows a limit from 0 to 255 while adding only one byte, and
putting it after the existing routing fields keeps the destination information
together.

The sender used an initial value of 4. The normal h1-to-h2/h3 paths need
two switches, while 4 is large enough to demonstrate forwarding through the
network without making a loop run for too long.

The tunnel-forwarding action checks the incoming value before forwarding.
If it is zero, the switch drops the packet. Otherwise it decrements the field
by one and forwards it. Thus a packet with limit 1 may cross one more switch,
then arrives with limit 0; a looping packet is eventually dropped at the next
switch.

The parser, deparser, `send.py`, and `receive.py` all use the same field
order and width. The sender sets the initial value, and the receiver prints
the value that arrives so the forwarding behavior can be verified.

## 3. Implementation

I added the 8-bit field to the P4 and Scapy tunnel headers in the same order.
The tunnel forwarding action drops a packet whose limit is zero; otherwise it
decrements the limit before selecting the egress port. `send.py` accepts
`--hops` and defaults it to 4, while `receive.py` decodes and prints the
remaining value. Plain IPv4 packets remain unchanged.

## 4. Evidence: it works

The evidence in `assignment/evidence/delivery.txt` shows tunnelled delivery
to both h2 and h3. Both receivers reported `hop_limit=2`, because the
starting value was 4 and the packets crossed two switches. The same file
shows that plain IPv4 still works: both pings to h3 succeeded with TTL 62.

The evidence in `assignment/evidence/loop-after.txt` shows the redirected
tunnel loop being forwarded four times, twice through each switch, and then
dropped on the fifth switch processing event when the incoming hop limit was
zero. This matches the design.

## 5. Evaluation

### Mixed rollout prediction (before running it)

With s1 running the new program and s2 running `tunnel_v1.p4`, I predict that
s2 will extract `proto_id` and `dst_id` from the first four tunnel bytes but
will not extract the new `hop_limit` byte. It will therefore begin parsing
IPv4 one byte too early, treating the sender's hop-limit byte as the first
IPv4 byte. However, because the old program still sees a valid tunnel header,
I predict that it will use its old tunnel table and forward the packet rather
than perform an IPv4 lookup. Because s2's old deparser also emits only the old
four-byte tunnel header, I predict that h2 will receive a one-byte-shifted,
malformed packet. For a loop packet, s2 will forward according to its old
tunnel fields rather than consume the new hop limit, so only the upgraded
switches in the loop will enforce the new limit.

### Safe rollout of a header change

The safest option is a versioned, dual-format rollout. I would first install
code that can parse and forward both the old and new header versions, selecting
the format with a version field or a distinct EtherType. I would then deploy
new senders and the new format, verify that all paths handle it, and only
after that remove old-format support. During the transition, an old switch
would continue to handle old packets, while an upgraded switch would not
silently interpret a new packet with the old field offsets.

A distinct EtherType is simpler operationally: old switches treat the packet
as an unknown protocol and can drop it rather than misparse it, while upgraded
switches select the new parser state. A version field is more space-efficient
when the EtherType is already fixed and supports several revisions under one
protocol, but every transitional switch must parse far enough to identify the
version. Keeping fields in an old-compatible order could avoid either change,
but it is only safe when added fields are after the data old switches consume
and old switches never need to forward packets with a changed total layout.
This experiment shows why simply inserting a byte is unsafe.

### Data-plane versus control-plane loop protection

The hop limit is a data-plane safeguard: every switch can enforce a finite
per-packet budget at line rate, so it catches forwarding loops caused by wrong
or stale table entries, including loops that arise after a topology change. It
does not prove that the configured network is loop-free, and an old or
incompatible switch may ignore or misparse the field.

Week 3's `check_paths()` is a control-plane graph check. It can detect a cycle
or an unreachable destination in the configured topology before traffic is
sent, and it can reason about all configured paths rather than waiting for a
packet. It cannot catch runtime-only faults such as a bad table update, a
failed link causing a temporary reconvergence loop, or a packet whose
forwarding state differs from the controller's model. Real networks use both:
`check_paths()` prevents known configuration errors, while the hop limit bounds
the damage from errors and changes that escape control-plane validation.

### Cost on a line-rate hardware switch

The change adds one byte to each tunnel header and requires the parser and
deparser to handle that byte. The forwarding action adds an 8-bit zero test,
an 8-bit decrement, and a conditional drop decision. These are small fixed
width operations that can normally be implemented in existing parser,
match-action, and arithmetic resources without changing the IPv4 pipeline.
The main costs are one byte of packet bandwidth and buffer/storage usage per
tunnel packet, a few bits of metadata/state in the pipeline, and a small
amount of power and area for the extra logic. It should not add a per-packet
software cost or reduce line rate unless the target hardware is already at a
parser or ALU resource limit.

## AI use

I used Copilot to help explain the BMv2 logs, review the mixed-version packet
behavior, and draft parts of the report. I implemented and tested the hop-limit
changes, packet delivery, loop prevention, and mixed-rollout experiment myself.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with the finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
