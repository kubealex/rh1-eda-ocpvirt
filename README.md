# Red Hat One - Automating the Unfortunate: Event-Driven Ansible for OpenShift Virtualization

Welcome to the repository for the session "Red Hat One - Automating the Unfortunate: Event-Driven Ansible for OpenShift Virtualization"

In this repository you will find the instructions and configuration to replicate the use cases in your environment.

- [Use cases](#use-cases)
  - [Creation and provisioning of a Virtual Machine in Openshift Virtualization](#creation-and-provisioning-of-a-virtual-machine-in-openshift-virtualization)
    - [VM Creation](#vm-creation)
    - [Day 2 Operations](#day-2-operations)
    - [Automatic Red Hat Insights advisory remediation](#automatic-red-hat-insights-advisory-remediation)
  - [Remediating an alert from OCP Virtualization VM with Service Now and EDA](#remediating-an-alert-from-ocp-virtualization-vm-with-service-now-and-eda)
    - [Trigger OCPVirtLowDisk Alert](#trigger-ocpvirtlowdisk-alert)
    - [Report the incident on ITSM and resolve it](#report-the-incident-on-itsm-and-resolve-it)
  - [Dynatrace reactive monitoring with EDA](#dynatrace-reactive-monitoring-with-eda)
    - [CPU Usage Anomaly detection](#cpu-usage-anomaly-detection)
    - [Reaction to the anomaly](#reaction-to-the-anomaly)
- [Configuration](#configuration)
- [Requirements](#requirements)
  - [Red Hat Openshift Container Platform](#red-hat-openshift-container-platform)
    - [Required operators](#required-operators)
    - [Network Configuration](#network-configuration)
    - [Storage Configuration](#storage-configuration)
    - [Monitoring Configuration](#monitoring-configuration)
  - [Red Hat Ansible Automation Platform](#red-hat-ansible-automation-platform)
    - [Ansible Automation Platform Controller and Event Driven Automation Controller](#ansible-automation-platform-controller-and-event-driven-automation-controller)

## Use cases

### Creation and provisioning of a Virtual Machine in Openshift Virtualization

The configuration will create a template, **[EDA][OCP] Create VM and configure monitoring** that will provide a cloud-like experience to create a virtual machine on Openshift Virtualization.

You need to fill _Provisioning Webhook_ to match your environment configuration when running it from the AAP Controller Console.

The use case is articulated in three phases:

#### VM Creation

The VM is created by instantiating a Virtual Machine template in OCP, the first configuration steps are taken using cloud-init. At the end of the configuration the _Provisioning Webhook_ is called to trigger EDA and start the day 2 operations.

#### Day 2 Operations

Once the VM provisioning webhook has been called, the VM is:

- registered on the Red Hat Network to attach a subscription
- configured to export metrics using node-exporter for Prometheus
- attached to Red Hat Insights for CVE detection and remediation

#### Automatic Red Hat Insights advisory remediation

Once registered to Red Hat Insights, the platform will send an event that is handled by the **eda-insights** rulebook activation in EDA.

The automation that is triggered will take care of:

- Check if the advisories contain attached security fixes (CVEs)
- Check which of the advisories that contain CVEs need a reboot
- Generate remediation playbooks on Red Hat Insights
- Generate a workflow containing the remediation playbooks
- Attach an approval step to the workflow if the operation requires a reboot

### Remediating an alert from OCP Virtualization VM with Service Now and EDA

This use case will focus on automatic remediation of problems detected on a Virtual Machine deployed on Openshift virtualization.

In this specific case, we will simulate a filesystem available space exhaustion by allocating more space than available and this event will be captured by the AlertManager rule we deploy along with the VM.
It will report the issue as critical and generate the OCPVirtLowDisk alert that will be sent to the listener on the AlertManager rulebook activation, which will take care of:

- React to the OCPVirtLowDisk alert coming from OCP
- Raise a ServiceNow incident
- Resolve the issue on the VM
- Notify the resolution closing the Incident

Note that the use case will take some time to be executed, as the "Firing" event takes 5 minutes to trigger upon detection.

#### Trigger OCPVirtLowDisk Alert

During the VM deployment, we also deployed a ServiceMonitor that allows Prometheus deployed on OCP to scrape metrics from the VM's node-exporter.
We also defined a PrometheusRule to generate an alert based on the above metrics, when available disk space on a VM reaches a critical level.

To trigger the alert, it is sufficient to generate a disk saturation. To do this, SSH in the newly created VM and run:

```bash
sudo fallocate -l 150G /myfilesystem/test
```

The alert will be visible in the "Observe" section in OCP and in 5 minutes without resolution, it will become "Firing", and it will be sent to Event Driven Automation Controller for further processing.

#### Trigger OCPVirtVMNodePressure Alert

During the VM deploy, we also deployed a ServiceMonitor that allows Prometheus deployed on OCP to scrape metrics from the VM's node-exporter.
Along with that we also defined a PrometheusRule to generate an alert based on the above metrics, when CPU levels of the node hosting a VM become critical, to make sure that the VM is then live-migrated to another node.

To trigger the alert, it is sufficient to spin-up a pod who starts raising the CPU level intensively.

A pod definition example could be as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: saturate-cpu
  namespace: rh1-demo
spec:
  containers:
  - name: saturate-cpu-container
    image: ubi9
    command: ["/bin/sh", "-c"]
    args:
      - openssl speed -multi $(grep -c ^processor /proc/cpuinfo)
    resources:
      limits:
        cpu: "6"
```

The alert will be visible in the "Observe" section in OCP and in 5 minutes without resolution, it will become "Firing", and it will be sent to Event Driven Automation Controller for further processing.

#### Report the incident on ITSM and resolve it

Once EDA has been notified, it will trigger a job template execution to report the incident on the ITSM platform, creating a ticket and proceed with the resolution.

For the sake of the use case, it is a simple resolution removing the file that caused the issue, but it can be extended using FS resizing, adding a disk, etc.

### Dynatrace reactive monitoring with EDA

The last use case will cover the reaction to an event on a monitored VM, specifically a CPU saturation.
THe Dynatrace agent will monitor the VM we created and we will simulate a CPU saturation event.

#### CPU Usage Anomaly detection

We will simulate the CPU saturation VM event in the VM that is now running in OCP Virtualization, where we previously installed the Dynatrace agent.
When the CPU usage reaches 100%, Dynatrace will generate a Problem reporting the issue, we will then use the API polling of the Event Driven Automation Source for Dynatrace to intercept the problem and react accordingly.

To generate the issue we can use the template **[EDA][Dynatrace] Saturate VM CPU**, which will leverage openssl command to saturate the CPU of the VM.

#### Reaction to the anomaly

After triggering the notification, EDA will analyze and enrich the information coming from Dynatrace, open a Service Now request, resolve the issue and then proceed with the resolution of the Service Now ticket.

## Configuration

## Requirements

In this section you will find the requirements to successfully run all the use cases.

### Red Hat Openshift Container Platform

A working OCP cluster is required to run the demo, as the use case will leverage Red Hat Openshift Virtualization to create a Virtual Machine.
The Virtual Machine template is configured to use a second network interface, bridged, to make use of DHCP and make the VM easily reachable from the outside.

You can use any alternative, like MetalLB operator, to assign the VM Service a fixed IP and adapt the configuration based on your needs.

#### Required operators

You will need:

- Openshift Virtualization
- Kubernetes NMState Operator - needed to use a bridged network to reach the VMs

#### Network Configuration

The Virtual Machine expects a bridged network using an additional NIC on the Openshift nodes for DHCP and IP reachability from Ansible Controller.

Example files are provided in the [eda-ocp-virt-automation/ocp-virt-bridged-network folder](./eda-ocp-virt-automation/ocp-virt-bridged-network/)

Ensure to adjust the name of the interface!

#### Storage Configuration

The VM will be configured using two disks, that will consume storage from a default storage class, ensure to have one before proceeding.

#### Monitoring Configuration

One use case relies on OCP Monitoring.
VM-related resources are created by the provisioning playbooks, but some additional steps should be taken:

- Configure monitoring for User Defined workloads
- Create an AlertManager receiver for user workloads

Configuration snippets can be found in the [eda-ocp-virt-automation/ocp-monitoring folder](./eda-ocp-virt-automation/ocp-monitoring/)

### Red Hat Ansible Automation Platform

#### Ansible Automation Platform Controller and Event Driven Automation Controller

Relevant configuration to run the demo is provided [in the aap-setup folder](./aap-setup/).
The playbooks in it will take care of provisioning the following resources:

_On AAP Controller_

- Dedicated project(s)
- Job Templates
- Credentials, including custom for Service Now, Openshift, Red Hat Subscription Manager and Dynatrace
- Workflows

_On EDA Controller_

- Dedicated project
- Token exchange with AAP Controller
- Decision Environment
- Rulebook activations for AlertManager, Dynatrace, Webhook and Insights

A minimum configuration is available in the [config-variables.yml file](./config-variables.yml) where you need to specify the following:

<details>
  <summary>Configuration details</summary>

```yaml
# AAP Controller information

aap2_controller_url:
aap2_controller_username:
aap2_controller_password:

# EDA Controller information

eda_controller_url:
eda_controller_user:
eda_controller_password:

# Service Now information

servicenow_instance_url:
servicenow_instance_username:
servicenow_instance_password:

# Openshift Container Platform information

openshift_api_url:
openshift_admin_username:
openshift_admin_password:

# Red Hat ID Credentials

rhsm_account_username:
rhsm_account_password:

# Dynatrace related variables

dynatrace_api_url:
dynatrace_api_token:

</details>

When the variables are in place, to run the configuration:

```bash
ansible-playbook -i aap-setup/inventory aap-setup/configure-aap.yml -e @config-variables.yml
```
