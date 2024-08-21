# Get AWS SSO Roles for .NET WinForms

## Overview

**Get AWS SSO Roles** is a .NET 8 WinForms application designed to streamline the process of managing AWS credentials, particularly for developers working on Terraform and AWS-based projects. The tool integrates with AWS Single Sign-On (SSO) to help you acquire and maintain AWS credentials with minimal hassle, ensuring your credentials are always up-to-date and ready for use.

This documentation provides comprehensive guidance for both end-users (developers) and contributors looking to extend or maintain the application.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Features](#features)
3. [User Guide](#user-guide)
   - [Installation](#installation)
   - [Configuration](#configuration)
   - [Using the Application](#using-the-application)
     - [Example Use Cases](#example-use-cases)
     - [Tips and Helpful Information](#tips-and-helpful-information)
4. [Developer Guide](#developer-guide)
   - [Project Structure](#project-structure)
   - [Code Overview](#code-overview)
   - [Building and Running](#building-and-running)
   - [Extending the Application](#extending-the-application)
5. [Dependencies](#dependencies)
6. [Troubleshooting](#troubleshooting)
7. [FAQ](#faq)
8. [Contributing](#contributing)
9. [Future Enhancements](#future-enhancements)

## Getting Started

### Prerequisites

Before using **Get AWS SSO Roles**, ensure your environment meets the following requirements:

- **Windows 10/11**
- **.NET 8 SDK**
- **Visual Studio 2022** (or later) for development (optional)
- **AWS CLI** (for command-line AWS management)
- **An AWS Account** with SSO enabled and access to the desired accounts and roles.

### Installation

1. Download the latest version of the application from the [releases page](#).
2. Extract the contents of the downloaded ZIP file to a desired location.
3. Run the `get-sso-roles-v2.exe` to launch the application.

### Dependencies

The following NuGet packages are required and included in the project:

- **AWSSDK.SSO**: AWS SDK for .NET (SSO)
- **AWSSDK.SSOOIDC**: AWS SDK for .NET (SSOOIDC)
- **Newtonsoft.Json**: JSON serialization/deserialization
- **System.Windows.Forms**: Windows Forms framework

### Features

- **AWS SSO Integration**: Seamlessly authenticate with AWS SSO and retrieve roles.
- **Credentials Management**: Automatically manage and update AWS credentials in the `.aws` folder.
- **Role Filtering**: Filter roles based on account and role names.
- **Token and Role Monitoring**: Automatically refresh tokens and monitor role credentials expiration.
- **User-Friendly Interface**: Simple and intuitive UI designed for developers.
- **Progress Monitoring**: Real-time updates on the process of retrieving roles and generating credentials.

## User Guide

### Installation

Follow the steps in the [Getting Started](#getting-started) section to install the application.

### Configuration

1. **Launch the Application**: Run `get-sso-roles-v2.exe`.
2. **Select Organization**: From the dropdown menu, choose the AWS organization you want to work with.
3. **Configure Filters**:
   - **Account Filter**: Specify a comma-separated list of account names to filter (e.g., `dev,staging,prod`). Use wildcards (`*`) cautiously as they can return a large number of results.
   - **Role Filter**: Specify a comma-separated list of role names to filter (e.g., `Admin,ReadOnly`). Similar caution should be applied when using wildcards.
   - **Default Role**: Set the role to use as the default in the AWS credentials file.
4. **Region Selection**: Select the AWS region from the dropdown. The default region is `us-east-1`.

### Using the Application

1. **Get Roles**:
   - Click the "Get Roles" button to authenticate with AWS SSO, retrieve available roles, and display them in the dropdown menu.
   - The process involves registering your device, authorizing it, and then retrieving the roles based on your filters.
   - *Tip*: Be mindful when using wildcards in your filters. For instance, using `*` in both account and role filters could lead to the retrieval of all available roles, which may take a significant amount of time.

2. **Select a Role**:
   - After roles are retrieved, select the desired role from the dropdown menu. This will generate the AWS credentials and save them to your `.aws/credentials` file.
   - *Tip*: You can set your most frequently used role as the default to streamline your workflow.

3. **Monitor Tokens**:
   - The application automatically monitors token expiration and prompts for renewal when necessary. You can also manually refresh the token if needed.
   - *Tip*: Enable auto-refresh in settings if you frequently work with long-running sessions.

4. **Clean Credentials**:
   - Use the "Clean Credentials" button to delete all AWS credentials and start fresh. This is useful if you need to reset the configuration or switch organizations.
   - *Tip*: Always back up your `.aws` folder before performing a clean if you have custom configurations or profiles.

### Example Use Cases

#### 1. Multi-Environment Development

A developer working across multiple AWS environments (dev, staging, prod) can use **Get AWS SSO Roles** to quickly switch between roles without manually editing the `.aws/credentials` file. By setting appropriate filters, they can limit the roles displayed to only those relevant to their current task.

**Steps**:
1. Set `dev,staging,prod` in the Account Filter.
2. Use `Admin` in the Role Filter.
3. Select `dev_Admin` from the dropdown and set it as default.

#### 2. Long-Running Sessions

For long-running Terraform or AWS CLI sessions, the application monitors and automatically refreshes tokens, ensuring that your credentials remain valid throughout the session without manual intervention.

**Steps**:
1. Enable auto-refresh in the settings.
2. Configure a notification to alert when the token is nearing expiration.

#### 3. Rapid Onboarding

New team members can quickly get set up by selecting their organization and role, with the tool handling all the configuration and credential generation behind the scenes.

**Steps**:
1. Provide new team members with the `.exe` and guide them to select the correct organization.
2. They can then select the necessary roles, set a default, and start working immediately.

### Tips and Helpful Information

- **Beware of Wildcards**: Using `*` in both account and role filters can lead to retrieving all possible roles, which may take a significant amount of time and cause performance issues.
- **Role Nicknames**: Role nicknames are automatically generated as `AccountName_RoleName`. This helps in identifying roles quickly.
- **Auto-Refresh**: Enable auto-refresh to prevent interruptions in long-running sessions, especially when working with Terraform or other AWS tools.
- **Logging**: All actions are logged in `app.log` in the application directory. Use this log to troubleshoot any issues.
- **Backup Configuration**: Before cleaning the `.aws` folder, back up your credentials and config files to avoid losing custom settings.

## Developer Guide

### Project Structure

```plaintext
├── Forms
│   └── frmMain.cs        # Main form containing the application's logic and UI elements.
├── Models
│   ├── AuthorizedClientInfo.cs  # Data model for storing authorization information.
│   ├── RegisteredClientInfo.cs  # Data model for storing registered client information.
│   └── RoleInfo.cs              # Data model for storing AWS role information.
├── Program.cs            # Application entry point.
└── AWSHelpers.cs         # Helper methods for AWS SSO and credentials management.
```

### Code Overview

The application is centered around the `frmMain` form, which houses the core logic and user interface. Key features include:

- **SSO Authentication**: Handled via `AmazonSSOClient` and `AmazonSSOOIDCClient`.
- **Credential Management**: Credentials are managed using standard AWS SDK methods and are stored in the `.aws/credentials` file.
- **Monitoring**: Both token and role credentials are monitored for expiration, with automatic renewal when needed.

### Building and Running

1. Clone the repository:  
   ```bash
   git clone https://github.com/your-repo/get-aws-sso-roles.git
   ```
2. Open the project in Visual Studio.
3. Restore NuGet packages.
4. Build the solution (`Ctrl + Shift + B`).
5. Run the application (`F5`).

### Extending the Application

To extend the functionality, consider the following:

- **Additional Filters**: Implement more advanced filtering options for roles, such as regex-based filtering.
- **Custom AWS Profiles**: Allow users to define custom AWS profiles beyond the default configurations.
- **Logging Enhancements**: Integrate a more robust logging framework like `Serilog` for better traceability.

## Dependencies

This application relies on the following dependencies:

- **AWSSDK.SSO**: AWS SDK for .NET (SSO)
- **AWSSDK.SSOOIDC**: AWS SDK for .NET (SSOOIDC)
- **Newtonsoft.Json**: JSON serialization/deserialization
- **System.Windows.Forms**: Windows Forms framework

These dependencies are

 managed via NuGet and should be restored automatically when the solution is built.

## Troubleshooting

### Common Issues

- **Unable to Retrieve Roles**: Ensure you are correctly authenticated via SSO and that your account has the necessary permissions.
- **Token Expiration Warnings**: The application will notify you when a token is about to expire. Use the refresh button or allow the application to auto-refresh.
- **Missing Dependencies**: If you encounter missing dependency errors, restore NuGet packages via Visual Studio.

### Logs

All actions and errors are logged to the `app.log` file in the application directory. You can review this log for troubleshooting and issue diagnosis.

## FAQ

**Q1:** What is AWS SSO?  
**A1:** AWS SSO provides a centralized way to manage access to multiple AWS accounts and applications.

**Q2:** Where are the credentials stored?  
**A2:** Credentials are stored in the `.aws/credentials` file within your user profile directory.

**Q3:** Can I use this tool for multiple AWS organizations?  
**A3:** Yes, simply select the appropriate organization from the dropdown.

## Contributing

Contributions are welcome! Please fork the repository and submit a pull request with your changes.

## Future Enhancements

### Possible Features

- **Profile Import/Export**: Enable users to import and export AWS profiles for easy sharing and backup.
- **Multi-Factor Authentication (MFA) Support**: Add support for MFA tokens during the SSO login process.
- **Custom Profile Paths**: Allow users to specify custom paths for the `.aws/credentials` file.
- **Session Management**: Provide more granular control over AWS session tokens, including custom expiration settings.
- **Parallel Role Retrieval**: Implement parallel processing for role retrieval to improve performance, especially for users with many accounts.
- **Enhanced UI/UX**: Revamp the user interface to be more modern and user-friendly, possibly introducing a dark mode.

### Bug Fixes

- **Role Retrieval Performance**: Improve the performance of role retrieval, especially for accounts with a large number of roles.
- **Token Expiration Timing**: Fine-tune the timing of token expiration warnings to ensure they are not triggered prematurely.
- **UI/UX Improvements**: Enhance the user interface for better usability, including better handling of edge cases like network interruptions during SSO login.
- **Better Error Handling**: Implement more comprehensive error handling to gracefully manage exceptions and provide users with actionable feedback.
