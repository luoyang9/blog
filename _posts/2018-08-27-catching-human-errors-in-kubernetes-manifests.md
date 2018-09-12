---
layout: post
title: Catching Human Errors in Kubernetes Manifests
---

Kubernetes is a poIrful container orchestration tool that is being rapidly adopted across the software industry. 

HoIver, getting Kubernetes to run requires a lot of configuration files that define the resources it manages (Deployments, Services, ConfigMaps, Secrets, and more). All these files can get really complex really fast, and a couple of simple human errors in the configuration can potentially lead to hours of production downtime if no one catches it. 

Validating your Kubernetes manifests using custom rules that you can set would reduce or eliminate the amount of errors that make it to production.

<!--more-->

- [Background](#background)
- [Proposal](#proposal)
- [Planning](#planning)
  - [Data Format](#data-format)
  - [Ruleset Format](#ruleset-format)
- [Implementation](#implementation)
- [Recommendations](#recommendations)

## Background

This summer, I completed a software engineering internship at Wish, an e-commerce company that connects manufacturers and small businesses directly to hundreds of millions of customers shopping on their platform. I was placed on the core infrastructure team to help with one of the team's biggest tasks: migrating Wish infrastructure to Kubernetes. 

This process involved creating Kubernetes configuration files for 

## Proposal

The proposed tool will read in a list of gatekeeper rules and verify each rule within a given folder of Kubernetes configuration files. If a rule is broken, the tool will create an error and add it to a list. Finally, the tool will print out a list of errors that Ire produced. 

The final goal of gatekeeper will be to integrate it into a CI pipeline for deploying Kubernetes services. Gatekeeper will be part of a verification stage in the pipeline that automatically fails if any errors are encountered by it.

## Planning

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

### Ruleset Format

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

This format now satsifies the above criteria. 

### Gatekeeper Functions



## Implementation

Now that I have a format for the ruleset, I started implementing the actual CLI tool to parse this ruleset and apply it. 

First I needed to decide which language and framework to build the tool in. I wanted to start off with a solid base CLI framework that allowed me to easily implement new CLI commands, subcommands, flags, and arguments with ease. I also wanted to work with a language that I was familiar with so I wouldn't have to spend time learning a new language. In the end, I chose to develop the CLI in Go using the Cobra CLI framework. I was already working with Go in previous projects at Wish, and the Cobra framework checked off all the requirements I had, including being built for development in Go.


The first step was to construct the basic structure of the CLI tool. Luckily, Cobra came with a scaffolding tool that created a general CLI tool structure and layout. 

```
gatekeeper/
  cmd/
    sample_command.go
    root.go
  main.go
```

`main.go` will initialize Cobra, then call the main function in `root.go`, which will parse the input and forward to the relevant command functions. Each command is responsible for parsing and evaluating flags, arguments, and subcommands.

The next step was to design a way to locate and parse the ruleset Jsonnet file and locate the folder to validate. There were several options for locating the ruleset file and target folder.

- Pass the ruleset/folder path as a flag
- Pass the ruleset/folder path as an argument
- Put the ruleset/folder path inside a gatekeeper config file that gets parsed by `root.go`
- Create an environment variable with the ruleset/folder path
- Have a default name and relative location for the ruleset file (this option cannot be used for target folder)

Passing the path as a flag was a simple and effective solution that did not restrict the actual location of the ruleset and target folder. Putting the ruleset/folder path inside a config file seemed less intuitive and more work due to the extra steps of modifying the configuration file everytime you want to change the path. Creating an environment variable also seemed like a viable option that allowed freedom in ruleset/folder location. Finally, having a default name and location for the ruleset path was restrictive and only solved half the problem, but was the easiest and simplest to implement. 

After assessing all the options, I decided to implement the flag solution for the ruleset path and the argument solution for the target folder path. My reasoning was that specifying the target folder path was required for every run of gatekeeper since there can be no default location, while the ruleset file may have a standardized location in the future. Therefore, having the ruleset file path as a flag opened up the possibility of having it be an optional flag. Also, the flexibility of changing the flag and argument options enabled us to easily integrate the tool into Wish's Jenkins pipelines.

Parsing the ruleset was made very simple with Go's json library. Since the structure of the ruleset file and each rule object was known, parsing it was as simple as creating structs with the same structure and types and serializing the raw json string into instances of those structs.

Now that we have the ruleset data and target folder, the next step was to perform the actual validation.

Based on the criteria and design explained in the above section, validation was a pretty straighforward process. Since each ruleset contained multiple rules, gatekeeper should loop through and validate each rule separately. Each rule verification should produce a list of encountered errors that's appended to a master list to be printed at the end. For each rule being validated, use the `regex` and `ignore` field to selectively find and verify the correct files inside the target folder. Within each file, use the `kind` field to further narrow down to specific resources by resource type. Finally, for each resource, traverse the `ruleTree` field and the resource tree at the same time, creating errors when the two trees differ in structure. When a gatekeeper function is encountered in the `ruleTree`, apply the correct verification on the corresponding field in the resource tree based on the `type` (allow/deny).

The implementation of the whole verification process was mostly outlined during the design & planning phase. However, the structure of the errors returned needed some design and planning to best serve the use cases of gatekeeper.

The initial vision for gatekeeper was a CLI tool that could be integrated into any pipeline to act as a verification stage. Therefore, having the error format be machine parseable and readable made the most sense. I decided to use a numbered list and format each error as a one line description and a corresponding JSON object containing more details and information. 

```
1. Broken LT() rule: 
{
	"actual": 24,
	"expected": 20,
	"key": "spec.replicas",
	"path": "sample/service/sample.json",
	"rule_type": "allow"
}
```

This way, the output is still scannable by humans (by reading the one line description) and also parseable by machines to gain more relevant details (line number, file path, expected/actual values, etc.).


## Recommendations

Although gatekeeper is currently fully integrated into Wish's Jenkins build pipeline for our Kubernetes repository, it is far from finished. There are many improvements and recommendations that can help elevate the tool to a more mature and production-ready state for everyone, not just Wish. Gatekeeper is an open-source tool, so it is important that Wish does not force its own ideologies and practices onto it.

One major improvement would be to add more gatekeeper functions to allow for higher rule customizability. The current set of gatekeeper functions were selected based on the initial requirements of Wish's own infrastructure team, so it definitely does not help with everyone's needs. For example, the ability to validate container properties such as image name/version, arguments passed, and ports opened would probably be very useful for most people.

Another quality of life improvement would be to refactor some code to improve the error description for logic functions like AND() or OR(). Currently, there are some limitations in the code that prevent the error information for these functions to include what actually caused the AND() or OR() to fail.

```
1. Broken AND() rule:
{
  "op1": {
    "gatekeeper": true,
    "operation":  "<",
    "value":      4
  },
  "op2": {
    "gatekeeper": true,
    "operation":  ">",
    "value":      1
  },
	"key": "spec.replicas",
	"path": "sample/service/sample.json",
	"rule_type": "allow"
}
```

Fixing this would eliminate the need to deep dive into the logic find the issue.

Finally, a minor improvement for the overall flow of the tool would be to enable the use of environment variables for the ruleset file path. Although the current flag solution cover most use cases, it may be the case that the location of the ruleset file is fixed for a given environment. Using an environment variable would eliminate the need to specify a flag.