# IdentifyActiveAPIConnections
When you create a connection with a service or system, Logic Apps creates an API Connection that is a separate resource in Azure, when you delete or just stop using an API Connection in your Logic App, the LA does not remove automatically that disconnected API Connection. If you have plenty LAs and API Connections it can be hard to manually identify Connected or Disconnected API Connections from the Azure Portal.  This PowerShell script is for knowing which API connections are in use and which ones are no longer in use. 



Before running the script check the following steps:
-------------------------------------------------------------------------------------------

1 - First you need to install the AZ module: (if already installed, skip this step)
Install-Module AZ
 
2 - Connect to your azure account where the Logic App is running:
Connect-AzAccount
 
3 - if you need to change the subscription run this command to see the subscriptions you have access to:
Get-AzSubscription

4 - then select your subscription running this command:
Select-AzSubscription -SubscriptionId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

If you cannot run scripts due to the execution policy run this command:
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope LocalMachine


