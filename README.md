# Support for automated on-demand execution of containers

This document intends to explain a possible method to automate the execution of containers on demand, in the EGI infrastructure. The idea is to use the [Infrastructure Manager](https://www.grycap.upv.es/im/index.php) (IM) to create virtual infrastructure and instruct it to run a container. Thanks to IM's [CLI](https://github.com/grycap/im-client) support, this execution can be easily automated in scripts.

The main steps are:
1. Installation and configuration of the authentication. This is done with the [oidc-agent](https://github.com/indigo-dc/oidc-agent).
2. Installation and configuration of [im-client](https://github.com/grycap/im-client).
3. Definition of a [TOSCA template](https://docs.oasis-open.org/tosca/TOSCA/v1.0/os/TOSCA-v1.0-os.html) to define the actions to perform.
4. Execution of the im-client with the TOSCA template.

**Steps 1 and 2** require to be done only once (unless there is a change of provider and new details have to be added). In **Step 3**, different templates will have to be created or modified for different container images. **Step 4** is the actual step that has to be run routinarily.

Specific instructions for each step are given below. You can skip some of the steps if, for example, you already have oidc-agent installed and configured in your system.

Note that these steps are prepared for an **Ubuntu system**. Other systems will require different but similar steps. **Python3** and **pip** must be available.

---

## 1. OIDC-AGENT

[oidc-agent](https://github.com/indigo-dc/oidc-agent) is a tool that allows you to use tokens from Check-in in, amongst others, in other applications. It is very convenient as it requests access tokens automatically when they expire.

### 1.1 Installation

```console
sudo sh -c "curl -N repo.data.kit.edu/repo-data-kit-edu-key.gpg | gpg --dearmor  > /usr/share/keyrings/kitrepo-archive.gpg"
sudo sh -c "echo 'deb [signed-by=/usr/share/keyrings/kitrepo-archive.gpg] https://repo.data.kit.edu/debian/testing ./' >> /etc/apt/sources.list"
sudo apt update
sudo apt install oidc-agent
```

*Note: Maybe in your system it is not necessary to add the keys. Herein, it is recommended to run only the last two commands and, if the installation fails for this reason, then run the four commands.*

### 1.2 Configuration

The name "**egi**" will be used for to reference this oidc-agent account. You can change this name for any other name of your preference.

```console
eval `oidc-keychain`
oidc-gen --pub --issuer https://aai.egi.eu/auth/realms/egi --scope "email eduperson_entitlement eduperson_scoped_affiliation eduperson_unique_id profile" egi
```

After this, a browser will open. You will have to complete the [Check-in](https://docs.egi.eu/users/aai/check-in/) authentication as usual.

### 1.3 Before execution

When a new terminal is open, these instructions must be executed to configure oidc-agent:

```console
eval `oidc-keychain`
export OIDC_AGENT_ACCOUNT=egi
```

(again, "**egi**" is the name of the oidc-agent account, and a different name may have been used).

These instructions can be permanently added *e.g.* to the `~/.profile` or `~/.bashrc` file, so that they are executed automatically on system startup.

### 1.4 Additional oidc-agent commands
- `oidc-gen -l`: Lists all available accounts
- `oidc-token egi`: Prints the token for the account "egi".
- ``fedcloud token check --oidc-access-token `oidc-token egi` ``: Checks token expiry time

OR: ``printf '%(%F %T)T\n' `oidc-token --env $OIDC_AGENT_ACCOUNT | grep -oP '(?<=OIDC_EXP=).*?(?=;)'` ``
- `oidc-gen egi --reauthenticate`: Obtain a new refresh token when it expires after (typically) 13 months.

---

## 2. IM-CLIENT

[im-client](https://github.com/grycap/im-client) is a tool to use the Infrastructure Manager from the command line.

There are two main ways to use im-client: directly via **Python** or via a **Docker container**. Both are equally valid but, since the current Docker image does not have oidc-agent preinstalled, this section will describe the use of the Python version.

### 2.1 Installation

For the installation, virtual environments will be used. Virtual environments help to keep a clean environment by managing Python packages for different projects. If this is not desired, this step can be skipped:

```console
python3 -m venv cli
source cli/bin/activate
```

Then, we proceed with the packages installation:

```console
pip install RADL
pip install requests
pip install IM-client
```

If virtual environments are used, note that the environment has to be activated via:

```console
source <PATH>/cli/bin/activate
```

Otherwise, the command _**im-client**_ will not be found outside the environment.

### 2.2 Configuration
To configure im-client, two files are needed:

- im_client.cfg. ([example file](example_files/im_client.cfg))
- auth.dat. ([example file](example_files/auth.dat))

*im_client.cfg* is fairly simple, and the one in the example can be used as it is, without any modification. The file must be placed either as `im_client.cfg` in the current directory where im-client is executed OR as `~\.im_client.cfg` (in the home directory of the user).

```
[im_client]
restapi_url=https://appsgrycap.i3m.upv.es:31443/im/
auth_file=auth.dat
```

*auth.dat* contains the details of the provider to be used. The first line is for the IM itself and can be kept unmodified. Then, a line must be added for each provider that will be used to deploy the virtual infrastructure (VMs).

```
id = im; type = InfrastructureManager; token = command(oidc-token egi)
id = iisas; type = OpenStack; host = https://cloud.ui.savba.sk:5000/v3/; username = egi.eu; tenant = openid; password = command(oidc-token egi); auth_version = 3.x_oidc_access_token; domain = vo.access.egi.eu
```
When using EGI infrastructure, the example line can be reused, but there are two attributes that must be modified according to the EGI provider that will be used:
- host, which is is the Openstack endpoint of the provider.
- domain, which is the name of the Opestack project for the VO. Note that this is normally the same as the VO (*e.g.* vo.access.egi.eu) but it might differ in some providers (*e.g.* VO:vo.access.egi.eu), so it is good to double check.

A convenient way to retrieve the information about the host and domain you want to use is with the [fedcloudclient](https://fedcloudclient.fedcloud.eu/) tool. If you need to install this tool, specific steps are given in [this section](https://github.com/EGI-ILM/fedcloud-terraform#3-install-fedcloudclient).

The **host** can be obtained via: `fedcloud endpoint list -a` under the `URL` column.

And the **domain** can be obtained via: `fedcloud endpoint projects -a` under the `Name` column.

---

## 3. TOSCA TEMPLATES

One way to instruct IM what to do is with a [TOSCA template](https://docs.oasis-open.org/tosca/TOSCA/v1.0/os/TOSCA-v1.0-os.html). TOSCA (*Topology and Orchestration Specification for Cloud Applications*) is an OASIS standard.

An example TOSCA template that can be used as a reference is provided as [an example](example_files/tosca_docker.yml). This template creates a VM, installs Docker inside and runs a container. In the example, the VM requested has 2 vCPU and 2 GB RAM (or the closest specs allowed by the provider) and runs the container `ghcr.io/ivoa/oligia-webtop:ubuntu-2022.01.13` (see: [Oligia repository](https://github.com/ivoa/ivoa-desktop)).

The main changes that can be made to this example template are described in the following subsections.

### 3.1 Number of CPUs:
```yaml
inputs:        
    num_cpus:
      type: integer
      description: Number of virtual cpus for the VM
      default: 2
```
### 3.2 Amount of RAM:
```yaml
    mem_size:
      type: scalar-unit.size
      description: Amount of memory for the VM
      default: 2 GB
```
### 3.3 Storage:
```yaml
    storage_size:
      type: scalar-unit.size
      description: Size of the extra HD added to the instance (Set 0 if disk is not needed)
      default: 10 GB
```
### 3.4 VM Image to use:
```yaml
    os_image:
      type: string
      description: OS Cloud base image
      default: appdb://IISAS-FedCloud/egi.ubuntu.18.04?vo.access.egi.eu
```

This is *important* as this is what tells IM which provider will be used. Of course, your `auth.dat` should have the details for this provider, and the user must have permissions on it (`oidc-agent` takes care of the access token). The string:

```appdb://IISAS-FedCloud/egi.ubuntu.18.04?vo.access.egi.eu```

is made of:
- The name of the provider.
- The VM image to use.
- The VO to use.

Although these details can be obtained via Fedcloudclient, another way to collect them, if they are not known, is via [AppDB](https://appdb.egi.eu/browse/vos/cloud). Example:
- `https://appdb.egi.eu/store/site/iisas-fedcloud`
- `https://appdb.egi.eu/store/vappliance/egi.ubuntu.18.04` OR `https://appdb.egi.eu/store/vappliance/egi.ubuntu.22.04`
- `https://appdb.egi.eu/store/vo/vo.access.egi.eu`

Not all sites support all VOs and all VM images, so this has to be found out in advance.

### 3.5 The Docker container to be used
```yaml
    ansible_tasks:
      type: string
      description: Ansible tasks (In case of using double quotes you have to escape it with \)
      default: |
        - name: Launch im container
          docker_container:
            name: oligia
            image: 'ghcr.io/ivoa/oligia-webtop:ubuntu-2022.01.13'
            state: started
            ports:
            - '3000:3000'
```

### 3.6 The port to be open for external access
```yaml
    ansible:
      type: tosca.nodes.ec3.Application
      capabilities:
        endpoint:
          properties:
            port: 3000
            protocol: tcp
```

---

## 4. CONTAINER EXECUTION

Once *oidc-agent* and *im-client* have been installed and a TOSCA template has been created, the setup can be considered finished, and it is time to do the routinary Docker execution.

## 4.1 Execute container

```console
im_client.py create tosca_docker.yml
```

This will create a VM and run the `oligia-webtop` Docker container inside. It will return an infrastructure identifier **<INFRA_ID>**.

## 4.2 Get VM IP address
```console
im_client.py getoutputs <INFRA_ID> | grep node_ip | cut -d' ' -f 3
```

_Note: If the VM is in the process of creation, the IP address might not be available. You might need to add a delay before trying the command or run it in a loop until the datum is available._

## 4.3 SSH into container
```console
im_client.py ssh <INFRA_ID>
```

## 4.4 Remove container (and VM)
```console
im_client.py destroy <INFRA_ID>
```

## 4.5 Other useful im-client commands
- `im_client.py list`: Lists all infrastructures available.
- `im_client.py getstate <INFRA_ID>`: Gets status of infrastructure.
- `im_client.py getvminfo <INFRA_ID> 0`: Gets info of the first VM (0) of an infrastructure.
- `im_client.py getoutputs <INFRA_ID>`: Gets all outputs from the infrastructure creation.

