# Scenario: HA routers on Linux Bridge using VXLAN tunnels

This configuration provides full L3 HA while using the simpler Linux Bridge environment. A minimum of two network nodes are required to provide HA fail over in the event that one network node fails.

Note: VXLAN tunnels require L2Population, due to bug -  ???? HA routers and L2Population do not work correctly. This bug is expected to be fixed in the Kilo release.

System environment:

![Neutron HA Router on Linux bridge Environment](images/netlha.eps "Neutron HA Router on Linux bridge Environment")

## Configuration:
