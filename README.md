# ms-commands
Repository of command line scripting for Microsoft services 




## Connecting: #

Windows PowerShell needs to be configured to run scripts
```
Set-ExecutionPolicy RemoteSigned
```
Connect to Microsoft Online 
```
Connect-MsolService
Get-MSOLUser
```
Connect to Azure AD 
```
Connect-AzureAD
Get-AzureADUser
```
Connect to Exchange Online using Windows PowerShell
```
$UserCredential = Get-Credential

$Session = New-PSSession –ConfigurationName Microsoft.Exchange –ConnectionUri https://outlook.office365.com/powershell-liveid/ -Credential $UserCredential –Authentication Basic –AllowRedirection

Import-PSSession $Session
```
Connect to Exchange Online using Windows PowerShell + MFA 
```
Connect-EXOPSSession -UserPrincipalName <UPN>
```
Closes one or more PowerShell session
```
Remove-PSSession $Session
Get-PSSession | Remove-PSSession
```
Kill all sessions by using the Revoke-AzureADUserAllRefreshToken cmdlet
```
Connect-AzureAD

Get-AzureADUser

Get-AzureADUser -SearchString user@user.com | Revoke-AzureADUserAllRefreshToken 
```
If you are investigating an incident and you believe a user’s token has been captured, you can invalidate a token with this AAD PowerShell cmdlet
```
$date =get-date; Set-Msoluser -UserPrincipalName (UPN) -StsRefreshTokensValidFrom $date
```
-----------------------------
## Auditing #
List all Office 365 mailboxes with mail addresses and stats 
```
Get-Mailbox | Select-Object DisplayName, PrimarySMTPAddress,TotalItemSize | Out-GridView
```
Check for soft deleted mailbox
```
Get-Mailbox -SoftDeletedMailbox | Select-Object Name,ExchangeGuid
```
Check mailbox audit delegation
```
Get-mailbox | select UserPrincipalName, auditenabled, AuditDelegate, AuditAdmin, AuditLogAgeLimit | Out-gridview
Get-Mailbox | FL name, audit*
```
Get mobile devices used 
```
 Get-MobileDevice -Mailbox "User Test" |  Select DeviceModel,DeviceUserAgent,DeviceAccessState,ClientType
```
Retrieve the permissions assigned  
https://practical365.com/exchange-server/list-users-access-exchange-mailboxes/
```
Get-Mailbox -RecipientType 'UserMailbox' -ResultSize Unlimited | Get-MailboxPermission
Get-Mailbox | Get-MailboxPermission | where {$_.user.tostring() -ne "NT AUTHORITY\SELF"}
Get-Mailbox | Get-MailboxPermission | where {$_.User.tostring() -Match "Disc*"}
Get-Mailbox | Get-MailboxPermission | where {$_.user.tostring() -ne "NT AUTHORITY\SELF" -and $_.IsInherited -eq $false} | Select Identity,User,@{Name='Access Rights';Expression={[string]::join(', ', $_.AccessRights)}} | Export-Csv -NoTypeInformation mailboxpermissions.csv
```
List all users and date of last password change
https://www.quadrotech-it.com/blog/list-all-users-and-date-of-last-password-change-in-office-365/
```
Get-MsolUser -All | select DisplayName, LastPasswordChangeTimeStamp | Out-GridView
Get-MsolUser -All | select DisplayName, LastPasswordChangeTimeStamp | Export-CSV LastPasswordChange.csv -NoTypeInformation
```
-----------------------------

## Authentication Protocols #
https://www.itpromentor.com/block-basic-auth/
Show mailbox services for a user
```
Get-CASMailbox -identity <username@domain.com> | fl Name,OwaEnabled,MapiEnabled,EwsEnabled,ActiveSyncEnabled,PopEnabled,ImapEnabled
```
Get mailbox EWS Access
```
 Get-CASMailbox "Ann Test" | Select *EWS*
```
Modify mailbox services in bulk
```
Get-Mailbox | Set-CasMailbox -PopEnabled $false -ImapEnabled $false 
-SmtpClientAuthenticationDisabled $true
```
Modify default mailbox services
```
Get-CASMailboxPlan | Set-CASMailboxPlan -ImapEnabled $false -PopEnabled $false
```
Show the remote PowerShell access status for all users
```
Get-User -ResultSize unlimited | Format-Table -Auto Name,DisplayName,RemotePowerShellEnabled
```
Show only users who don't have access to remote PowerShell
```
Get-User -ResultSize unlimited -Filter {RemotePowerShellEnabled -eq $false}
```
Get user with auth policy
```
Get-User -Filter {AuthenticationPolicy -eq 'No Basic Auth'}
```
Get authentication policy
```
Get-AuthenticationPolicy | Format-List Name,DistinguishedName
```
Checking Applied Policies to Accounts and output to file 
```
Get-Recipient -RecipientTypeDetails UserMailbox -ResultSize Unlimited | Get-User | Format-Table DisplayName, AuthenticationPolicy, Sts* > D:\output.txt
```
Set authentication policy to allow basic authentication for a protocol
```
Set-AuthenticationPolicy -Identity "No Basic Auth" -AllowBasicAuthPop:$True
```
Set authentication policy for all users  
```
Get-User -ResultSize unlimited | Set-User -AuthenticationPolicy “Block Basic Auth”
```
Set authentication policy for a user and reset it's token enforcing the policy 
```
get-user RandomUser |  Set-User -AuthenticationPolicy "No Basic Auth" -STSRefreshTokensValidFrom $([System.DateTime]::UtcNow)
```

