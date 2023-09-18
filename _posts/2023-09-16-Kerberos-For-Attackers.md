As a domain user we need to access resources within the environment. These resources are available via services such as HTTP, SMB, and LDAP. Active Directory (AD) uses Kerberos to manage authentication to these services. 

## Example

We're the domain user, `skyler.knecht@rayke.local`, and we would like to authenticate to the SMB service on `ws01.rayke.local`.

Typically we'd authenticate with plaintext credentials when using the NTLM protocol, however, with Kerberos we'll need a Ticket Granting Ticket (TGT).

To obtain a TGT we must negotiate with the Key Distribution Center's (KDCs) Authentication Service (AS) with a tool such as [getTGT.py](https://github.com/fortra/impacket/blob/master/examples/getTGT.py) from the Impacket suite.

```console
getTGT.py rayke.local/skyler.knecht:Password1! -dc-ip 192.168.1.200 
```

This will generate a ccache file entitled, `skyler.knecht.ccache`. We can use this file to authenticate by assigning it to the environment variable `KRB5CCNAME` and using a "Kerberos aware" tool such as [CrackMapExec](https://github.com/Porchetta-Industries/CrackMapExec) (CME).

```console
export KRB5CCNAME=skyler.knecht.ccache
```

To signal CME to use Kerberos authentication we must provide the command line argument, `-k`. 

```
skyler@debian:~$ crackmapexec smb ws01.rayke.local -k
SMB         ws01.rayke.local 445    WS01             [*] Windows 10.0 Build 19041 x64 (name:WS01) (domain:rayke.local) (signing:False) (SMBv1:False)
... 
OSError: [Errno Connection error (RAYKE.LOCAL:88)]
```

> We must always use the target's Fully Qualified Domain Name (FQDN) when authenticating with Kerberos.
{: .prompt-warning }

Surprisingly, CME makes another request to the KDC to obtain additional information prior to authentication. Since we did not provide CME where to locate the KDC the authentication fails.

## Why did this fail?

When we requested a TGT, `getTGT.py` made an Authentication Service Request (AS-REQ) to the KDC. This request includes a Client Name (cname) and Service Name (sname). In this case the cname would be `skyler.knecht` and the service name would be `krbtgt/rayke.local`. 

The KDC processes this information and determined that pre authentication is required, thus made an Authentication Service Response (AS-REP) with the error, `KRB5KDC_ERR_PREAUTH_REQUIRED`.

Pre Authentication data (PA-data) is a timestamp encrypted with the client's kerberos secret, typically the user's password. 

`getTGT.py` used the plaintext credentials we provided and made another AS-REQ with the PA-data appended. 

The KDC processed the PA-authentication data by decrypting the timestamp using the client's kerberos secret.

> Since a KDC requires access to the client's kerberos secret it is typically located on a Domain Controller. 
{: .prompt-info }

Upon successful decryption, The KDC made an Authentication Service Response (AS-REP) with a TGT and session key appended. `getTGT.py` processes this response and writes the ccache file, `skyler.knecht.ccache`, containing our TGT to disk.

Unlike NTLM, we cannot use our credentials, the TGT, to directly authenticate to a service. Instead we need to use our TGT to obtain a Service Ticket (ST). CME identifies this an attempts to request a ST from the KDC. However, CME cannot locate the KDC and the authentication fails. 

## How do we obtain a Service Ticket?

We can request a ST by making an AS-REQ with the sname set to the target service.

Services are identified by their Service Principal Name (SPN). In this case we use `host/ws01.rayke.local` as this SPN is used to provide authentication to the SMB service. 

`getTGT.py` permits us to make an AS-REQ with this configuration by providing the command line argument, `-service`.

```console
getTGT.py rayke.local/skyler.knecht:Password1! -dc-ip 192.168.1.200 -service host/ws01.rayke.local
```

With new ccache file we can authenticate successfully. 

```
skyler@debian:~$ crackmapexec smb ws01.rayke.local -k
SMB         ws01.rayke.local 445    WS01             [*] Windows 10.0 Build 19041 x64 (name:WS01) (domain:rayke.local) (signing:False) (SMBv1:False)
SMB         ws01.rayke.local 445    WS01             [+] rayke.local\skyler.knecht (Pwn3d!)
```

This is inefficient as we'll need to obtain a new TGT for every ST. This includes encountering the `KRB5KDC_ERR_PREAUTH_REQUIRED` error and encrypting a timestamp. 

Alternatively, we can negotiate with the Key Distribution Center's (KDCs) Ticket Granting Service (TGS) with a tool such as [getST.py](https://github.com/fortra/impacket/blob/master/examples/getST.py) from the Impacket suite.


```console
getST.py rayke.local/skyler.knecht -spn host/ws01.rayke.local -k -no-pass -dc-ip 192.168.1.200
```

This permits us to request STs up to the expiration date of our TGT skipping the TGT-generation process. 

To obtain a Service Ticket, `getST.py` made an Ticket Granting Service Request (TGS-REQ). This request includes an authenticator, our TGT and the sname, `host/ws01.rayke.local`. 

The authenticator is the session key from the AS-REP encrypted with our kerberos secret. 

The TGT is a session key along with other metadata encrypted with the KDC's kerberos secret, typically the krbtgt user's password.

The sname is the SPN of the service that we're authenticating to. 

To verify the authenticity of the request, the KDC will begin by decrypting both the authenticator and TGT recovering a session key from both. If the session keys are the same then the request is authentic.

Once authenticated, the KDC will create a ST for the sname we provided and a session key. The ST contains information from the TGT along with other meta data such as the user's groups. The ST is encrypted with the service owner's kerberos secret. In this case `ws01$@rayke.local`. Once encrypted the KDC will make an TGS-REP. The contents of this response are both the session key and ST encrypted with the user's kerberos secret. 

`getTGT.py` processes this response and writes the ccache to disk. We can assign this to the `KRB5CCNAME` environment variable and successfully authenticate to the service. 

```
skyler@debian:~$ export KRB5CCNAME=skyler.knecht@host_ws01.rayke.local@RAYKE.LOCAL.ccache
skyler@debian:~$ crackmapexec smb ws01.rayke.local -k
SMB         ws01.rayke.local 445    WS01             [*] Windows 10.0 Build 19041 x64 (name:WS01) (domain:rayke.local) (signing:False) (SMBv1:False)
SMB         ws01.rayke.local 445    WS01             [+] rayke.local\skyler.knecht (Pwn3d!)
```

Thankfully, most tools will automated ST-requesting process given a TGT and the location of the KDC. We can provide CME with the command line argument, `--kdcHost` to achieve this.

```
skyler@debian:~$ crackmapexec smb ws01.rayke.local -k --kdcHost 192.168.1.200
SMB         ws01.rayke.local 445    WS01             [*] Windows 10.0 Build 19041 x64 (name:WS01) (domain:rayke.local) (signing:False) (SMBv1:False)
SMB         ws01.rayke.local 445    WS01             [+] rayke.local\skyler.knecht (Pwn3d!)
```

## What about authorization?

The Kerberos protocol provides a means of verifying the authenticity principals and relies on the service for authorization. CrackMapExec will use our ccache file to make an Application Server Request (AS-REQ) to the service. This request contains our ST and an authenticator. 

The authenticator contains metadata such as a sequence number and is encrypted with the session key obtained from the TGS-REP. 

The Application Server will decrypt the ST recovering the session key and the groups associated with the user. The Application Server will use the session key to decrypt the authenticator and recover the sequence number. If the sequence number has already been received the Application Server the request will be dropped. Otherwise, the Application Server will parse the user's groups and make an AS-REP either informing the user the status of their authorization.




