---
slug: sso-for-ssh-which-tool-to-use
title: SSO for SSH - Which tool to use?
authors: [scheibling]
tags: [ ssh, openssh, oidc, bitwarden, smallstep, step-ca, opkssh, openpubkey, vaultwarden ]
---

You want to add extra security and/or Single Sign-On to your SSH server(s), but how? Here is a short summary of the most common solutions

<!-- truncate -->

OpenSSH security is a complex topic, it's an old piece of software and the amount of solutions and tips available is massive. There all kinds of variations, from hardware keys or encrypted keys, SSH agents or certificates and much more.

### Changes/Additions
2025-04-15: Added more details after suggestions on Reddit (OPKSSH, Bastion/Jumphosts, PAM, clarification on some sections. You can see the entire changelog on [github.com/idpea/idpea](https://github.com/idpea/idpea))


### Why do I need anything else than username/password?
There's no general rule that says you need to use anything else. All security measures are a trade-off between security and usability, if you only run your local raspberry pi ssh server which can only be reached from your network, setting up OPKSSH or step-ca for authentication might be a bit overkill (but it's very useful for learning). But generally, SSH along with RDP and other remote management protocols are some of the most common attack surfaces.

## Types of authentication
Generally, there are a couple different types of authentication for SSH:

### Username/Password
The simplest one, and the one usually enabled by default for non-root users. You specify the username in the SSH command, type the password when connecting and you're in. This option can be combined with other security measures like MFA, Ratelimiting, IP allow/deny lists and so on.

### SSH Keys (+ SSH Agent)
You generate a set of keys, a public one and a private one for one-way encryption. During authentication, your client negotiates with the server whether to use a key, and how to use it, which ciphers and algorithms to deploy. You send your public key to the server, the server sends their public key to you, you sign data with the servers public key which the server can then decrypt, and the server signs data with your public key that your private key can decrypt.

There are loads of variants here regarding key storage, encryption and management; more about that below. The SSH agent is in this category, since it's (very simplified) a secure storage facility for your private keys.

### SSH Certificates
An extension of SSH Keys, where a central authority takes in your public key, writes a document of sorts describing "this key is allowed to connect under these circumstances" and signs it with the CA key. When it's sent to the server, the server validates that it was issued by a trusted CA, and which access it grants the user. This is a bit more complex, but allows for more flexibility and functionality like short-term (minutes, hours, days) access which needs to be frequently renewed, or temporary access to a server.

### PAM-based authentication
On the linux side, OpenSSH can use PAM (Pluggable Authentication Modules) to authenticate users. This allows for a lot of flexibility, since PAM can be configured to use all kinds of authentication methods, including LDAP, Kerberos, Yubikeys and so on. There are loads of subcategories, more below.

### X509 Certificates/Smartcards
Not quite as common for SSH, but there are some implementations of X509 certificates for SSH. This works more or less similarily to the SSH Certificates, but it's a different format.

### Zero Trust/Zero Trust Network Access-type solutions
The term Zero Trust means "nothing should be automatically trusted", so all forms of access are authenticated, from network to application. The applications in this category do not necessarily secure all types of access towards SSH, but are in the general vicinity of Zero Trust-type software. Some, like OpenZiti, offer customized clients that secure the entire connection process, Tailscale SSH can do network-level authentication, session recording and SSH Key Management while others like Headscale and Netbird are able to limit access on the network level alone.

### Jumphosts and Bastion Hosts, PAM
Intermediary servers or software that is used to "proxy"/forward connections to servers inside fenced-off networks. There are different levels of seurity and functionality, but the general idea is to have a single point of access through which all connections to other servers go. A relatively normal implementation is to have a web-based interface that allows you to connect to servers, but there alternatives available that provide console-based access too.

# SSO for SSH
In this article, we'll focus on the SSO aspect of SSH authentication. The general requirements we're going for are:

- We assume that an IDP is already in place (OIDC). There are options for SAML, but they're far less common nowadays
- Easy to use and minimal long-term management
- Easily scaleable

Type: The type of method
Name: The name of the method
OIDC: Supports authorization via OIDC
Extra Server: Requires a separate server to use
Extra CLI: Requires additional client software
Access Control: Can be used to limit access to specific servers or groups of servers
Authorization: Can be used to limit access to specific users or groups of users
Notes: Notes about the method


| Type | Name | OIDC | Extra Server | Extra CLI | Access Control | Authorization | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| SSH Keys | SSH Agent | No | Yes | No | n/a | n/a | Client-side solution, not applicable to access control/authorization in itself |
| SSH Keys | Bitwarden,Vaultwarden,1Password | No* | Yes | Yes | n/a | n/a | Client-side solution, not applicable to access control/authorization in itself |
| SSH Keys | Yubikeys, Smartcards | No* | Yes | Yes | n/a | n/a | Client-side solution, not applicable to access control/authorization in itself |
| SSH Keys | AuthorizedKeysCommand | No | No | No | Manually | Manually | Can technically be used for temporary access, but you'd have to manage the start/end manually. Can be used to set specific permissions |
| SSH Keys | SSH Key Authority | LDAP only | Yes | No | Yes | Yes | Also has logging and auditing |
| SSH Keys | OPKSSH | Yes (only Gitlab, MSFT, Google) | No* | Yes | Yes | Yes | Requires a custom CLI on the client and server side. Access control is managed in configuration files on each server. | 
| SSH Certificates | Smallstep StepCA | Yes | Yes | Yes | Yes | Yes | Authorization is configured on the server based on attributes sent in the certificate from StepCA like usernames, groups, etc. |
| SSH Certificates | Hashicorp Vault | Yes | Yes | Yes | Yes | Yes | - |
| SSH Certificates | Gravitational Teleport | Limited (Github only in FOSS) | Yes (or SAAS) | Yes | Yes | Yes | - |
| SSH Certificates | Hashicorp Boundary | Yes | Yes | Yes | Yes | Yes | - |
| PAM | PAM_OAUTH2_DEVICE | Yes | No | No | Yes | Yes | - |
| X509 | PKIXSSH | No | No | Yes (custom SSH client) | Manual | Manual | Requires custom build of SSH, not compatible with bundled servers (VSCode, etc.) |
| Zero Trust | Netbird | Yes | Yes | Yes | Yes | Yes* (network-level only) | - |
| Zero Trust | OpenZiti | Yes | Yes | Yes | Yes | Yes | Custom SSH client providing deeper level of integration |
| Zero Trust | Headscale | Yes | Yes | Yes | Yes | Yes* (network-level only) | - |
| Zero Trust | Tailscale | Yes | Yes | Yes | Yes | Yes* | Tailscale SSH can provide additional application-level authorization |
| Zero Trust | Ionscale | Yes | Yes | Yes | Yes | Yes*  | Tailscale SSH implementation can provide additional application-level authorization |
| Zero Trust | Pomerium | Yes | Yes | Yes | Yes | Yes* (network-level and/or app-level) | - |
| Jumphosts | Guacamole | Yes | Yes | No | Yes | Yes | Web-based, so won't work with other SSH clients |
| Jumphosts | Bastillion | No | Yes | No | Yes | Yes | Web-based access, but can also be used to manage SSH keys for access with other clients |
| Jumphosts | Jumpserver | Enterprise only | Yes | No | Yes | Yes | Web-based access |
| Jumphosts | OVH "The Bastion" | No | Yes | No | Yes | Yes | Terminal/TUI-based, so no web console access. Can be used with existing SSH clients, the "bastion" command is just an alias for SSH |


## SSH Key Management (User Side)

### SSH Agent
The SSH agent is used to securely store keys on your device and use them on your machine and other machines you SSH to without copying the key files anywhere. It's bundled with most OpenSSH distributions, keys are added via ```ssh-add``` and you can forward the agent by using the -A flag when connecting, meaning you can jump through multiple servers while still securely using your local agent and not providing private keys to the remote servers.

### Bitwarden, Vaultwarden, 1Password, etc.
Bitwarden, Vaultwarden and 1Password have a built-in SSH agent that can be used to store and use SSH keys. The upside with this over the previous method is that you can also use this to synchronize your keys between devices along with your passwords and other secrets.

A note here though is that Bitwarden and Vaultwarden allow exporting keys, which the SSH Agent does not. Generally, you'd want your private keys to be non-exportable, because it means someone entering your machine wouldn't be able to exfiltrate the keys from your machine. But it's a bit of a trade-off between usability; you might want to be able to copy your keys out from your password manager to use them in another context, and security.

### Yubikeys and other hardware keys
You can load your private key onto a yubikey or smartcard, and use that to authenticate. It's not very complex, and can be done with relatively little effort, but there are some caveats: lack of proper WSL/WSL2 support, requiring software for usage. There are some ways to achieve this:

PIV: You create a SSH CA, and use it to sign arbitrary public keys for accessing a server. This allows for a lot of flexibility, and you can create time-limited access on any machine which expires minutes or hours later.

PGP: Depending on the client, you can use the PGP feature of the yubikey to store SSH Keys

FIDO2: OpenSSH 8.3+ supports FIDO2 authentication, which allows you to use a FIDO2 key (like a Yubikey) to authenticate to SSH servers. You can also require touch ("verify-required"), and require the key to have been generated on the yubikey ("resident") and not exported later.


## SSH Key Authorization (Server Side)

### AuthorizedKeysCommand (Accesss Control)
If you want to start off real easy, you can configure the AuthorizedKeysCommand in /etc/ssh/sshd_config. This command allows you to specify a script or program that will run when a user tries to log in to determine if the key the user is sending is authorized for the server in question.

With this method, you could query Github or Gitlab for the public keys belonging to a specific username, for example

```ini
AuthorizedKeysCommand /usr/bin/curl -s https://api.github.com/users/%u/keys | jq -r '.[].key'
```

This command will get the list of a users (%u is replaced with the username logging in) public keys from the Github API, and returns them to the SSH server. The server then checks if one of those keys is sent by the client, if successful the user is logged in. You could also replace the %u with your github username, that way only the keys you have on your account are accepted for authentication.

You could also use basically any static web server to host a list of public keys, and use that as the AuthorizedKeysCommand.

### [Operasoftware SSH Key Authority (Access Control) (Link)](https://github.com/operasoftware/ssh-key-authority)
SSH Key Authority is a central server that can be used to manage access to a fleet of servers, set permissions without having to reconfigure individual servers. The user logs in and adds their own SSH Keys, which are then distributed to all servers the user has access to.

### [OPKSSH (OpenPubkey)](https://github.com/openpubkey/opkssh/)
This is a quite recent release by Cloudflare, since they acquired BastionZero they also now own the protocol, and has now been relesed under the umbrella of the OpenPubkey project. It enables an existing Oauth2/OIDC IDP to issue SSH access via tokens, where the token describes that a certain key is allowed to access one or multiple servers under certain conditions.

There are still some challenges with this method, you still need to manually manage access on each server, it still only supports a small subset of OIDC IDPs (*"Currently opkssh is compatible with Google, Microsoft/Azure and Gitlab OpenID Providers (OP). If you have a gmail, microsoft or a gitlab account you can ssh with that account."*), but it's an interesting new approach.

Note: While this solution *technically* uses SSH Certificates, it does so only to "smuggle"[sic] the tokens over to the server, where they are verified against the IDP.


## SSH Certificates 

### [Smallstep StepCA (link)](https://github.com/smallstep/certificates)
Step-ca has built-in functionality for issuing SSH certificates and can be used on a larger scale to manage access to multiple servers. There is support for limited and temporary access (e.g. User X on server Y for 1 hour between 09:00 and 10:00 on the 5th of November 2029). This still uses SSH Keys, but the keys are only valid once a certificate has been created by step-ca meaning you can just generate new keys on the fly instead of moving your private keys around.

This also supports authentication to get the certificate based with an OIDC provider, and can set different levels of access for different users, groups and attributes. Step-ca requires a separate server if you want to be able to use it from everywhere, you could install it on a USB stick and start as-needed but that wouldn't provide much benefit over just using regular SSH keys.

There are no IDP limitations on the community/free edition, you can use any OIDC-compatible IDP.

### [Hashicorp Vault](https://www.hashicorp.com/en/products/vault)
Similar to StepCA, can also issue SSH certificates based on OIDC authentication. This one is a bit more complex to set up, manage and maintain. It also includes a lot of other functionality like central secret storage which you might not need in this context, but is a solid solution.

### [Gravitational Teleport](https://goteleport.com)
Teleport takes a similar approach as StepCA, but with more of a full-service solution for connecting to machines, managing permissions and access. [The community edition is limited to Github SSO.](https://goteleport.com/docs/feature-matrix/). Could technically also fit into the categories of Zero Trust and Bastion/PAM.

### [Hashicorp Boundary](https://www.hashicorp.com/en/products/boundary)
Boundary is similar to Teleport, and provides more of a full-service solution for access management. The feature set of HCP boundary is a lot less limited than Teleport, but it does require a fair bit more maintenance and is trickier to set up and maintain.


## PAM
### [PAM_OAUTH2_DEVICE](https://github.com/ICS-MU/pam_oauth2_device)
This is a small module that enables the use for OIDC authentication for SSH. It uses the device flow to authenticate users, and works something like this:

1. You sign in to the server (username@servername)
2. You are presented with a QR code and/or link to the IDP
3. You scan the code or open the link and authenticate
4. You return to the server and confirm that you've logged in
5. The server checks with the IDP to see the permissions for the user and maps them to the local user accounts
6. The user is now logged in

There are some downsides to this method - one of them being it does not really work with VSCode, but it can easily be combined with other authentication methods like keys or certificates for automatic access. It requires no extra server components apart from the IDP, just two files (program + config) on the server.

## X509 Certificates/Smartcards
### [PKIXSSH](https://roumenpetrov.info/secsh/index.html)
This one requires a custom build of SSH, since the official OpenSSH implementation doesn't support this method out of the box. It allows for the use of X.509 certificates for SSH authentication, providing a more standardized approach to managing SSH keys and access control. This also means it's not really compatible with a lot of bundled SSH servers out there, like the one that comes with VSCode, but you can add other authentication methods so that those work as well.

### PAM_OAUTH2_DEVICE, StepCA, HCP Boundary, etc.
If you have Certificate Authentication enabled on your IDP, you can use any of these methods to in practice use X509 certificates for SSH Authentication.

The official SSH Certificate implementation uses another type of certificate which is not cross-compatible with X509 certificates.

## Zero Trust
### [Netbird](https://netbird.io)
Netbird is a zero trust VPN/Mesh network solution that allows you to limit SSH access based on network policies.

### [OpenZiti](https://openziti.github.io)
OpenZiti is a zero trust networking solution that allows you to limit SSH access based on network policies. It can be used to create a secure mesh network between devices, and can also be used to limit access to specific applications or services.

### Headscale, Tailscale, Ionscale
Headscale-compatible services work in the same way as Netbird, and can provide network-level authentication for access to SSH servers. In addition, Tailscale SSH will do authorization using some kind of traffic interception to ensure that the policy permits you to connect to that specific server with that specific username.

While Headscale does not have suport for Tailscale SSH, Ionscale provides an open source implementation of the control server with that support added.

### [Pomerium](https://pomerium.com)
Pomerium is also a Zero-Trust solution, and provides secure SSH access via a jump host where the connection is authenticated via your IDP. It also seems to be limited to Google, Okta, OneLogin, JumpCloud, Auth0, Azure, Slack and Duo SSO.


## Jumphosts and Bastion Hosts (PAM)
### [Apache Guacamole](https://guacamole.apache.org/)
Guacamole is an open source web-based gateway for accessing servers via protocols like SSH, RDP and VNC. You can set it up with OpenID Connect or SAML for single sign-on, and access servers via a browser. There are also a heap of other features like session recording, auditing and more. It can be combined with SSH Keys, SSH Certificates, and other authentication methods like pam_oauth2_device.

### [Bastillion](https://www.bastillion.io/)
Bastillion is similar to Guacamole, but more focused on the SSH protocol. It can also be used to manage and distribute SSH Keys, and has a web-based interface for managing access to servers. It doesn't currently support OpenID Connect or SAML-based authentication, but it does support LDAP/Active directory.

### [Jumpserver](https://github.com/jumpserver/jumpserver)
An open source jumphost/privileged access management tool, provides a web interface similar to the previous two options. Supports a range of protocols in addition to SSH including RDP, VNC, Database access and more. Requires the enterprise variant for a lot of the features like SSO via OIDC or SAML, but the community edition is still a solid option for managing access to servers. 

### [OVH The Bastion](https://github.com/ovh/the-bastion)
An open source bastion developed by OVH, which works a bit differently from the other bastion hosts above. It works with existing clients, adding a layer of authentication and security to the connection and forwarding it to the real server. It supports a range of features like PIV Authentication Yubico PIV attestation checking ("is this key stored on a yubikey"), realms (groups of servers), and more. It's completely terminal-based, so no web interface.


## Conclusion
The options are endless, and there are surely a lot I've missed on this list (feel free to send me a message and I'll add your suggestions! ). But the question is really - what do you need?

While Zero Trust is quickly becoming an industry standard, setting it up for accessing a handful of servers is, while being a fun challenge, probably unnecessary. An easier alternative might be to just store your SSH keys in Bitwarden/Vaultwarden for added security and portability, or setting up PAM_OAUTH2_DEVICE for a quick and easy solution.

But likely, nobody else can answer this for you. Do you want a challenge setting upp HCP Boundary? Do you want something that just works? Do you want to learn something new? 

I might do some more articles about how to set up and use some of these methods in the future, but for now, signing off!