# Reference — Decision Guides

Agent reference for technology selection decisions in Talos Linux Kubernetes clusters. Each guide provides structured comparison of alternatives with concrete recommendations per use case.

---

## How to Use

1. **Identify the decision point** — what the user needs to choose (CNI, CSI, topology, etc.)
2. **Open the relevant guide** and walk through it with the user
3. **Match their requirements** against the comparison tables
4. **Follow the recommendation** for their specific use case
5. **Note constraints** — some decisions are bootstrap-time only and cannot be changed later

---

## Guide Template

Every decision guide follows a consistent structure:

1. **Context** — Why this decision matters, when it must be made, and whether it can be changed later.
2. **Options** — Each option described with its strengths, weaknesses, and Talos-specific considerations.
3. **Comparison Table** — Side-by-side feature comparison across all options.
4. **Recommendation by Use Case** — Specific recommendation for common scenarios (homelab, production, edge, etc.).
5. **Migration Path** — How to move between options if the initial choice needs to change.

---

## Available Guides

| Guide | Decision | Must Decide By |
|-------|----------|----------------|
| [CNI Selection](./cni-selection.md) | Container Network Interface (Cilium, Flannel, Calico) | Bootstrap time (permanent) |
| [CSI Selection](./csi-selection.md) | Container Storage Interface (Ceph, Longhorn, Local, etc.) | Post-bootstrap (changeable) |
| [Cluster Topology](./topology.md) | Number of nodes, roles, sizing | Cluster creation (expandable) |
| [HA Endpoint Strategy](./ha-endpoint.md) | Control plane endpoint HA (VIP, LB, DNS) | Bootstrap time (changeable with effort) |
| [GitOps Tooling](./gitops-tooling.md) | Flux vs ArgoCD | Post-bootstrap (changeable) |
