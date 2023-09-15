---
author: "Chendrayan Venkatesan"
title: "GitLab Custom SAST Analyzer for PowerShell Project"
description: "demonstration of implementing a custom SAST analyzer for PowerShell projects..."
date: 2023-09-14T07:56:42+01:00
draft: false
tags: ["GitLab", "DevSecOps" , "SAST" , "CICD" , "PowerShell" , "PSScriptAnalyzer"]
thumbnail: "/2023/09/09-blog-cover.jpg"
---

# Introduction

In the ever-evolving landscape of infrastructure as code, software development and cybersecurity, ensuring the robust security of your codebase has become paramount. My journey into enhancing the security posture of my PowerShell projects led me to explore the world of GitLab Custom SAST Analyzers.

As a GitLab Security Specialist, I was well-versed in the importance of securing code repositories and the pivotal role that Static Application Security Testing (SAST) plays in this endeavour. GitLab's commitment to providing a secure DevOps platform is evident, yet I found a particular gap regarding PowerShell projects. There was no default SAST analyzer readily available for this scripting language.

Driven by the need to bridge this gap and bolster the security of my PowerShell projects, I embarked on a journey of innovation and customization. I had already cleared the GitLab Security Specialist certification, which empowered me with the knowledge and expertise to address this challenge effectively.

In this journey, I discovered the power of custom SAST analysis using PSScript Analyzer. Armed with this open-source static code analysis tool designed specifically for PowerShell, I crafted a tailored solution to scan my PowerShell projects for potential security vulnerabilities. PSScriptAnalyzer module offered the flexibility and extensibility needed to create a custom SAST analyzer aligned with my project's unique requirements.

In this exploration, I enhanced the security of my PowerShell code and realized the potential to contribute to the broader GitLab community by sharing my custom SAST analyzer. This journey represents the essence of DevSecOps - continuous improvement, collaboration, and empowerment. 

In the following sections, I will delve into the process of creating a GitLab Custom SAST Analyzer for PowerShell projects, highlighting the steps, challenges, and insights gained along the way. Together, we'll unlock the power of customized security analysis in the GitLab ecosystem, strengthening the defences of our code one script at a time.

# Video 

I respect your time and recorded the demonstration of implementing a custom SAST analyzer for the PowerShell project. 

{{<youtube Z-60XQxtoF0>}}

# What is SAST? 

**SAST** stands for `Static Application Security Testing.` It is a security testing methodology used in software development and cybersecurity to identify and mitigate security vulnerabilities in source code or compiled binaries before deploying the application. SAST is also known as "white-box testing" because it analyzes an application's code's internal structure and logic without executing the software. Highlights of `SAST` are as follows. 

- Early Detection of Vulnerabilities
- Source Code Analysis
- Automation
- Customizable Rules
- False Positives
- Integration with Development Tools
- Complementing Other Testing Methods
- Regulatory Compliance

# What is PSScriptAnalyzer? 

`PSScriptAnalyzer` is a valuable module in PowerShell scripting and development. It is a powerful static code analysis tool for PowerShell scripts and modules. PSScriptAnalyzer empowers developers and administrators to enhance their PowerShell code's quality, readability, and security.

I need to implement custom PowerShell rules for the organization's needs. How to do it? Refer this [link](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/create-custom-rule?view=ps-modules)  

# Custom SAST Analyzer report schema 

