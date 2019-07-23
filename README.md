## Project
This project provides an Ansible playbook for the installation of Globus
Connect Server (GCS) v5 onto an OSN pod. This effort provides the top layer
of a functional OSN data stack for use in the OSN pilot phase. It was derived
from early work in the project 'ansible-osn'. That project was later divided
to form middleware-ceph and globus-connect-server. The purpose of the split was
to allow development in parallel of both middleware components. This was an
early consideration before the creation of ceph-ansible-root which now
supercedes middleware-ceph.

The supplied configuration will install Globus Connect Server onto the
designated monitor node, configure Globus for access to the local Ceph
cluster, create the Globus endpoint definition, setup an HTTPS interface
for data access, enable authorized access for users having globusid.org
accounts and set user 'osn@globusid.org' as the endpoint administrator.

This playbook provides a single role, globus-connect-server, which is a 
distillation of the public instructions found at 
https://docs.globus.org/globus-connect-server-v5-installation-guide/.

#### globus-connect-server role
This role will perform the installation of Globus Connect Server in a two-pass
pattern. The first pass will install all software components and create the 
endpoint definition within the Globus service at globus.org. An intermediate,
one-time, manual step is required to log into the Globus website and flag the
endpoint as 'managed' which enables additional features for the endpoint (ex.
web-based transfer monitoring, roles to delegate endpoint management). On
subsequent runs of the playbook, the Globus software will be configured to
operate with Ceph and grant user 'endpoint_admin' as the administrator of the
endpoint which allows the account to further configure and monitor the endpoint
directly through the Globus website and CLI.

## Terminology

#### Globus Connect Server
Globus Connect Server (GCS) is the name for the software stack that end users
and system administrators install and configure in order to connect local 
data storage (file system, Ceph, etc) to the Globus managed transfer service
and to take advantage of the Globus Auth service.

#### Globus Auth
Globus Auth is an authorization service layer that provides an OAuth2-compliant
interface to identity providers from thousands of research institutions. Through
Globus Auth, users can authenticate directly with their home institution and 
receive access tokens which can be introspected by services developed by third
parties.

#### Linked Accounts
Users of Globus Auth may have credentials at more than one institution. Using
Globus Auth, it is possible to 'link' these accounts so that they may be used
interchangeably throughout the Globus platform.

                        (Endpoint)
    |--------------- Pod Monitor Node ---------------|--- Globus Transfer ---|
    ceph bucket --- collection --- storage gateway ---- endpoint definition

#### endpoint
An endpoint refers to the machine(s) on which Globus Connect Server is
installed. The endpoint definition exists within the Globus Transfer service
and desribes the endpoint's location, access controls and roles.

#### storage gateway
A storage gateway defines the type of backend storage that is to be used by GCS
and how that storage is accessed. For an OSN pod, the storage gateway defines
the location of the Rados Gateway and lists users allowed to create collections
on the Ceph storage cluster.

#### collection
A collection, formerly called 'shares', are a single folder within the Ceph
storage cluster and all of its contents, recursively. The collection will
have access control lists defining who can acess the data within the
collection and the type(s) of actions permitted on that data.

#### Management Console
Web-based interface through which the admin of an endpoint can monitor and 
manage transfer activities.

## Overview
All endpoint defintions are owned by a shared administrative account,
'osn@globusid.org'. This account has the capability to create and delete
OSN endpoints, configure endpoints for paid features, monitor endpoint data
transfers, create collections, set access controls on collections and delegate
these responsibilities to other Globus identities.

Each OSN pod will be a single, independent Globus endpoint using the naming
convention \<site\>.osn (ex jhu.osn) making the endpoints easier to idenfity
by OSN users. Users searching for data within OSN can simply search for the
'osn' endpoint suffix using the Globus web or CLI clients and drill down into
collections available at each endpoint. Each endpoint will offer managed
transfers via the Globus Transfer service as well as HTTP (eventually S3)
transfers. Each POD's endpoint definition will be created on the initial
Ansible run.

Each dataset offered by OSN will likely reside within a single bucket owned by
Ceph user 'osn' and that bucket will be the root of a collection used to hold
the dataset. For example, allocatee Zhang may have a public dataset named
"Gravitational Waves". Users browsing OSN endpoints may find the corresponding
collection named "Gravitional Waves 2019". Browsing of that collection will
transparently redirect the user to the appropriate POD endpoint and bucket that
hold the dataset.

OSN users will access data via the Globus Transfer service or by using the HTTPs
interface provided by GCS. Access controls set by the OSN shared administrative
account on a collection are in effect for both access mechanisms.

All managed transfers originating through the web interface (globus.org) or
Globus CLI can be monitored via the management console,
https://www.globus.org/app/console/endpoints, accessible to the shared OSN
account 'osn@globusid.org'. This functionality can be shared with additional
OSN staff through assignment of the 'Activity Manager' and 'Activity Monitor'
roles.