-----------------------------

## Collecing #
Export users  and mailboxes to csv 
```
Foreach-Object ($mailbox in (Get-Mailbox))
{
	Get-MsolUser –UserPrincipalName $mailbox.userprincipalname
}
Get-MsolUser -All | Export-Clixml .\MsolUser.XML
Get-Mailbox –ResultSize Unlimited | Export-CliXML .\Mailboxes.XML
```
use Import-CliXML to import the Mailboxes.XML file
```
Import-CliXML .\Mailboxes.XML
```
Get the audit log data by email (for TestUser1 mailbox)
```
New-MailboxAuditLogSearch –Mailboxes “TestUser1” –LogonTypes admin, Delegate –StartDate 07/12/2017 –EndDate 11/12/2017 –StatusMailRecipientsadministrator@www.domain.com –ShowDetails
```
-----------------------------
## Mailbox Auditing 

Enable mailbox audit logging for all the user mailboxes in your organization.
```
Get-Mailbox -ResultSize Unlimited -Filter {RecipientTypeDetails -eq "UserMailbox"} | Set-Mailbox -AuditEnabled $true
Get-Mailbox -Filter {AuditEnabled -eq $False} -RecipientTypeDetails UserMailbox, SharedMailbox | Set-Mailbox -AuditEnabled $True –AuditDelegate Create, FolderBind, SendAs, SendOnBehalf, SoftDelete, HardDelete, Update, Move, MoveToDeletedItems, UpdateFolderPermissions
```
Run the Get-MailboxAuditBypassAssociation cmdlet and filter for accounts with a bypass
```
Get-MailboxAuditBypassAssociation | ? {$_.AuditBypassEnabled -eq $True} | Format-Table Name, WhenCreated, AuditBypassEnabled 
```
Search MailboxAuditlog
```
Search-MailboxAuditLog -Identity Dimitri Schmitz -LogonTypes Admin,Delegate -StartDate 06/01/2018 -EndDate 07/20/2018 -ResultSize 2000
```
Verify mailbox auditing on by default is turned on
```
Get-OrganizationConfig | Format-List AuditDisabled
```
-----------------------------

## References  
https://github.com/OfficeDev/O365-InvestigationTooling

### Connecting:
Connect to Exchange Online PowerShell using multi-factor authentication
https://docs.microsoft.com/en-us/powershell/exchange/exchange-online/connect-to-exchange-online-powershell/mfa-connect-to-exchange-online-powershell

Killing Sessions to a Compromised Office 365 Account  
https://blogs.technet.microsoft.com/cloudyhappypeople/2017/10/05/killing-sessions-to-a-compromised-office-365-account/

Remove-PSSession  
https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/remove-pssession

### Mailbox:
A Step-By-Step Guide to Enable Mailbox Auditing of Exchange Online (Office 365)  
https://www.lepide.com/how-to/enable-mailbox-auditing-of-exchange-online.html

How to Track Who Accessed Mailboxes in Exchange Server 2016  
https://www.lepide.com/how-to/track-who-accessed-mailboxes-in-exchange-server-2016.html

Manage mailbox auditing  
https://docs.microsoft.com/en-us/office365/securitycompliance/enable-mailbox-auditing

Office 365: Grant a User Full Mailbox Access to all Mailboxes 
https://www.petenetlive.com/KB/Article/0001466

Exchange: Mailbox Delegate Report using PowerShell  
https://gallery.technet.microsoft.com/office/Exchange-Mailbox-Delegate-27d1e4fb

Exchange: Mailbox Delegate Report using PowerShell  
https://social.technet.microsoft.com/wiki/contents/articles/51341.exchange-mailbox-delegate-report-using-powershell.aspx

### Access:
Manage Remote PowerShell Access to Exchange Online  
https://www.petri.com/managing-remote-powershell-access-exchange-online

Control remote PowerShell access to Exchange servers  
https://docs.microsoft.com/en-us/powershell/exchange/exchange-server/control-remote-powershell-access-to-exchange-servers?view=exchange-ps

Control access to EWS in Exchange  
https://docs.microsoft.com/en-us/exchange/client-developer/exchange-web-services/how-to-control-access-to-ews-in-exchange

Disabling Basic Authentication for Exchange Online (Preview)  
https://office365itpros.com/2018/10/24/disable-basic-authentication-exchange-online/

### Misc:
Defending against Evilginx2 in Office 365  
http://www.thecloudtechnologist.com/category/security/

The top 6 PowerShell commands you need to know to manage Office 365  
https://practical365.com/microsoft-365/the-top-6-powershell-commands-you-need-to-know-to-manage-office-365/

