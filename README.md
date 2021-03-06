# ACI Networking Integration
![Automatic ACI Configuration](/images/automatic-aci-configuration.png)

This integration extends Apprenda's automated application deployment to include automated application-level network configuration. It takes advantage of the flexibility of ACI to not only provide developers with multiple modes of application isolation, but also to connect their Kubernetes applications with on or off platform non-Kubernetes applications. Details of the installation process can be found in [**Cisco's CNI documentation**](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/kb/b_Kubernetes_Integration_with_ACI.html).

* [**Installation Video**](https://apprenda.wistia.com/medias/9cecof71wi#)
* [**Usage Video**](https://apprenda.wistia.com/medias/15fp6svfes#)
* [**Overview Video**](https://apprenda.wistia.com/medias/rvowaka2ym#)

#### Supported Software Versions
|Software|Min Version|Max Version|
|-|-|-|
|Apprenda|8.0|8.2|
|Kubernetes|1.6.X|1.7.X|
|ACC|1.6.0|1.7.1|
|ACI|3.0(1j)|3.1(1i)|
|CentOS|7|7|

## Prerequisites
* An Apprenda instance networked to EPG(s) in ACI.
* A contract residing in the common tenant for communication with the Apprenda instance.
* A L3 out configured in ACI.
* A dedicated AD account for the integration.
* A unassigned NIC cabled to leaf(s) in the ACI fabric on each ESXi Kubernetes node host.
* An AEP configured in ACI for the ports cabled to the ESXi hosts.
* Valid certificates for all encrypted connections.

## ACI Configuration
1. Use pip to install ```acc-provision``` on the master node:

        pip install acc-provision

2. Generate the sample provisioning YAML:

        acc-provision --sample > aci-containers-config.yaml

3. Edit the provisioning YAML for your environment.
    * The infrastructure VLAN ID must match that of your ACI environment.
4. Use ```acc-provision``` with the edited ```aci-containers-config.yaml``` to configure ACI:

        acc-provision -c aci-containers-config.yaml -o aci-containers.yaml -a -u username -p password

5. Configure the ```kube-nodes``` and ```kube-system``` EPGs in the provisioned tenant to consume the Apprenda contract.

## vCenter Configuration
1. Create a distributed switch configured for 1600 MTU and assign it to the prerequisite unused NIC(s).
2. Create a distributed port group configured for VLAN trunking and enable forged transmits.
3. Create additional NIC(s) on all Kubernetes nodes configured for the distributed port group.

## Node Configuration
*This process installs specific versions of Docker and Kubenetes. Any previous versions must be uninstalled before running ```config.sh```. These steps must be completed on all Kubernetes nodes.*

1. Edit the provided ```config.sh``` with the details of the respective node.
2. Run the edited ```config.sh```.
3. Mark all management interfaces as not the default route.
    * The Kubernetes API VLAN interface should be the only interface **not** marked as not the default route.
4. Reboot the node.

## Kubernetes Installation
*We'll be using ```kubeadm``` for installing Kubernetes. These steps take place on the master node unless specified otherwise.*

1. Edit the provided ```config.yaml``` and ```basic_auth.csv``` for your environment.
    * ABAC must be specified as the authorization mode for the cluster to be usable by Apprenda.
2. Copy the provided ```abac_policy.json``` and the edited ```basic_auth.csv``` to ```/etc/kubernetes```.
3. Use ```kubeadm``` with the edited ```config.yaml``` to install Kubernetes:

        kubeadm init --config config.yaml

4. Follow the instructions printed to the console by ```kubeadm init```.
    * Copy ```/etc/kubernetes/admin.conf``` to your home directory.
    * Set the ```KUBECONFIG``` environment variable.
    * Execute the ```kubeadm join``` command on each of the worker nodes.
5. Use ```kubectl``` to apply the CNI YAML generated by ```acc-provision```:

        kubectl apply -f aci-containers.yaml

6. Install ```/etc/kubernetes/pki/ca.crt``` on all Windows application and web nodes as a local computer trusted system root.
7. Add the cluster to Apprenda using the Clouds page in the SOC.

## Integration Installation
*These steps take place on the LM node unless specified otherwise. These steps require the Apprenda SDK on the LM node.*

1. Run ```generate-certificate.ps1``` to generate the credentials-encrypting certificate.
2. Copy the certificate created by step 1 and ```ProvisionNode.ps1``` to each Windows application and web node.
3. Run ```provision-node.ps1``` on each Windows application and web node to import the certificate, grant the AD account read access to the private key, and grant "Log on as service" to the AD account.
    * Pass the prerequisite AD account credentials to the script.
4. Run ```customize-archive.ps1``` to embed the AD account credentials in the archive.
5. Run ```install-integration.ps1``` to create and promote the APIC Proxy application, create all the necessary custom properties, and create the APIC Proxy bootstrapper.
    * Pass the optional consumed or provided services to add them as options to the respective custom property.
    * This script requires the [**PSApprendaApi**](https://www.powershellgallery.com/packages/PSApprendaApi) module for PowerShell. It can be installed with the following command:

            Install-Module PSApprendaApi

## Configuration
1. Launch the APIC Proxy UI and enter/save the credentials for your APIC and Kubernetes cluster.

## Verification
1. Review the APIC Proxy log (/log) and verify that the service is watching for changes to replica sets.
2. Create a new application using the provided ```connectivity-pod.yaml```.
3. Promote the application to sandbox.
4. Use APIC to verify that an ANP, EPG, contract, and filter were created.
5. Verify that the Connectivity Pod UI can be accessed by launching the application from the developer portal.
6. Use the Connectivity Pod UI to verify connectivity with external resources.
7. Demote the application.
8. Use APIC to verify that the ANP, EPG, contract, and filter were removed.

## Usage
![Component Custom Properties](/images/component-custom-properties.png)

The integration provides the following kubernetes-specific component custom properties to developers. These custom properties direct automatic ACI configuration on application promotion. ACI objects created by the integration for an application are automatically removed on demotion or deletion. An application must be patched or promoted for any changes to custom properties to take effect.

1. **APIC Tenant** - Used for specifying the tenant to configure in APIC. This must coincide with the cluster that the developer's application will be deployed to.
2. **Application Ports** - Used for specifying the set of ports and protocols over which the developer's application will accept traffic. Each entry must be supplied in the format: name/port/protocol ([a-z0-9-]+/1-65535|unspecified/tcp|udp|icmp|unspecified). "unspecified" may be used as a wildcard for the port and/or protocol. The port must be set to "unspecified" when the protocol is ICMP or unspecified.
3. **Apprenda Network** - Used for specifying the group of applications that will be allowed access to the developer's application. Selecting "Common" will result in the application being added to the "kube-default" EPG.
4. **Consumed Services** - Used for specifying services external to the specified Apprenda Network that the developer's application requires access to. These options must correspond to contracts residing in the specified APIC tenant or in the common tenant.
5. **Provided Services** - Used for specifying services external to the specified Apprenda Network that the developer's application provides. These options must correspond to contracts residing in the specified APIC tenant or in the common tenant.

## Troubleshooting
The log can be accessed by launching the APIC Proxy and navigating to /log.

* Failure to save credentials:
    1. Check that all Windows nodes have the APICProxy certificate installed under LocalMachine/Personal/Certificates.
* Failure to promote:
    1. Check that all Windows nodes can access the developer portal.
    2. Check that the correct APIC URL and credentials are configured.
    3. Check that all Windows nodes have the APICProxy certificate installed under LocalMachine/Personal/Certificates and that the AD account has been granted read access to the private key.
    4. Check that the ACI CNI has been correctly configured for the ACI environment.
* Failure to remove ACI objects on demote:
    1. Check that the workload had been removed from Kubernetes.
    2. Check that the correct Kubernetes URL and credentials are configured.
    3. Check that all Windows nodes have the APICProxy certificate installed under LocalMachine/Personal/Certificates and that the AD account has been granted read access to the private key.
    4. Check that the AD account has been granted "Log on as service" in Local Group Policy on all Windows nodes.

## Known Issues
* [APPRENDA-24382](https://apprenda.atlassian.net/browse/APPRENDA-24382): During Apprenda installation, host entries for the default cloud are added the hosts file on all Windows nodes. These entries must be removed for the integration to connect to the developer portal.
* [APPRENDA-25141](https://apprenda.atlassian.net/browse/APPRENDA-25141): The first consumed or provided service cannot be toggled in the developer portal UI.
* [APPRENDA-25147](https://apprenda.atlassian.net/browse/APPRENDA-25147): Editing a custom property with multiple default values untags all but one of the values as a default.
* [AI-594](https://apprenda.atlassian.net/browse/AI-594): Stopped or failed promotions from sandbox to published may result in missing network configuration in ACI. Patching or promoting the application fixes the issue.
