---
title: Output Formatting
description: Format Falco Alerts for Containers and Kubernetes
linktitle: Output Formatting
aliases:
- ../alerts/formatting
weight: 20
---

Previous guides introduced the [Output Fields of Falco Rules](/docs/rules/basic-elements/#output) and provided [Guidelines](/docs/rules/style-guide/#output-fields) on how to use them. This section specifically highlights additional global formatting options for your deployment, complementing the information previously provided.

First, note that you can always manually edit any rule to output more or fewer fields by simply editing `output` section for a given rule within the rules' YAML file.


```yaml
# Original rule output
output: Executing binary not part of base image (proc_sname=%proc.sname user=%user.name process=%proc.name proc_exepath=%proc.exepath parent=%proc.pname command=%proc.cmdline terminal=%proc.tty %container.info)

# Remove output fields
output: %proc.sname %user.name

# Add more output fields again
output: %proc.sname %user.name %proc.is_exe_upper_layer
```

However, adding the same output field to multiple rules manually can be tedious. Thankfully Falco has a shortcut to make this eaier.

Falco inherently supports event decoration for associated [Container](https://falco.org/docs/reference/rules/supported-fields/#field-class-container) and [Kubernetes](https://falco.org/docs/reference/rules/supported-fields/#field-class-k8s) metadata using a special placeholder field  (`%container.info`) in a rule's output section.

To take advantage of event decoration, you need to run Falco with either the `-pk` or `-pc` command-line option.

If you have an even more customized use case, Falco also provides a `-p` flag where you can define additional custom output fields to be included in each rule.

## Example Rule

```yaml
- rule: Drop and execute new binary in container
  desc: SKIPPED
  condition: 
    spawned_process
    and container
    and proc.is_exe_upper_layer=true 
    and not container.image.repository in (known_drop_and_execute_containers)
  output: Executing binary not part of base image (proc_sname=%proc.sname user=%user.name process=%proc.name proc_exepath=%proc.exepath parent=%proc.pname command=%proc.cmdline terminal=%proc.tty %container.info)
  priority: CRITICAL
  tags: [maturity_stable, container, process, mitre_persistence, TA0003, PCI_DSS_11.5.1]
```

*Scenario 1*

The rule outputs include `%container.info`, but Falco is started without any command line flags:

```bash 
sudo /usr/bin/falco  -c /etc/falco/falco.yaml -r falco_rules_test.yaml
```

In this case Falco will output `%container.id` and `%container.name` but no other container metadata will be displayed:

```bash
03:00:45.104332605: Critical Executing binary not part of base image (proc_sname=bash user=root process=sleep proc_exepath=/tmp/sleep parent=bash command=sleep 10000 terminal=34816 container_id=0fdb3cd5b5fc container_name=optimistic_newton)
```

*Scenario 2*

The rule outputs include `%container.info`, and Falco is started with the `-pc` flag:

```bash 
sudo /usr/bin/falco  -c /etc/falco/falco.yaml -r falco_rules_test.yaml -pc
```

The output includes the default container fields:

```bash
03:02:52.019002207: Critical Executing binary not part of base image (proc_sname=bash user=root process=sleep proc_exepath=/tmp/sleep parent=bash command=sleep 10000 terminal=34816 container_id=0fdb3cd5b5fc container_image=ubuntu container_image_tag=latest container_name=optimistic_newton)
```

*Scenario 3*

The rule outputs include `%container.info`, and Falco is started with the `-pk` flag:

```bash 
sudo /usr/bin/falco  -c /etc/falco/falco.yaml -r falco_rules_test.yaml -pk
```

Output includes the default container fields and the default Kubernetes fields:

```bash
03:03:23.573329751: Critical Executing binary not part of base image (proc_sname=bash user=root process=sleep proc_exepath=/tmp/sleep parent=bash command=sleep 10000 terminal=34816 container_id=0fdb3cd5b5fc container_image=ubuntu container_image_tag=latest container_name=optimistic_newton k8s_ns=my_ns k8s_pod_name=my_pod_name)
```

*Scenario 4*

The rule outputs include `%container.info`, and you run Falco with the `-p` flag while providing custom output fields:

```bash 
sudo /usr/bin/falco  -c /etc/falco/falco.yaml -r falco_rules_test.yaml -p "k8s_pod_uid=%k8s.pod.uid proc_pexepath=%proc.pexepath"
```

The output includes your custom output fields along with the default `%container.id` and `%container.name` because the rule still contained the `%container.info` placeholder field:

```bash
03:05:34.475000383: Critical Executing binary not part of base image (proc_sname=bash user=root process=sleep proc_exepath=/tmp/sleep parent=bash command=sleep 10000 terminal=34816 container_id=0fdb3cd5b5fc container_name=optimistic_newton) k8s_pod_uid=my_pod_uid proc_pexepath=/usr/bin/bash
```
