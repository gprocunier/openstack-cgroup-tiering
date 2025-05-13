
Handling Hardware Generations in the Gold/Silver/Bronze Tiering Model
===

Context
-------
*   The proposed model ties service‑tier fairness to *per‑host* Placement inventories (`CUSTOM_VCPU_GOLD/SILVER/BRONZE`) and flavour CPU‑share weights.
*   Every host advertises the same 1:1:1 domain capacity (e.g. 112 : 112 : 112)
  so that *each* tier can consume at most one physical core’s worth of load under full contention.
*   In 1–3 years you will add denser nodes (more cores / RAM).  Old hardware will
  eventually be retired.

Primary goals for lifecycle management
--------------------------------------
1. **Keep deterministic SLOs** — a Gold vCPU always gets ≈50 % of a core on
   *whatever* host it lands.
2. **Minimise operational friction** — avoid bulk cold migrations when possible.
3. **Keep the flavour catalogue understandable** — no explosion of nearly
   identical flavours.

Key decision axis
-----------------
| Question                                | Option A                               | Option B                          |
|-----------------------------------------|----------------------------------------|-----------------------------------|
| How to expose bigger per‑host pools?    | Increase each new host’s inventory for `CUSTOM_VCPU_*` in Placement (e.g. 160/160/160 instead of 112/112/112). **Flavours stay unchanged.** | Create new *generation* traits (or aggregates) and generation‑specific flavours (`gold_v2_xlarge`) that are scheduled only to Gen‑2 hosts. |
| Can old flavours land on new hosts?     | Yes — they just consume fewer of the larger pool. | Only if you let the flavour omit the “generation” trait; otherwise they’ll stay on Gen‑1. |
| Operational churn when retiring Gen‑1   | None for VMs that are already on Gen‑2.  Drain Gen‑1 hosts as they empty. | Must *resize* or migrate VMs that still use Gen‑1 flavours (e.g. `gold_v1_xlarge`) before you can retire hosts. |
| User cognitive load                     | One set of tiered flavours for *all* time; hardware difference is invisible. | Users must understand v1 vs v2 flavour lines. |
| Scheduler complexity                    | None beyond bigger inventory numbers.   | Need traits / aggregate filters in every generation flavour. |

Recommended baseline
--------------------
### 1. **Keep a single flavour line per tier/size**  
*Rely on heterogeneous Placement inventories rather than flavour per generation.*

*   New compute nodes advertise larger inventory numbers (e.g. 160 Gold units
    instead of 112).  That is a one‑line change in each node’s `provider.yaml`.
*   Flavours never change; a `gold.xlarge` that requests 4 `CUSTOM_VCPU_GOLD`
    works everywhere.
*   SLOs remain deterministic per host because the **ratio** of Gold:Silver:Bronze
    slots on every host is still 1 : 1 : 1.  A Gold vCPU never competes with more
    than one peer Gold vCPU for its 50 % share on *that* host.

### 2. **Capacity‑planning guard‑rails**
*   *Head‑room:* Maintain ≤ 90 % allocation of any custom resource cluster‑wide
    so you can live‑migrate off a host for maintenance even when tiers are full.
*   *Drain procedure:* Before hardware retirement set the host’s inventories to
    **0** (or put it in maintenance); Placement will live‑migrate / evacuate the
    residual VMs onto newer hosts while still respecting per‑host limits.

### 3. **Optional generation traits for feature gaps**
If new CPUs add an instruction set (AVX‑512, AMX) that *some* workloads need:
*   Tag those hosts with `TRAIT_CPU_GEN2`, create *opt‑in* flavours that
    `required=TRAIT_CPU_GEN2`.  Only workloads that care will use them.
*   Everyone else keeps using the tiered “gen‑agnostic” flavours.

Enhancements & alternatives
---------------------------
1. **Soft‑drain with Placement ratios**  
   Instead of hard 112/112/112 on Gen‑1 and 160/160/160 on Gen‑2 you can slowly
   *decrease* Gen‑1 Gold capacity (e.g. to 80) to encourage new workloads onto
   Gen‑2 while old ones age out naturally.  Placement rejects new Gold VMs on
   Gen‑1 once below 80 free units; no user impact.

2. **Watcher‑assisted “retire hosts” audit**   
   A custom strategy can gradually move VMs off least‑capable hosts, shrink their
   inventories to zero, and mark the node for decommission.

3. **Flavour aliasing via metadata**  
   If UX absolutely requires “v2 sizing”, keep a *single* internal flavour and
   expose Horizon names like “Gold v2” via the `description` field rather than
   separate IDs.  Avoids Placement double‑maintenance.

4. **Memory‑dense nodes**  
   If new nodes also have higher RAM‑per‑core, expose bigger `MEMORY_MB` totals
   (and adjust `ram_allocation_ratio` locally).  Existing flavours transparently
   benefit — nothing extra is needed as long as you keep the same Gold/Silver/
   Bronze share split.

Action checklist
----------------
1. **Automate provider‑inventory generation** so each new host publishes the
   correct `CUSTOM_VCPU_*` totals on initial boot.
2. **Document a host‑retirement run‑book**: set inventories to zero → evacuate →
   remove from scheduler/cluster.
3. **Maintain head‑room**: alert when any tier > 90 % cluster utilisation.
4. **Keep flavours immutable**; never mutate v1 to v2 in place — create new
   flavours only if absolutely required for feature traits.
5. **Track CPU feature divergence** with Placement traits, not separate
   tier‑pools, unless the feature materially changes the SLO maths.

Bottom line
-----------
*Heterogeneous Placement inventories* give us “buy‑new‑hardware‑and‑it‑just‑works”
semantics: old flavours, new cores, same deterministic fairness.  We only introduce
generation‑specific flavours/traits when we truly need to expose a hardware
capability difference, not merely core count.

