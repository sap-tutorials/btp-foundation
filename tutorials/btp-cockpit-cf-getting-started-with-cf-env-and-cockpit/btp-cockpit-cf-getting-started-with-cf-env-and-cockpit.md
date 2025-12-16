---
title: Getting Started with Cloud Foundry Environment and SAP BTP Cockpit
description: Learn the key concepts for working with SAP BTP Cloud Foundry environment and the SAP BTP cockpit.
parser: v2
auto_validation: true
time: 5
tags: [tutorial>beginner, software-product>sap-business-technology-platform, software-product-function>sap-btp-cockpit, software-product>sap-btp--cloud-foundry-environment]
primary_tag: software-product-function>sap-btp-cockpit
author_name: Dragomir Anachkov
author_profile: https://github.com/danachkov
---


## You will learn

- What you need to do before you start this learning journey
- What SAP BTP cockpit is
- What Cloud Foundry environment is
- About the connection between the Cloud Foundry environment and the SAP BTP cockpit
- How to try out the tutorials

## Prerequisites

**Note**: This tutorial is part of a learning journey. <!-- See [](Add link to the group). -->
- You have a global account and you've **created a subaccount** in it. If you're not sure how to set up your subaccount, go to [Create a Subaccount | SAP Help Portal](https://help.sap.com/docs/btp/sap-business-technology-platform/create-subaccount).
- In the subaccount, you've **enabled Cloud Foundry** and **created an organization (org)**. For more information, see [Create Orgs | SAP Help Portal](https://help.sap.com/docs/btp/sap-business-technology-platform/create-orgs).
- Each tutorial requires you to have **specific Cloud Foundry roles**. For more information, refer to the **Prerequisites** section of each tutorial.

### What is SAP BTP cockpit?

The **SAP BTP cockpit** is a web-based administration interface for managing SAP Business Technology Platform (SAP BTP) account and all the resources in it. It provides a central entry point where you can:

- Manage your global accounts, subaccounts, organizations (orgs), and spaces

- Manage user roles and permissions

- Deploy and monitor applications

- Create and bind services

### What is Cloud Foundry environment?

The **Cloud Foundry environment** allows you to deploy, run, and scale business applications. It supports multiple programming languages and related libraries. It also allows you to consume services such as databases and APIs by binding them to these applications.

The Cloud Foundry environment is available in the SAP BTP cockpit as a service (Cloud Foundry runtime service) which is based on the open-source platform managed by the Cloud Foundry Foundation.

The Cloud Foundry environment is suitable for companies using a diverse array of technologies allowing them to focus on the development of their applications rather than delving into the infrastructure side of the process.

### Understand the connection between the Cloud Foundry environment and SAP BTP cockpit

The SAP BTP cockpit is the place where you can organize, monitor, and configure resources, while Cloud Foundry acts as one of the runtime environments available within SAP BTP for deploying and running applications.

The Cloud Foundry environment in the SAP BTP cockpit is structured as follows:

1. At the top level, you have the **global account**, where you oversee your entire SAP BTP landscape, including billing and entitlements. Entitlements are the service plans that you're entitled to use.

2. Within the global account, you create **subaccounts**. Subaccounts help you organize resources based on different criteria, such as projects or departments. Each subaccount can have its own settings and configurations.

3. Inside each subaccount where Cloud Foundry is enabled, there is a single **organization (org)**. Each org contains one or more spaces and helps you manage users and resources. Orgs provide a way to group spaces that share common purposes or functions.

4. **Spaces** are the final level in this structure. This is where you deploy applications and bind services to them. Spaces allow you to manage your deployed applications and their configurations such as memory and instances.

### Try out the tutorials with companion applications

We've prepared two sample applications developed in Java and Node.js - **java-demo-app** and **nodejs-demo-app**: [github.com/SAP-samples/btp-cf-uis](https://github.com/SAP-samples/btp-cf-uis). You can test and explore our tutorials with them.

However, if you have your own application for testing purposes, feel free to use it instead.