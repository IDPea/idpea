---
slug: bitwarden-vaultwarden-ssh-keys
title: Bitwarden/Vaultwarden - Finally with SSH keys (and agent)
authors: [scheibling]
tags: [bitwarden, vaultwarden, openssh]
---

Bitwarden, and by extension Vaultwarden, is *finally* receiving support for managing SSH keys AND authenticating via a built-in SSH agent!

<!-- truncate -->

So I kind of stumbled over this as I was scrolling through the changelog for the client a few days ago. There hasn't been any kind of public announcement yet, but I've followed the thread with Quextens contribution of SSH Key support and the SSH agent integration with Bitwarden/vaultwarden (https://community.bitwarden.com/t/ssh-key-support/49460/22) for a couple of weeks now, and eagerly awaiting to try it out.

And now, I finally got to try it out. And it works amazingly well out of the box.

## Enabling the features

### On Bitwarden Server

The Bitwarden server has yet to receive the feature flags, they are planned for the January release.

### On Vaultwarden Server

To start using the functionality, you need to set the following feature flags in the Vaultwarden (server) configuration:

- ssh-key-vault-item
- ssh-agent

This is done by setting the environment variable `EXPERIMENTAL_CLIENT_FEATURE_FLAGS` to `ssh-key-vault-item,ssh-agent` in the Vaultwarden configuration, and restarting the container/service/server.

### On the client

**You need to be on a supported client version to use the feature. The Bitwarden desktop client and browser plugin version 2024.12.0 and later support this feature. Vaultwarden is currently shipped with an older version of the web client, so the keys are not showing up in the web UI yet.**

![Enable SSH Agent](image-3.png)

On supported clients, there is a setting between the Theme and Hardware Acceleration settings where you can enable the SSH Agent. The vault item is enabled by default after adding the feature flags.

## The functionality

If you want more in-detail information about the feature, check out the link above where Quexten goes into more detail about the feature and how everything is implemented. There are two main parts to the feature:

### SSH Key Vault Item

![SSH Key Vault Item](image-1.png)

This is the actual vault item type that is created when you add an SSH key to the vault.

![Edit SSH Key Vault Item](image.png)

When creating or editing the vault item, it will by default generate a key for your. If you want to store your own key, you need to copy the private key and then click the "Import key from clipboard" button. If your private key is encrypted with a passphrase, it will prompt you for that when you import the key.

![Prompt for passphrase](image-2.png)

The rest is basically the same as with other vault items, set a folder, name, collections etc.

Important to note is that the private key needs to be in the OpenSSH private key format (-----BEGIN OPENSSH PRIVATE KEY-----) and not in the PEM format (-----BEGIN RSA/EC PRIVATE KEY-----). If you try to import a PEM key, it will be rejected.

### SSH Agent

There is also a new SSH agent feature for using those keys together with your clients. Out of the box, once I'd imported my keys into Bitwarden, I could already use them with OpenSSH in Windows (after enabling the agent in the client settings).

![Showing the keys in ssh-add](image-4.png)

When connecting to a server, you'll receive a popup in your Bitwarden client asking you to confirm the connection.

![Confirm key usage](image-5.png)

The SSH agent is a drop-in replacement (well, implementation) of the standard ssh-agent, meaning you can forward it to other machines, use it with GIT and other things in the same way that you'd previously used the ssh-agent and ssh-add commands.

## Usage in WSL2

I'm an avid user of WSL2 and Windows Terminal, so naturally I wanted to try out the feature there as well. The issue is that the SSH agent from Windows is not forwarded to the WSL2 environment.

The easiest way to do it would be just using `ssh-add.exe -l` and `ssh.exe` to manage keys and connect to servers, but those directly use the Windows SSH client and not the OpenSSH client in the WSL2 environment.

I did find some ways to forward the agent, but they are not very elegant. I'll keep searching for a better solution and post that in a part 2.

## Conclusion

I'm personally very excited about this feature. I've been using Bitwarden and Vaultwarden for a long time now in many different circumstances, and this feature has always been high up on my wishlist. 

Apart from the key import functionality being a bit buggy at times (restarting the client usually works), and the WSL2 issues (which have nothing to do with Bitwarden and everything to do with Windows and WSL2), the feature works amazingly well and saves me a lot of grief moving keys back and forth between my machines.
