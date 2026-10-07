# My BGP Questions & Answers — Networking Journal

**Author:** SGV Prasad  
**Purpose:** Personal BGP learning journal  
**Style:** My actual questions → technically correct answer → key takeaway

> **Note:** The questions in this document are based only on questions I actually asked during my BGP discussions. The wording is preserved as closely as possible; minor grammar cleanup is used only where needed for readability.

---

## 1. BGP Fundamentals & Design

### Q1. Why was BGP designed as a path-vector protocol?

**Answer:**

BGP was designed as a path-vector protocol because inter-domain routing needs more than simply knowing the shortest path.

Different autonomous systems need to apply **policy**, and they also need a mechanism for **loop prevention**.

By carrying path information such as `AS_PATH`, BGP can:

- Record which ASes a route has traversed.
- Detect AS-level routing loops.
- Apply policy based on the path.
- Select among multiple paths according to attributes.

**Key takeaway:**

> BGP is not simply trying to find the shortest path. It is designed for **policy + path information + loop prevention** between autonomous systems.

---

## 2. BGP Neighbor Formation & Routing Table Exchange

### Q2. Once BGP peers form a neighbor relationship, they share their full routing table ???

**Answer:**

Not exactly.

After the BGP session reaches **Established**, the peers exchange the BGP routes that they are permitted and able to advertise for the relevant address family.

It is not:

> "Send everything in my routing table."

BGP only advertises routes according to:

- BGP best-path rules
- Export policy
- Address-family activation
- iBGP/eBGP rules
- Route filtering
- Other BGP policy

So the initial exchange can be large, but it is the **BGP routing information eligible for advertisement**, not the entire IP routing table.

**Key takeaway:**

> BGP exchanges **eligible BGP routes**, not the complete RIB/FIB blindly.

---

### Q3. By default BGP full mesh is expected ?

**Answer:**

For traditional **iBGP**, yes, a full mesh was the original requirement.

The reason is the iBGP split-horizon rule:

> A route learned from one iBGP peer is not normally advertised to another iBGP peer.

Therefore, if there are three routers:

```text
R1 ---- R2
 \      /
   \  /
    R3
```

all three need direct iBGP relationships in the traditional design.

This becomes difficult to scale.

---

### Q4. full mesh is not mandatory right

**Answer:**

Correct.

Full-mesh iBGP is **not mandatory** in modern networks.

Two major scaling mechanisms are:

- Route Reflectors
- BGP Confederations

Route Reflectors are particularly common.

Instead of:

```text
R1 <----> R2
 |  \    / |
 |   \  /  |
 |    \/   |
R3 <----> R4
```

we can use:

```text
       RR
     / | \
   R1  R2  R3
```

The RR reflects routes between its clients according to route-reflection rules.

**Key takeaway:**

> Full mesh is the traditional solution. Route Reflection is the common scalable solution.

---

## 3. BGP OPEN Message & eBGP

### Q5. subnet mask is not there in OPEN msg ..Even for eBGP there is no restriction to be on same subnet ??

**Answer:**

Correct.

The BGP OPEN message does **not** contain an IP subnet mask used to determine whether the peers are on the same subnet.

BGP establishes its session over **TCP port 179**.

For directly connected eBGP peers, they are commonly in the same subnet, but BGP itself does not have a universal rule saying:

> "eBGP peers must always be in the same subnet."

eBGP peers can be multiple hops away when the design uses features such as `ebgp-multihop`.

What matters is that TCP connectivity to the configured neighbor exists and the BGP session parameters are acceptable.

**Key takeaway:**

> BGP neighbor reachability is a TCP/IP connectivity issue; the OPEN message itself does not carry a subnet mask.

---

## 4. MP-BGP

### Q6. neighbor activate command is needed when we use MPBGP , Not when default unicast ipv4

**Answer:**

Yes, this is the important distinction in common Cisco-style configurations.

For traditional IPv4 unicast BGP, the IPv4 unicast address family is commonly activated by default depending on the platform/configuration model.

With MP-BGP, different address families such as:

