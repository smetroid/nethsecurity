# ns-dpi

Manage network traffic using DPI on network flows.

How it works:
- Netify flow actions plugin adds a label to matching connections
- nft rules can block or change priority (`dscp`) to connections with labels

To enable traffic processing:
- configure `dpi` UCI database (see below for an example)
- enable DPI service:
  ```
  uci set dpi.config.enabled=1
  uci commit dpi
  /etc/init.d/dpi restart
  service netifyd reload
  ```

Global options:

- `enabled`: can be `0` or `1`, if set to `1` enable the service
- `log_blocked`: can be `0` or `1`, if set to `1` blocked connections will be logged
- `firewall_exemption`: can be `0` or `1`, if set to `1` all firewall IP addresses will be
  added to global exemption list and will not match DPI rules

Rule options:

- `criteria`: DPI expression to match the traffic
  - the criteria must terminate with `;` when using complex expressions
  - use the `"` symbol to enclose strings, double-qoutes will be then translated to `'` inside the plugin configuration file (`/etc/netify.d/netify-flow-actions.json`)
- `action`: valid actions are:
  - `block`: matching traffic will be blocked
  - `bulk`: matching traffic will be moved to low priority QoS class named `Bulk`
  - `best_effort`: matching traffic will be moved to average priority QoS class named `Best Effort`
  - `video`: matching traffic will be moved to high priority QoS class named `Video`
  - `voice`: matching traffic will be moved to very high priority QoS class named `Voice`
- `description`: an optional rule description
- `enabled`: can be `0` or `1`, if set to `1` the rule will be enabled

All enabled rules are always evaluated by netifyd, the rule order doesn't matter.

Example of `/etc/config/dpi`:
```
config main 'config'
	option log_blocked '1'
	option enabled '1'
	option firewall_exemption '1'

config rule
	option action 'block'
	option criteria 'ai:netify.twitter'
	option description 'Block Twitter for everyone'
	option enabled 1

config rule
	option action 'bulk'
	option criteria 'local_ip == 192.168.100.22 && application == "netify.facebook";'
	option description 'Low priority for 192.168.100.22 when accessing Facebook'
	option enabled 1
```

QoS rules do not have any effect if qosify is not enabled.
To enable qosify use:
```
uci set qosify.wan.disabled=0
uci commit qosify
/etc/init.d/qosify restart
```

To inspect qosify status use:
```
qosify-status
```

Check if traffic is matching:
```
nft list table inet dpi
```

List connections with `block` labels:
```
conntrack -L -o label -l block
```

If needed, start netifyd in debug mode:
```
netifyd -R -d -I br-lan -E eth1
```