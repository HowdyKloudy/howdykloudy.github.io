---
author: "Chendrayan Venkatesan"
title: "Using the GitLab GraphQL API in PowerShell"
description: "demonstration of using GraphQL API with PowerShell..."
date: 2023-10-14T11:20:02+01:00
draft: false
tags: ["GitLab", "DevSecOps" , "GraphQL" , "PowerShell"]
thumbnail: "/2023/10/blog-header.jpg"
---

# Introduction

GraphQL is a revolutionary query language and runtime for API interactions in web development. It doesn't just transform how we request and manage data—it reshapes the entire API landscape. In this blog post, we'll unveil the power of GitLab GraphQL API, exploring how it offers a comprehensive and user-friendly approach to data manipulation and API evolution.

At its core, GraphQL serves as a versatile tool for querying APIs and orchestrating data retrieval. It introduces a structured, coherent description of your API's data, allowing clients to wield precise control over the information they receive. But GraphQL's influence extends beyond efficient data queries; it empowers seamless API evolution and equips developers with a potent arsenal of tools.

I appreciate your time and recorded the demo 

{{<youtube zHynYT_cjk4>}}

## What is GitLab GraphQL API? 

The GitLab GraphQL API is an alternative to the traditional REST API, designed to provide developers with precise control over the data they request and manipulate. It is based on the GraphQL query language, enabling you to specify what information you need from GitLab, simplifying data retrieval and enhancing the efficiency of your interactions.

Sufficient theory can be tedious; let's commence with the exercise instead.

## Getting started

To access the GitLab GraphQL API, you will need to:

- Have a GitLab account. [Signup Link](https://gitlab.com/users/sign_up)
- Generate a personal access token with the read_api scope. [PAT](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)

Once you have a personal access token, you can authenticate to the API. There are a few different ways to do this:

- Bearer header in your requests.
- Set the GITLAB_TOKEN environment variable.
- Third-party tools such as GraphiQL or Postman.

To write basic queries, you will need to learn the GraphQL syntax. There are many tutorials and resources available online.

Here is an example of a basic GraphQL query to retrieve current user information 

```GraphQL
query currentUserBasicInformation {
  currentUser {
    id
    name
    twitter
    groupCount
    starredProjects {
      count
    }
    jobTitle
    organization
  }
}
```

The response is below

```JSON
{
  "data": {
    "currentUser": {
      "id": "gid://gitlab/User/12345678",
      "name": "Chendrayan Venkatesan",
      "twitter": "@ChendrayanV",
      "groupCount": 9,
      "starredProjects": {
        "count": 1
      },
      "jobTitle": "Technology Lead",
      "organization": "Freelancer"
    }
  }
}
```

### Get a list of all projects

```GraphQL
query getProjectInformation($fullPath: ID!) {
  project(fullPath: $fullPath) {
    id
    name
  }
}
```

- The keyword `query` in GraphQL is to define a query operation. A query operation is a request to fetch data from a GraphQL server.
- The string literal `getProjectInformation` is the friendly name of the operation. `$fullPath` is the query variable. 
- The word `project`` is the TYPE
- The id and name are the FIELDS. 

> Query Variables 

```GraphQL
{
  "fullPath": "group/fullPath"
}
```

As we all know, the preceding code retrieves the project information, but how is it executed? Through GraphiQL or PowerShell? Both are correct 

> GraphiQL

![GraphiQL](/2023/10/image-graphiql.png)


> PowerShell

```PowerShell
param (
    $orgnizationName = "gitlab.com"
    $fullPath = "group/project"
)

$query = [pscustomobject]@{
    operationName = "getProjectInformation"
    query         = $(Get-Content -path .\query.graphql -Raw)
    variables     = @{
        fullPath = $($fullPath)
    }
} | ConvertTo-Json -Compress 

$PrivateToken = 'SUPER-SECRET'
$Result = Invoke-RestMethod -Uri "https://$($orgnizationName)/api/graphql" -Headers @{Authorization = "Bearer $($PrivateToken)" } -Method Post -Body $query -ContentType 'application/json'
$Result.data
```

# Some Common Queries 

From now on, the PowerShell code remains as is! Yes, only the `query` that gets change (based on requirements)

## List project merge requests

```GraphQL
query getProjectInformation($fullPath: ID!) {
  project(fullPath: $fullPath) {
    id
    name
    mergeRequests(state: all) {
      nodes {
        id
        state
      }
    }
  }
}
```

Hey, you know I see only the first `100` records. How do I retrieve all the field objects? Do `Pagination`! Here is the PowerShell script to pull the MR information with pagination 

```PowerShell
$PrivateToken = 'SUPER-SECRET'
$query = @{
    query = @"
        query {
            project(fullPath: "group/project") {
                id
                name
            mergeRequests(first: 5, after: null) {
                    pageInfo {
                        hasNextPage
                        endCursor
                    }
                    nodes {
                        id
                        state
                    }
            }
            }
        }
"@
} | ConvertTo-Json 
$collection = @()

while ($true) {
    $response = Invoke-RestMethod -Uri "https://gitlab.com/api/graphql" -Headers @{Authorization = "Bearer $($PrivateToken)" } -Method Post -Body $query -ContentType 'application/json' -Verbose
    $query = @{
        query = @"
        query {
            project(fullPath: "group/project") {
                id
                name
            mergeRequests(first: 5, after: "$($response.data.project.mergeRequests.pageInfo.endCursor)") {
                    pageInfo {
                        hasNextPage
                        endCursor
                    }
                    nodes {
                        id
                        state
                    }
            }
            }
        }
"@
    } | ConvertTo-Json 

    $collection += $response
    if ($response.data.project.mergeRequests.pageInfo.hasNextPage -eq $false) {
        break 
    }
}
$collection
```

# Comparing two projects

```GraphQL
{
  leftProject: project(fullPath: "group/project") {
    starCount
  }
  rightProject: project(fullPath: "group/project") {
    starCount
  }
}

```

Hey, you know what? I have to compare more fields, and is there any shorthand? Yes, try using a fragment.

```GraphQL
fragment compareFields on Project {
  starCount
  forksCount
  wikiEnabled
  statistics {
    commitCount
  }
  archived
}

{
  leftProject: project(fullPath: "group/project") {
    ...compareFields
  }
  rightProject: project(fullPath: "group/project") {
    ...compareFields
  }
}

```

- `leftproject` and `rightproject` are aliases. 

Now that you know the query operation, work on the scenario, and if you need assistance with an advanced query, feel free to reach me.  


# Mutations 

`Mutation` is to do change the state of the object. Yes, something to create or update. Can we keep it short? Why, not? here is the example

> Create an issue 

```GraphQL
mutation {
  createIssue(
    input: {title: "test -1", projectPath: "group/project", description: "test 1 description"}
  ) {
    issue {
      id
      type
      title
    }
  }
}
```

> Close the issue (Update Demo)

```GraphQL
mutation {
  updateIssue(
    input: {iid: "2", projectPath: "group/project", stateEvent: CLOSE}
  ) {
    issue {
      id
      title
    }
  }
}
```

# Recommendations

- Use variables to make your queries more reusable and dynamic.
- Use fragments to encapsulate related sets of fields.
- Use nested fields to fetch data from related entities.
- Use filters and sorting to narrow down your results.
- Use the GraphiQL explorer to test your queries.
- Read the GitLab GraphQL documentation.

# GraphQL over RESTAPI (GitLab)

- More flexible and expressive than REST.
- More performant than REST for complex queries.
- Easier to maintain than REST.
- More secure and scalable than REST.
- Self-documenting, which means that the schema provides all of the information that you need to understand and use the API.
- Supports powerful features such as filtering, sorting, and pagination.
- Language-agnostic, which means that you can use it with any programming language.

# References

- [GraphQL](https://graphql.org/)
- [GitLab GraphQL](https://docs.gitlab.com/ee/api/graphql/)
- [PowerShell](https://docs.gitlab.com/ee/api/graphql/)


# Summary

GitLab GraphQL API is a powerful tool that allows you to fetch data from GitLab flexibly and efficiently. Using GraphQL, you can write queries to retrieve only the data you need and conveniently format the results. 

In my next blog post, I will demonstrate building a user experience to view the data that helps to visualize the GitLab information. 