```text
VPNv4
VPNv6
EVPN
IPv6 unicast
```

are explicitly activated for the neighbor.

For example, conceptually:

```text
router bgp 65000

 address-family vpnv4
   neighbor X activate
```

The TCP/BGP neighbor relationship and the address-family activation are separate concepts.

**Key takeaway:**

> A BGP session can exist while a particular address family is not activated for that neighbor.

---

# 5. BGP Path-Vector & Attributes

### Q7. Why is LOCAL_PREF preferred over AS_PATH?

**Answer:**

Because BGP's decision process is designed to allow **local policy to override path length**.

Consider:

```text
Path A:
LOCAL_PREF = 100
AS_PATH = 65010

Path B:
LOCAL_PREF = 200
AS_PATH = 65020 65030 65040
```

Even though Path B has a longer AS_PATH, Path B can be selected because LOCAL_PREF is considered before AS_PATH.

This is intentional.

An organization may say:

> "I prefer this provider/link regardless of the number of AS hops."

LOCAL_PREF provides that internal policy control.

**Key takeaway:**

> BGP is **policy driven**, not simply "shortest AS_PATH wins."

---

### Q8. What kind of attributes are originator ID , cluster list ?

**Answer:**

`ORIGINATOR_ID` and `CLUSTER_LIST` are **optional non-transitive BGP attributes** associated primarily with Route Reflection.

They are used for **loop prevention in route-reflector environments**.

They are not ordinary attributes used to describe the Internet path in the same way as AS_PATH.

---

## 6. BGP Attribute Classification

### Q9. nontransitive means ? Out of the AS it never goes right ?

**Answer:**

Broadly, yes.

A **non-transitive** attribute is not required to be propagated unchanged across an AS boundary to external BGP peers.

The important idea is:

```text
Transitive
    ↓
may be carried onward by BGP

Non-transitive
    ↓
does not have to be propagated onward
```

For interview purposes, don't interpret "non-transitive" as "the attribute can never appear outside the original AS under any possible implementation." The protocol's rule is about whether it is required/allowed to be propagated as a transitive attribute.

---

### Q10. well known , discretionary means ..Its shd be known by router ..But router need not pass it right ?

**Answer:**

Not quite.

These are two separate classification dimensions.

### Well-known

The attribute is recognized by all BGP implementations.

### Optional

The attribute does not have to be implemented by every BGP implementation.

Then we have:

### Mandatory

The attribute must be present in relevant BGP updates.

### Discretionary

The attribute does not have to be present in every applicable UPDATE.

So:

```text
Well-known + Mandatory
Well-known + Discretionary
Optional + Transitive
Optional + Non-transitive
```

are different concepts.

**Key takeaway:**

> "Well-known" is about whether all BGP implementations are expected to recognize it.  
> "Mandatory/discretionary" is about whether it must be present.

---

# 7. iBGP Split Horizon

### Q11. Why doesn't iBGP advertise routes learned from one iBGP peer to another?

**Answer:**

Because of the **iBGP split-horizon rule**.

Suppose:

```text
R1 ---- R2 ---- R3
 \              /
    same AS
```

If R2 learns a route from R1 via iBGP, R2 normally does not advertise that route to R3 via iBGP.

Why?

Because AS_PATH is not changed as a route travels between iBGP peers.

Without the rule, routes could circulate inside the AS without the same AS-level loop protection that eBGP gets from AS_PATH modification.

The solution was traditionally:

> Every iBGP speaker should have a direct iBGP relationship with every other iBGP speaker.

That is the origin of the traditional full-mesh requirement.

**Key takeaway:**

> The full-mesh requirement exists **because of the iBGP split-horizon rule**.

---

### Q12. BGP requires full mesh due to split horizon rule ? OR the vice versa ..I wanna understand which came 1st

**Answer:**

The logical order is:

```text
iBGP split-horizon rule
        ↓
iBGP-learned route cannot be advertised to another iBGP peer
        ↓
every iBGP speaker needs to learn routes directly
        ↓
traditional full-mesh requirement
```