[sast-report-format](https://gitlab.com/gitlab-org/security-products/security-report-schemas/-/blob/master/dist/sast-report-format.json)

![JSON Schema](/2023/09/OP-1.png)

```JSON
{
    "version": "",
    "scan": {
        "start_time": "",
        "end_time": "",
        "status": "success",
        "type": "sast",
        "scanner": {
            "id": "",
            "name": "",
            "url": "",
            "version": "",
            "vendor": {
                "name": ""
            }
        },
        "analyzer": {
            "id": "",
            "name": "",
            "url": "",
            "vendor": {
                "name": ""
            },
            "version": ""
        }
    },
    "vulnerabilities": [
        {
            "id": "",
            "name": "",
            "severity": "",
            "cve": "",
            "location": {
                "file": "",
                "commit": ""
            },
            "start_line": ,
            "identifiers": [
                {
                    "type": "",
                    "name": "",
                    "value": ""
                }
            ],
            "scanner": {
                "id": "",
                "name": ""
            }
        }
    ]
}
```

# PowerShell Script to generate GL-SAST-REPORT.JSON

```PowerShell
Install-Module -Name PSScriptAnalyzer -Scope CurrentUser -Repository PSGallery -Force
$Collection = @()
$vulnerabilities = (Get-ChildItem -Path . -Exclude test.ps1 | Invoke-ScriptAnalyzer )
for ($i = 0; $i -lt $vulnerabilities.Count; $i++) {  
    $vulnerability = [PSCustomObject]@{
        id          = $(New-Guid).Guid
        name        = $($vulnerabilities[$i].RuleName)
        severity    = $(
            switch ($vulnerabilities[$i].Severity.ToString()) {
                'Error' {
                    'Critical'
                }
                'Warning' {
                    'Low'
                }
                'Information' {
                    'Info'
                }
                default {
                    'Unknown'
                }
            }
        )
        cve         = $([string]::Concat('id:', $((New-Guid).Guid), ':' , $(Get-Random -Maximum 10 -Minimum 1)))
        location    = [pscustomobject]@{
            file   = $([string]::Concat($($vulnerabilities[$i].ScriptName), '-' , $($vulnerabilities[$i].Line)))
            commit = '0000000'
        }
        start_line  = $($vulnerabilities[$i].Line)
        identifiers = @([pscustomobject]@{
                type  = $(New-Guid).Guid
                name  = $($vulnerabilities[$i].RuleName)
                value = $([string]::Concat('id:', $((New-Guid).Guid), ':' , $(Get-Random -Maximum 10 -Minimum 1)))
            })
        scanner     = [pscustomobject]@{
            id   = $(New-Guid).Guid
            name = "Gitleaks"
        }
    }
    $Collection += $vulnerability
}

$jsonObject = [PSCustomObject]@{
    version         = "15.0.4"
    scan            = [pscustomobject]@{
        start_time = $(Get-Date -Format 'yyyy-MM-dd%THH:mm:ss')
        end_time   = $(Get-Date -Format 'yyyy-MM-dd%THH:mm:ss')
        status     = "success"
        type       = "sast"
        scanner    = [pscustomobject]@{
            id      = $(New-Guid).Guid
            name    = "PowerShell Script Analyzer - $((New-Guid).Guid)"
            url     = "https://learn.microsoft.com/en-us/powershell/module/psscriptanalyzer/?view=ps-modules"
            version = "development"
            vendor  = [pscustomobject]@{
                name = "Microsoft"
            }
        }
        analyzer   = [PSCustomObject]@{
            "id"      = $(New-Guid).Guid
            "name"    = "PSScriptAnalyzer"
            "url"     = "https://gitlab.com/gitlab-org/security-products/analyzers/secrets"
            "vendor"  = [PSCustomObject]@{
                "name" = "Microsoft"
            }
            "version" = "1.21.0"
    
        }
    }
    vulnerabilities = $Collection
}
$jsonObject | ConvertTo-Json -Depth 9 | Out-File "gl-sast-report.json"
```

## Install PSScriptAnalyzer

> This command installs the `PSScriptAnalyzer` module from the PowerShell Gallery for the current user.

```PowerShell
Install-Module -Name PSScriptAnalyzer -Scope CurrentUser -Repository PSGallery -Force
```

## Initialize Variables

> This initializes an empty array called `$Collection` to store vulnerability information.

```PowerShell
$Collection = @()
```

## Run Script Analyzer

> This command runs `PSScriptAnalyzer` on all PowerShell script files in the current directory (excluding `test.ps1`) and stores the results in the `$vulnerabilities` variable.
Iterate Through Vulnerabilities

```PowerShell
$vulnerabilities = (Get-ChildItem -Path . -Exclude test.ps1 | Invoke-ScriptAnalyzer)
```


```PowerShell 
for ($i = 0; $i -lt $vulnerabilities.Count; $i++) {
```

This starts a loop to iterate through the detected vulnerabilities.

## Create Vulnerability Object

```PowerShell
$vulnerability = [PSCustomObject]@{ ... }
```

Inside the loop, a custom object is created to represent each vulnerability. The object includes fields such as ID, name, severity, location, identifiers, and scanner information. Severity is also mapped from 'Error', 'Warning', and 'Information' to 'Critical', 'Low', and 'Info', respectively.

## Add Vulnerability to Collection

```PowerShell
$Collection += $vulnerability
```

Each vulnerability object is added to the `$Collection` array.

## Create JSON Report

```PowerShell
$jsonObject = [PSCustomObject]@{ ... }
```

After processing all vulnerabilities, a JSON report object is created. It includes version information, scan details, and the list of vulnerabilities from the `$Collection` array.
Convert to JSON and Save

```PowerShell
$jsonObject | ConvertTo-Json -Depth 9 | Out-File "gl-sast-report.json"
```

The report object is converted to JSON with a depth of 9 (usually sufficient), and the JSON content is saved to a file named "gl-sast-report.json."
When you run this PowerShell script, it will analyze your PowerShell scripts using `PSScriptAnalyzer` module , convert the results into a GitLab-compatible SAST report in JSON format, and save it to the specified file. You can then use this report to integrate security analysis into your GitLab CI/CD pipelines.

# GitLab Pipeline

> .gitlab-ci.yml 

```YAML 
image: mcr.microsoft.com/powershell
 
stages:
  - "ps_sast"
 
ps_sast:
  stage: ps_sast
  script: 
    - pwsh -f ".\test.ps1"
  artifacts:
    reports:
      sast: gl-sast-report.json
    paths:
      - gl-sast-report.json
```

# Output 

![Output](/2023/09/OP-2.png)

# References

- [GitLab SAST](https://docs.gitlab.com/ee/user/application_security/sast/)
- [SAST Analyzers](https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html)
- [Custom Analyzers](https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html#custom-analyzers)
- [PSScriptAnalyzer](https://github.com/PowerShell/PSScriptAnalyzer)
- [Learn PSScriptAnalyzer](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview?view=ps-modules)
- [PSScriptAnalyzer Custom Rules](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/create-custom-rule?view=ps-modules)

> I wanted to take a moment to express my deepest gratitude to each one of you who has been a part of our blog's journey. Your support and readership have been invaluable, and I cannot thank you enough for being a part of our technical community.