# Port Analysis — Ubuntu Wazuh Server

This table compares services listening locally on the Ubuntu server with ports that were reachable from the Windows endpoint using Nmap.

| Port | Seen in `ss`? | Seen in Nmap? | Expected? | Purpose |
|---|---|---|---|---|
| 22/tcp | Yes | Yes | Yes | SSH remote administration |
| 443/tcp | Yes | Yes | Yes | Wazuh Dashboard over HTTPS |
| 1514/tcp | Yes | No | Yes | Wazuh agent event forwarding |
| 1515/tcp | Yes | No | Yes | Wazuh agent enrollment |
| 55000/tcp | Yes | No | Yes | Wazuh API |
| 9200/tcp | Yes | No | Yes | Wazuh Indexer API |
| 9300/tcp | Yes | No | Yes | Wazuh Indexer internal communication |

## Analyst Notes

The `ss -tulnp` command showed services listening locally on the Ubuntu server.

The Nmap scan from the Windows endpoint only showed ports 22 and 443 as reachable.

This means some services were listening on the Ubuntu server but were not reachable from the Windows endpoint through the network. This is important because local listening ports and externally reachable ports are not always the same.

From an analyst perspective, ports 22 and 443 were expected because SSH is used for remote administration and HTTPS is used for the Wazuh Dashboard.
