- [Overview](#overview)
  - [Scope](#scope)
  - [Summary of the existing everflow test plan](#summary-of-the-existing-everflow-test-plan)
  - [Extend the test plan to cover both ingress and egress ACL](#extend-the-test-plan-to-cover-both-ingress-and-egress-acl)
  - [Related **DUT** CLI commands](#related-dut-cli-commands)
    - [sonic-cfggen](#sonic-cfggen)
    - [acl-loader](#acl-loader)
    - [aclshow](#aclshow)
    - [config mirror_session](#config-mirror_session)
- [Test structure](#test-structure)
  - [Overall structure](#overall-structure)
  - [Setup configuration](#setup-configuration)
    - [Scripts for generating configuration on SONIC](#scripts-for-generating-configuration-on-sonic)
    - [Ansible scripts to setup and run test](#ansible-scripts-to-setup-and-run-test)
      - [everflow_testbed.yml](#everflow_testbedyml)
    - [Setup of DUT switch](#setup-of-dut-switch)
- [PTF Test](#ptf-test)
  - [Input files for PTF test](#input-files-for-ptf-test)
  - [Traffic validation in PTF](#traffic-validation-in-ptf)
- [Test cases](#test-cases)
  - [Test case \#1 - Resolved route](#test-case-1---resolved-route)
    - [Test objective](#test-objective)
    - [Test steps](#test-steps)
  - [Test case \#2 - Longer prefix route with resolved next hop](#test-case-2---longer-prefix-route-with-resolved-next-hop)
    - [Test objective](#test-objective-1)
    - [Test steps](#test-steps-1)
  - [Test case \#3 - Remove longer prefix route.](#test-case-3---remove-longer-prefix-route)
    - [Test objective](#test-objective-2)
    - [Test steps](#test-steps-2)
  - [Test case \#4 - Change neighbor MAC address.](#test-case-4---change-neighbor-mac-address)
    - [Test objective](#test-objective-3)
    - [Test steps](#test-steps-3)
  - [Test case \#5 - Resolved ECMP route.](#test-case-5---resolved-ecmp-route)
    - [Test objective](#test-objective-4)
    - [Test steps](#test-steps-4)
  - [Test case \#6 - ECMP route change (add next hop).](#test-case-6---ecmp-route-change-add-next-hop)
    - [Test objective](#test-objective-5)
    - [Test steps](#test-steps-5)
  - [Test case \#7 - ECMP route change (remove next hop used by session).](#test-case-7---ecmp-route-change-remove-next-hop-used-by-session)
    - [Test objective](#test-objective-6)
    - [Test steps](#test-steps-6)
  - [Test case \#8 - ECMP route change (remove next hop not used by session).](#test-case-8---ecmp-route-change-remove-next-hop-not-used-by-session)
    - [Test objective](#test-objective-7)
    - [Test steps](#test-steps-7)
  - [Other possible tests](#other-possible-tests)
- [TODO](#todo)
- [Open Questions](#open-questions)

## Overview
The purpose is to test functionality of Everflow on the SONIC switch DUT with and without LAGs configured, closely resembling production environment.
The test assumes all necessary configuration, including Everflow session and ACL rules, LAG configuration and BGP routes, are already pre-configured on the SONIC switch before test runs.

### Scope
The test is targeting a running SONIC system with fully functioning configuration.
The purpose of the test is not to test specific SAI API, but functional testing of Everflow on SONiC system, making sure that traffic flows correctly, according to BGP routes advertised by BGP peers of SONIC switch, and the LAG configuration.

NOTE: Everflow+LAG test will be able to run **only** in the testbed specifically created for LAG.

### Summary of the existing everflow test plan

The existing everflow test plan only covers ingress ACL. And ingress everflow ACL table is binded to all ports. ACL rules are added to that ACL table. Then the PTF container is used for injecting packets. Two packets directions are tested: SPINE -> TOR ports and TOR -> SPINE ports. When the injected packets hit any of the configured ACL rules, the packets will be mirrored to configured mirror session. GRE tunnel is used for sending mirrored packets with src, dst IP and other parmeters configured in the mirror session. Below test cases are used for verifying that the DUT switch can properly forwared the mirrored packets according to different routing configurations for mirror session destination.

* testcase 1 - Resolved route
* testcase 2 - Longer prefix route with resolved next hop
* testcase 3 - Remove longer prefix route
* testcase 4 - Change neighbor MAC address
* testcase 5 - Resolved ECMP route
* testcase 6 - ECMP route change (add next hop)
* testcase 7 - ECMP route change (remove next hop used by session)
* testcase 8 - ECMP route change (remove next hop not used by session)

### Extend the test plan to cover both ingress and egress ACL

After egress ACL is supported, this test plan needs to cover both ingress and egress ACL. But for supporting swiches that do not support egress yet, an ansible playbook variable is required for indicating egress ACL capability. Value for the variable can be passed from CLI.

To cover egress ACL everflow testing, the ACL tables should not be binded to all ports. And different ACL tables for ingress and egress ACL need to be tested. Based on this requirement, totally 4 ACL tables should be configured before testing:

ACL Table | Type | Bind to | Stage | Packet flow | Description
----------|------|---------|-------|-------------|------------
TORIN | MIRROR | TOR ports | Ingress | TOR -> SPINE | Ingress ACL, packets injected into TOR ports, hit ACL, mirrored to mirror session destination
SPINEIN | MIRROR | SPINE ports | Ingress | SPINE -> TOR | Ingress ACL, packets injected into SPINE ports, hit ACL, mirrored to mirror session destination
TOROUT | MIRROR | TOR ports | Egress | SPINE -> TOR | Egress ACL, packets injected into SPINE ports, forwarded to TOR ports, hit ACL, mirrored to mirror session destination
SPINEOUT | MIRROR | SPINE ports | Egress | TOR -> SPINE | Egress ACL, packets injected into TOR ports, forwarded to SPINE ports, hit ACL, mirrored to mirror session destination

For switches do not support egress ACL, only the first two ACL tables will be configured for testing.

### Related **DUT** CLI commands

The existing test plan uses a script calling swsssdk to configure mirror session. Calling SDK to configure DUT switch is not user friendly. Now DUT switch supports configuring mirror session using CLI command `config mirror_session`.

Summary of the CLI commands that will be used for configuring DUT.

#### sonic-cfggen

It is for configuring ACL tables reading current ACL configuration. For example:
```
# sonic-cfggen -j <configuration_json_file> --write-to-db
# sonic-cfggen -d -v ACL_TABLE
# sonic-cfggen -d -v ACL_RULE
```

#### acl-loader
 
This tool is for configuring ACL rules. For example:
```
# acl-loader update full <acl_rule_configuration_json_file>
```

#### aclshow
This tool is for collecting ACL rule couters. For examle:
```
# aclshow -a
```

#### config mirror_session
This tool is for configuring everflow mirror session. For example:
```
# config mirror_session add <session_name> <src_ip> <dst_ip> <dscp> <ttl> [gre_type] [queue]
```

## Test structure 

### Overall structure

The ansible test playbook will have 3 parts:
1. Add everflow configuration
2. Run everflow testing
3. Clear everflow configuration

Everflow test consists of a number of test cases, and each of them will include the following steps:

1. Run lognanalyzer 'init' phase
2. Run Everflow test case
3. Run loganalyzer 'analyze' phase

### Setup configuration

Everflow configuration should be created using ansible playbook on the DUT before running the test.

#### Scripts for generating configuration on SONIC

There will be j2 template files for generating ACL tables and ACL rules configurations. Ansible playbook will generate ACL tables and ACL rules json configuration files to DUT based on these templates, switch capability and running topology. Then use `sonic-cfggen` and `acl-loader` CLI commands to load the configurations.

When ingress ACL rules are used, they should be associated with ACL table which only binded to ports that the PTF packets will be injected into. Similarly, when egress ACL rules are used, they should be associated with ACL table which only binded to egress ports of the injected PTF packets.

#### Ansible scripts to setup and run test

##### everflow_testbed.yml

When run test_sonic.yml with testcase_name=everflow_testbed, the everflow_testbed.yml script will run and do the following:

1. Generate JSON files and apply them on the switch.
2. Run test.
3. Clean up dynamic configuration and temporary configuration on the DUT.

Different sets of JSON file will be generated for different test scenarios:
1. Ingress ACL on TOR ports, inject packets into TOR ports. Packets hit ACL, then mirrored to mirror session destination
2. Ingress ACL on SPINE ports, inject packets into SPINE ports. Packets hit ACL, then mirrored to mirror session destination
3. Egress ACL on TOR ports, inject packets into SPINE ports. Packets forwarded to TOR ports, hit ACL, then mirrored to mirror session destination
4. Egress ACL on SPINE ports, inject packets into TOR ports. Packets forwarded to SPINE ports, hit ACL, then mirrored to mirror session destination

Everflow test consists of a number of subtests, and each of them will include the following steps:

1. Run lognanalyzer 'init' phase
2. Run Everflow Sub Test
3. Run loganalyzer 'analyze' phase

Everflow subtests will be implemented in the PTF (everflow_tb_test.py). Every subtest will be implemented in a separate class.

#### Setup of DUT switch
Setup of SONIC DUT will be done by Ansible script. During setup Ansible playbook will generate ACL tables and ACL rules json configuration files to DUT based on these templates, switch capability and running topology. Then use `sonic-cfggen` and `acl-loader` CLI commands to load the configurations. Mirror session can be configured using `config mirror_session`.

Add mirror_session:
```
# config mirror_session add session_1 1.1.1.1 2.2.2.2 8 64 0x6558 0
# acl-loader show session
Name       Status    SRC IP     DST IP    GRE     DSCP    TTL    Queue
---------  --------  ---------  --------  ------  ------  -----  -------
session_1  inactive  1.1.1.1    2.2.2.2   0x6558  8       64     0
```

Sample json configuration files for adding ACL tables. Use CLI `sonic-cfggen -j <acl_table_configuration_filename> --write-to-db` to load the configurations.

everfow_acl_table.json, ingress, tor->spine
```
{
    "ACL_TABLE": {
        "TORIN": {
            "policy_desc": "TOR ports ingress", 
            "ports": [
                "Ethernet64", "Ethernet68", "Ethernet72", "Ethernet76", "Ethernet80", "Ethernet84", "Ethernet88", "Ethernet92", "Ethernet96", "Ethernet100", "Ethernet104", "Ethernet108", "Ethernet112", "Ethernet116", "Ethernet120", "Ethernet124"
            ], 
            "type": "MIRROR"
        }
    }
}
```

everfow_acl_table.json, ingress, spine->tor
```
{
    "ACL_TABLE": {
        "SPINEIN": {
            "policy_desc": "SPINE ports ingress", 
            "ports": [
                "Ethernet0", "Ethernet4", "Ethernet8", "Ethernet12" "Ethernet16", "Ethernet20", "Ethernet24", "Ethernet28", "Ethernet32", "Ethernet36", "Ethernet40", "Ethernet44", "Ethernet48", "Ethernet52", "Ethernet56", "Ethernet60"
            ], 
            "type": "MIRROR"
        }
    }
}
```

everfow_acl_table.json, egress, tor->spine
```
{
    "ACL_TABLE": {
        "SPINEOUT": {
            "policy_desc": "SPINE ports egress", 
            "ports": [
                "Ethernet0", "Ethernet4", "Ethernet8", "Ethernet12" "Ethernet16", "Ethernet20", "Ethernet24", "Ethernet28", "Ethernet32", "Ethernet36", "Ethernet40", "Ethernet44", "Ethernet48", "Ethernet52", "Ethernet56", "Ethernet60"
            ], 
            "type": "MIRROR"
        }
    }
}
```


everfow_acl_table.json, egress, spine->tor
```
{
    "ACL_TABLE": {
        "TOROUT": {
            "policy_desc": "TOR ports egress", 
            "ports": [
                "Ethernet64", "Ethernet68", "Ethernet72", "Ethernet76", "Ethernet80", "Ethernet84", "Ethernet88", "Ethernet92", "Ethernet96", "Ethernet100", "Ethernet104", "Ethernet108", "Ethernet112", "Ethernet116", "Ethernet120", "Ethernet124"
            ], 
            "type": "MIRROR"
        }
    }
}
```

everflow_acl_rule.json
For different test scenario, acl rule should be added for different ACL rule table. Below example is for the ingress tor->spine scenario, ACL table is TORIN. CLI command for loading this config: `acl-loader update full <acl_rule_configuration_json_file> --session_name=<mirror_session_name`.

```
{
    "acl": {
        "acl-sets": {
            "acl-set": {
                "TORIN": {
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
                            }
                        }
                    }
                }
            }
        }
    }
}
```
## PTF Test

### Input files for PTF test

PTF test will generate traffic between ports and make sure it mirrored according to the configured Everflow session and ACL rules. Depending on the testbed topology and the existing configuration (e.g. ECMP, LAGS, etc) packets may arrive to different ports. Therefore ports connection information will be generated from the minigraph and supplied to the PTF script.

### Traffic validation in PTF
Depending on the test PTF test will verify the packet arrived or dropped.

## Test cases

Each test case will be additionally validated by the loganalizer utility.

Each test case will add dynamic Everflow ACL rules at the beginning and remove them at the end.

Each test case will run traffic for persistent and dynamic Everflow ACL rules.

Each test case will analyze Everflow packet header and payload (if mirrored packet is equal to original).

### Test case \#1 - Resolved route

#### Test objective

Verify that session with resolved route has active state.

#### Test steps

- Create route that matches session destination IP with unresolved next hop.
- Resolve route next hop.
- Send packets that matches each Everflow ACL rule.
- Verify that packet mirrored to appropriate port.
- Analyze mirrored packet header.
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
- Verify that mirrored packets payload is equal to sent packet.

### Test case \#6 - ECMP route change (add next hop).

#### Test objective

Verify that insertion of additional next hop to ECMP group doesn't affects session DST MAC and port.

#### Test steps

- Create ECMP route that matches session destination IP with two unresolved next hops.
- Resolve route next hops.
- Send packets that matches each Everflow ACL rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packet header.
- Verify that mirrored packet payload is equal to sent packet.
- Add resolved next hop to ECMP route.
- Send packets that matches each Everflow rule.
- Verify that packets are mirrored to the same port.
- Verify that mirrored packets have the same DST MAC.

### Test case \#7 - ECMP route change (remove next hop used by session).

#### Test objective

Verify that removal of next hop that is not used by session doesn't cause DST port and MAC change.

#### Test steps

- Create ECMP route that matches session destination IP with two unresolved next hops.
- Resolve route next hops.
- Send packets that matches each Everflow rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packet header.
- Verify that mirrored packets payload is equal to sent packets.
- Remove next hop that is not used by session.
- Send packets that matches each Everflow rule.
- Verify that packets are mirrored to the same port.
- Verify that mirrored packets have the same DST MAC.

### Test case \#8 - ECMP route change (remove next hop not used by session).

#### Test objective

Verify that after removal of next hop that was used by session from ECMP route session state is active.

#### Test steps

- Create ECMP route that matches session destination IP with two unresolved next hops.
- Resolve route next hops.
- Send packets that matches each Everflow ACL rule.
- Verify that packets mirrored to appropriate port.
- Analyze mirrored packet header.
- Verify that mirrored packets payload is equal to sent packets.
- Remove next hop that is used by session.
- Send packets that matches each Everflow ACL rule.
- Verify that packets are mirrored and destination port changed accordingly.

### Other possible tests

## TODO
- Everflow+VLAN test configuration and testcases (Add VLAN, move destination port in VLAN, test everflow; move destination port out of VLAN, test everflow)
- Everflow+LAG test configuration and testcases (separate ansible playbook)

## Open Questions
