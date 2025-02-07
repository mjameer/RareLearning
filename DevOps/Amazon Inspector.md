# Application Security Remediation Process

## Overview

This document outlines the process for identifying and fixing security vulnerabilities in container images using **AWS Amazon Inspector**. The initial focus is on addressing **Critical** and **High** vulnerabilities to ensure application security and compliance.

## Prerequisites

Before starting, ensure you have the following:
- AWS account with necessary permissions to access Amazon Inspector
- IAM permissions to view container repository findings
- Access to the relevant AWS container repository (Amazon Elastic Container Registry - ECR)

## Step-by-Step Process

### 1. Access Amazon Inspector

- In the **AWS Console**, search for **Amazon Inspector** in the search bar.
- Click on **Amazon Inspector** from the search results.

### 2. View Findings

- In the **Amazon Inspector** dashboard, navigate to **Findings**.
- Click on **By container repository**.
- Use the **Filter by repository name** option:
  - Select **Starts with**
  - Enter your repository name
- Click on the **Image Tag** associated with your container image.

### 3. Identify Vulnerabilities

- The page will display vulnerabilities categorized as:
  - **Critical**
  - **High**
  - **Medium**
  - **Low**
- During the **first wave**, focus on fixing **Critical** and **High** vulnerabilities.

### 4. Fix Vulnerabilities

- Analyze the reported vulnerabilities for your container image.
- Update dependencies and packages to patched versions.
- Rebuild the container image using the updated dependencies.
- Push the updated image to the repository.
- Redeploy the updated container to your environment.

### 5. Verify Fixes

- Redeploy your app to **Dev** or **UAT**, this process should re-run **Amazon Inspector** scans on the updated container image.
- Ensure that **Critical** and **High** vulnerabilities are resolved.

## Notes

- Medium and Low vulnerabilities can be addressed in subsequent remediation waves.
- Ensure application functionality is tested post-fix before deployment.
- Maintain compliance by continuously monitoring vulnerabilities.
