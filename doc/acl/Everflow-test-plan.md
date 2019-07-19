
- [Overview](#Overview)
  - [Scope](#Scope)
  - [Summary of the existing everflow test plan](#Summary-of-the-existing-everflow-test-plan)
  - [Extend the test plan to cover both ingress and egress mirroring](#Extend-the-test-plan-to-cover-both-ingress-and-egress-mirroring)
    - [What new enhancements need to be covered?](#What-new-enhancements-need-to-be-covered)
      - [Egress ACL table](#Egress-ACL-table)
      - [Egress mirroring](#Egress-mirroring)
    - [Some existing areas not covered by the existing scripts](#Some-existing-areas-not-covered-by-the-existing-scripts)
      - [ACL rule for matching "IN_PORTS"](#ACL-rule-for-matching-IN_PORTS)
      - [IPv6 everflow](#IPv6-everflow)
    - [How to extend the testing](#How-to-extend-the-testing)
  - [Test configurations](#Test-configurations)
    - [ACL table configurations](#ACL-table-configurations)
  - [Related **DUT** CLI commands](#Related-DUT-CLI-commands)
    - [`sonic-cfggen`](#sonic-cfggen)
    - [`acl-loader`](#acl-loader)
    - [`aclshow`](#aclshow)
    - [`config mirror_session`](#config-mirror_session)
- [Test structure](#Test-structure)
  - [Overall structure](#Overall-structure)
  - [Prepare some variables for testing](#Prepare-some-variables-for-testing)
  - [Add everflow configuration](#Add-everflow-configuration)
    - [ACL tables](#ACL-tables)
    - [Mirror sessions](#Mirror-sessions)
    - [ACL rules](#ACL-rules)
  - [Run test](#Run-test)
    - [PTF Test](#PTF-Test)
- [Test cases](#Test-cases)
  - [Test case \#1 - Resolved route](#Test-case-1---Resolved-route)
    - [Test objective](#Test-objective)
    - [Test steps](#Test-steps)
  - [Test case \#2 - Longer prefix route with resolved next hop](#Test-case-2---Longer-prefix-route-with-resolved-next-hop)
    - [Test objective](#Test-objective-1)
    - [Test steps](#Test-steps-1)
  - [Test case \#3 - Remove longer prefix route.](#Test-case-3---Remove-longer-prefix-route)
    - [Test objective](#Test-objective-2)
    - [Test steps](#Test-steps-2)
  - [Test case \#4 - Change neighbor MAC address.](#Test-case-4---Change-neighbor-MAC-address)
    - [Test objective](#Test-objective-3)
    - [Test steps](#Test-steps-3)
  - [Test case \#5 - Resolved ECMP route.](#Test-case-5---Resolved-ECMP-route)
    - [Test objective](#Test-objective-4)
    - [Test steps](#Test-steps-4)
  - [Test case \#7 - ECMP route change (remove next hop not used by session).](#Test-case-7---ECMP-route-change-remove-next-hop-not-used-by-session)
    - [Test objective](#Test-objective-5)
    - [Test steps](#Test-steps-5)
  - [Test case \#8 - Policer enforced DSCP value/mask test.](#Test-case-8---Policer-enforced-DSCP-valuemask-test)
    - [Test objective](#Test-objective-6)
    - [Test steps](#Test-steps-6)
- [TODO](#TODO)
- [Open Questions](#Open-Questions)

## Overview

This document is an updated version of the existing everflow test plan: https://github.com/Azure/SONiC/wiki/Everflow-test-plan

The purpose is to test functionality of Everflow on the SONIC switch DUT with and without LAGs configured, closely resembling production environment.
The test assumes all necessary configuration, including Everflow session and ACL rules, LAG configuration and BGP routes, are already pre-configured on the SONIC switch before test runs.

### Scope
The test is targeting a running SONIC system with fully functioning configuration.
The purpose of the test is not to test specific SAI API, but functional testing of Everflow on SONiC system, making sure that traffic flows correctly, according to BGP routes advertised by BGP peers of SONIC switch, and the LAG configuration.

NOTE: Everflow+LAG test will be able to run **only** in the testbed specifically created for LAG.

### Summary of the existing everflow test plan

The existing everflow scripts:
```
ansible/
    roles/
        test/
            tasks/
                everflow_testbed.yml
                everflow_testbed/
                    apply_config/
                        acl_rule_persistent.json
                        expect_messages.txt
                    del_config/
                        acl_rule_persistent-del.json
                        acl_rule_persistent.json
                        acl_table.json
                        expect_messages.txt
                        session.json
                    apply_config.yml
                    del_config.yml
                    get_neighbor_info.yml
                    get_port_info.yml
                    get_session_info.yml
                    run_test.yml
                    testcase_1.yml
                    testcase_2.yml
                    testcase_3.yml
                    testcase_4.yml
                    testcase_5.yml
                    testcase_6.yml
                    testcase_7.yml
                    testcase_8.yml
            files/
                acstests/
                    everflow_tb_test.py
                    everflow_policer_test.py
```

The existing everflow test plan only covers ingress mirroring. ACL rules are added to ACL table of type "MIRROR". And by default, packets are only checked against the ACL rules on ingress stage. On ports that are bound to everflow ACL table, any ingress packets hitting the ACL rules are copied to associated mirror destination in GRE tunnel.

Two packet directions are covered in the exist testing: SPINE -> TOR ports and TOR -> SPINE ports. When the injected packets hit any of the configured ACL rules in the ingress stage, the packets will be mirrored to configured mirror destination. GRE tunnel is used for sending mirrored packets with src, dst IP and other parameters configured in the mirror session. Below test cases are used for verifying that the DUT switch can properly forwarded the mirrored packets according to different routing configurations for mirror session destination.

Example ACL table in config_db for everflow testing, type of the table is `MIRROR`:
```
{
    "ACL_TABLE": {
        "EVERFLOW": {
            "policy_desc": "EVERFLOW",
            "ports": [
                "Ethernet100",
                "Ethernet104",
                "Ethernet92",
                "Ethernet96",
                "Ethernet84",
                "Ethernet88",
                "Ethernet76",
                "Ethernet80",
                "Ethernet108",
                "Ethernet112",
                "Ethernet64",
                "Ethernet60",
                "Ethernet52",
                "Ethernet48",
                "Ethernet44",
                "Ethernet40",
                "Ethernet36",
                "Ethernet120",
                "Ethernet116",
                "Ethernet56",
                "Ethernet124",
                "Ethernet72",
                "Ethernet68",
                "Ethernet24",
                "Ethernet20",
                "Ethernet16",
                "Ethernet12",
                "Ethernet8",
                "Ethernet4",
                "Ethernet0",
                "Ethernet32",
                "Ethernet28"
            ],
            "type": "MIRROR"
        }
    }
}
```

Example ACL rules in config_db for everflow testing, action field and value of the ACL rules is `MIRROR_ACTION: <session_name>`:
```
{
    "ACL_RULE": {
        "EVERFLOW|RULE_1": {
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9999",
            "SRC_IP": "20.0.0.10/32"
        },
        "EVERFLOW|RULE_2": {
            "DST_IP": "30.0.0.10/32",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9998"
        },
        "EVERFLOW|RULE_3": {
            "L4_SRC_PORT": "4661",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9997"
        },
        "EVERFLOW|RULE_4": {
            "L4_DST_PORT": "4661",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9996"
        },
        "EVERFLOW|RULE_5": {
            "ETHER_TYPE": "4660",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9995"
        },
        "EVERFLOW|RULE_6": {
            "IP_PROTOCOL": "126",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9994"
        },
        "EVERFLOW|RULE_7": {
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9993",
            "TCP_FLAGS": "0x12/0x12"
        },
        "EVERFLOW|RULE_8": {
            "L4_SRC_PORT_RANGE": "4672-4681",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9992"
        },
        "EVERFLOW|RULE_9": {
            "L4_DST_PORT_RANGE": "4672-4681",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9991"
        },
        "EVERFLOW|RULE_10": {
            "DSCP": "51",
            "MIRROR_ACTION": "test_session_1",
            "PRIORITY": "9990"
        }
    }
}
```

Examples of show the ACL table and rules configuration from command line:
```
$ show acl table EVERFLOW
Name      Type    Binding      Description
--------  ------  -----------  -------------
EVERFLOW  MIRROR  Ethernet0    EVERFLOW
                  Ethernet4
                  Ethernet8
                  Ethernet12
                  Ethernet16
                  Ethernet20
                  Ethernet24
                  Ethernet28
                  Ethernet32
                  Ethernet36
                  Ethernet40
                  Ethernet44
                  Ethernet48
                  Ethernet52
                  Ethernet56
                  Ethernet60
                  Ethernet64
                  Ethernet68
                  Ethernet72
                  Ethernet76
                  Ethernet80
                  Ethernet84
                  Ethernet88
                  Ethernet92
                  Ethernet96
                  Ethernet100
                  Ethernet104
                  Ethernet108
                  Ethernet112
                  Ethernet116
                  Ethernet120
                  Ethernet124
$ show acl rule
Table     Rule       Priority  Action                  Match
--------  -------  ----------  ----------------------  ----------------------------
EVERFLOW  RULE_1         9999  MIRROR: test_session_1  SRC_IP: 20.0.0.10/32
EVERFLOW  RULE_2         9998  MIRROR: test_session_1  DST_IP: 30.0.0.10/32
EVERFLOW  RULE_3         9997  MIRROR: test_session_1  L4_SRC_PORT: 4661
EVERFLOW  RULE_4         9996  MIRROR: test_session_1  L4_DST_PORT: 4661
EVERFLOW  RULE_5         9995  MIRROR: test_session_1  ETHER_TYPE: 4660
EVERFLOW  RULE_6         9994  MIRROR: test_session_1  IP_PROTOCOL: 126
EVERFLOW  RULE_7         9993  MIRROR: test_session_1  TCP_FLAGS: 0x12/0x12
EVERFLOW  RULE_8         9992  MIRROR: test_session_1  L4_SRC_PORT_RANGE: 4672-4681
EVERFLOW  RULE_9         9991  MIRROR: test_session_1  L4_DST_PORT_RANGE: 4672-4681
EVERFLOW  RULE_10        9990  MIRROR: test_session_1  DSCP: 51
```

Existing test cases:
* testcase 1 - Resolved route
* testcase 2 - Longer prefix route with resolved next hop
* testcase 3 - Remove longer prefix route
* testcase 4 - Change neighbor MAC address
* testcase 5 - Resolved ECMP route
* testcase 6 - ECMP route change (remove next hop not used by session)
* testcase 7 - ECMP route change (remove next hop used by session)
* testcase 8 - Policer enforced DSCP value/mask test

### Extend the test plan to cover both ingress and egress mirroring

#### What new enhancements need to be covered?

##### Egress ACL table
In Jan 2019, egress ACL table support is added (https://github.com/Azure/SONiC/pull/322, https://github.com/Azure/sonic-swss/pull/741) to SONiC. Then ACL table can have an extra field `stage` indicting on which stage will the ACL rules be checked against packets. If the `stage` is ignored or is set to 'ingress', the behavior is same as before, ingress packets will be checked against ACL rules. If the `stage` field is set to 'egress', then on ports bound to ACL table, egress packets will be checked against the ACL rules and will be handled according to the action configured for the ACL rules. The action could be `PACKET_ACTION` or `MIRROR_ACTION`.

##### Egress mirroring
Besides the egress ACL table support, a recent enhancement (Design: https://github.com/Azure/SONiC/pull/411 Implementations: https://github.com/Azure/sonic-swss/pull/963 https://github.com/Azure/sonic-utilities/pull/575) added egress mirroring support. This enhancement added two ACL rule action types based on the existing mirroring action `MIRROR_ACTION`:
* MIRROR_INGRESS_ACTION
* MIRROR_EGRESS_ACTION

The `MIRROR_INGRESS_ACTION` type new. But its behavior is same as the existing ingress mirroring. Packets hit ACL rule will be mirrored at the ingress stage.
The `MIRROR_EGRESS_ACTION` is a new action type which is for egress mirroring. It means that on ports bound to everflow ACL table, when packets hit ACL rules of that table, the packets will be mirrored at the egress stage. The original `MIRROR_ACTION` is kept for backward compatibility and it is implicitly set to "ingress" by default.

Combining these two enhancements, there are 4 scenarios for everflow.
1. ACL table of type `MIRROR` have `stage` field ignored or set to "ingress". Action type of ACL_RULE is `MIRROR_ACTION` or `MIRROR_INGRESS_ACTION`.
2. ACL table of type `MIRROR` have `stage` field ignored or set to "ingress". Action type of ACL_RULE is `MIRROR_EGRESS_ACTION`.
3. ACL table of type `MIRROR` have `stage` field set to "egress". Action type of ACL_RULE is `MIRROR_ACTION` or `MIRROR_INGRESS_ACTION`.
4. ACL table of type `MIRROR` have `stage` field set to "egress". Action type of ACL_RULE is `MIRROR_EGRESS_ACTION`.

Expected behaviors for the combinations:

- | ACL table stage: ingress | ACL table stage: egress
-|-|-
Action type: MIRROR_INGRESS_ACTION | Ingress packets hit ACL rules, mirrored at ingress stage    | Not applicable
Action type: MIRROR_EGRESS_ACTION | Ingress packets hit ACL rules, mirrored at egress stage | Egress packets hit ACL rules, mirrored at egress stage

Since not all the combinations are supported by all vendors, the enhancement also added ACL capability detection. The supported ACL action types at different stage are detected and stored in redis. The below is an example of showing detected capabilities:
```
$ redis-cli -n 6 hgetall 'SWITCH_CAPABILITY|switch'
 1) "MIRROR"
 2) "true"
 3) "MIRRORV6"
 4) "true"
 5) "ACL_ACTIONS|INGRESS"
 6) "PACKET_ACTION,REDIRECT_ACTION,MIRROR_INGRESS_ACTION"
 7) "ACL_ACTIONS|EGRESS"
 8) "PACKET_ACTION,MIRROR_EGRESS_ACTION"
 9) "ACL_ACTION|PACKET_ACTION"
1)  "DROP,FORWARD"
```
In the above example output, two everflow combinations are supported on the platform being checked:
* INGRESS stage, MIRROR_INGRESS_ACTION
* EGRESS stage, MIRROR_EGRESS_ACTION

This test plan needs to be extended to cover both the existing everflow function and the newly added capabilities.

#### Some existing areas not covered by the existing scripts

##### ACL rule for matching "IN_PORTS"
Now the SONiC ACL rules support matching "IN_PORTS". New ACL rule for matching "IN_PORTS" need to be added and covered.

##### IPv6 everflow
The existing scripts only covered IPv6. IPv6 is also supported by SONiC now. The scripts need to be extended to cover IPv6 everflow too. To cover IPv6:
* Everflow ACL table of type "MIRRORV6" needs to be defined and loaded during testing.
* Different stages (ingress & egress) also need to be covered.
* Different ACL rule mirror actions also need to be covered.
* New set of IPv6 ACL rules need to be defined and loaded during testing.
* The PTF script needs to be extended to inject and monitor IPv6 packets.

#### How to extend the testing

To cover the new enhancements, the existing scripts need to be extended:
* The existing structure, sub-tests and PTF scripts can be reused.
* Add new sets of configurations.
* We can refactor the run_test.yml to run the existing sub-tests in multiple iteration. Each iteration loads a different set of ACL table and ACL rules configuration.
* Adjust initialization of variables used in testing if needed.
* Add a new class in the existing PTF script to inject and monitor IPv6 packets.
* Add a sub-test and add a new class in the PTF script to cover ACL rule matching "IN_PORTS".

In a summary, the extended scripts need to do below work:
1. Firstly the script need to get capability info of the DUT from hard coded resource, then check against the detected capabilities in DB. Fail the test if capabilities do not match.
2. Create everflow ACL table with different stage setting. Load ACL rules with different action type.
3. Run test cases in the existing scripts (8 test cases at the time of writing).
4. Ensure that each combination of the supported ACL table stages and ACL rule action types are covered.

Summary of the possible combinations:

Combinations | ACL table stage: ingress | ACL table stage: egress
-|-|-
ACL Rule MIRROR_INGRESS_ACTION | [x] | N/A
ACL Rule MIRROR_EGRESS_ACTION | [x] | [x]

Totally there are 3 possible combinations. Not all the combinations are supported by all platforms. The actual combinations to be tested are determined the actual DUT platform.

Switch's ACL capability can be queried from DB table `SWITCH_CAPABILITY|switch`. The testing script can query ACL capability first. And then run the supported combinations, skip the unsupported combinations.

For example, if query SWITCH_CAPABILITY got below results:
```
$ redis-cli -n 6 hgetall "SWITCH_CAPABILITY|switch"
1) "ACL_ACTIONS|INGRESS"
2) "PACKET_ACTION,REDIRECT_ACTION,MIRROR_ACTION_INGRESS"
3) "ACL_ACTIONS|EGRESS"
4) "PACKET_ACTION,MIRROR_ACTION_EGRESS"
...
```
Then the platform only supports two combinations:
* ACL table stage ingress + ACL Rule MIRROR_INGRESS_ACTION
* ACL table stage egress + ACL Rule MIRROR_EGRESS_ACTION

The third combination would be skipped on this platform.

### Test configurations

#### ACL table configurations

New ACL tables of type MIRROR and MIRRORv6 need to be created in testing. The new ACL tables also need to set different values for their "stage" attribute.

### Related **DUT** CLI commands

Summary of the CLI commands that will be used for configuring DUT.

#### `sonic-cfggen`

It is for adding configuration for ACL tables. For example:
* `sonic-cfggen -j <configuration_json_file> --write-to-db`: Load configuration in json format to config_db. The json file could be ACL table configuration.
* `sonic-cfggen -d -v ACL_TABLE`: Dump current ACL_TABLE configuration from config_db.
* `sonic-cfggen -d -v ACL_RULE`: Dump current ACL_RULE configuration from config_db.

#### `acl-loader`

It is for configuring ACL rules. For example:
* `acl-loader update full <acl_rule_configuration_json_file> [--session_name=<session_name> --mirror_stage=<ingress|egress>]`: Load acl rules specified in a json file to config_db.

#### `aclshow`
This tool is for collecting ACL rule counters. For example:
* `aclshow -a`

#### `config mirror_session`

This tool is for configuring everflow mirror session. For example:
* `config mirror_session add <session_name> <src_ip> <dst_ip> <dscp> <ttl> [gre_type] [queue]`

## Test structure

### Overall structure

The extended ansible test playbook will have below parts:
1. Prepare some variables for testing
2. Add everflow configuration
3. Run everflow sub-tests
4. Clear everflow configuration
5. Repeat steps 1-3 for other configuration scenarios

The subsequent sections will have more detailed description of part

### Prepare some variables for testing

Firstly, some variables need to be prepared for testing. For example, the source ports for injecting traffic. The expected destination ports for mirrored packets.

### Add everflow configuration

Before run the sub-tests for each scenario, the scripts need to setup by loading configurations to DUT for the scenario to be covered.

There will be j2 template files for generating ACL tables and ACL rules configurations. Ansible playbook will generate ACL tables and ACL rules json configuration files to DUT based on these templates, switch capability and running topology. Then commands `sonic-cfggen` and `acl-loader` can be used for loading the configurations.

Different sets of configuration files will be generated for different test scenarios:
* IPv4
  * ACL table: MIRROR, implicit ingress; ACL rules, MIRROR_INGRESS_ACTION
  * ACL table: MIRROR, implicit ingress; ACL rules, MIRROR_EGRESS_ACTION
  * ACL table: MIRROR, ingress; ACL rules, MIRROR_INGRESS_ACTION
  * ACL table: MIRROR, ingress; ACL rules, MIRROR_EGRESS_ACTION
  * ACL table: MIRROR, egress; ACL rules, MIRROR_EGRESS_ACTION
* IPv6
  * ACL table: MIRRORV6, implicit ingress; ACL rules, MIRROR_INGRESS_ACTION
  * ACL table: MIRRORV6, implicit ingress; ACL rules, MIRROR_EGRESS_ACTION
  * ACL table: MIRRORV6, ingress; ACL rules, MIRROR_INGRESS_ACTION
  * ACL table: MIRRORV6, ingress; ACL rules, MIRROR_EGRESS_ACTION
  * ACL table: MIRRORV6, egress; ACL rules, MIRROR_EGRESS_ACTION

#### ACL tables

For each test scenario, an ACL table configuration need to be created and loaded to DUT. Command `sonic-cfggen` can be used to load the ACL table configuration into config_db. Command syntax:  `sonic-cfggen -j <acl_table_configuration_filename> --write-to-db`.

Example ACL table configuration files to be generated for each scenario:
* ACL table: MIRROR, implicit ingress
```
{
    "ACL_TABLE": {
        "EF_INGRESS1": {
            "policy_desc": "EVERFLOW implicit ingress",
            "ports": [
                "Ethernet100", "Ethernet104", "Ethernet92", "Ethernet96", "Ethernet84", "Ethernet88", "Ethernet76", "Ethernet80", "Ethernet108", "Ethernet112", "Ethernet64", "Ethernet60", "Ethernet52", "Ethernet48", "Ethernet44", "Ethernet40", "Ethernet36", "Ethernet120", "Ethernet116", "Ethernet56", "Ethernet124", "Ethernet72", "Ethernet68", "Ethernet24", "Ethernet20", "Ethernet16", "Ethernet12", "Ethernet8", "Ethernet4", "Ethernet0", "Ethernet32", "Ethernet28"
            ],
            "type": "MIRROR"
        }
    }
}
```
* ACL table: MIRROR, ingress
```
{
    "ACL_TABLE": {
        "EF_INGRESS2": {
            "policy_desc": "EVERFLOW ingress",
            "ports": [
                "Ethernet100", "Ethernet104", "Ethernet92", "Ethernet96", "Ethernet84", "Ethernet88", "Ethernet76", "Ethernet80", "Ethernet108", "Ethernet112", "Ethernet64", "Ethernet60", "Ethernet52", "Ethernet48", "Ethernet44", "Ethernet40", "Ethernet36", "Ethernet120", "Ethernet116", "Ethernet56", "Ethernet124", "Ethernet72", "Ethernet68", "Ethernet24", "Ethernet20", "Ethernet16", "Ethernet12", "Ethernet8", "Ethernet4", "Ethernet0", "Ethernet32", "Ethernet28"
            ],
            "type": "MIRROR",
            "stage": "ingress"
        }
    }
}
```
* ACL table: MIRROR, egress
```
{
    "ACL_TABLE": {
        "EF_EGRESS": {
            "policy_desc": "EVERFLOW egress",
            "ports": [
                "Ethernet100", "Ethernet104", "Ethernet92", "Ethernet96", "Ethernet84", "Ethernet88", "Ethernet76", "Ethernet80", "Ethernet108", "Ethernet112", "Ethernet64", "Ethernet60", "Ethernet52", "Ethernet48", "Ethernet44", "Ethernet40", "Ethernet36", "Ethernet120", "Ethernet116", "Ethernet56", "Ethernet124", "Ethernet72", "Ethernet68", "Ethernet24", "Ethernet20", "Ethernet16", "Ethernet12", "Ethernet8", "Ethernet4", "Ethernet0", "Ethernet32", "Ethernet28"
            ],
            "type": "MIRROR",
            "stage": "egress"
        }
    }
}
```
* ACL table: MIRRORV6, implicit ingress
```
{
    "ACL_TABLE": {
        "EFV6_INGRESS1": {
            "policy_desc": "EVERFLOW IPv6 implicit ingress",
            "ports": [
                "Ethernet100", "Ethernet104", "Ethernet92", "Ethernet96", "Ethernet84", "Ethernet88", "Ethernet76", "Ethernet80", "Ethernet108", "Ethernet112", "Ethernet64", "Ethernet60", "Ethernet52", "Ethernet48", "Ethernet44", "Ethernet40", "Ethernet36", "Ethernet120", "Ethernet116", "Ethernet56", "Ethernet124", "Ethernet72", "Ethernet68", "Ethernet24", "Ethernet20", "Ethernet16", "Ethernet12", "Ethernet8", "Ethernet4", "Ethernet0", "Ethernet32", "Ethernet28"
            ],
            "type": "MIRRORV6"
        }
    }
}
```
* ACL table: MIRRORV6, ingress
```
{
    "ACL_TABLE": {
        "EFV6_INGRESS2": {
            "policy_desc": "EVERFLOW IPv6 ingress",
            "ports": [
                "Ethernet100", "Ethernet104", "Ethernet92", "Ethernet96", "Ethernet84", "Ethernet88", "Ethernet76", "Ethernet80", "Ethernet108", "Ethernet112", "Ethernet64", "Ethernet60", "Ethernet52", "Ethernet48", "Ethernet44", "Ethernet40", "Ethernet36", "Ethernet120", "Ethernet116", "Ethernet56", "Ethernet124", "Ethernet72", "Ethernet68", "Ethernet24", "Ethernet20", "Ethernet16", "Ethernet12", "Ethernet8", "Ethernet4", "Ethernet0", "Ethernet32", "Ethernet28"
            ],
            "type": "MIRRORV6",
            "stage": "ingress"
        }
    }
}
```
* ACL table: MIRRORV6, egress
```
{
    "ACL_TABLE": {
        "EFV6_EGRESS": {
            "policy_desc": "EVERFLOW IPv6 egress",
            "ports": [
                "Ethernet100", "Ethernet104", "Ethernet92", "Ethernet96", "Ethernet84", "Ethernet88", "Ethernet76", "Ethernet80", "Ethernet108", "Ethernet112", "Ethernet64", "Ethernet60", "Ethernet52", "Ethernet48", "Ethernet44", "Ethernet40", "Ethernet36", "Ethernet120", "Ethernet116", "Ethernet56", "Ethernet124", "Ethernet72", "Ethernet68", "Ethernet24", "Ethernet20", "Ethernet16", "Ethernet12", "Ethernet8", "Ethernet4", "Ethernet0", "Ethernet32", "Ethernet28"
            ],
            "type": "MIRRORV6",
            "stage": "egress"
        }
    }
}
```

#### Mirror sessions

The script will configure mirror session using `config mirror_session`.

Add mirror_session for IPv4 testing:
```
$ config mirror_session add session_1 1.1.1.1 2.2.2.2 8 64 0x6558 0
$ acl-loader show session
Name       Status    SRC IP     DST IP    GRE     DSCP    TTL    Queue
---------  --------  ---------  --------  ------  ------  -----  -------
session_1  inactive  1.1.1.1    2.2.2.2   0x6558  8       64     0
```

Add mirror_session for IPv6 testing:
```
$ config mirror_session add session_2 2000::1:1:1:1 2000::2:2:2:2 8 64 0x6558 0
$ acl-loader show session
Name       Status    SRC IP          DST IP          GRE     DSCP    TTL    Queue
---------  --------  -------------   -------------   ------  ------  -----  -------
session_2  inactive  2000::1:1:1:1   2000::2:2:2:2   0x6558  8       64     0
```

#### ACL rules

Generate different sets of ACL rules from template. Load the ACL rules using below command:
`acl-loader update full <acl_rule_configuration_json_file> --session_name=<mirror_session_name> --mirror_stage=<ingress|egress>`


For IPv4 testing, the ACL rules template:
```
{
    "acl": {
        "acl-sets": {
            "acl-set": {
                "{{ acl_table_name }}": {
                    "acl-entries": {
                        "acl-entry": {
                            "1": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 1
                                },
                                "ip": {
                                    "config": {
                                        "source-ip-address": "20.0.0.10/32"
                                    }
                                }
                            },
                            "2": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 2
                                },
                                "ip": {
                                    "config": {
                                        "destination-ip-address": "192.168.0.10/32"
                                    }
                                }
                            },
                            "3": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 3
                                },
                                "transport": {
                                    "config": {
                                        "source-port": "4661"
                                    }
                                }
                            },
                            "4": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 4
                                },
                                "transport": {
                                    "config": {
                                        "destination-port": "4661"
                                    }
                                }
                            },
                            "5": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 5
                                },
                                "l2": {
                                    "config": {
                                        "ethertype": "4660"
                                    }
                                }
                            },
                            "6": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 6
                                },
                                "ip": {
                                    "config": {
                                        "protocol": 126
                                    }
                                }
                            },
                            "7": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 7
                                },
                                "transport": {
                                    "config": {
                                        "tcp-flags": ["TCP_ACK", "TCP_SYN"]
                                    }
                                }
                            },
                            "8": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 8
                                },
                                "transport": {
                                    "config": {
                                        "source-port": "4672..4681"
                                    }
                                }
                            },
                            "9": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 9
                                },
                                "transport": {
                                    "config": {
                                        "destination-port": "4672..4681"
                                    }
                                }
                            },
                            "10": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 10
                                },
                                "ip": {
                                    "config": {
                                        "dscp": "51"
                                    }
                                }
                            },
                            "11": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 10
                                },
                                "input_interface": {
                                    "interface_ref": {
                                        "config": {
                                            "interface": "{{ acl_in_ports }}"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

For IPv6 testing, the ACL rules template:
```
{
    "acl": {
        "acl-sets": {
            "acl-set": {
                "{{ acl_table_name }}": {
                    "acl-entries": {
                        "acl-entry": {
                            "1": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 1
                                },
                                "ip": {
                                    "config": {
                                        "source-ip-address": "2000::20:0:0:10/64"
                                    }
                                }
                            },
                            "2": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 2
                                },
                                "ip": {
                                    "config": {
                                        "destination-ip-address": "fe80::192:168:0:10/64"
                                    }
                                }
                            },
                            "3": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 3
                                },
                                "transport": {
                                    "config": {
                                        "source-port": "4661"
                                    }
                                }
                            },
                            "4": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 4
                                },
                                "transport": {
                                    "config": {
                                        "destination-port": "4661"
                                    }
                                }
                            },
                            "5": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 5
                                },
                                "l2": {
                                    "config": {
                                        "ethertype": "4660"
                                    }
                                }
                            },
                            "6": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 6
                                },
                                "ip": {
                                    "config": {
                                        "protocol": 126
                                    }
                                }
                            },
                            "7": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 7
                                },
                                "transport": {
                                    "config": {
                                        "tcp-flags": ["TCP_ACK", "TCP_SYN"]
                                    }
                                }
                            },
                            "8": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 8
                                },
                                "transport": {
                                    "config": {
                                        "source-port": "4672..4681"
                                    }
                                }
                            },
                            "9": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 9
                                },
                                "transport": {
                                    "config": {
                                        "destination-port": "4672..4681"
                                    }
                                }
                            },
                            "10": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 10
                                },
                                "ip": {
                                    "config": {
                                        "dscp": "51"
                                    }
                                }
                            },
                            "11": {
                                "actions": {
                                    "config": {
                                        "forwarding-action": "ACCEPT"
                                    }
                                },
                                "config": {
                                    "sequence-id": 10
                                },
                                "input_interface": {
                                    "interface_ref": {
                                        "config": {
                                            "interface": "{{ acl_in_ports }}"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

For example, if load ACL rules for IPv4 everflow egress and MIRROR_EGRESS_ACTION, they should like below in config_db:
```
{
    "ACL_RULE": {
        "EF_EGRESS|RULE_1": {
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9999",
            "SRC_IP": "20.0.0.10/32"
        },
        "EF_EGRESS|RULE_2": {
            "DST_IP": "30.0.0.10/32",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9998"
        },
        "EF_EGRESS|RULE_3": {
            "L4_SRC_PORT": "4661",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9997"
        },
        "EF_EGRESS|RULE_4": {
            "L4_DST_PORT": "4661",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9996"
        },
        "EF_EGRESS|RULE_5": {
            "ETHER_TYPE": "4660",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9995"
        },
        "EF_EGRESS|RULE_6": {
            "IP_PROTOCOL": "126",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9994"
        },
        "EF_EGRESS|RULE_7": {
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9993",
            "TCP_FLAGS": "0x12/0x12"
        },
        "EF_EGRESS|RULE_8": {
            "L4_SRC_PORT_RANGE": "4672-4681",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9992"
        },
        "EF_EGRESS|RULE_9": {
            "L4_DST_PORT_RANGE": "4672-4681",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9991"
        },
        "EF_EGRESS|RULE_10": {
            "DSCP": "51",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9990"
        },
        "EF_EGRESS|RULE_11": {
            "IN_PORTS": "Ethernet4,Ethernet8",
            "MIRROR_EGRESS_ACTION": "session_1",
            "PRIORITY": "9989"
        }
    }
}
```

### Run test

For each configuration scenario, we need to run all the sub-tests. Everflow sub-tests consists of a number of test cases. Each of the test case is executed with log analyzer enabled, for example:

1. Run loganalyzer 'init' phase
2. Run a everflow test case
3. Run loganalyzer 'analyze' phase

Each test case may involve with with one or more classes defined in the PTF script.

#### PTF Test

The everflow test cases eventually call the ptf scripts to do the actual testing. The PTF scripts inject packets into DUT and validate traffic forwarded by DUT.

PTF test will generate traffic between ports and make sure it mirrored according to the configured Everflow session and ACL rules. Depending on the testbed topology and the existing configuration (e.g. ECMP, LAGS, etc) packets may arrive to different ports. Therefore ports connection information will be generated from the minigraph and supplied to the PTF script.

The `EverflowTest` class in everflow_tb_test.py need to be extended to cover IPv6 testing. Need some new methods for sending and validating IPv6 packets.

## Test cases

Each test case will be additionally validated by the loganalyzer utility.

Each test case will add dynamic Everflow ACL rules at the beginning and remove them at the end.

Each test case will run traffic for persistent and dynamic Everflow ACL rules.

Each test case will analyze Everflow packet header and payload (if mirrored packet is equal to original). In case of egress mirroring, verify that TTL of the mirrored packet in GRE tunnel is decremented comparing with the injected packet.

### Test case \#1 - Resolved route

#### Test objective

Verify that session with resolved route has active state.

#### Test steps

- Create route that matches session destination IP with unresolved next hop.
- Resolve route next hop.
- Send packets that matches each Everflow ACL rule.
- Verify that packet mirrored to appropriate port.
- Analyze mirrored packet header.
- In case of egress mirroring, verify that TTL of the mirrored packet is decremented comparing with the injected packet.
- Verify that mirrored packet payload is equal to sent packet.
- Verify that counters value of each Everflow ACL rule is correct.

### Test case \#2 - Longer prefix route with resolved next hop

#### Test objective

Verify that session destination port and MAC address are changed after best match route insertion.

#### Test steps

- Create route that matches session destination IP with unresolved next hop.
- Resolve route next hop.
- Send packets that matches each Everflow ACL rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packet header.
- In case of egress mirroring, verify that TTL of the mirrored packet is decremented comparing with the injected packet.
- Verify that mirrored packet payload is equal to sent packet.
- Create best match route that matches session destination IP with unresolved next hop.
- Send packets that matches each Everflow ACL rule.
- Verify that packets are mirrored to the same port.
- Resolve best match route next hop (neighbor should be on different port).
- Send packets that matches each Everflow ACL rule.
- Verify that packets are mirrored and destination port changed accordingly.

### Test case \#3 - Remove longer prefix route.

#### Test objective

Verify that session destination port and MAC address are changed after best match route removal.

#### Test steps

- Create route that matches session destination IP with unresolved next hop.
- Resolve route next hop.
- Send packets that matches each Everflow ACL rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packet header.
- In case of egress mirroring, verify that TTL of the mirrored packet is decremented comparing with the injected packet.
- Verify that mirrored packet payload is equal to sent packet.
- Create best match route that matches session destination IP with unresolved next hop.
- Resolve best match route next hop (neighbor should be on different port).
- Send packets that matches each Everflow ACL rule.
- Verify that packets are mirrored and destination port changed accordingly.
- Remove best match route.
- Send packets that matches each Everflow ACL rule.
- Verify that packets are mirrored and destination port changed accordingly.

### Test case \#4 - Change neighbor MAC address.

#### Test objective

Verify that session destination MAC address is changed after neighbor MAC address update.

#### Test steps

- Create route that matches session destination IP with unresolved next hop.
- Resolve route next hop.
- Send packets that matches each Everflow ACL rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packet header.
- In case of egress mirroring, verify that TTL of the mirrored packet is decremented comparing with the injected packet.
- Verify that mirrored packet payload is equal to sent packet.
- Change neighbor MAC address.
- Send packets that matches each Everflow ACL rule.
- Verify that DST MAC address in mirrored packet header is changed accordingly.

### Test case \#5 - Resolved ECMP route.

#### Test objective

Verify that session with resolved ECMP route has active state.

#### Test steps

- Create ECMP route that matches session destination IP with two unresolved next hops.
- Resolve route next hops.
- Send packets that matches each Everflow ACL rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packets header.
- In case of egress mirroring, verify that TTL of the mirrored packet is decremented comparing with the injected packet.
- Verify that mirrored packets payload is equal to sent packet.

### Test case \#7 - ECMP route change (remove next hop not used by session).

#### Test objective

Verify that after removal of next hop that was used by session from ECMP route session state is active.

#### Test steps

- Create ECMP route that matches session destination IP with two unresolved next hops.
- Resolve route next hops.
- Send packets that matches each Everflow ACL rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packet header.
- In case of egress mirroring, verify that TTL of the mirrored packet is decremented comparing with the injected packet.
- Verify that mirrored packets payload is equal to sent packets.
- Remove next hop that is used by session.
- Send packets that matches each Everflow ACL rule.
- Verify that packets are mirrored and destination port changed accordingly.

### Test case \#8 - Policer enforced DSCP value/mask test.

#### Test objective

#### Test steps


## TODO
- Everflow+VLAN test configuration and test cases (Add VLAN, move destination port in VLAN, test everflow; move destination port out of VLAN, test everflow)
- Everflow+LAG test configuration and test cases (separate ansible playbook)

## Open Questions
