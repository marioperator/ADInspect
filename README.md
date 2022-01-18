# AD*Inspect*

# Purpose
Following in the footsteps of Soteria's 365Inspect tool, AD-Inspect aims to further the state of Active Directory security by authoring a PowerShell script that automates the security assessment of Microsoft Active Directory environments.


# Setup
AD*Inspect* requires that the Active Directory and LAPS PowerShell modules be installed, and that the auditor has sufficient rights to perform all required queries.
Domain Administrator is recommended as certain modules connect to Domain Controllers to audit software and services installed.


# Usage
To run AD*Inspect*, open a PowerShell console as a Domain Admin, using either a Privileged Administrator Workstation or RunAs, and navigate to the AD*Inspect* folder:
```
cd ADInspect
```
You will interact with AD*Inspect* by executing the main script file, ADInspect.ps1, from within the PowerShell command prompt.

All AD*Inspect* requires to inspect your Active Directory environment is access via an account with proper permissions and a defined path for the report that will be generated by the audit.

Execution of AD*Inspect* looks like this:

```powershell
.EXAMPLE
  PS> .\ADInspect.ps1 -OutPath "C:\Reports"

.EXAMPLE
  PS> .\ADInspect.ps1 -Username admin -OutPath "C:\Reports"
```


# Developing Inspector Modules
AD*Inspect* is designed to be easy to expand, with the hope that it enables individuals and organizations to either utilize their own AD*Inspect* modules internally, or publish those modules for the O365 community.

All of AD*Inspect*'s inspector modules are stored in the .\inspectors folder.

It is simple to create an inspector module. Inspectors have two files:

ModuleName.ps1: the PowerShell source code of the inspector module. Should return a list of all O365 objects affected by a specific issue, represented as strings.
ModuleName.json: metadata about the inspector itself. For example, the finding name, description, remediation information, and references.
The PowerShell and JSON file names must be identical for AD*Inspect* to recognize that the two belong together. There are numerous examples in AD*Inspect*'s built-in suite of modules, but we'll put an example here too.

Example .ps1 file, Inspect-PasswordLongevity.ps1:

```powershell
# Define a function that we will later invoke.
# ADInspect's built-in modules all follow this pattern.
<#
.SYNOPSIS
    Gather information about Active Directory accounts with long-lived passwords
.DESCRIPTION
    This script checks Active Directory Accounts for long-lived passwords and offers remediation steps
.COMPONENT
    PowerShell, Active Directory PowerShell Module, and sufficient rights to change admin accounts
.ROLE
    Domain Admin or Delegated rights
.FUNCTIONALITY
    Gather information about Active Directory accounts with long-lived passwords
#>

Function Inspect-PasswordLongevity{
    $pwdNotchanged = Get-ADUser -filter {(PasswordLastSet -lt (Get-Date).adddays(-120)) -and (PasswordNeverExpires -eq $false)} -Properties PasswordLastSet
    
    if ($pwdNotchanged.count -ne '0' -and $pwdNotchanged.enabled -eq $true){
        Return $pwdNotchanged
    }
}

Return Inspect-PasswordLongevity
```

Example .json file, Inspect-PasswordLongevity.json:

```json
{
	"FindingName": "Accounts with Long-lived Passwords Found",
	"Description": "Accounts with passwords that are older than 120 days were identified. Passwords that remain unchanged for long periods of time are more susceptable to being cracked",
	"Remediation": "Identified accounts should have their passwords changed. This can be accomplished by setting the 'Must change password at next logon' flag, or running the following PowerShell command: 'Get-ADUser -filter {(PasswordLastSet -lt (Get-Date).adddays(-120)) -and (PasswordNeverExpires -eq $false)} | Set-ADUser -ChangePasswordAtLogon $true'. Additionally, Group Policy Objects and Domain Password Policies should be enforced on all accounts.",
	"AffectedObjects": "",
	"References": [
		{
			"Url": "https://activedirectorypro.com/how-to-configure-a-domain-password-policy/",
			"Text": "How To Configure a Domain Password Policy"
		}
	]
}
```

Once you drop these two files in the .\inspectors folder, they are considered part of AD*Inspect*'s module inventory and will run the next time you execute AD*Inspect*.

You have just created the Inspect-PasswordLongevity Inspector module. That's all!

AD*Inspect* will throw a pretty loud and ugly error if something in your module doesn't work or doesn't follow AD*Inspect* conventions, so monitor the command line output.


# Output
AD*Inspect* creates the directory specified in the out_path parameter. This directory is the result of the entire AD*Inspect* inspection. It contains three items of note:

 - Report.html: graphical report that describes the Active Directory security issues identified by AD*Inspect*, lists AD objects that are misconfigured, and provides remediation advice.
 - Various text files named [Inspector-Name]: these are raw output from inspector modules and contain a list (one item per line) of misconfigured AD objects that contain the described security flaw. For example, if a module Inspect-FictionalPasswordSettings were to detect all users who do not have passwords, the file "Inspect-FictionalPasswordSettings" in the report ZIP would contain one user per line who does not have a password. This information is only dumped to a file in cases where more than 15 affected objects are discovered. If less than 15 affected objects are discovered, the objects are listed directly in the main HTML report body.
 - Report.zip: zipped version of this entire directory, for convenient distribution of the results in cases where some inspector modules generated a large amount of findings.


# About Security
AD*Inspect* is a script harness that runs other inspector script modules stored in the .\inspectors folder. As with any other script you may run with elevated privileges, you should observe certain security hygiene practices:

No untrusted user should have write access to the AD*Inspect* folder/files, as that user could then overwrite scripts or templates therein and induce you to run malicious code.
No script module should be placed in .\inspectors unless you trust the source of that script module.
