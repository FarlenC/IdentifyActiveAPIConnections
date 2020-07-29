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



PowerShell Script:
--------------------------------------------------------------------------------------------


#Variables
$logicApps = ""
$APIConnections = ""
$Parameters = ""
$logicAppName = @()
$logicAppCounter = 0
$ConnectionIDs = @()
$ConnectedAPI = ""
$ConnectedAPIO = @()
$DisconnectedAPI = ""
$DisconnectedAPIO = @()
$Pcounter = 0
$connectedAPICount = 0
$disconnectedAPICount = 0
$APIConnectionCounter = 0
$APIFound = 0
$option = ""
$exportConnectedAPIs = $home+"\Desktop\ConnectedAPIs.csv"
$exportDisconnectedAPIs = $home+"\Desktop\DisconnectedAPIs.csv"


#script
Write-Host "Searching for API Connection resources in the subscription..."
#get all API Connection resources in a subscription
$APIConnections = Get-AzResource | where ResourceType -EQ Microsoft.Web/connections

Write-Host "Searching for Logic Apps in the subscription..."
#get all Logic Apps in a subscription
$logicApps = Get-AzLogicApp

Write-Host "Identifying active connections... please wait"
#access to the workflow definition for each logic app found in the subscription to get all the API Connections in the parameters section
foreach($LA in $logicApps){
    if($LA.Parameters.Keys.Contains('$connections')){
    $Collection = (($LA.Parameters.Values | ConvertTo-Json) | ConvertFrom-Json).value
        foreach($CollectIDs in $Collection){
        $ConnectionIDs += $CollectIDs.Split(':').Split(' ')[2].Replace('"','')
            $APIConnectionCounter = $APIConnections.Count
            $logicAppName = $logicAppName + $logicApps[$logicAppCounter].Name
        }

    }
    $logicAppCounter += 1
}

#having a list of existing API Connection resources in the subscription and a list of all Connection IDs found in the parameters section 
#for all logic apps in the subscription, now the script search for matches to identify API Connections that are in use for any Logic App.
foreach($APIConnection in $APIConnections){

    $APIFound = 0
    $position = 0

    foreach($ConnectionID in $ConnectionIDs){

        If($APIConnection.ResourceId -eq $ConnectionID){
            $APIFound += 1
            $connectedAPICount += 1
            $ConnectedAPI = "" | Select ConnectionId, LogicApp
            $ConnectedAPI.ConnectionId = $APIConnection.ResourceId
            $ConnectedAPI.LogicApp = $logicAppName[$position]

            $ConnectedAPIO += $ConnectedAPI
        }

    $position += 1

    }
  
    if($APIFound -eq 0){
        $disconnectedAPICount += 1
        $DisconnectedAPI = "" | Select ConnectionId
        $DisconnectedAPI.ConnectionId = $APIConnection.ResourceId

        $DisconnectedAPIO += $DisconnectedAPI
    }
  
}

#output a label including the number of connected API Connections found
Write-Host "`nResults:`n"
Write-Host $connectedAPICount "Connected API Connection resources found: `n" -BackgroundColor Black -ForegroundColor Green

#output the connected API Connection resource IDs and the logic app to which it is connected
for($i = 0; $i -lt $ConnectedAPIO.Count; $i++){
    Write-Host $ConnectedAPIO[$i].ConnectionId -BackgroundColor Black -ForegroundColor Green
    Write-Host "Status: Connected to Logic App : " $ConnectedAPIO[$i].LogicApp `n -BackgroundColor Black -ForegroundColor Green
}

#validates if there are disconnected API Connections to show the output
if($DisconnectedAPIO.Length -eq 0){
    Write-Host "No disconnected API connection resources found" -BackgroundColor White -ForegroundColor Blue

    #user menu if no disconnected API connections found
    $option = Read-Host "`n`nMenu:`n`n1 = Export connected API connections to csv file`n    Or press any other key to exit`n",
    "`nTo continue select a menu option and press enter`n"
    

    Switch($option){
        1{$ConnectedAPIO | Select @{N="API Connection Resource Id";E={$_.ConnectionId}}, @{N="Connected to Logic App";E={$_.LogicApp}} | Export-Csv -Path $exportConnectedAPIs -NoTypeInformation
        Write-Host "File has been exported in this path:" $exportConnectedAPIs }
        default{break}
    }
}
else{
    #output a label including the number of disconnected API Connections found
    Write-Host $disconnectedAPICount "Disconnected API Connection resources found: `n" -BackgroundColor Black -ForegroundColor Red

    #output the disconnected API Connection resource IDs
    for($i = 0; $i -lt $DisconnectedAPIO.Count; $i++){
    Write-Host $DisconnectedAPIO[$i].ConnectionId -BackgroundColor Black -ForegroundColor Red
    }

    #User menu
    $option = Read-Host "`n`nMenu:`n`n",
    "1 = Export only connected API connections to csv file`n",
    "2 = Export only disconnected API connections to csv file`n",
    "3 = Delete all disconnected API Connection resources found`n",
    "4 = Exit`n`n",
    "To continue select a menu option and press enter`n"

    Switch($option){

        1{$ConnectedAPIO | Select @{N="API Connection Resource Id";E={$_.ConnectionId}}, @{N="Connected to Logic App";E={$_.LogicApp}} | Export-Csv -Path $exportConnectedAPIs -NoTypeInformation
        Write-Host "File has been exported in this path:" $exportConnectedAPIs }

        2{$DisconnectedAPIO | Select @{N="API Connection Resource Id";E={$_.ConnectionId}} | Export-Csv -Path $exportDisconnectedAPIs -NoTypeInformation 
        Write-Host "File has been exported in this path:" $exportDisconnectedAPIs }

        3{$option4 = Read-Host "`nDo you really want to delete all Disconnected API connection resources found?`n`ny = Yes`nn = No, cancel the process`n`nSelect an option to confirm`n"
             
             Switch($option4){
             
             y{Foreach($id in $DisconnectedAPIO){
                    Remove-AzResource -ResourceId $id.ConnectionId -Force -ErrorAction SilentlyContinue
                 }
                 Write-Host "All Disconnected API Connection resources found have been deleted" -ForegroundColor White -BackgroundColor Blue
             }

             n{Write-Host "You have cancelled the deletion process" -ForegroundColor White -BackgroundColor Blue }

             default{Write-Host "Your input did not confirm or deny, the script has cancelled the deletion process" -ForegroundColor White -BackgroundColor Blue}

             }
             
         } 
           
        4{break}

        Default{Write-Host "Invalid input" -BackgroundColor Black -ForegroundColor Red}
    }
}
