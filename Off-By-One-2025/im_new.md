## Newcomers

Welcome to Active Directory! This page is designed for beginners and those with limited experience in Active Directory environments - including individuals familiar with entry-level certifications like the OSCP.  The goal of this page is to break down the learning curve of Active Directory, and hopefully help you get the first flag!

## Support 

If you've read through this guide, and are facing an issue - feel free to reach out to anyone on the team at the conference!

We have [@gatari (Zavier)](https://twitter.com/gatariee), [@Sora (Jun Yu)](https://www.linkedin.com/in/sorahehe/), [@Gladiator (Cher Boon)](https://twitter.com/cherboon) and a couple of others physically on-site!

## Jump Host

In most Active Directory assessments, organizations typically grant testers access to the internal environment through a designated machine. This is usually either a [VDI Machine](https://azure.microsoft.com/en-us/resources/cloud-computing-dictionary/what-is-virtual-desktop-infrastructure-vdi) or an [NUC](https://en.wikipedia.org/wiki/Next_Unit_of_Computing) in the environment that you'll be given SSH access to, alternatively you may be placed directly into the environment (on-site access) - in which case, you may disregard the use of a jump host.

Similarly, in this CTF, you'll be given SSH access to `172.16.10.10` with the credentials: `svc_jmp:8iP0TbO5mQmS`. You may access the machine with the following command (note that `sshpass` is only used for this guide to prevent confusion, you should omit this in actual testing):

```sh
sshpass -p '8iP0TbO5mQmS' ssh svc_jmp@172.16.10.10
```

If you aren't able to SSH into the machine, but you are able to ping the machine: simply attempt to SSH with no creds, to facilitate known key checks.

To access internal machines via the jump host, you have two main options:
1. Execute commands directly within the SSH session
2. Proxy traffic through the jump host

The second method is more versatile, as it lets testers work from the comfort and familiarity of their own testing machine. 

### Proxychains

You can use [proxychains (Linux)](https://github.com/haad/proxychains) or [proxifier (Windows)](https://www.proxifier.com/) to "force" traffic to go through the jump host via SSH, known as "proxying". This guide will use `proxychains`, ensure that `proxychains` is installed before continuing: `sudo apt-get install proxychains`.

`proxychains` functions as a wrapper for any command you run in your terminal. For example, by using `proxychains <command>`, it will direct traffic through the specified SOCKS4/5 proxy server. For instance, `proxychains firefox` will launch a new firefox instance that is wrapped around the proxy.

#### Setting up the Proxy

Firstly, we should configure `proxychains` to tunnel all traffic to `127.0.0.1:1081` (this can be changed later on).

```
sudo nano /etc/proxychains4.conf
```

At the end of the file, add the following line to specify that the SOCKS4 proxy is listening on `127.0.0.1:1081`.

```
socks4 127.0.0.1 1081
```

Additionally, since SOCKS4 doesn't support DNS proxying by default - you'll need to comment out the `proxy_dns` flag as well.

```
# proxy_dns
```

Next, we can use `ssh` to start a SOCKS4 proxy on `127.0.0.1:1081` and route the traffic through the `ssh` tunnel associated with the jump host. The syntax is as follows:

```sh
ssh <user>@<host> -D 1081
```

Additionally, the `-N` flag can be added to cancel the interactive session. Otherwise, an SSH session will be initiated along with the socks proxy. The following command can be used in the context of the CTF:
  
```sh
sshpass -p '8iP0TbO5mQmS' ssh svc_jmp@172.16.10.10 -D 1081 -N  
```

#### Testing the Proxy
  
After starting the SSH tunnel, we can now attempt to connect to the DC using [netexec](https://www.netexec.wiki/).
  
```sh
proxychains nxc smb 10.2.30.51  
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  10.2.30.51:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  10.2.30.51:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  10.2.30.51:445  ...  OK
SMB         10.2.30.51      445    KING-DC          [*] Windows Server 2022 Build 20348 x64 (name:KING-DC) (domain:async.empire) (signing:True) (SMBv1:False)   
```

As you may infer from the `[proxychains] Strict chain  ...  127.0.0.1:1081  ...  10.2.30.51:445  ...  OK`, the connection chain roughly resembles the following:
  
```
proxychains nxc smb 10.2.30.51  -> 127.0.0.1:1081 -> svc_jump@172.16.10.10 <-> 10.2.30.51:445
```
  
As a result, all internal traffic will appear to originate from the jump host (`172.16.10.1`).
  
## Kerberos
  
The primary authentication protocol in Active Directory is [Kerberos](https://en.wikipedia.org/wiki/Kerberos_(protocol)), if you're interested in learning the applications of Kerberos - I highly recommend the following [blog](https://ericesquivel.github.io/posts/kerberos) post by [Eric Esquivel](https://ericesquivel.github.io/). There are _many_ caveats in the protocol, but I will address the 2 main issues that you will almost definitely face while doing the lab.

1. DNS
2. Time Synchronization
  
### DNS
  
Active Directory is heavily dependent on DNS services for its proper functioning. DNS plays a critical role in locating domain controllers and other resources within the network. If Kerberos authentication is attempted and you have misconfigured DNS services, your operating system will not know which KDC to authenticate with.
  
In practice, if you attempt to authenticate to the `async.empire` domain controller with Kerberos authentication - you will find the following error.

```
proxychains -q nxc smb 10.2.30.51 -u 'Towns_Merchant_7' -p 'Hq7L2wPzX9' --kerberos
SMB         10.2.30.51      445    KING-DC          [*] Windows Server 2022 Build 20348 x64 (name:KING-DC) (domain:async.empire) (signing:True) (SMBv1:False) 
SMB         10.2.30.51      445    KING-DC          [-] async.empire\Towns_Merchant_7:Hq7L2wPzX9 [Errno Connection error (ASYNC.EMPIRE:88)] [Errno -2] Name or service not known  
```
  
This error occurs because your OS recognizes that the KDC is `async.empire`, however they do not have an IP to associate with that name. In most cases, the KDC is the Domain Controller - so we can use `/etc/hosts` to set this manually. You can add the following entry into `/etc/hosts`:
  
```
10.2.30.51   async.empire
```
  
On the next attempt, there should be no error as you're able to resolve the KDC to the right host.
  
 ```
 proxychains -q nxc smb 10.2.30.51 -u 'Towns_Merchant_7' -p 'Hq7L2wPzX9' --kerberos        
SMB         10.2.30.51      445    KING-DC          [*] Windows Server 2022 Build 20348 x64 (name:KING-DC) (domain:async.empire) (signing:True) (SMBv1:False) 
SMB         10.2.30.51      445    KING-DC          [+] async.empire\Towns_Merchant_7:Hq7L2wPzX9 
 ```

However, if we attempt to obtain a TGT and access the SMB shares on the DC using a tool such as [impacket](https://github.com/fortra/impacket). You may find the following error:
  
```
proxychains -q getTGT.py 'async.empire'/'Towns_Merchant_7':'Hq7L2wPzX9'            
Impacket v0.13.0.dev0+20250422.104055.27bebb13 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in Towns_Merchant_7.ccache
  
export KRB5CCNAME='Towns_Merchant_7.ccache'

proxychains -q smbclient.py -k -no-pass 10.3.30.51                     
Impacket v0.13.0.dev0+20250422.104055.27bebb13 - Copyright Fortra, LLC and its affiliated companies 
[-] Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
```
  
This error occurs because Kerberos is very particular about the hostnames associated with connection strings, and we are attempting to connect with a raw IP address. This issue can be resolved by writing FQDN entries for the domain-joined hosts. The general format is: `<hostname>.<domain-name>`, such as `10.2.30.51  King-DC.async.empire`.
  
After adding the above to `/etc/hosts`, we should use the `FQDN` with the ticket to access the service and there should be no errors.
  
  ```
proxychains -q smbclient.py -k -no-pass King-DC.async.empire
Impacket v0.13.0.dev0+20250422.104055.27bebb13 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# shares
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
# 
  ```
  
### Time Synchronization
  
Kerberos relies heavily on timestamps to encrypt and decrypt tickets. As a result, if the system time on the client machine is too far off from the time on the Domain Controller (DC), authentication will fail. This is a common issue, especially in scenarios where the client and DC are in different time zones or when working with overseas clients.

Consider the following, where I set my local time to 24 hours ahead of GMT+8 (Singapore Time) to intentionally de-sync with the Domain Controller:
  
```
sudo date --set "$(date -d '+24 hours')"
Thu May  8 12:16:32 PM EDT 2025 
```
  
If we attempt to do Kerberos authentication like before, we will get the following error:

```
proxychains -q smbclient.py -k -no-pass King-DC.async.empire
Impacket v0.13.0.dev0+20250422.104055.27bebb13 - Copyright Fortra, LLC and its affiliated companies 

[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```
  
The most common way to mitigate this, is to use `ntpdate` to re-synchronize our time with the domain controller using the NTP protocol.

```
sudo proxychains -q ntpdate King-DC.async.empire
[sudo] password for kali: 
2025-05-07 12:20:49.32724 (-0400) -86396.211347 +/- 0.003057 King-DC.async.empire 10.3.30.51 s1 no-leap
CLOCK: time stepped by -86396.211347

proxychains -q smbclient.py -k -no-pass King-DC.async.empire
Impacket v0.13.0.dev0+20250422.104055.27bebb13 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# shares
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
# 
```
  
The second way, is to use the SMB protocol with `net`. This is useful if you're in an environment where UDP traffic is blocked, but SMB is very unlikely to be blocked.
  
```
sudo proxychains -q net time set -S King-DC.async.empire
proxychains -q smbclient.py -k -no-pass King-DC.async.empire
Impacket v0.13.0.dev0+20250422.104055.27bebb13 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# shares
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
# 
```