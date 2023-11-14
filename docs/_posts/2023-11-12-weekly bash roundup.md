---
layout: post
title:  "Weekly bash roundup"
date:   2023-11-12 08:30:00 -0800
---
I wrote a few cool bash snippets last Friday while refactoring some terraform

## Removing aws eventbridge rule targets so that Terraform can delete them

While running a `terraform apply` after removing some eventbridge rules,
I was confronted by an error stating that the rules could not be deleted while
they still had targets.

```bash
 aws events list-rules --query 'Rules[*].[Name]' --output text > allrules.txt
 grep $PATTERN_ONE < allrules.txt >> rules.txt
 grep $PATTERN_TWO < allrules.txt >> rules.txt

 xargs -I {} sh -c 'aws events remove-targets \
  --rule "$1" \
  --ids $(aws events list-targets-by-rule \
      --rule "$1" \
      --query "Targets[*].Id" \
      --output text)' - {} < rules.txt
```

## moving terraform state after renaming a module

In this case, I renamed a terraform module from "buckets" to "stage."
This changes the ID of the resources within that module.

If we run `terraform plan` without doing anything else, terraform will attempt to
destroy and re-create the resources. This is non-ideal for 2 reasons.
First, re-creating this resource may have side-effects.
Second, because Terraform does not know that these resources are related,
it does not know that the destruction must occur *before* re-creation, and may error out.

The solution is to move the state using `terraform state mv`

We have existing addresses in the state file which look like this:

module.main.[...].module.<span style="color:red">buckets</span>["restricted"].snowflake_stage.this

and we need to change them as such

module.main.[...].module.<span style="color:green">stages</span>["restricted"].snowflake_stage.this

```bash
terraform state list |
rex '(\S*\.module\.)buckets(\S*)' |
awk '{print "terraform state mv \047" $1 "buckets" $2 "\047 \047" $1 "stages" $2 "\047"}' > move.sh

sh move.sh
```

rex is a [command line utility I wrote](https://github.com/DaraDadachanji/gorex) to perform regex extractions in the terminal. Let's look at this one:

`(\S*\.module\.)buckets(\S*)`

it's comprised of two capture groups. The first captures everything *before* the
name of the module we are renaming, the second captures everything afterwards.

the output of the gorex program looks like this:

```text
module.main.[...].module. ["restricted"].snowflake_stage.this
module.main.[...].module. ["restricted"].snowflake_stage_grant.ownership
```

Each line contains two fields separated by a space. We can pipe this into `awk`
and begin constructing our terraform commands.

`\047` prints a single quote `'`.
We wrap each address in single quotes since the addresses contain double quotes

### What if there are spaces in the terraform addresses?

Sometimes terraform addresses can have spaces in them.
Usually as a result of `for_each` blocks. Let's say our addresses looked like this

`module.main.["this has spaces"].module.stages["this also has spaces"].snowflake_stage.this`

How would we alter our script to account for this? Simple: we change the field separator.

```bash
terraform state list |
rex -F ::: '(\S*\.module\.)buckets(\S*)' |
awk -F ::: '{print "terraform state mv \047" $1 "buckets" $2 "\047 \047" $1 "stages" $2 "\047"}' > move.sh
```

both `awk` and `rex` can change the field separator used in their inputs and outputs (respectively)
using the `-F` flag.
