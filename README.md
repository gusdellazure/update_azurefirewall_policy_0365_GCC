#GCC json
https://endpoints.office.com/endpoints/worldwide?clientrequestid=b10c5ed1-bad1-445f-b386-b919946339a7

Here is the complete PowerShell script:
#set Firewall rules subcription 
set-AzContext -Subscription  Yourazsubcription

```powershell
# Define the path to the JSON file
$jsonFilePath = "path_to_your_json_file.json"

# Import the JSON file
$jsonContent = Get-Content -Path $jsonFilePath -Raw | ConvertFrom-Json

# Define variables for the existing Azure Firewall policy
$resourceGroupName = "YourResourceGroupName"
$firewallPolicyName = "YourFirewallPolicyName"

# Function to create network rule
function Create-NetworkRule {
    param (
        [string]$Name,
        [string]$SourceAddresses,
        [string]$DestinationAddresses,
        [string]$Protocols,
        [string]$DestinationPorts,
        [int]$Priority,
        [string]$Action,
        [string]$RuleCollectionName
    )
    $command = @"
az network firewall policy rule-collection group collection rule add `
    --policy-name $firewallPolicyName `
    --resource-group $resourceGroupName `
    --rule-collection-group-name $RuleCollectionName `
    --rule-collection-name $RuleCollectionName `
    --name $Name `
    --rule-type NetworkRule `
    --rule-name $Name `
    --priority $Priority `
    --action $Action `
    --rule-protocols $Protocols `
    --source-addresses $SourceAddresses `
    --destination-addresses $DestinationAddresses `
    --destination-ports $DestinationPorts
"@
    try {
        Invoke-Expression $command
        Write-Host "Network rule '$Name' created successfully."
    } catch {
        Write-Host "Failed to create network rule '$Name'. Error: $_"
    }
}

# Function to create application rule
function Create-ApplicationRule {
    param (
        [string]$Name,
        [string]$SourceAddresses,
        [string]$TargetFqdns,
        [string]$Protocols,
        [int]$Priority,
        [string]$Action,
        [string]$RuleCollectionName
    )
    $command = @"
az network firewall policy rule-collection group collection rule add `
    --policy-name $firewallPolicyName `
    --resource-group $resourceGroupName `
    --rule-collection-group-name $RuleCollectionName `
    --rule-collection-name $RuleCollectionName `
    --name $Name `
    --rule-type ApplicationRule `
    --rule-name $Name `
    --priority $Priority `
    --action $Action `
    --rule-protocols $Protocols `
    --source-addresses $SourceAddresses `
    --target-fqdns $TargetFqdns
"@
    try {
        Invoke-Expression $command
        Write-Host "Application rule '$Name' created successfully."
    } catch {
        Write-Host "Failed to create application rule '$Name'. Error: $_"
    }
}

$priority = 100
$action = "Allow"

# Loop through each rule in the JSON file
foreach ($rule in $jsonContent) {
    $name = "Rule$($rule.id)"
    $ruleCollectionName = "Collection$($rule.id)"
    $sourceAddresses = "*"
    
    if ($rule.ips) {
        $destinationAddresses = $rule.ips -join ","
    } else {
        $destinationAddresses = "*"
    }

    $protocols = @()

    if ($rule.tcpPorts) {
        $protocols += "Tcp"
    }
    if ($rule.udpPorts) {
        $protocols += "Udp"
    }

    $protocols = $protocols -join ","
    $destinationPorts = "$($rule.tcpPorts),$($rule.udpPorts)" -replace " ", ""

    if ($rule.urls) {
        $targetFqdns = $rule.urls -join ","
        Create-ApplicationRule -Name $name -SourceAddresses $sourceAddresses -TargetFqdns $targetFqdns -Protocols $protocols -Priority $priority -Action $action -RuleCollectionName $ruleCollectionName
    } else {
        Create-NetworkRule -Name $name -SourceAddresses $sourceAddresses -DestinationAddresses $destinationAddresses -Protocols $protocols -DestinationPorts $destinationPorts -Priority $priority -Action $action -RuleCollectionName $ruleCollectionName
    }

    $priority += 100
}

Write-Host "Firewall rules have been created."
```

### Explanation:
1. **Import JSON File**: Read and convert the JSON file into a PowerShell object.
2. **Define Variables**: Set the resource group and firewall policy names.
3. **Functions**: Define `Create-NetworkRule` and `Create-ApplicationRule` functions to add rules using Azure CLI.
4. **Command Construction**: Construct the `az` command as a multi-line string using `@` and `"` for readability.
5. **Error Handling**: Use `try` and `catch` blocks to handle errors and provide feedback.
6. **Loop Through Rules**: Loop through each rule in the JSON file and call the appropriate function based on whether URLs are present (for application rules) or not (for network rules).

### Steps to Run the Script:
1. **Update Variables**: Replace `path_to_your_json_file.json`, `YourResourceGroupName`, and `YourFirewallPolicyName` with your actual values.
2. **Save Script**: Save the script as `CreateAzureFirewallRules.ps1`.
3. **Execute Script**:
   - Open PowerShell.
   - Navigate to the script's directory.
   - Run the script:
     ```powershell
     .\CreateAzureFirewallRules.ps1
     ```

This script should correctly create the necessary firewall rules in your Azure Firewall policy based on the provided JSON file. If you encounter any specific errors, please provide the error messages for further troubleshooting.
