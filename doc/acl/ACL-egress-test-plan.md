## [DRAFT, UNDER DEVELOPMENT]

- [[DRAFT, UNDER DEVELOPMENT]](#draft-under-development)
- [The existing test plan and scripts](#the-existing-test-plan-and-scripts)
  - [Problems of existing ingress ACL testing](#problems-of-existing-ingress-acl-testing)
- [Ingress & egress ACL testing strategy](#ingress--egress-acl-testing-strategy)
  - [The testing strategy](#the-testing-strategy)
  - [Work need to be done](#work-need-to-be-done)
  - [ACL tables and ACL rules](#acl-tables-and-acl-rules)
  - [ACL tests](#acl-tests)

## The existing test plan and scripts

The existing acl test scripts covered ingress ACL on SONiC switch. Supported topo: t1, t1-lag, t1-64-lag

```
$ acl-loader show rule
Table    Rule          Priority    Action    Match
-------  ------------  ----------  --------  ----------------------------
DATAACL  RULE_1        9999        FORWARD   SRC_IP: 10.0.0.2/32
DATAACL  RULE_2        9998        FORWARD   DST_IP: 192.168.0.16/32
DATAACL  RULE_3        9997        FORWARD   DST_IP: 172.16.2.0/32
DATAACL  RULE_4        9996        FORWARD   L4_SRC_PORT: 4661
DATAACL  RULE_5        9995        FORWARD   IP_PROTOCOL: 126
DATAACL  RULE_6        9994        FORWARD   TCP_FLAGS: 0x12/0x12
DATAACL  RULE_7        9993        DROP      SRC_IP: 10.0.0.3/32
DATAACL  RULE_8        9992        FORWARD   SRC_IP: 10.0.0.3/32
DATAACL  RULE_9        9991        FORWARD   L4_DST_PORT: 4661
DATAACL  RULE_10       9990        FORWARD   L4_SRC_PORT_RANGE: 4656-4671
DATAACL  RULE_11       9989        FORWARD   L4_DST_PORT_RANGE: 4640-4687
DATAACL  RULE_12       9988        FORWARD   IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.2/32
DATAACL  RULE_13       9987        FORWARD   IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.2/32
DATAACL  DEFAULT_RULE  1           DROP      ETHER_TYPE: 2048
```

The existing acl testing script inserts a set of rules into the DATAACL table of type "l3". A default rule is always added by acl-loader.
Any packets that do not match higher priority rules will hit the default rule and be dropped.
To verify that the ingress ACL rules are working, the PTF script send various packets matching higher priority rules. These packets should pass through the rules and be forwarded by switch. The PTF script then can verify appearance of these packets on corresponding egress ports.

### Problems of existing ingress ACL testing

* Most of the rules covered FORWARD action. Coverage of DROP action is not enough.
* "aclshow -a" can show counters of ACL rules. Checking counters is not covered.
* The packets intended for matching RULE_12 and RULE_13 are matched by RULE_1 firstly. RULE_12 and RULE_13 are never hit.
* The ACL table is binded to all ports.
* Logging of the PTF script needs improvement. If a case failed, failed case is not in ansible log. Need to check PTF log to find out exactly which case failed.

## Ingress & egress ACL testing strategy

### The testing strategy

In a summary, to cover egress ACL testing, we plan to:
* Improve the existing ingress ACL scripts to address the current problems
* Extend the scripts to cover egress ACL testing

To test egress ACL, we need to tell whether it is the egress or ingress rule taking effect. Thus we should not bind ACL table to all ports. To test ingress ACL, ACL table should only be binded to ports that receive packts. To test egress ACL, ACL talbe should only be binded to ports that send packets.

According to supported t1, t1-lag and t1-64-lag topology. Half of the DUT ports are connected to TOR routers. The other half are connected to Spine routers. The existing ingress ACL test cases cover two directions of packets injection:
* TOR ports -> Spine ports
* Spine ports -> TOR ports

To cover egress ACL testing, the existing ingress ACL testing need to be updated to have similar pattern with the new egress ACL testing. The overall proposed stategy:
* Do not use the pre-defined ACL tables. Add 4 new ACL tables for testing.
  * TORIN - Bind to TOR ports only, stage=INGRESS (If stage is not specified, applied to ingress packets by default)
  * TOROUT - Bind to TOR ports only, stage=EGRESS
  * SPINEIN - Bind to Spine ports only, stage=INGRESS (If stage is not specified, applied to ingress packets by default)
  * SPINEOUT - Bind to Spine ports only, stage=EGRESS
* While testing ingress ACL:
  * Load ACL rules to TORIN table.
  * Run PTF script to inject packets into TOR ports and capture on Spine ports.
  * Check counters of ACL rules.
  * Clear ACL rules. Load ACL rules to SPINEIN table.
  * Run PTF script to inject packets into Spine ports and capture on TOR ports.
  * Check counters of ACL rules.
  * Clear ACL rules.
* While testing egress ACL:
  * Load ACL rules to TOROUT table.
  * Run PTF script to inject packets into Spine ports and capture on TOR ports.
  * Check counters of ACL rules.
  * Clear ACL rules. Load ACL rules to SPINEOUT table.
  * Run PTF scripts to inject packets into TOR ports and capture on Spine ports.
  * Check counters of ACL rules.
  * Clear ACL rules.

### Work need to be done

Work need to be done based on this strategy and existing scripts:
* Update the existing acltb.yml script:
  * Backup config_db. 
  * Add an ansible variable to indicate whether egress ACL is supported by the current platform. If not, only run the ingress ACL testing. This variable can get value from ansible-playbook command line.
  * Create ACL tables before testing.
  * Load different rules to different ACL tables during testing, cover ingress and egress.
  * Run PTF scripts to cover different packet injection directions, cover ingress and egress.
  * Restore configuration after testing.
  * Keep the existing tests for loading ACL rule part1&part2.
  * Keep the existing tests for testing ACL after reboot.
* Update the PTF script
  * Currently the PTF script covers the TOR->Spine and Spine->TOR directions in single run. Need to add a new parameter. PTF script should be able to cover specified direction or both according to parameter value.
* The same set of existing ACL rules could be reused. Load the same set of rules to different tables during testing.
* Improve the existing ACL rules to address issue that RULE_12 and RULE_13 are not hit.
* Extend the existing ACL rules to cover more DROP action. The PTF script should be extended accordingly too.
* Add a new ansible module for gathering ACL counters in DUT switch.
* Check counters of ACL rules after each PTF script execution.
* Improve logging of the PTF script. Output more detailed information of failed case in ansible log.

### ACL tables and ACL rules
The existing ACL tables will not be used. Will add 4 new L3 ACL tables for testing. The ACL rules will be updated too. RULE_12 and RULE_13 should use different source IP address, for example 10.0.0.4/32. Otherwise packets with source IP 10.0.0.2/32 would always match RULE_1 and never hit RULE_12 and RULE_13. The PTF script testing case 10 and 11 also need to use this new source IP address for the injected packets.

Example of updated ACL tables and rules. They should not be all loaded at the same time.

The TORIN ACL table and its ruls should only be loaded when testing：
* Ingress
* TOR -> SPINE
```
$ acl-loader show rule
Table    Rule          Priority    Action    Match
-------  ------------  ----------  --------  ----------------------------
TORIN    RULE_1        9999        FORWARD   SRC_IP: 10.0.0.2/32
TORIN    RULE_2        9998        FORWARD   DST_IP: 192.168.0.16/32
TORIN    RULE_3        9997        FORWARD   DST_IP: 172.16.2.0/32
TORIN    RULE_4        9996        FORWARD   L4_SRC_PORT: 4661
TORIN    RULE_5        9995        FORWARD   IP_PROTOCOL: 126
TORIN    RULE_6        9994        FORWARD   TCP_FLAGS: 0x12/0x12
TORIN    RULE_7        9993        DROP      SRC_IP: 10.0.0.3/32
TORIN    RULE_8        9992        FORWARD   SRC_IP: 10.0.0.3/32
TORIN    RULE_9        9991        FORWARD   L4_DST_PORT: 4661
TORIN    RULE_10       9990        FORWARD   L4_SRC_PORT_RANGE: 4656-4671
TORIN    RULE_11       9989        FORWARD   L4_DST_PORT_RANGE: 4640-4687
TORIN    RULE_12       9988        FORWARD   IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.4/32
TORIN    RULE_13       9987        FORWARD   IP_PROTOCOL: 17
TORIN    RULE_14       9986        DROP      SRC_IP: 10.0.0.6/32
TORIN    RULE_15       9985        DROP      DST_IP: 192.168.0.17/32
TORIN    RULE_16       9984        DROP      DST_IP: 172.16.3.0/32
TORIN    RULE_17       9983        DROP      L4_SRC_PORT: 4761
TORIN    RULE_18       9982        DROP      IP_PROTOCOL: 127
TORIN    RULE_19       9981        DROP      TCP_FLAGS: 0x24/0x24
TORIN    RULE_20       9980        FORWARD   SRC_IP: 10.0.0.7/32
TORIN    RULE_21       9979        DROP      SRC_IP: 10.0.0.7/32
TORIN    RULE_22       9978        DROP      L4_DST_PORT: 4761
TORIN    RULE_23       9977        DROP      L4_SRC_PORT_RANGE: 4756-4771
TORIN    RULE_24       9976        DROP      L4_DST_PORT_RANGE: 4740-4787
TORIN    RULE_25       9975        DROP      IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.8/32
TORIN    RULE_26       9974        DROP      IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.8/32                                             
TORIN    DEFAULT_RULE  1           DROP      ETHER_TYPE: 2048
```

The TOROUT ACL table and its ruls should only be loaded when testing：
* Egress
* SPINE -> TOR
```
$ acl-loader show rule
Table    Rule          Priority    Action    Match
-------  ------------  ----------  --------  ----------------------------
TOROUT   RULE_1        9999        FORWARD   SRC_IP: 10.0.0.2/32
TOROUT   RULE_2        9998        FORWARD   DST_IP: 192.168.0.16/32
TOROUT   RULE_3        9997        FORWARD   DST_IP: 172.16.2.0/32
TOROUT   RULE_4        9996        FORWARD   L4_SRC_PORT: 4661
TOROUT   RULE_5        9995        FORWARD   IP_PROTOCOL: 126
TOROUT   RULE_6        9994        FORWARD   TCP_FLAGS: 0x12/0x12
TOROUT   RULE_7        9993        DROP      SRC_IP: 10.0.0.3/32
TOROUT   RULE_8        9992        FORWARD   SRC_IP: 10.0.0.3/32
TOROUT   RULE_9        9991        FORWARD   L4_DST_PORT: 4661
TOROUT   RULE_10       9990        FORWARD   L4_SRC_PORT_RANGE: 4656-4671
TOROUT   RULE_11       9989        FORWARD   L4_DST_PORT_RANGE: 4640-4687
TOROUT   RULE_12       9988        FORWARD   IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.4/32
TOROUT   RULE_13       9987        FORWARD   IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.4/32
TOROUT   RULE_14       9986        DROP      SRC_IP: 10.0.0.6/32
TOROUT   RULE_15       9985        DROP      DST_IP: 192.168.0.17/32
TOROUT   RULE_16       9984        DROP      DST_IP: 172.16.3.0/32
TOROUT   RULE_17       9983        DROP      L4_SRC_PORT: 4761
TOROUT   RULE_18       9982        DROP      IP_PROTOCOL: 127
TOROUT   RULE_19       9981        DROP      TCP_FLAGS: 0x24/0x24
TOROUT   RULE_20       9980        FORWARD   SRC_IP: 10.0.0.7/32
TOROUT   RULE_21       9979        DROP      SRC_IP: 10.0.0.7/32
TOROUT   RULE_22       9978        DROP      L4_DST_PORT: 4761
TOROUT   RULE_23       9977        DROP      L4_SRC_PORT_RANGE: 4756-4771
TOROUT   RULE_24       9976        DROP      L4_DST_PORT_RANGE: 4740-4787
TOROUT   RULE_25       9975        DROP      IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.8/32
TOROUT   RULE_26       9974        DROP      IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.8/32                                             
TOROUT   DEFAULT_RULE  1           DROP      ETHER_TYPE: 2048
```

The SPINEIN ACL table and its ruls should only be loaded when testing：
* Ingress
* SPINE -> TOR
```
$ acl-loader show rule
Table    Rule          Priority    Action    Match
-------  ------------  ----------  --------  ----------------------------
SPINEIN  RULE_1        9999        FORWARD   SRC_IP: 10.0.0.2/32
SPINEIN  RULE_2        9998        FORWARD   DST_IP: 192.168.0.16/32
SPINEIN  RULE_3        9997        FORWARD   DST_IP: 172.16.2.0/32
SPINEIN  RULE_4        9996        FORWARD   L4_SRC_PORT: 4661
SPINEIN  RULE_5        9995        FORWARD   IP_PROTOCOL: 126
SPINEIN  RULE_6        9994        FORWARD   TCP_FLAGS: 0x12/0x12
SPINEIN  RULE_7        9993        DROP      SRC_IP: 10.0.0.3/32
SPINEIN  RULE_8        9992        FORWARD   SRC_IP: 10.0.0.3/32
SPINEIN  RULE_9        9991        FORWARD   L4_DST_PORT: 4661
SPINEIN  RULE_10       9990        FORWARD   L4_SRC_PORT_RANGE: 4656-4671
SPINEIN  RULE_11       9989        FORWARD   L4_DST_PORT_RANGE: 4640-4687
SPINEIN  RULE_12       9988        FORWARD   IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.4/32
SPINEIN  RULE_13       9987        FORWARD   IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.4/32
SPINEIN  RULE_14       9986        DROP      SRC_IP: 10.0.0.6/32
SPINEIN  RULE_15       9985        DROP      DST_IP: 192.168.0.17/32
SPINEIN  RULE_16       9984        DROP      DST_IP: 172.16.3.0/32
SPINEIN  RULE_17       9983        DROP      L4_SRC_PORT: 4761
SPINEIN  RULE_18       9982        DROP      IP_PROTOCOL: 127
SPINEIN  RULE_19       9981        DROP      TCP_FLAGS: 0x24/0x24
SPINEIN  RULE_20       9980        FORWARD   SRC_IP: 10.0.0.7/32
SPINEIN  RULE_21       9979        DROP      SRC_IP: 10.0.0.7/32
SPINEIN  RULE_22       9978        DROP      L4_DST_PORT: 4761
SPINEIN  RULE_23       9977        DROP      L4_SRC_PORT_RANGE: 4756-4771
SPINEIN  RULE_24       9976        DROP      L4_DST_PORT_RANGE: 4740-4787
SPINEIN  RULE_25       9975        DROP      IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.8/32
SPINEIN  RULE_26       9974        DROP      IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.8/32                                             
SPINEIN  DEFAULT_RULE  1           DROP      ETHER_TYPE: 2048
```

The SPINEOUT ACL table and its ruls should only be loaded when testing：
* Egress
* TOR -> SPINE
```
$ acl-loader show rule
Table    Rule          Priority    Action    Match
-------  ------------  ----------  --------  ----------------------------
SPINEOUT RULE_1        9999        FORWARD   SRC_IP: 10.0.0.2/32
SPINEOUT RULE_2        9998        FORWARD   DST_IP: 192.168.0.16/32
SPINEOUT RULE_3        9997        FORWARD   DST_IP: 172.16.2.0/32
SPINEOUT RULE_4        9996        FORWARD   L4_SRC_PORT: 4661
SPINEOUT RULE_5        9995        FORWARD   IP_PROTOCOL: 126
SPINEOUT RULE_6        9994        FORWARD   TCP_FLAGS: 0x12/0x12
SPINEOUT RULE_7        9993        DROP      SRC_IP: 10.0.0.3/32
SPINEOUT RULE_8        9992        FORWARD   SRC_IP: 10.0.0.3/32
SPINEOUT RULE_9        9991        FORWARD   L4_DST_PORT: 4661
SPINEOUT RULE_10       9990        FORWARD   L4_SRC_PORT_RANGE: 4656-4671
SPINEOUT RULE_11       9989        FORWARD   L4_DST_PORT_RANGE: 4640-4687
SPINEOUT RULE_12       9988        FORWARD   IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.4/32
SPINEOUT RULE_13       9987        FORWARD   IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.4/32
SPINEOUT RULE_14       9986        DROP      SRC_IP: 10.0.0.6/32
SPINEOUT RULE_15       9985        DROP      DST_IP: 192.168.0.17/32
SPINEOUT RULE_16       9984        DROP      DST_IP: 172.16.3.0/32
SPINEOUT RULE_17       9983        DROP      L4_SRC_PORT: 4761
SPINEOUT RULE_18       9982        DROP      IP_PROTOCOL: 127
SPINEOUT RULE_19       9981        DROP      TCP_FLAGS: 0x24/0x24
SPINEOUT RULE_20       9980        FORWARD   SRC_IP: 10.0.0.7/32
SPINEOUT RULE_21       9979        DROP      SRC_IP: 10.0.0.7/32
SPINEOUT RULE_22       9978        DROP      L4_DST_PORT: 4761
SPINEOUT RULE_23       9977        DROP      L4_SRC_PORT_RANGE: 4756-4771
SPINEOUT RULE_24       9976        DROP      L4_DST_PORT_RANGE: 4740-4787
SPINEOUT RULE_25       9975        DROP      IP_PROTOCOL: 1
                                             SRC_IP: 10.0.0.8/32
SPINEOUT RULE_26       9974        DROP      IP_PROTOCOL: 17
                                             SRC_IP: 10.0.0.8/32                                             
SPINEOUT DEFAULT_RULE  1           DROP      ETHER_TYPE: 2048
```

Show counters of ACL rules:
```
$ aclshow -a
RULE NAME     TABLE NAME    TYPE      PRIO  ACTION    PACKETS COUNT    BYTES COUNT
------------  ------------  ------  ------  --------  ---------------  -------------
RULE_1        TORIN         L3        9999  FORWARD   0                0
RULE_2        TORIN         L3        9998  FORWARD   N/A              N/A
RULE_3        TORIN         L3        9997  FORWARD   N/A              N/A
RULE_4        TORIN         L3        9996  FORWARD   N/A              N/A
RULE_5        TORIN         L3        9995  FORWARD   N/A              N/A
RULE_6        TORIN         L3        9994  FORWARD   N/A              N/A
RULE_7        TORIN         L3        9993  DROP      N/A              N/A
RULE_8        TORIN         L3        9992  FORWARD   N/A              N/A
RULE_9        TORIN         L3        9991  FORWARD   N/A              N/A
RULE_10       TORIN         L3        9990  FORWARD   0                0
RULE_11       TORIN         L3        9989  FORWARD   0                0
RULE_12       TORIN         L3        9988  FORWARD   N/A              N/A
RULE_13       TORIN         L3        9987  FORWARD   N/A              N/A
...
DEFAULT_RULE  TORIN         L3           1  DROP      0                0
...
```

### ACL tests

For each packet direction of ingress and egress testing, all of these tests must be executed in the PTF script:

* Test 0 - unmatched packet - dropped

* Test 1 - source IP match - forwarded
* Test 2 - destination IP match - forwarded
* Test 3 - L4 source port match - forwarded
* Test 4 - L4 destination port match - forwarded
* Test 5 - IP protocol match - forwarded
* Test 6 - TCP flags match - forwarded
* Test 7 - source port range match - forwarded
* Test 8 - destination port range match - forwarded
* Test 9 - rules priority - dropped
* Test 10 - ICMP source IP match - forwarded
* Test 11 - UDP source IP match - forwarded

* Test 12 - source IP match - dropped
* Test 13 - destination IP match - dropped
* Test 14 - L4 source port match - dropped
* Test 15 - L4 destination port match - dropped
* Test 16 - IP protocol match - dropped
* Test 17 - TCP flags match - dropped
* Test 18 - source port range match - dropped
* Test 19 - destination port range match - dropped
* Test 20 - rules priority - forwarded
* Test 21 - ICMP source IP match - dropped
* Test 22 - UDP source IP match - dropped
