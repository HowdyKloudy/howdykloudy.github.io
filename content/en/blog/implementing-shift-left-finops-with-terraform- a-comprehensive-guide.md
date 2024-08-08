---
author: "Chendrayan Venkatesan"
title: "Implementing Shift Left FinOps with Terraform: A Comprehensive Guide"
date: 2024-08-07
description: "Shift Left FinOps for Terraform"
tags: ["Azure", "Terraform" , "Shift Left", "PowerShell", "FinOps"]
thumbnail: /2024/08/blog-cover-finops.jpg
---

# Introduction

As organizations strive to enhance their cloud infrastructure management, Shift Left and FinOps concepts have gained significant traction. Shift Left emphasizes integrating various aspects of the software development lifecycle earlier, identifying and resolving issues sooner. FinOps, on the other hand, focuses on the financial management of cloud services, ensuring cost efficiency and transparency.

Combining these two practices, Shift Left FinOps aims to incorporate financial considerations into the development and deployment stages, allowing teams to predict and optimize costs proactively. This blog post will explore how you can achieve Shift Left FinOps using the Infracost tool and PowerShell.

> **Disclaimer:** This is just the beginning - stay tuned for more scripts and insights around the shift left for Terraform.  

## Addressing Questions

> How about Shift Left FinOps?  

***Answer: This blog post will cover Shift Left FinOps using the Infracost tool with PowerShell. Infracost provides cost estimates for cloud resources defined in your Terraform projects, allowing you to make informed decisions about your infrastructure's financial impact before deployment.***

> Why the Type has Error, Medium, and Critical – It's confusing.  

***Answer: Yeah, it is! Let me fix it in the next blog post and deliver the best result.***

> Our organization needs custom validation, not generic.  

***Answer: We can achieve this using TFSec. I have added it to my backlog and will cover it in a future post.*** :wink:

> Is there any way to get to know the misconfigurations/vulnerabilities in real time rather than by executing a script?  

***Answer: Well, it's a big ASK. Let me work towards it! It's my aim for this quarter.*** :star_struck: :star_struck: :star_struck:

## Implementing Shift Left FinOps with Infracost and PowerShell

> Step 1: Install Infracost

```PowerShell
choco install infracost
```

> Step 2: Check the version

```PowerShell
infracost --version
```

> Step 3: Login

```PowerShell
infracost auth login # The credentials (api key) gets stored in the location C:\Users\<USERNAME>\.config\infracost
```

> Step 4: Develop with PowerShell 

```PowerShell
Push-Location -Path '.\src\environments\dev'

function Write-ColorText {
    param(
        [Parameter(Mandatory)]
        [ValidateSet('Red', 'Green', 'Blue', 'Yellow', 'Magenta', 'Cyan')]
        [string]
        $Color,

        [Parameter(Mandatory)]
        [string]
        $Text
    )

    switch ($Color) {
        'Red' {
            "$([char]0x1b)[91m$Text$([char]0x1b)[0m"
        }
        'Green' {
            "$([char]0x1b)[92m$Text$([char]0x1b)[0m" 
        }
        'Blue' {
            "$([char]0x1b)[94m$Text$([char]0x1b)[0m" 
        }
        'Yellow' {
            "$([char]0x1b)[93m$Text$([char]0x1b)[0m"
        }
        'Magenta' {
            "$([char]0x1b)[95m$Text$([char]0x1b)[0m" 
        }
        'Cyan' {
            "$([char]0x1b)[96m$Text$([char]0x1b)[0m" 
        }
    }
}

$cost = infracost breakdown --path='.' --format json --log-level error | ConvertFrom-Json 

[PSCustomObject]@{
    Currency              = $cost.currency
    TotalHourlyCost       = $(
        if ($($cost.totalHourlyCost) -le 1) {
            Write-ColorText -Color Green -Text $cost.totalHourlyCost 
        }
        else {
            Write-ColorText -Color Cyan -Text  $cost.totalHourlyCost
        }
    )
    TotalMonthlyCost      = $(
        if ($($cost.totalMonthlyCost) -le 500) {
            Write-ColorText -Color Green -Text $cost.totalMonthlyCost 
        }
        else { 
            Write-ColorText -Color Red -Text $cost.totalMonthlyCost 
        }
    )
    TotalMonthlyUsageCost = $cost.totalMonthlyUsageCost
    TimeGenerated         = $cost.timeGenerated
} | Format-Table -AutoSize -Expand Both
Pop-Location
```

## Output 

![img](/2024/08/InfraCost.png)

## Reference

- [Get Starterd With InfaCost](https://www.infracost.io/docs/)  
- [InfraCost](https://www.infracost.io/)  
- [InfraCost CLI Commands](https://www.infracost.io/docs/features/cli_commands/)  

## Conclusion

Infracost CLI is an essential tool for teams looking to optimize their cloud spending. Integrating cost estimation into the development workflow empowers teams to make informed decisions, ensuring cost-efficiency and transparency. Whether you are a developer, a DevOps engineer, or part of the finance team, understanding and utilizing Infracost can significantly enhance your cloud cost management strategy.