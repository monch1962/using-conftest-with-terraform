# Using conftest to validate Terraform
Notes on how to use Conftest to validate Terraforms

## Overview
There's plenty of resources describing how to use conftest (https://www.conftest.dev/) to validate Terraforms, however they all seem to be written at a high level for trivial Terraforms. When defining IaaS using Terraform to create the infrastructure for a large project, the process of writing conftest policies is considerably more complicated than those simple examples would have you believe.

This document goes through some of the challenges I've found, and outlines processes to address them

## Getting started

conftest wants to work with something like a JSON file, not Terraforms. To convert those Terraforms into a format that conftest can consume, run the following commands from within the directory containing your Terraform files:
```
$ terraform init
$ terraform plan -out=tfplan_2_resources_planned
$ terraform show -json ./tfplan_2_resources_planned | jq > tfplan.json
```

This will create a `tfplan.json` file which can be checked by conftest. The following sections assume you have a `tfplan.json` file and you want to check that it complies with certain standards

Note that running these commands will use an existing `.tfstate` file that describes the current infrastructure configuration if it exists. If this is the case, then `tfplan.json` will contain both `before` and `after` sections that document the changes that would be made to the infrastructure if you ran `$ terraform apply`

## Challenges

### Terraform plan JSONs don't follow a fixed schema or hierarchy, and have an arbitrary number of levels based on the complexity of the environment being built

It'd be great if Terraform worked to a known hierarchy, but infrastructure doesn't work that way. In practice, you might find VPCs inside VPCs inside VPCs so it's turtles all the way down.

One consequence of this is that you can't just look at e.g. `enable_key_rotation` values at a single level in your Terraform JSON - they could be sprinkled all through the `tfplan.json`, at different levels of hierarchy

One approach for identifying all the `enable_key_location` keys inside `tfplan.json` is to leverage `jq` (https://stedolan.github.io/jq/) as follows

```
$ cat tfplan.json| jq -c "paths | select(.[-1] == \"enable_key_rotation\")";
```

This command will identify all the keys in `tfplan.json`, and print out JSONpath-like details of how to access them.

The output should look something like this:

```
["planned_values","root_module","child_modules",0,"resources",17,"values","enable_key_rotation"]
["planned_values","root_module","child_modules",0,"child_modules",0,"resources",82,"values","enable_key_rotation"]
["resource_changes",17,"change","before","enable_key_rotation"]
["resource_changes",17,"change","after","enable_key_rotation"]
["resource_changes",320,"change","before","enable_key_rotation"]
["resource_changes",320,"change","after","enable_key_rotation"]
["prior_state","values","root_module","child_modules",0,"resources",17,"values","enable_key_rotation"]
["prior_state","values","root_module","child_modules",0,"child_modules",4,"resources",82,"values","enable_key_rotation"]
["configuration","root_module","module_calls","env","module","resources",17,"expressions","enable_key_rotation"]
["configuration","root_module","module_calls","env","module","module_calls","fargate_tasks","module","resources",82,"expressions","enable_key_rotation"]
```

The above output contains 10 lines, indicating that the key `enable_key_rotation` exists at 10 different places inside `tfplan.json`.

The first line `["planned_values","root_module","child_modules",0,"resources",17,"values","enable_key_rotation"]` indicates that there's an `enable_key_rotation` key sitting at JSONpath `.planned_values.root_module.child_modules[0].resources[17].values.enable_key_rotation`. You can interpret the remaining 9 lines of output similarly.

These JSONpaths are what you'll want to use within your conftest checks, which leads us to the next challenge

### Terraform plans have no defined sequence, and running `terraform plan` multiple times against the same TF files is likely to change the order in which the items in that plan are created inside `tfplan.json`

Following on from the above point, we can see that the first instance of `enable_key_rotation` is currently at JSONpath `.planned_values.root_module.child_modules[0].resources[17].values.enable_key_rotation`. However, if you were to make an unrelated change to that Terraform, the `child_modules[0]` and `resources[17]` parts of that JSON path may change as the `tfplan.json` file isn't ordered in any way.

To work around this inside your conftest policies, you may want to implement a check such as 
```
deny[msg] {
    disabled_key_rotations := {k | k := input.planned_values.root_module.child_modules[_].resources[_].values; k.enable_key_rotation == false}
    count(disabled_key_rotations) != 0
    msg := "3. Key rotation not enabled at .planned_values.root_module.child_modules[_].resources[_].values"
}
```
which will look all through the `tfplan.json` file for any violations regardless of how those 2 arrays in the JSONpath have been sequenced.

### Identifying what's going to be changed if you apply this Terraform plan

Terraform is a declarative tool, meaning it will make the minimal changes to an environment to get it into the desired state. To identify only the changes that will be made to the current state of the environment, you can use `jq` as follows:
- Resource changes: `cat tfplan.json | jq .resource_changes`
- Type of resources being changed: `cat tfplan.json | jq '.resource_changes[].type'`
- Resource action (change, create, destroy): `cat tfplan.json | jq '.resource_changes[].change.actions[]'`

Full disclosure: I've lifted these commands from https://dev.to/lucassha/don-t-let-your-terraform-go-rogue-with-conftest-and-the-open-policy-agent-233b - I only repeat them here in case that page disappears at some point.

These steps can come in handy if you only want to use conftest to check the changes about to be applied to the environment, rather than the entire environment. Possible reasons for doing this may be as follows:
- you might have known non-compliant elements in your existing infrastructure, and you don't want to keep reporting those infrastructure elements as non-compliant every time you run tests. This is particularly handy if your conftest compliance checks are run as part of a CI pipeline; in this case, you might not want "known problems" to block your Terraform deployment
- you might only be interested in ensuring _changes_ to the infrastructure are compliant. This could be the case if you're retrospectively introducing controls to existing infrastructure, with the aim of bringing that infrastructure into a compliant state over a period of time

### Naming conventions for particular fields within Terraform (or its providers) aren't consistent

To be fair, this is more likely to be a problem with Terraform providers, and as such is likely to remain a problem for the foreseeable future

When I look through the `enable_key_rotation` keys inside my `tfplan.json` file, in one place I see:
```
"enable_key_rotation": true,
```
and in another place I see
```
"enable_key_rotation": { "constant_value": true },
```

Yes, this is inside the same `tfplan.json` file, and is yet another hurdle that needs to be addressed in your conftest checks.

Once you know this problem exists, it's relatively easy to alter the check in your REGO files to account for it. However, you need to know that it exists, otherwise you might find some of your tests pass when they should be failing.

## Handy references

These are some of the handy references to refer to:
- overview of important security parameters for the AWS provider (https://faun.pub/managing-aws-best-practices-using-terraform-6c1a4fc517c4)
- Terraform best practices for AWS users (https://github.com/ozbillwang/terraform-best-practices)

Bridgecrew has a bunch of interesting content that's worth checking out:
- https://bridgecrew.io/blog/infrastructure-security-at-scale-with-bridgecrew-for-terraform-cloud/
- https://workshop.bridgecrew.io/terraform.html
- https://bridgecrew.io/blog/tutorial-find-fix-and-prevent-terraform-misconfigurations-with-bridgecrew/
- scanning Serverless framework's YAML files for security problems (https://bridgecrew.io/blog/securing-serverless-framework-configurations-for-aws-using-bridgecrew/)
- checking Terraforms using checkov (https://bridgecrew.io/blog/terraform-variable-evaluation/)

Checkov (https://www.checkov.io/) by Bridgecrew looks like a great option; need ot investigate this further:
- git repo https://github.com/bridgecrewio/checkov

