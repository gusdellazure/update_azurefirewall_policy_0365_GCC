To create a PowerShell script that adds Azure Firewall policy rules based on the provided JSON file, follow these steps:

get the firewall information from Microsoft 
https://endpoints.office.com/endpoints/worldwide?clientrequestid=b10c5ed1-bad1-445f-b386-b919946339a7

1. Parse the JSON file.
2. Loop through each rule and create the corresponding network and application rules in the Azure Firewall policy.

Here is a PowerShell script to achieve this:

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
    az network firewall policy rule-collection group collection rule add \
        --policy-name $firewallPolicyName \
        --resource-group $resourceGroupName \
        --rule-collection-group-name $RuleCollectionName \
        --rule-collection-name $RuleCollectionName \
        --name $Name \
        --rule-type NetworkRule \
        --rule-name $Name \
        --priority $Priority \
        --action $Action \
        --rule-protocols $Protocols \
        --source-addresses $SourceAddresses \
        --destination-addresses $DestinationAddresses \
        --destination-ports $DestinationPorts
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
    az network firewall policy rule-collection group collection rule add \
        --policy-name $firewallPolicyName \
        --resource-group $resourceGroupName \
        --rule-collection-group-name $RuleCollectionName \
        --rule-collection-name $RuleCollectionName \
        --name $Name \
        --rule-type ApplicationRule \
        --rule-name $Name \
        --priority $Priority \
        --action $Action \
        --rule-protocols $Protocols \
        --source-addresses $SourceAddresses \
        --target-fqdns $TargetFqdns
}

$priority = 100
$action = "Allow"

# Loop through each rule in the JSON file
foreach ($rule in $jsonContent) {
    $name = "Rule$($rule.id)"
    $ruleCollectionName = "Collection$($rule.id)"
    $sourceAddresses = "*"
    $destinationAddresses = $rule.ips -join ","
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
1. **Import JSON File**: The script reads and converts the JSON file into a PowerShell object.
2. **Define Variables**: Set the resource group name and firewall policy name.
3. **Functions**: Define functions to create network and application rules using Azure CLI.
4. **Loop Through Rules**: The script loops through each rule in the JSON file and creates the corresponding network or application rule based on the presence of URLs and IP addresses.
5. **Priority and Action**: Set the priority and action for each rule. Increment the priority for each subsequent rule to ensure unique priorities.

### Usage:
- Save the script as a `.ps1` file (e.g., `CreateAzureFirewallRules.ps1`).
- Update the variables `$jsonFilePath`, `$resourceGroupName`, and `$firewallPolicyName` with your actual values.
- Run the script in PowerShell: `.\CreateAzureFirewallRules.ps1`.

This script will create the necessary firewall rules in your Azure Firewall policy based on the provided JSON file.
