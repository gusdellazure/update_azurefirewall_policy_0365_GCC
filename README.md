# update_azurefirewall_policy_0365_GCC
A simple way to update your firewall policy for GCC 
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
