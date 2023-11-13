---
layout: post
title:  "Weekly bash roundup"
date:   2023-11-12 08:30:00 -0800
categories: bash regex terraform
---
I wrote a few cool bash snippets last week while refactoring some terraform

## Removing aws eventbridge rule targets so that Terraform can delete them

While running a `terraform apply` after removing some eventbridge rules,
I was confronted by an error stating that the rules could not be deleted while
they still had targets.

{% highlight bash %}
 aws events list-rules --query 'Rules[*].[Name]' --output text > allrules.txt
 grep catalog < allrules.txt >> rules.txt
 grep data_quality < allrules.txt >> rules.txt

 xargs -I {} sh -c 'aws events remove-targets \
  --rule "$1" \
  --ids $(aws events list-targets-by-rule \
      --rule "$1" \
      --query "Targets[*].Id" \
      --output text)' - {} < rules.txt
{% endhighlight %}

## moving terraform state after renaming a module

In this case, I renamed a terraform module from "buckets" to "stage."
This changes the ID of the resources within that module

{% highlight bash %}
tf plan | tee plan.json

gorex '(module\.main\.\S*\.module\.)buckets(\[[^\]]*\]\.snowflake_stage\.\S*)\s*will be destroyed' < plan.json |
  escape |
  awk '{print "terraform state mv " $1 "buckets" $2 " " $1 "stages" $2}' > move.sh
{% endhighlight %}

gorex is a command line utility I wrote to do regex extractions. Let's look at this one:

{% highlight regex %}
(module\.main\.\S*\.module\.)buckets(\[[^\]]*\]\.snowflake_stage\.\S*)\s*will be destroyed
{% endhighlight %}

it's comprised of two capture groups