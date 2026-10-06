#Why RPF needed in PIM ?, When the PIM Tree with *PIM states/signalling* is Built to be loop free already 

PIM Tree or the PIM Forwarding states are in *Control Plane*.
In *Data Plane* While the Actual data packets still needs a check while forwarding.
The PIM tree defines downstream interfaces (OIL — Outgoing Interface List). But a router can receive the same multicast stream on multiple interfaces — especially during:
	• SPT switchover (from RPT to Shortest Path Tree)
	• Topology changes / reconvergence
	• Redundant links
  Without RPF, the router has no way to know which incoming copy is legitimate
	•To AVoid Transient Loops During Convergence
   Even if the steady-state tree is loop-free, during link failures or routing reconvergence, packets can temporarily arrive on unexpected interfaces. 
    RPF acts as a real-time sanity check on every packet, not just at tree-build time.
 •Misdelivered Traffic
  Without RPF, a multicast packet injected or reflected from the wrong part of the network could be forwarded endlessly — RPF is the data-plane gate.
Think of the PIM tree as a road map — it tells you which roads exist. RPF is the checkpoint at each intersection that verifies traffic is actually coming from the right direction, not just that the road exists. A correct map doesn't eliminate the need for directional validation at runtime.

Summary
The PIM tree being loop-free is a topological guarantee (control plane). RPF is a per-packet forwarding correctness check (data plane). 
They solve two different problems — one builds the right structure, the other ensures only legitimate traffic flows through it.
PIM Tree   →   Builds the correct highway system
RPF Check  →   Traffic police at every on-ramp, 