So the **split-horizon rule came first conceptually**.

Full mesh is the consequence/design solution, not the reason for the split-horizon rule.

Later, Route Reflection and Confederations provided scalable alternatives.

**Key takeaway:**

> **Split horizon → full mesh requirement.**

---

# 8. Route Reflectors

### Q13. how dual route reflector works in ibgp?

**Answer:**

A common design uses two Route Reflectors for redundancy.

For example:

```text
             RR1
            / | \
           /  |  \
          R1  R2  R3
           \  |  /
            \ | /
             RR2
```

Clients can establish iBGP sessions to both RR1 and RR2.

If one RR fails, the other can continue distributing routes.

The important point is that the two RRs need to be designed so that their route-reflection behavior and loop-prevention mechanisms work correctly.

**Key takeaway:**

> Dual RRs provide **control-plane redundancy**; clients do not depend on a single RR.

---

### Q14. what kind of attributes are originator ID , cluster list ?

**Answer:**

They are optional non-transitive attributes used for **Route Reflection loop prevention**.

- `ORIGINATOR_ID` identifies the router that originally introduced the route into the route-reflection system.
- `CLUSTER_LIST` records the Route Reflector cluster IDs through which the route has been reflected.

---

### Q15. So cluster ID is ID of the RR group ? Or onlr RR ?.. And a client in 1 cluster can be RR in another group ?

**Answer:**

A Cluster-ID identifies a **Route Reflector cluster**.

It is associated with the RR cluster rather than meaning simply "the ID of one RR."

A router can participate in different route-reflector relationships and can itself act as an RR in another relationship/design.

So RR/client roles are based on the configured peering relationships and are not a permanent property of a router.

**Key takeaway:**

> Think of Cluster-ID as identifying the **RR cluster**, not simply "the RR's router ID."

---

### Q16. cluster ID is only for avoiding loops ?

**Answer:**

Its primary purpose in Route Reflection is **loop prevention**.

When an RR reflects a route, its Cluster-ID can be added to the Cluster-List.

If the route later returns to an RR that sees its own Cluster-ID in the list, the RR can identify the reflection loop and reject the route.

---

# 9. BGP Path Selection

### Q17. What happens if two paths have the same attributes?

**Answer:**

BGP continues through the remaining decision criteria/tie-breakers.

If the important path attributes are equal, later criteria such as:

- eBGP vs iBGP
- IGP cost to the next hop
- Router ID
- Neighbor address

may determine the winner, depending on the implementation and configuration.

If the platform supports BGP multipath and the paths satisfy the required conditions, multiple paths may instead be installed.

**Key takeaway:**

> Equal visible attributes do not necessarily mean BGP has no way to choose. It continues through tie-breakers.

---

# 10. AS_SET and Summarization

### Q18. BGP attributes msgs FSM and RR concepts & AS set summarization dampening etc are enuf as fundamentals?

**Answer:**

For BGP fundamentals, those areas provide a strong foundation:

```text
BGP messages
BGP FSM
BGP attributes
Path selection
iBGP/eBGP
iBGP split horizon
Route Reflection
AS_SET / aggregation
Route dampening
```

For a networking interview, however, understanding **why** each mechanism exists is more valuable than memorizing only definitions.

For example:

```text
Why full mesh?
Why Route Reflector?
Why AS_PATH?
Why LOCAL_PREF before AS_PATH?
Why Cluster-List?
Why next-hop-self?
```

Those reasoning questions demonstrate deeper protocol understanding.

---

# 11. BGP / MP-BGP / EVPN Questions

### Q19. type 2, vni info not present ... ?

**Answer:**

VNI information should not be thought of as a universal field that appears in every EVPN route.

EVPN routes carry control-plane information defined for their particular route type.

The VXLAN VNI is associated with the VXLAN forwarding context/data plane, while EVPN control-plane routes provide the information needed for VTEPs to construct forwarding state.

---

### Q20. type 2 update will be sent to all peers , irrespective of their vni ?

**Answer:**