It is recommended that the number of admins sharing the OSN account be minimal.
Each POD/site then may elect a number of local admins which can each be granted
additional roles (activity monitor, endpoint administrator, etc) on the POD
endpoint. This gives each site full administrative control over their own
endpoint and allows the admin to view overall OSN POD activity all while 
maintaining restricted access to the shared osn account .

## Architecture

                   |--------- Pod Monitor Node ---------|--- External Services ---|
    
                                   --- GridFTP ---              --- Globus Transfer
                                   |             |              |
    Data Nodes --- Rados Gateway ---             --- Port 443 ---
                    (localhost)    |             |              |
                                   ---- HTTPS ----              --- HTTPS Clients


GCS configures the local httpd process to listen on port 443 and direct requests
to the module 'mod-globus'. This single port is used for both requests from
HTTPs clients (ex curl, chrome) and the Globus Transfer service. Using SSL ALPN,
mod-globus is able to detect the request protocol and direct traffic to
either the local GridFTP service for managed transfers or directly to the Rados
Gateway for other client HTTP requests (ex. GET, PUT). Care must be taken so
that other local uses of the httpd process do not interfere with this
configuration.

During installation, GCS allocates one or more DNS entries from the glob.us
domain in order to automate SSL configuration. Each entry has the form
[0-9a-f]+.[0-9a-f]+.dn.glob.us. A unique DNS entry is created for each endpoint
and each collection on each endpoint. All DNS entries are covered by a 
LetsEncrypt SSL certificate that is created during installation and managed
by GCS processes.

## Authorization Model
The illustration below should serve as a visual aid for understanding the
authorization model, wrt to the relationship between accounts, used with the
installation of GCS.

    |-- Ceph --|----- Globus Auth Accounts, linked to user's home institution -----|

                                    --- Delegation --- joe@college.edu
                                    |                       |
    Ceph Acct --- user@idp_domain ---                       |
                                    |                       V
                                    -----------------> Sharing ACLs --> public(read)
                                                            |
                                                            |
                                                            V
                                                    project member(write)

#### Ceph Acct
This is an account known to the Ceph RadosGW, created with 
'radosgw-admin user create <uid> ...'. All access to the Ceph RadosGW will
masquerage as this account. This playbook relies implicitly on the Ceph RGW
account 'osn'; the account is not created by this playbook.

#### user@idp_domain
The Globus Auth account that maps to the local Ceph Account. From the
perspective of the Globus platform, all access to Ceph masquerades as this user
and therefore appears to Ceph as 'Ceph Acct'. The user prefix must be identical
to the Ceph Acct (ie both must be 'osn').

The basic premise of this design is to permit authorized users of OSN to access
OSN-hosted datasets using the user's credentials of choice (home institution,
Google accounts, etc) and to limit OSN's operational burden associated with
account management. For example, user Jill can be granted write access to a
dataset based on authentication with her preferred institutional credentials
jill@college.edu. All responsibitilies for ensuring the validity of the account
reside with the identity provider college.edu. At no point would OSN be
responsible for distribution of account credentials (password reset, rotation
of secrets, etc). OSN simply needs to guarantee that ACLs are updated accurately
to reflect the allocation and current policies.

The name chosen for the Ceph account must share a naming convention with the
Globus identity choosen for user@idp_domain; the 'user' prefix of the Globus ID
must match the name chosen for the Ceph account. When a request is made from
GCS to Ceph, GCS will lookup up the 'user' prefix for the Globus ID in Ceph and
conduct all access as the mapped Ceph account. In this playbook's current
configuration, the choosen Ceph account is 'osn', therefore the Globus ID is
osn@idp_domain.

idp_domain may be chosen from any established identity provider. The chosen IdP
will be responsible for the creation and maintenance of the associated user
account within the domain. In this initial configuration, the domain
'globusid.org' was chosen for simplicity; user accounts can be created within
the GlobusID domain without burdensome institutional hoops. Any IdP can be used
and the chosen IdP need not be consistent across all PODs. However, there
doesn't appear to be any advantage to using an IdP other that GlobusID.

The account 'osn@globusid.org' is a shared account operated by specific members
of OSN implementers. Besides servings at the default account that maps to Ceph,
it also serves other purposes such as endpoint configuration and monitoring and
is therefore the likely candidate.

#### Delegation
The owner of a collection (in this example osn@globusid.org) can delegate full
operational control of the collection (dataset) to the allocatee while retaining
ownership. This greatly simplifies account management since the allocatee will 
have the same full access as ownership but without creating additional Ceph
user accounts. All access from the delegated account as performed as the
delegatee's account.

