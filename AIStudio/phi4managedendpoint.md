# Microsoft PHI 4 Managed Endpoint in Azure AI Foundry

## Introduction

- How to deploy Microsoft PHI 4 model on Azure AI Studio using Management Endpoint
- Need GPU compute
- SKU - Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk) or even 24 or 48 cores
- Using Managed Endpoint with 1 instance

## Prerequisites

- Azure subscription
- Azure AI Studio
- GPU Compute
- SKU - Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk)

## Steps

- First log into Azure AI Studio
- Need a project, i created one in Sweden Central
- Go to Model Catalog
- Select PHI 4 model
- Click on Deploy

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-1.jpg 'RagChat')

- Accept the terms and conditions

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-2.jpg 'RagChat')

- Give the name
- Select the compute
- Give endpoint a name and deployment a name
- Select to enable inference metric to track usage and performance of API
- Click on Deploy

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-4.jpg 'RagChat')

- Wait for deployment to complete, might take 5 to 10 minutes

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-5.jpg 'RagChat')

- Click Test

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-6.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-7.jpg 'RagChat')

- Click Monitoring

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-8.jpg 'RagChat')

- Now Delete the Endpoint

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIStudio/images/phi-4-10.jpg 'RagChat')