BGP distribution is based on BGP peering, address-family activation and routing policy.

It is not simply:

```text
same VNI → send
different VNI → don't send
```

An EVPN UPDATE can be received by an EVPN BGP peer, after which the receiving VTEP determines whether and how the route is relevant to its local EVPN/VXLAN configuration and policy.

**Key takeaway:**

> BGP peering/distribution and VNI participation are separate concepts.

---

### Q21. Vni comes in data plane only then ?

**Answer:**

The VNI is fundamentally associated with the **VXLAN data-plane forwarding context**.

The EVPN control plane distributes reachability information that allows VTEPs to build the appropriate forwarding state.

So it is useful to separate:

```text
BGP EVPN
   ↓
Control plane: "How/where is this endpoint or prefix reachable?"

VXLAN
   ↓
Data plane: "Which VNI/tunnel/encapsulation should carry the packet?"
```

---

### Q22. Vni is not present in any route in evpn

**Answer:**

The important correction is that this statement is too absolute.

VNI-related information can be associated with EVPN route encoding/attributes depending on the route type and implementation. What is important is that **VNI is not a universal field that appears identically in every EVPN route type**.

For interview purposes, avoid saying:

> "VNI never appears in EVPN routes."

Instead say:

> "VNI is not a universal field in every EVPN route type; EVPN route types carry different control-plane information, while VNI represents the VXLAN forwarding context."

---

### Q23. Mp bgp evpn routers send the update to all other mp bgp routers irrespective of their vni ?

**Answer:**

The BGP control plane distributes EVPN routes according to the BGP session, address family and policy.

It is not necessary for the sending router to establish a separate BGP session for every VNI.

The receiving VTEP processes the EVPN information according to its local EVPN/VXLAN configuration.

---

### Q24. Vni is data plane...so it need not be present in ctrl plane ..as it's not found in any evpn route types msgs

**Answer:**

The first part is a useful mental model, but the conclusion is too strong.

Yes, VNI represents a VXLAN forwarding context in the data plane.

However, the EVPN control plane needs enough information to associate advertised reachability with the relevant EVPN/VXLAN service context. Therefore, it is not correct to conclude that the control plane has no VNI-related information simply because VNI is primarily a data-plane identifier.

The safest interview explanation is:

> **VNI is a VXLAN data-plane identifier, while EVPN/BGP is the control plane that distributes the reachability information required to build the VTEP forwarding state.**

---

# 12. BGP Questions to Continue Adding

This section is intentionally left open.

Future entries should be added in the same format:

```text
### Q. <My actual question>

**Answer:**

<technical explanation>

**Key takeaway:**

<one-line mental model>
```

The journal should remain a record of **actual questions I asked**, rather than becoming a generic BGP study guide.

---

# My BGP Mental Models

## 1. Full Mesh

```text
iBGP split horizon
        ↓
iBGP route cannot be passed iBGP → iBGP
        ↓
traditional full mesh
        ↓
scaling problem
        ↓
Route Reflectors / Confederations
```

## 2. BGP Path Selection

```text
Policy
  ↓
LOCAL_PREF
  ↓
AS_PATH
  ↓
ORIGIN
  ↓
MED
  ↓
other tie-breakers
```

The important lesson:

> **BGP is policy driven, not simply shortest-path driven.**

## 3. Route Reflection

```text
Normal iBGP
    ↓
iBGP → iBGP advertisement blocked

Route Reflector
    ↓
controlled reflection
    ↓
Originator-ID + Cluster-List
    ↓
loop prevention
```

## 4. MP-BGP

```text
BGP session
    ↓
Address Family
    ↓
IPv4 / IPv6 / VPNv4 / EVPN / etc.
```

A BGP session can exist while a particular address family is not activated.

---

# Changelog

### 2026-10-07

Rebuilt this journal to follow the user's requirement:

- Only user-originated BGP questions.
- No generic assistant-created interview questions.
- User wording preserved wherever available.
- Answers added below each question.
- Focus on the actual doubts raised during BGP discussions.
