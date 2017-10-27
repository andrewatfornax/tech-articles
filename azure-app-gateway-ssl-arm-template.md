# Deploying an Azure Application Gateway with an existing SSL Certificate from an ARM Template

The [Azure Application Gateway FAQ](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-faq) states that Application Gateways do not integrate natively with Key Vaults. What this appears to mean is that you cannot use [Key Vault Certificates](https://docs.microsoft.com/en-us/rest/api/keyvault/about-keys--secrets-and-certificates#key-vault-certificates) with an Application Gateway, to allow for SSL termination.

While documentation exists for how to upload an existing SSL Certificate to an Application Gateway that has already been created, using either PowerShell or the Azure CLI tools, the documentation that exists on how to create an Application Gateway that performs SSL termination via an ARM Template is not at all clear on how this might be done in a way that allows the SSL certificate information to managed securely.

Here's a solution to that problem that appears to work:

## Convert PEM certificate to PFX

If the existing SSL certificate is in PEM format, it will need to be converted to PFX format first, as Application Gateways require PFX formatted certificates.

```
openssl pkcs12 -export -out certificate.pfx -inkey privateKey.key -in certificate.crt
```

Note that you *must* create the PFX certificate with a password. The Application Gateway definition for ARM Templates appears to *require* that a password for the certificate be supplied when deploying.

## Upload PFX certificate to Key Vault

Rather than uploading the PFX certificate to the Key Vault as a [Certificate](https://docs.microsoft.com/en-us/rest/api/keyvault/about-keys--secrets-and-certificates#key-vault-certificates), instead, the certificate data needs to be uploaded as a [Secret](https://docs.microsoft.com/en-us/rest/api/keyvault/about-keys--secrets-and-certificates#key-vault-secrets). This way, the ARM Template can access the certificate data during deployment of the Application Gateway.

The certificate also needs to be converted into Base64 encoded format, but this can be performed as part of the upload process, using the Azure CLI tool:

```
az keyvault secret set --vault-name KEY_VAULT_NAME --encoding base64 --description text/plain --name CERT_SECRET_NAME --file certificate.pfx
```

You should also store the PFX certificate password created as a Secret in the Key Vault as well.

## Reference certificate secret

The PFX certificate data, in Base64 encoded format, and the certificate password, can now both be passed to the ARM Template as a reference (via a parameters file).

### ARM Template

The template assumes that an Azure VNet and Subnet are already in place. Note that you can only deploy an Application Gateway into a Subnet that contains no other resources (apart from other Application Gateways).

*app-gateway.json*
```
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "sslCertificateData": {
            "type": "string",
            "metadata": {
                "description": "The base-64 encoded SSL certificate PFX data. Must be supplied via a parameters file references to a Key Vault / Secret Name."
            }
        },
        "sslCertificatePassword": {
            "type": "securestring",
            "metadata": {
                "description": "The SSL certificate password. Must be supplied via a parameters file references to a Key Vault / Secret Name."
            }
        },
        "vNetId": {
            "type": "string",
            "metadata": {
                "description": "The ID of the VNet."
            }
        },
        "subnetName": {
            "type": "string",
            "metadata": {
                "description": "The name of the DMZ Subnet."
            }
        }

    },
    "variables": {
        "networkApiVersion": "2017-04-01",

        "subnetId": "[concat(parameters('vNetId'), '/subnets/', parameters('subnetName'))]",

        "appGatewayPublicIpAddressId": "[resourceId('Microsoft.Network/publicIPAddresses', 'appGatewayPublicIpAddress')]",

        "appGwId": "[resourceId('Microsoft.Network/applicationGateways', 'appGateway')]",

        "appGwSize": "Standard_Small",
        "appGwTier": "Standard",
        "appGwCapacity": 5,
        "appGwFePort": 443,
        "appGwFeProtocol": "Https",
        "appGwBePort": 80,
        "appGwBEProtocol": "Http"
    },
    "resources": [
        {
            "type": "Microsoft.Network/publicIPAddresses",
            "name": "appGatewayPublicIpAddress",
            "location": "[resourceGroup().location]",
            "apiVersion": "[variables('networkApiVersion')]",
            "comments": "This creates a single, dynamically allocated public IP address for use by the Application Gateway.",
            "properties": {
                "publicIPAllocationMethod": "Dynamic"
            }
        },
        {
            "type": "Microsoft.Network/applicationGateways",
            "name": "appGateway",
            "location": "[resourceGroup().location]",
            "apiVersion": "[variables('networkApiVersion')]",
            "comments": "This creates the Application Gateway.",
            "dependsOn": [
                "[concat('Microsoft.Network/publicIPAddresses/', 'appGatewayPublicIpAddress')]"
            ],
            "properties": {
                "sku": {
                    "name": "[variables('appGwSize')]",
                    "tier": "[variables('appGwTier')]",
                    "capacity": "[variables('appGwCapacity')]"
                },
                "gatewayIPConfigurations": [
                    {
                        "name": "gatewayIpCofig",
                        "properties": {
                            "subnet": {
                                "id": "[variables('subnetId')]"
                            }
                        }
                    }
                ],
                "frontendIPConfigurations": [
                    {
                        "name": "frontendIpConfig",
                        "properties": {
                            "PublicIPAddress": {
                                "id": "[variables('appGatewayPublicIpAddressId')]"
                            }
                        }
                    }
                ],
                "frontendPorts": [
                    {
                        "name": "frontendPort",
                        "properties": {
                            "Port": "[variables('appGwFePort')]"
                        }
                    }
                ],
                "sslCertificates": [
                    {
                        "name": "appGwSslCertificate",
                        "properties": {
                            "data": "[parameters('sslCertificateData')]",
                            "password": "[parameters('sslCertificatePassword')]"
                        }
                    }
                ],
                "backendAddressPools": [
                    {
                        "name": "BackendAddressPool"
                    }
                ],
                "backendHttpSettingsCollection": [
                    {
                        "name": "HttpSettings",
                        "properties": {
                            "Port": "[variables('appGwBePort')]",
                            "Protocol": "[variables('appGwBeProtocol')]"
                        }
                    }
                ],
                "httpListeners": [
                    {
                        "name": "HttpListener",
                        "properties": {
                            "FrontendIPConfiguration": {
                                "Id": "[concat(variables('appGwId'), '/frontendIPConfigurations/frontendIpConfig')]"
                            },
                            "FrontendPort": {
                                "Id": "[concat(variables('appGwId'), '/frontendPorts/frontendPort')]"
                            },
                            "Protocol": "[variables('appGwFeProtocol')]",
                            "SslCertificate": {
                                "id": "[concat(variables('appGwId'), '/sslCertificates/appGwSslCertificate')]"
                            }
                        }
                    }
                ],
                "requestRoutingRules": [
                    {
                        "Name": "RoutingRule",
                        "properties": {
                            "RuleType": "Basic",
                            "httpListener": {
                                "id": "[concat(variables('appGwId'), '/httpListeners/HttpListener')]"
                            },
                            "backendAddressPool": {
                                "id": "[concat(variables('appGwId'), '/backendAddressPools/BackendAddressPool')]"
                            },
                            "backendHttpSettings": {
                                "id": "[concat(variables('appGwId'), '/backendHttpSettingsCollection/HttpSettings')]"
                            }
                        }
                    }
                ]
            }
        }
    ]
}
```

### ARM Template Parameter File

*app-gateway-parameters.json*
```
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "sslCertificateData": {
            "reference": {
                "keyVault": {
                    "id": "/subscriptions/SUBSCRIPTION_ID/resourcegroups/RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/KEY_VAULT_NAME"
                },
                "secretName": "CERT_SECRET_NAME"
            }
        },
        "sslCertificatePassword": {
            "reference": {
                "keyVault": {
                    "id": "/subscriptions/SUBSCRIPTION_ID/resourcegroups/RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/KEY_VAULT_NAME"
                },
                "secretName": "CERT_PASSWORD_SECRET_NAME"
            }
        },
        "vNetId": {
            "value": "/subscriptions/SUBSCRIPTION_ID/resourceGroups/RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/VNET_NAME"
        },
        "subnetName": {
            "value": "SUBNET_NAME"
        }
    }
}
```

### Deploy Command

```
az group deployment create --resource-group RESOURCE_GROUP --template-file app-gateway.json --parameters @app-gateway-parameters.json
```