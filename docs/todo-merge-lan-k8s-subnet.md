# TODO: Merge LAN and k8s network into 10.1.0.0/23

## Current state
The LAN (10.1.0.0/24: clients, TrueNAS) and the k8s network (10.1.1.0/24) share one L2 segment but are separate subnets, so traffic between them goes through the router.
The router does Source NAT in both directions between them, so replies take the same path as requests.
Without it, routing was asymmetric: small requests worked, but larger transfers stalled (Immich uploads hung after ~1 MB and ended in a 502).

The downside: all traffic between the two networks, including NFS, takes a detour through the router, and Immich/Traefik only see the router's IP as client.

## Target
One subnet 10.1.0.0/23, so hosts reach each other directly and the router only handles other networks and the internet.

1. Router: set the LAN interface to /23 (keep both gateway IPs, so hosts keep their gateway) and the DHCP subnet mask to 255.255.254.0.
2. Every host with a static IP (prod and rancher nodes, TrueNAS): change the prefix to /23; IPs and gateways stay. DHCP clients pick it up on lease renewal.
3. Verify from both halves, including prod node → TrueNAS: `ip route get <ip-in-other-half>` shows no `via`, `tracepath -n <ip-in-other-half>` shows one hop.
4. Remove the Source NAT rules and test an Immich upload. If it hangs again, a host is still on /24 or has a static route through the router.

No repo change is needed: node IPs stay, and both MetalLB pools (10.1.0.235-245, 10.1.1.x) lie inside the /23. DHCP ranges must not overlap the pools.