#### Sharing ACLs
Permissions can be granted to users based on their chosen home institutional
credentials without conseding ownership or any administrative privileges of
the collection. 

## Getting Started
This Ansible configuration was developed for execution from a mock command
center node, citadel, using a SSH-push model from the local user 'ansible'
home directory to install on the JHU pod. Each target pod node is required
to permit passwordless SSH access from ansible@citidel and the local ansible
account must have passwordless sudo capabilities.

## Prerequisites
* The command center node is required to have Ansible 2.4.2 or newer.
* SSH public key authentication must be enabled for the user 'ansible' on each
of the pod nodes.
  * ansible@citadel:~/.ssh/osn-ansible <--- Private SSH key
  * On each pod node: ~ansible/.ssh/authorized_keys <--- Public SSH key
* The ansible user must be able to perform passwordless sudo on the pod nodes:
  * /etc/sudoers

  > \#\# Allows people in group wheel to run all commands  
  > \#%wheel ALL=(ALL)       ALL
  >
  > \#\# Same thing without a password  
  > %wheel  ALL=(ALL)       NOPASSWD: ALL

  * /etc/group
  > `wheel:x:10:ansible`
* Functional Ceph Rados Gateway service. It can operate on localhost on the
target node with or without SSL.
* Set firewall rules to allow external communication with GCS as described in
the [online documentation](https://docs.globus.org/globus-connect-server-v5-installation-guide/#open-tcp-ports_section):
  * Ports 50000—51000 inbound and outbound to/from ANY
  * Port 443 inbound and outbound to/from ANY
* Each GCS installation (ie each pod) must have a unique Globus Auth client id
and secret as described in [online documentation](https://docs.globus.org/globus-connect-server-v5-installation-guide/#register_the_endpoint_and_obtain_credentials):
  1. Log into the Globus Developers Console, developers.globus.org, using the
shared OSN administrative account.
  2. Click "Register your app with Globus."
  3. Click "Add another project" and fill out the form. This project will be
used to track your Globus Connect Server registrations. Keep it separate
from any other projects you might have.
  4. Use the "Add…" menu to add other appropriate users in your organization
as administrators of the project. Adding other administrators helps your
organization avoid losing administrative control should any one administrator
leave your organization.
  5. From the "Add…" menu for the project click "Add a new Globus Connect
Server" and fill out the form. For display name, use the syntax <site>.osn.
  6. Click "Generate a New Client Secret" and fill out the form.
  7. Save the Client ID and Client Secret values. You will use them soon when
configuring the playbook.
* The Ceph account 'osn' must already exist.
  * Create with radosgw-admin user create --uid=osn --display-name "osn@globusid.org"

## Configuration
Below are the most relevant configuration options necessary for review prior to
installation. This playbook has several configuration options which do not
appear here but are documented within each of the configuration files.

#### ansible.cfg  
 `remote_user`:  Change this to the account local to the pod nodes  
 `private_key_file`: Set to the location of your SSH private key 

#### group_vars/all.yml  
 `ceph_url`: host and port of the Ceph Rados Gateway.
  ceph_allowed_users: User(s) at the configured idp_domain that can create
  collections.

#### group_vars/jhu.yml  
  endpoint_name: Change this to the endpoint's name using the syntax <site>.osn.
  For example, the pod at JHU would be jhu.osn.
  client_id: Globus Auth client id to be used for this endpoint. This is the
  client id generated in the prerequisites.
  client_secret: Globus Auth client secret to be used for this endpoint. This is
  the client secret generated in the prerequisites.

#### host_vars/jhu.yml  
  public_interface: fully qualified domain name of the target node's public
  interface on the external DMZ that should be used for client requests.

#### inventory  
  This file is a standard Ansible definition of target hosts. See site.yml for
  how hosts are mapped to roles.

#### site.yml  
  Maps hosts to roles. All hosts in the ceph group with the '-mon2' infix will
  have Globus Connect Server installed.

## Deployment
  The following commands will perform the installation of GCS on the target Pod
  node and ensure that GCS services are operational. It is a 3-part process. The
  initial run will install and configure GCS. Then a one-time manual step is
  required to configure the endpoint as managed (enabling paid features).
  Subsequent runs will configure osn@globusid.org as the endpoint's administrator
  and ensure that services are operating.

  Install and perform initial configuration of GCS:
  > ansible-playbook site.yml

  Configure the endpoint has managed which enables paid features:
  * Log into globus.org as the shared administrative account 'osn@globusid.org'
  * Search on the endpoint by the name <site>.osn. Select the endpoint to view
    its configuration.
  * Select 'Edit Attributes' on the right hand side.
  * Set 'Managed Endpoint' to yes.
  * Select 'Save Changes'.

  Run the playbook again to complete configuration and start services:
  > ansible-playbook site.yml
