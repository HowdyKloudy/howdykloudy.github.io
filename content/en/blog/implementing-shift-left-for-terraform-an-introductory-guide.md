---
author: "Chendrayan Venkatesan"
title: "Implementing Shift Left for Terraform: An Introductory Guide"
date: 2024-08-05
description: "Shift Left for Terraform"
tags: ["Azure", "Terraform" , "Shift Left", "PowerShell"]
thumbnail: /2024/08/new_blog_img.jpg
---

# Introduction

In today's fast-paced development environment, ensuring the security and quality of your infrastructure as code (IaC) is crucial. When I mentioned implementing Shift Left for Terraform to a few colleagues, many responded with, "Say what?" The concept of shifting security and quality checks to the earlier stages of the development lifecycle is still new to some. So, let's dive into what Shift Left for Terraform means and explore a practical approach to implementing it.

## What is Shift Left?

Shift Left is a practice in software development that involves performing tests and checks earlier in the development process rather than waiting until later stages or post-deployment. By shifting left, teams can identify and resolve issues sooner, reducing the risk of costly and time-consuming fixes down the line. This approach enhances the overall quality and security of the codebase and infrastructure.

## Why Shift Left for Terraform?

Terraform is a popular IaC tool used to define and provision infrastructure consistently declaratively. As with any code, Terraform configurations can contain errors, security vulnerabilities, and misconfigurations. Shifting left with Terraform means integrating checks and validations into the development process, ensuring these issues are caught early.

## Real-World Scenarios

### Scenario 1: Early Detection of Misconfigurations with TFLint

TFLint is a linter for Terraform that checks for potential issues and enforces best practices. Integrating TFLint into your CI/CD pipeline allows you to catch misconfigurations and style violations before they make it to production. For example, TFLint can identify unused variables, deprecated syntax, and potential security risks in your Terraform code.

```PowerShell
$(tflint --format json --force | ConvertFrom-Json)
```

### Scenario 2: Validating Terraform Configurations with TFValidate

Terraform's built-in validation tool, terraform validate, ensures that your configuration files are syntactically valid and internally consistent. This step is crucial for catching errors that could prevent your infrastructure from being provisioned correctly. 

```PowerShell
$(terraform validate -json | ConvertFrom-Json)
```

### Scenario 3: Enhancing Security with TFSec

TFSec is a static analysis tool that checks for security issues in your Terraform code. It can be used generically and with custom policies to enforce your organization's security standards. By integrating TFSec, you can identify insecure configurations, such as open security groups or unencrypted storage, and address them before deployment.

```PowerShell
$(tfsec --format json | ConvertFrom-Json)
```

## Putting It All Together
Here's a basic script that combines TFLint, TFValidate, and TFSec to pull out issues identified at different stages:

```PowerShell
terraform init 
function Yellow ($Text) {
    "$([char]0x1b)[93m$Text$([char]0x1b)[0m"
}
function Red ($Text) {
    "$([char]0x1b)[91m$Text$([char]0x1b)[0m"
}

function Magenta ($Text) {
    "$([char]0x1b)[95m$Text$([char]0x1b)[0m"        
}

$results = [PSCustomObject]@{
    TFLint     = $(tflint --format json --force | ConvertFrom-Json)
    TFValidate = $(terraform validate -json | ConvertFrom-Json)
    TFSecurity = $(tfsec --format json | ConvertFrom-Json)
}

$Ti = [cultureinfo]::new('en-US', $false).TextInfo
    
$flatResults = @()

foreach ($item in $results.TFLint.issues) {
    $flatResults += [PSCustomObject]@{
        Tool    = 'TFLint'
        Type    = $(
            switch ($Ti.ToTitleCase($item.rule.severity)) {
                'Error' {
                    Red $Ti.ToTitleCase($item.rule.severity)
                }
                'Warning' {
                    Yellow $Ti.ToTitleCase($item.rule.severity)
                }

                'MEDIUM' {
                    Magenta $Ti.ToTitleCase($item.rule.severity)
                }
                'CRITICAL' {
                    Red $Ti.ToTitleCase($item.rule.severity)
                }
            }
        )
        File    = $item.range.filename
        Line    = $item.range.start.line
        Message = $($item.Message)
           
    }
}

foreach ($item in $results.TFLint.errors) {
    $flatResults += [PSCustomObject]@{
        Tool    = 'TFLint'
        Type    = 'Error'
        File    = $item.File
        Line    = $item.Line
        Message = $item.Message
           
    }
}

foreach ($item in $results.TFValidate.diagnostics) {
    $flatResults += [PSCustomObject]@{
        Tool    = 'TFValidate'
        # Type    = $($Ti.ToTitleCase($item.severity))
        Type    = $(
            switch ($Ti.ToTitleCase($item.severity)) {
                'Error' {
                    Red $Ti.ToTitleCase($item.severity)
                }
                'Warning' {
                    Yellow $Ti.ToTitleCase($item.severity)
                }

                'MEDIUM' {
                    Magenta $Ti.ToTitleCase($item.severity)
                }
                'CRITICAL' {
                    Red $Ti.ToTitleCase($item.severity)
                }
            }
        )
        File    = $item.range.filename
        Line    = $item.range.start.line
        Message = $item.summary
            
    }
}

foreach ($item in $results.TFSecurity.results) {
    $flatResults += [PSCustomObject]@{
        Tool    = 'TFSec'
        # Type    = $($Ti.ToTitleCase($item.severity))
        Type    = $(
            switch ($Ti.ToTitleCase($item.severity)) {
                'Error' {
                    Red $Ti.ToTitleCase($item.severity)
                }
                'Warning' {
                    Yellow $Ti.ToTitleCase($item.severity)
                }

                'MEDIUM' {
                    Magenta $Ti.ToTitleCase($item.severity) 
                }
                'CRITICAL' {
                    Red $Ti.ToTitleCase($item.severity)
                }
            }
        )
        File    = ($item.location.filename -split '\\')[-1]
        Line    = $item.location.start_line
        Message = $item.description
           
    }
}

$flatResults | Format-Table -AutoSize -Expand Both

```

## Output 

![img](/2024/08/image1.png)

## Benefits of Shifting Left for Terraform

- **Early Issue Detection:** Identify and resolve misconfigurations, syntax errors, and security vulnerabilities early in development.  
- **Increased Security:** Enforce best practices and custom policies to secure your infrastructure.  
- **Improved Code Quality:** Maintain a high standard of code quality by adhering to best practices and avoiding deprecated syntax.  
- **Cost and Time Savings:** Reduce the risk of costly fixes and rework by catching issues early.  


## Conclusion

Implementing Shift Left for Terraform is a proactive approach to managing your infrastructure as code. Integrating tools like TFLint, TFValidate, and TFSec into your development workflow ensures your Terraform configurations are robust, secure, and ready for deployment. Start shifting left today and experience the benefits of early issue detection and improved code quality in your Terraform projects.
