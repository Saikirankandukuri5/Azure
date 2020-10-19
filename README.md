{
    "name": "pipeline1",
    "properties": {
        "activities": [
            {
                "name": "Call dataset refresh",
                "type": "WebActivity",
                "dependsOn": [
                    {
                        "activity": "Get AAD Token",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    }
                ],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": false,
                    "secureInput": true
                },
                "userProperties": [],
                "typeProperties": {
                    "url": {
                        "value": "@concat('https://api.powerbi.com/v1.0/myorg/groups/',pipeline().parameters.workspaceGuid,'/datasets/',pipeline().parameters.datasetGuid,'/refreshes')",
                        "type": "Expression"
                    },
                    "method": "POST",
                    "headers": {
                        "Authorization": {
                            "value": "@concat(string(activity('Get AAD Token').output.token_type),' ',string(activity('Get AAD Token').output.access_token))",
                            "type": "Expression"
                        }
                    },
                    "body": {
                        "notifyOption": "NoNotification"
                    }
                }
            },
            {
                "name": "Until dataset refresh completion",
                "type": "Until",
                "dependsOn": [
                    {
                        "activity": "Call dataset refresh",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    }
                ],
                "userProperties": [],
                "typeProperties": {
                    "expression": {
                        "value": "@not(equals(first(json(string(activity('Get dataset refresh status').output)).value).status,'Unknown'))",
                        "type": "Expression"
                    },
                    "activities": [
                        {
                            "name": "Wait 15 seconds",
                            "type": "Wait",
                            "dependsOn": [],
                            "userProperties": [],
                            "typeProperties": {
                                "waitTimeInSeconds": 15
                            }
                        },
                        {
                            "name": "Get dataset refresh status",
                            "type": "WebActivity",
                            "dependsOn": [
                                {
                                    "activity": "Wait 15 seconds",
                                    "dependencyConditions": [
                                        "Succeeded"
                                    ]
                                }
                            ],
                            "policy": {
                                "timeout": "7.00:00:00",
                                "retry": 0,
                                "retryIntervalInSeconds": 30,
                                "secureOutput": false,
                                "secureInput": true
                            },
                            "userProperties": [],
                            "typeProperties": {
                                "url": {
                                    "value": "@concat('https://api.powerbi.com/v1.0/myorg/groups/',pipeline().parameters.workspaceGuid,'/datasets/',pipeline().parameters.datasetGuid,'/refreshes?$top=1')",
                                    "type": "Expression"
                                },
                                "method": "GET",
                                "headers": {
                                    "Authorization": {
                                        "value": "@concat(string(activity('Get AAD Token').output.token_type),' ',string(activity('Get AAD Token').output.access_token))",
                                        "type": "Expression"
                                    },
                                    "Content-Type": "application/json"
                                }
                            }
                        }
                    ],
                    "timeout": "0.03:00:00"
                }
            },
            {
                "name": "If dataset refresh failed",
                "type": "IfCondition",
                "dependsOn": [
                    {
                        "activity": "Until dataset refresh completion",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    }
                ],
                "userProperties": [],
                "typeProperties": {
                    "expression": {
                        "value": "@equals(first(json(string(activity('Get dataset refresh status').output)).value).status,'Failed')",
                        "type": "Expression"
                    },
                    "ifTrueActivities": [
                        {
                            "name": "SaveErrorMessages",
                            "description": "In case of an error in processing of the dataset, the actual messages are saved in the pipeline variable \"ProcessingErrors\".",
                            "type": "SetVariable",
                            "dependsOn": [],
                            "userProperties": [],
                            "typeProperties": {
                                "variableName": "ProcessingErrors",
                                "value": {
                                    "value": "@string(json(first(json(string(activity('Get dataset refresh status').output)).value).serviceExceptionJson))",
                                    "type": "Expression"
                                }
                            }
                        }
                    ]
                }
            },
            {
                "name": "Get AAD Token",
                "type": "WebActivity",
                "dependsOn": [
                    {
                        "activity": "Get Secret from AKV",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    },
                    {
                        "activity": "Get ClientId from AKV",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    },
                    {
                        "activity": "Get TenantId from AKV",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    }
                ],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": true,
                    "secureInput": true
                },
                "userProperties": [],
                "typeProperties": {
                    "url": {
                        "value": "@concat('https://login.microsoftonline.com/',activity('Get TenantId from AKV').output.value,'/oauth2/token')",
                        "type": "Expression"
                    },
                    "method": "POST",
                    "headers": {
                        "Content-Type": "application/x-www-form-urlencoded"
                    },
                    "body": {
                        "value": "@concat('grant_type=client_credentials&resource=https://analysis.windows.net/powerbi/api&client_id=',activity('Get ClientId from AKV').output.value,'&client_secret=',encodeUriComponent(activity('Get Secret from AKV').output.value))",
                        "type": "Expression"
                    }
                }
            },
            {
                "name": "Get TenantId from AKV",
                "type": "WebActivity",
                "dependsOn": [],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": true,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "url": {
                        "value": "@concat(pipeline().parameters.KeyVaultDNSName,'secrets/',pipeline().parameters.SecretName_TenantId,'/?api-version=7.0')",
                        "type": "Expression"
                    },
                    "method": "GET",
                    "body": {
                        "simple": "body"
                    },
                    "authentication": {
                        "type": "MSI",
                        "resource": "https://vault.azure.net"
                    }
                }
            },
            {
                "name": "Get ClientId from AKV",
                "type": "WebActivity",
                "dependsOn": [],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": true,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "url": {
                        "value": "@concat(pipeline().parameters.KeyVaultDNSName,'secrets/',pipeline().parameters.SecretName_SPClientId,'/?api-version=7.0')",
                        "type": "Expression"
                    },
                    "method": "GET",
                    "body": {
                        "simple": "body"
                    },
                    "authentication": {
                        "type": "MSI",
                        "resource": "https://vault.azure.net"
                    }
                }
            },
            {
                "name": "Get Secret from AKV",
                "type": "WebActivity",
                "dependsOn": [],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": true,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "url": {
                        "value": "@concat(pipeline().parameters.KeyVaultDNSName,'secrets/',pipeline().parameters.SecretName_SPSecret,'/?api-version=7.0')",
                        "type": "Expression"
                    },
                    "method": "GET",
                    "body": {
                        "simple": "body"
                    },
                    "authentication": {
                        "type": "MSI",
                        "resource": "https://vault.azure.net"
                    }
                }
            }
        ],
        "parameters": {
            "SecretName_TenantId": {
                "type": "String",
                "defaultValue": "TenantID"
            },
            "SecretName_SPClientId": {
                "type": "String",
                "defaultValue": "ClientID"
            },
            "SecretName_SPSecret": {
                "type": "String",
                "defaultValue": "Secret"
            },
            "KeyVaultDNSName": {
                "type": "string",
                "defaultValue": "https://keyvaultforadf-powerbi.vault.azure.net/"
            },
            "workspaceGuid": {
                "type": "string",
                "defaultValue": "987f86ce-3e59-4c05-94bb-01d857c37a64"
            },
            "datasetGuid": {
                "type": "string",
                "defaultValue": "74255cc2-738b-4997-b0ee-67bfb7ee5d3f"
            },
            "SecretVersion_TenantId": {
                "type": "string",
                "defaultValue": "c77cbe4ae4834d948b658c044d48b38c"
            },
            "SecretVersion_SPClientId": {
                "type": "string",
                "defaultValue": "411d3f137b714deb970ba50907894e12"
            },
            "SecretVersion_SPSecret": {
                "type": "string",
                "defaultValue": "95721d6d416a447f855ae8f901e7b40f"
            }
        },
        "variables": {
            "ProcessingErrors": {
                "type": "Boolean"
            }
        },
        "folder": {
            "name": "50-powerbi"
        },
        "annotations": [],
        "lastPublishTime": "2020-10-19T08:27:56Z"
    },
    "type": "Microsoft.DataFactory/factories/pipelines"
}
