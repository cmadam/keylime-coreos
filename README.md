### Setup Environment

* Platform - Amazon AWS
* Operating System - Red Hat Enterprise Linux CoreOS release 4.10

### Keylime Server

*Important*: prior to installation, make sure that both server
(verifier/registrar) and client (agent/tenant) images support the 
same Keylime API version (e.g. 2.1).  Setups running different
(incompatible) API versions on the server and client sides will fail.

The recommended way to install software in CoreOS is to run it in
containers.  In this installation we manage the containers using
Podman.

#### Server-Side Configuration Changes
Download the Keylime verifier and registrar images from quay.io:

```
podman pull quay.io/keylime/keylime_verifier
podman pull quay.io/keylime/keylime_registrar
```

Create a `keylime.verifier.conf` file to be used by the verifier.
This file is produced from the original `keyline.conf` file in the
keylime distribution, with the following modifications under the
`[cloud_verifer]` section:

1. Replace `cloudverifier_ip = 127.0.0.1` with `cloudverifier_ip =
0.0.0.0`.
2. Replace `registrar_ip = 127.0.0.1` with `registrar_ip = <IP Address
of the host>`

Create a `keylime.registrar.conf` file to be used by the registrar.
This file is produced from the original `keyline.conf` file in the
keylime distribution, with the following modifications under the
`[registrar]` section:

1. Replace `registrar_ip = 127.0.0.1` with `registrar_ip =
0.0.0.0`.
2. Replace
   ```
   ca_cert = default
   my_cert = default
   private_key = default
   ```
   with:
   ```
   ca_cert = cacert.crt
   my_cert = 6f4aef70a0d0-cert.crt
   private_key = 6f4aef70a0d0-private.pem
   ```
   where `6f4aef70a0d0` is the ID of the verifier container.

#### Starting the verifier and the registrar

Start the verifier with the following command:
```
mkdir -p /tmp/keylime
rm -rf /tmp/keylime/*
podman run -p 0.0.0.0:8881:8881 -v /tmp/keylime:/var/lib/keylime:z  -v /tmp/keylime.verifier.conf:/etc/keylime.conf:z -d quay.io/keylime/keylime_verifier
```

Start the registrar with the following command:
```
podman run -p 0.0.0.0:8890:8890 -p 0.0.0.0:8891:8891 -v /tmp/keylime:/var/lib/keylime:z  -v /tmp/keylime.registrar.conf:/etc/keylime.conf:z -d quay.io/keylime/keylime_registrar
```

The verifier and registrar startup commands expose the ports 8881 (for
the verifier) and 8890/8891 (for the registrar).  The first volume,
`/tmp/keylime` will contain, upon the verifier launch, the verifier
and registrar databases, and the `cv_ca` directory which includes the
CA certificate, the client certificate and the client private key.
The second volume maps the keylime configuration files defined in the
previous section to the `/etc/keylime.conf` standard location.


### Keylime Agent

Let `AGENT_IP` denote the IP address where the keylime agent is
running, and `SERVER_IP` denote the IP address where the keylime
verifier and registrar are running.

#### Copy the Certificates and Keys from Keylime Server

Copy the entire `cv_ca` directory that was created during the verifier
startup in the local `/var/lib/keylime/` directory.  For our example,
we assume that we copied the contents of the server
`/var/lib/keylime/cv_ca` directory in the agent machine in the
directory `/var/lib/keylime/aws/keylime/cv_ca/`

#### General Configuration Changes

In the `[general]` section of the `/etc/keylime.conf` configuration
file, set the `receive_revocation_ip` parameter.  Replace the line:
```
receive_revocation_ip = 127.0.0.1
```
with:
```
receive_revocation_ip = <SERVER_IP>
```

#### Configuration Changes for the Agent

Under the `[cloud_agent]` section, make the following changes:

1. Replace `cloudagent_ip = 127.0.0.1` with `cloudagent_ip = 0.0.0.0`

2. Replace `agent_contact_ip = 127.0.0.1` with `agent_contact_ip =
<AGENT_IP>`

3. Replace `registrar_ip = 127.0.0.1` with `registrar_ip = <SERVER_IP>`

4. Replace `keylime_ca = default` with `keylime_ca = /var/lib/keylime/aws/keylime/cv_ca/cacert.crt`


#### Configuration Changes for the Tenant

Under the `[tenant]` section, make the following changes:

1. Replace `cloudverifier_ip = 127.0.0.1` with `cloudverifier_ip =
<SERVER_IP>`

2. Replace `registrar_ip = 127.0.0.1` with `registrar_ip = <SERVER_IP>`

3. Replace `tls_dir = default` with `tls_dir = /var/lib/keylime/aws/keylime/cv_ca/`

4. Replace:
   ```
   ca_cert = default
   my_cert = default
   private_key = default
   ````
   with:
   ```
   ca_cert = cacert.crt
   my_cert = client-cert.crt
   private_key = client-private.pem
   ```

5. Replace `registrar_tls_dir = CV` with `registrar_tls_dir =
/var/lib/keylime/aws/keylime/cv_ca/`

6. Replace:
   ```
   registrar_ca_cert = default
   registrar_my_cert = default
   registrar_private_key = default
   ```
   with:
   ```
   registrar_ca_cert = cacert.crt
   registrar_my_cert = client-cert.crt
   registrar_private_key = client-private.pem
   ```

### Keylime Rust Agent

To install the Rust Keylime Agent in CoreOS, first clone the github repo
```
git clone git clone  https://github.com/keylime/rust-keylime.git
```
Then build the Rust agent image:
```
cd rust-keylime/docker/fedora
podman build . -f keylime_rust.Dockerfile
```
