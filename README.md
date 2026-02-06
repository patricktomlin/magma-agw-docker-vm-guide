# Deploying Magma AGW in Docker with Docker-Based Orc8r

## Table of Contents

* [Purpose](#purpose)
* [Prerequisites](#prerequisites)
* [High-Level Architecture Basics](#high-level-architecture-basics)
* [Networking and DNS Requirements](#networking-and-dns-requirements)
* [Explanation of Certificates](#explanation-of-certificates)
* [Important Orc8r Ports](#important-orc8r-ports)
* [Step-by-Step AGW Deployment](#step-by-step-agw-deployment)
  * [Installing Ubuntu in a VM](#installing-ubuntu-in-a-vm)
  * [Installing the Magma Prerequisites](#installing-the-magma-prerequisites)
  * [Deploying AGW](#deploying-agw)
* [Troubleshooting](#troubleshooting)

---

## Purpose

This guide is intended for contributors to Magma Core who want to run a Magma Access Gateway (AGW) using Docker inside a virtual machine and connect it to a Docker-based deployment of the Magma Orchestrator (Orc8r).

Other deployment types may work with similar steps, but this is not guaranteed. **This guide is not suitable for production use**: it relies on self-signed certificates and does not focus on security hardening.

## Prerequisites

* A working Magma Orc8r and NMS deployment running in Docker.
* An organization and user already created in NMS.
* Network connectivity between the AGW VM and the Orc8r host.
* A static IP address for the Orc8r host.

If you do not yet have Orc8r and NMS running, the following guide is recommended:

* [https://medium.com/@pauloskidus48/a-beginner-friendly-git-and-docker-workflow-for-contributing-to-magmas-nms-fba541eb8443](https://medium.com/@pauloskidus48/a-beginner-friendly-git-and-docker-workflow-for-contributing-to-magmas-nms-fba541eb8443)
 
### Official Reference Documentation

* Magma AGW (Docker): [https://magma.github.io/magma/docs/lte/deploy_install_docker](https://magma.github.io/magma/docs/lte/deploy_install_docker)
* Magma prerequisites: [https://magma.github.io/magma/docs/basics/prerequisites](https://magma.github.io/magma/docs/basics/prerequisites)

---

## High-Level Architecture Basics 

In this deployment:

* Orc8r runs in Docker inside a Linux virtual machine
* AGW runs in Docker inside a separate Linux virtual machine

---

## Networking and DNS Requirements

If you are using **bridged networking**, the Orc8r host **must** have a static IP address assigned by your router.

If both Orc8r and AGW are running as VMs on the same physical machine and your host uses Wi‑Fi, bridged networking may not be possible for both VMs. In that case, you must use **internal networking**.

When using internal networking in VirtualBox:

* Create an internal network
* Disable DHCP
* Manually assign IP addresses inside each VM

You must know the IP address of the Orc8r host before proceeding.

> **Important**: Local DNS (for example, Pi-hole or router-based DNS records) will *not* work. AGW uses a hardcoded DNS server and will ignore local resolvers. Local hostname resolution must be done via `/etc/hosts` on the AGW VM.

---

## Explanation of Certificates

Magma uses TLS certificates extensively, and understanding their roles is critical for a successful deployment.

When Orc8r is deployed, it generates self-signed certificates located at:

```
magma/.cache/test_certs/
```

Key files:

* **rootCA.pem**
  The root Certificate Authority file for orc8r.

* **admin_operator.pfx**
  A client certificate used for client authentication to Orc8r's Swagger UI.

The `rootCA.pem` file is generated on the Orc8r host but must be copied to the AGW VM.

Installing `admin_operator.pfx` in your browser is optional for the purposes of this guide and only required for accessing Swagger; this is outside the scope of this guide.

---

## Important Orc8r Ports  

This deployment relies on two Orc8r ports that are not clearly explained in the official documentation as of the time of this writing:

* **7443** — AGW → Orc8r controller (normal operation)
* **7444** — AGW → Orc8r bootstrapper (bootstrapping)

Both ports must be configured correctly. If either is incorrect:

* nginx may route traffic to the wrong backend
* AGW will not connect to orc8r
* Bootstrap and certificate issuance will fail

---

## Step-by-step AGW Deployment  

### Installing Ubuntu in a VM  

Install **Ubuntu 20.04** inside a virtual machine. These instructions target VirtualBox and Proxmox, but other hypervisors should work similarly.

Because Ubuntu 20.04 is end-of-life, you must download it from a mirror:

* [https://launchpad.net/ubuntu/+cdmirrors](https://launchpad.net/ubuntu/+cdmirrors)

Look for:

```
ubuntu-20.04.6-desktop-amd64.iso
```

Required VM resources:

* 2 network interfaces
* 4 GB RAM minimum
* ~30 GB storage

Configure the network according to the instructions in [Networking and DNS Requirements](#networking-and-dns-requirements).

--- 

### Installing the Magma Prerequisites  

Once Ubuntu is installed, follow the official Magma prerequisites documentation:

* [https://magma.github.io/magma/docs/basics/prerequisites](https://magma.github.io/magma/docs/basics/prerequisites)

Follow the instructions from **Ubuntu** through **Downloading Magma**.

> You may skip installing Vagrant and VirtualBox.

Do not forget to install **docker-compose**. If it is missing, `agw_install_docker.sh` will fail.

After cloning Magma, copy it to `/var/opt`:

```bash
sudo cp -r ~/magma /var/opt/
```

AGW expects Magma files to be located under `/var/opt/magma`.

---

### Deploying AGW  

Follow the official AGW Docker deployment guide along with the instructions below:

* [https://magma.github.io/magma/docs/lte/deploy_install_docker](https://magma.github.io/magma/docs/lte/deploy_install_docker)

**Important**: If you just follow the Magma documentation without the below modifications, the deployment will fail because they are not tailored to this use-case.

#### Copy the Root CA

The `rootCA.pem` file the documentation refers to is located on the Orc8r host at:

```
magma/.cache/test_certs/rootCA.pem
```

Copy this file to the AGW VM at:

```
/var/opt/magma/certs/rootCA.pem
```

#### Configure Hostname Resolution

Do **not** create `control_proxy.yml` as described in the official docs.

Instead, begin by adding the required hostnames to `/etc/hosts` on the AGW VM:

```bash
nano /etc/hosts
```

Add the following lines, replacing `<orc8r_ip>` with the IP address of your Orc8r host:

```
<orc8r_ip>  controller.magma.test
<orc8r_ip>  bootstrapper-controller.magma.test
```

These hostnames must be used because the certificates generated by Orc8r expect the `*.magma.test` domain. The subdomains must also match exactly. The deployment will fail unless you use these domains.

#### Create control_proxy.yml

Edit the configuration file:

```bash
nano /var/opt/magma/configs/control_proxy.yml
```

Paste the following:

```yaml
cloud_address: controller.magma.test
cloud_port: 7443

bootstrap_address: bootstrapper-controller.magma.test
bootstrap_port: 7444

fluentd_address: controller.magma.test
fluentd_port: 24224

rootca_cert: /var/opt/magma/certs/rootCA.pem
```

Continue following the official AGW deployment steps.

After deployment completes, verify AGW health:

```bash
cd /var/opt/magma/docker
sudo docker exec magmad checkin_cli.py
```

---

## Troubleshooting

This section lists common failure modes encountered when deploying AGW in Docker inside a VM, along with the most likely causes and concrete diagnostic steps.

Unless otherwise specified, all commands are to be run on the AGW as the root user.

### Common Diagnostic Commands

#### Check AGW container status
```bash
cd /var/opt/magma/docker
docker compose ps
```
All services (except `liagentd`) should be **Up** and none should be unhealthy.

#### View AGW logs

```bash
docker compose logs -f --tail=0 magmad
```

#### Run AGW connectivity self-check

```bash
docker exec magmad checkin_cli.py
```

#### Show AGW hardware ID and challenge key

```bash
docker exec magmad show_gateway_info.py
```

Ensure these values match what is registered in NMS.

#### Verify root CA is visible inside the container

```bash
docker exec magmad ls -l /var/opt/magma/certs
```

`rootCA.pem` must exist.

#### Test TLS + gRPC ALPN from AGW host

```bash
openssl s_client \
  -connect controller.magma.test:7443 \
  -servername controller.magma.test \
  -alpn h2 \
  -CAfile /var/opt/magma/certs/rootCA.pem
```

A correct endpoint will negotiate `ALPN protocol: h2` and verify successfully.

#### Restart Docker Containers

This restarts all services in the current docker-compose project and is safe to run during troubleshooting.

Move to the docker directory, `/var/opt/magma/docker` on the AGW machine and `magma/orc8r/cloud/docker` on the Orc8r machine

```bash
docker compose restart
```

---

### Symptom: `invalid value key=content-type value=text/html`

**Seen in**:

* `magmad` logs
* `checkin_cli.py`

**Cause**:

AGW is connecting to an HTTP endpoint instead of a gRPC endpoint, usually due to:

* Incorrect port configuration
* Incorrect hostname causing nginx to select the default server block

**Fix**:

* Verify `control_proxy.yml` uses port `7443` for `controller.magma.test` and port `7444` for `bootstrapper-controller.magma.test`
* Confirm `controller.magma.test` and `bootstrapper-controller.magma.test` resolve correctly
* Restart AGW and Orc8r [Restart Docker Containers](#restart-docker-containers)

---

### Symptom: `client sent no required SSL certificate`

**Cause**:

AGW connected to a controller endpoint that requires a client certificate before bootstrap completed.

**Fix**:

* Verify `control_proxy.yml` uses port `7443` for `controller.magma.test` and port `7444` for `bootstrapper-controller.magma.test`
* Ensure AGW can reach the bootstrapper on port `7444`
* Verify `rootCA.pem` is correct
* Check the output of
```bash
docker exec magmad checkin_cli.py
```
and see if it has a recommended fix

This error should disappear after successful bootstrap.

---

### Symptom: `gRPC error: Received RST_STREAM with error code 0` or `status = StatusCode.UNKNOWN`

**Cause**:

AGW fails to connect to Orc8r after generating certificates

**Fix**:

* Restart **both** AGW and Orc8r [Restart Docker Containers](#restart-docker-containers)

---

### If You Are Completely Stuck

Before asking for help, collect the following:

* Output of `checkin_cli.py`
* Last ~10 lines of `magmad` logs
* Relevant `orc8r_nginx` log entries
* Contents of `control_proxy.yml`
* Confirmation of exposed ports on the Orc8r host
