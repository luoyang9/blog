---
layout: post
title: Catching Human Errors in Kubernetes Manifests
---

Kubernetes is a poIrful container orchestration tool that is being rapidly adopted across the software industry. 

HoIver, getting Kubernetes to run requires a lot of configuration files that define the resources it manages (Deployments, Services, ConfigMaps, Secrets, and more). All these files can get really complex really fast, and a couple of simple human errors in the configuration can potentially lead to hours of production downtime if no one catches it. 

Validating your Kubernetes manifests using custom rules that you can set would reduce or eliminate the amount of errors that make it to production.

<!--more-->

## Background

This summer, I completed a software engineering internship at Wish, an e-commerce company that connects manufacturers and small businesses directly to hundreds of millions of customers shopping on their platform. I was placed on the core infrastructure team to help with one of the team's biggest tasks: migrating Wish infrastructure to Kubernetes. 

This process involved creating Kubernetes configuration files for 

## Proposal

The proposed tool will read in a list of gatekeeper rules and verify each rule within a given folder of Kubernetes configuration files. If a rule is broken, the tool will create an error and add it to a list. Finally, the tool will print out a list of errors that Ire produced. 

The final goal of gatekeeper will be to integrate it into a CI pipeline for deploying Kubernetes services. Gatekeeper will be part of a verification stage in the pipeline that automatically fails if any errors are encountered by it.

## Planning Phase

I initially started out by brainstorming the possible rules that my team would want to implement in gatekeeper. After some thoughtful discussions with my co-workers, I came up with a list of options that should be easily customizable to best fit the use cases.

1. A rule should be customizable to only apply to certain files and ignore other files
2. A rule should be customizable to only apply to certain Kubernetes resources and ignore other Kubernetes resources
3. A rule should either verify that all gatekeeper functions are met or verify that no gatekeeper functions are met
4. A rule should specify which gatekeeper functions apply to which fields in the actual JSON

The next step was to create a ruleset format that can capture these criteria. 

### Data Format

First, I had to determine which data format to use. I considered popular formats such as JSON, YAML, and XML as well as more powerful templating formats such as Jsonnet. In the end, I decided on Jsonnet to take advantage of its built-in functions feature. This way, I can easily implement new gatekeeper functions by creating predefined Jsonnet functions that return a JSON object containing the function information. Then I can simply inject them into the actual ruleset file to allow developers to create complex rule definitions using these predefined functions. 

Here's a sample of the gatekeeper.jsonnet file that defines the gatekeeper functions.

```
// LT() checks if the selected field is less than the given value
local LT(value=0) = {
  gatekeeper: true,
  operation: "<",
  value: value
};

// GT() checks if the selected field is greater than the given value
local GT(value=0) = {
  gatekeeper: true,
  operation: ">",
  value: value
};

// EQ() checks if the selected field is equal to the given value
local EQ(value="") = {
  gatekeeper: true,
  operation: "=",
  value: value
};

// AND() checks if both op1 and op2 are satisfied
local AND(op1, op2) = {
  gatekeeper: true,
  operation: "&",
  op1: op1,
  op2: op2,
};
```

Each function encapsulates the function type, description, and parameters so that in the actual ruleset file, you can simply use these functions like so:

```
spec: {
  replicas: AND(GT(2), LT(25))
},
```

In this rule, I verify that the `spec.replicas` field is greater than `2` and less than `25`. 

Other data formats such as JSON, YAML, or XML were too simple and would have bloated the ruleset file with function definitions and parameters. Jsonnet allowed me to abstract the preset gatekeeper functions out and keep the ruleset file clean, readable, and descriptive.

### Ruleset Jsonnet Format

Now that I have the data format, the next step was to design the layout of the ruleset.jsonnet file. For this, I used the ruleset criteria that I created during my brainstorming session.

First, since a ruleset is composed of multiple rules, there should be a `rules` field that contains an array of rule objects.

```
{
  rules: [
    ...
  ]
}
```

For each rule object, we want the ability to only verify specific files, as defined by criterion 1. For this, I came up with a `regex` field that was used to match file names. The rule would only apply to files that match the rule regex.

```
{
  rules: [
    {
      regex: "*namespace.json"
    },
    ...
  ]
}
```

After some consideration, I also added an `ignore` field in the root object that contains an array of filenames to ignore. I realized that there was no clean way to ignore certain files using the current regex matching solution. For example, in our Kubernetes setup, there was a specific `channel.yaml` file that we always wanted to ignore because it wasn't actually a Kubernetes file.

```
{
  ignore: ["channel.yaml"],
  rules: [
    {
      regex: ".*namespace.json"
    },
    ...
  ]
}
```

Criterion 1 is now satisfied. Developers can create custom rules that match specific files using regex and also ignore special files. The next criterion specifies that rules should only apply to certain Kubernetes resources, so I created a `kind` field in the rule object that specifies the kind of resource this rule applied to.

```
{
  ignore: ["channel.yaml"],
  rules: [
    {
      regex: "*namespace.json",
      kind: "Namespace"
    },
    ...
  ]
}
```

The third criterion was easily covered by adding another field called `type`. A value of `allow` would indicate a rule that verified all gatekeeper functions were satisfied, while a value of `deny` would indicate a rule that verified no gatekeeper functions were satisfied.

```
{
  ignore: ["channel.yaml"],
  rules: [
    {
      regex: "*namespace.json",
      kind: "Namespace",
      tyoe: "allow",
    },
    ...
  ]
}
```

Finally, to satisfy the fourth criterion, I added a `ruleTree` field that contained a JSON object describing the structure of the corresponding Kubernetes resource and applying gatekeeper functions to specific fields.

```
{
  ignore: ["channel.yaml"],
  rules: [
    {
      regex: "*namespace.json",
      kind: "Namespace",
      tyoe: "allow",
      ruleTree: {
        metadata: {
          labels: {
            name: AND(TAG("namespace"), PATH(1))
          },
          name: TAG("namespace")
      }
    },
    ...
  ]
}
```

## Implementation

Gatekeeper's process in verifying each rule.

CLI specification and how to use gatekeeper.

How to implement the various parts of each rule. How to implement each gatekeeper function. How to parse unstructured JSON in Go. Traversing two ruletrees.


## Recommendations & Potential Improvements

Add more gatekeeper rules to allow for more customizable rules
Refactor some code to allow for more descriptive error messages
Improve test coverage
