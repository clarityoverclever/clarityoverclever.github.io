---
title: "Secure PowerShell Automation: Credential Management Across Windows and Linux"
author: "Keith Marshall"
date: 2025-11-19
categories: [tutorial]
tags: [patterns]
excerpt: "Learn how to safely store and retrieve credentials in PowerShell using DPAPI on Windows, custom key derivation on Linux/macOS, and the cross‑platform SecretManagement module."
layout: post
---

## Credential Management
Automation often requires securely storing credentials so scripts can interact with protected systems. Because credentials are sensitive, mishandling them can expose entire systems to compromise. We all know that hardcoding credentials into automations is risky, so the real challenge is finding the right balance: how do we keep PowerShell automations efficient while ensuring strong security?  

## Storing Credentials
When storing credentials on a Windows system, PowerShell can leverage the built-in Data Application Programming Interface (DPAPI) which allows applications to securely encrypt and decrypt sensitive data using keys derived from the user profile and machine; however, there is no direct equivilant to DPAPI on Linux/Unix systems which can lead to unexpected behavior if those cmdlets are used on such systems. Let's explore how this works in practice:

### Windows PowerShell (Windows)
Consider the following functions that prompt a Windows user for a credential, exports it into an XML file comprising of a username string, a SecureString password, and some metadata.

We can later import the XML file and read the credential directly.

Since DPAPI leverages OS level interactions and encrypts the password in a way that binds it to the machine and user, it makes for a simple method of storing credentials for PowerShell automations on a Windows system.

```powershell
function New-CredentialToFile {
    param (
        [Parameter(Position = 0, Mandatory = $true)]
        [string] $KeyPath,

        [Parameter(Position = 1, Mandatory = $true)]
        [string] $KeyName
    )

    [pscredential] $credential = Get-Credential

    $credential | Export-Clixml $(Join-Path -Path $KeyPath -ChildPath $KeyName)
}

function Import-CredentialFromFile {
    param (
        [Parameter(Position = 0, Mandatory = $true)]
        [string] $Path
    )

    # verify the file exist
    if (-not Test-Path -Path $Path) {
         throw "Supplied path cannot be found: $Path"
    }

    [pscredential] $credential = $(Import-Clixml -Path $Path)

    # validate the credential object
    if (-not ($Credential -is [System.Management.Automation.PSCredential])) {
        throw "Parameter must be a PSCredential object."
    }

    return $credential
}

function Read-SecureString {
    param (
        [Parameter(Position = 0, Mandatory = $true)]
        [securestring] $SecureString
    )

    try {
        $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecureString)
        return [System.String][Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)   
    } finally {
        # zero out $bstr to prevent sensitive data from lingering in memory
        if ($bstr -ne [IntPtr]::Zero) {
            [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
        }
    }
}
```
> The credential username is not considered sensitive and can be read like any System.String while the transformation of the SecureString password into plaintext is done with:
> 
> [System.String][Runtime.InteropServices.Marshal]::PtrToStringAuto(
>   [Runtime.InteropServices.Marshal]::SecureStringToBSTR($Credential.Password)
> )
>
{: .notice--danger}
> ⚠️ **Security Note**
> Decrypting the SecureString is not always necessary as some applications take SecureStrings or PsCredential objects as parameters. When it is necessary to decrypt and assign a SecureString, make sure to destroy the variable after use.  
> Remove-Variable -Name _variable_ -Force

### PowerShell Core (Linux/Unix)
To approximate the same behavior on a non-Windows system, we will need to generate an encryption key using collectable identifiers and passing that key as a parameter during the SecureString creation.

```powershell
function Get-UserMachineKey {
    # collect unique identifiers and combine them
    $user = $env:USER ?? $env:USERNAME
    $machine = $env:HOSTNAME ?? $env:COMPUTERNAME ?? (& hostname)
    $raw = "$user@$machine"

    # generate a hash from the collected values
    $sha256 = [System.Security.Cryptography.SHA256]::Create()
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($raw)
    $hash = $sha256.ComputeHash($bytes)

    # return 32 bytes of the hash for use as a symmetric encryption key
    return $hash[0..31]  # 32-byte AES key
}

function New-CredentialToFile {
    param (
        [Parameter(Position = 0, Mandatory = $true)]
        [string] $Path
    )

    $credential = Get-Credential
    $key = Get-UserMachineKey

    $secureText = $credential.Password | ConvertFrom-SecureString -Key $key

    $object = [PSCustomObject]@{
        UserName = $credential.UserName
        Password = $secureText
    }

    $object | Export-Clixml $Path
}

function Import-CredentialFromFile {
    param (
        [Parameter(Mandatory = $true)]
        [string] $Path
    )

    $credential = Import-Clixml -Path $Path
    $key = Get-UserMachineKey

    $secureString = $credential.Password | ConvertTo-SecureString -Key $key
    return New-Object System.Management.Automation.PSCredential($credential.UserName, $secureString)
}
```
> - When collecting identifiers, I’m using the PowerShell Core **null‑coalescing operator (`??`)** to fall back gracefully if an environment variable doesn’t exist. This ensures the function always resolves a value without throwing errors.

> - In my example, I limited the identifiers to the logged‑in user and machine name; however, you can substitute any values that are reasonably unique in your environment. 

> - As an alternative, you could generate a random hash once, store it in a file with strict permissions (similar to an SSH private key, e.g. `chmod 600`), and then import it as the encryption key:
>   ```powershell
>   $key = Get-Content -Path $KeyPath -Raw
>   ```


### Crossplatform Combined
On Windows, the Linux/Unix approximation function above will work as expected, but it is not as strong as invoking DPAPI. We can; however, combine the operating system specific functions together and select which to use based on the calling OS using some PowerShell built-ins.
```powershell
if ($IsWindows) {
    # use DPAPI-backed functions
    $cred = Import-CredentialFromFile -Path "C:\keys\myCred.xml"
} elseif ($IsLinux -or $IsMacOS) {
    # use the custom key-based functions
    $cred = Import-CredentialFromFile -Path "/home/user/myCred.xml"
}
```
{: .notice--info}
> 💡 **Cross‑platform Note**  
> Both the Windows (DPAPI) and Linux/macOS (key‑based) implementations return a `PSCredential` object. 
> This means your downstream automation logic can remain platform‑agnostic.

### Crossplatform Compromise
Microsoft offers a SecretManagement module which can be installed from the PSGallery on both Windows and Linux/Unix machines. This module creates a secrets store that can house sensitive data, and encrypts that store with an symmetric key.

The obvious disadvantage to this method is that you will need to either unlock the vault manually per session, or automate the unlock with one of the methods from above.

```powershell
# install the module from PSGallery
Install-Module Microsoft.PowerShell.SecretManagement
Install-Module Microsoft.PowerShell.SecretStore

# register a secrets vault
Register-SecretVault -Name MyVault -ModuleName Microsoft.PowerShell.SecretStore -DefaultVault

# store credential
$credential = Get-Credential
Set-Secret -Name MyServiceCred -Secret $credential

# retrieve credential
$credential = Get-Secret -Name MyServiceCred
```

## Final Thoughts
In this post I attempted to illustrate some of the options available to you for secrets management. Like so many things, there is no "one right solution" and each approach comes with trade-offs that you will need to balance against your needs, and your environment.

As you design PowerShell automations, aim to keep credentials out of plain text, prefer `PSCredential` and `SecureString` objects where possible, and consider adopting SecretManagement for long‑term, cross‑platform workflows. By balancing efficiency with security, you can build automations that are both powerful and responsible.
