---
title: Managing User-Provided Environment Variables
description: Find out how to add and manage your own environment variables in the SAP BTP cockpit. You will customize your application settings as needed. 
parser: v2
auto_validation: true
time: 7
tags: [tutorial>beginner, software-product>sap-business-technology-platform, software-product-function>sap-btp-cockpit]
primary_tag: software-product-function>sap-btp-cockpit
author_name: Dragomir Anachkov
author_profile: https://github.com/danachkov
---


## You will learn

- What a user-provided environment variable is
- How to create a user-provided environment variable
- How to delete a user-provided environment variable

## Prerequisites

**Note**: This tutorial is part of a learning journey. <!-- See [](Add link to the group). -->
- Make sure **you've fulfilled all prerequisites** in [Getting Started with Cloud Foundry Environment and SAP BTP Cockpit](https://developers.sap.com/tutorials/btp-cockpit-cf-getting-started-with-cf-env-and-cockpit.html).
- You have either the **Space Developer** or **Space Supporter** role.

### What is a user-provided environment variable?

User-provided variables are your custom environment variables, created as key-value pairs. You define them manually to pass data such as:

- API tokens
- Feature flags
- Configuration values
- External URLs

**Caution:** Avoid using user-provided environment variables for security-sensitive information like credentials. They might unintentionally appear in cf CLI output and Cloud Controller logs.

#### Example

Our sample applications are implemented to expect the variable `DARK_MODE`. This variable allows you to customize the UI theme of an app. Here's a snippet from the source code of the sample Node.js app:

```
const DARK_MODE = process.env.DARK_MODE === 'true';
```

### Create a user-provided environment variable

Let's create an environment variable that switches the theme of the sample app from the default to a dark theme. This is how our sample app looks like by default:

<!-- border; size:540px --> ![Default Theme of the Sample Application](./sample-app-default-theme.png)

1. Go to the **Cloud Foundry > Spaces** in the left navigation menu.

    <!-- border; size:540px --> ![Go to Spaces](./go-to-spaces-enter-space-1.png)

2. Go to a space. This opens the **Applications** page.

3. Click the name of the application which you want to customize with your own variable.

    <!-- border; size:540px --> ![Go to Application Overview](./go-to-app-overview-2.png)

4. Go to **User-Provided Variables** in the left navigation menu.

    <!-- border; size:540px --> ![Go to User-Provided Variables](./go-to-user-provided-variables-1.png)

5. Choose **Create Variable**.

    <!-- border; size:540px --> ![Choose Create Variable](./choose-create-variable-1.png)

6. Provide a **Key** and a **Value**. These values are case-sensitive.

    <!-- border; size:540px --> ![Enter Key and Value](./enter-key-and-value-1.png)

7. Choose **Create**.

    The user-provided variable is now created.

    <!-- border; size:540px --> ![User-Provided Variable Created](./user-provided-variable-created.png)

    To apply the change, you need to restart your application.

8. Go to the **Overview** page of your application.

    <!-- border; size:540px --> ![Go to Application Overview](./user-provided-variable-created-1.png)

9. Choose **Restart**.

    <!-- border; size:540px --> ![Choose Restart](./click-app-url-1.png)

10. Click the URL of the application.

    As you can see, the interface of the app has switched to the dark theme.

    <!-- border; size:540px --> ![Dark Theme of the Sample Application](./sample-app-dark-theme.png)

### Delete a user-provided environment variable

When you no longer need the variable and the customization it brings, delete it:

1. On the **User-Provided Variables** page, choose the **Delete** button in the **Actions** column.

    <!-- border; size:540px --> ![Choose Delete](./choose-delete-1.png)

2. Choose **Delete** to confirm the action.

    <!-- border; size:540px --> ![Deleting a Variable](./choose-delete-3.png)

After you delete the variable, restart the application as described in steps 8-10 of **Create a user-provided environment variable** or restage it. Otherwise, the change won't take effect.