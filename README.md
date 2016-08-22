# Chef Compliance Patterns

As part of your organization's Security Compliance efforts, you will want to be
able to quickly understand what risks your infrastructure has so that you are
able to remediate and eliminate them from the environment.  Chef provides all
the components necessary to automate this process.

## The Idea
Let's keep it really simple.

 **Scan -> Add Hardening -> Repeat**

We'll begin by creating a Hardening Cookbook and a Audit Cookbook for our
organization.

Assuming we are "BrewInc." and we run RHEL 7 internally, we shall christen them
`brewinc_rhel_hardening` and `brewinc_rhel_auditing`.

The hardening cookbook will house all recipes for ensuring the system adheres to
our Compliance controls. The auditing cookbook will provide all the inspec
tests for testing our organization's RHEL 7.x image. It will function as a
`Role Cookbook` wrapping the https://github.com/chef-cookbooks/audit cookbook.

Our node's will likely utilize a base Role Cookbook to set up the run_list order
which is significant because we want to first harden the OS, followed by
applying any other Application specific cookbooks, then finally we want our
auditing cookbook to run last, reporting on the final state of the node.

```ruby
# BrewInc base Role Cookbook
incude_recipe "brewinc_rhel_hardening"
incude_recipe "appFoo_setup"
incude_recipe "appFoo_deploy"
...
incude_recipe "brewinc_rhel_auditing"
```

With run_list controlled in this way, every time your fleet converges, Chef
executes the hardening recipes, inspec locally scans the systems and then
reports back on how compliant they are.

## Scanning
Inspec is the toolset that allows you to easily scan your entire infrastructure
for risks and compliance issues and report on them.

As you might suspect, the place to get started with **Compliance as Code** is in
local development, for instance utilizing: ChefDK's `test-kitchen`.

First, let's review how the CIS benchmarks are categorized.  Quite simply, they
are grouped by Operating System flavor and then broken out into Level 1 and
Level 2.  Level 1 recommendations are settings that should be configured at a
minimum on a server or a database, and should cause little or no interruption of
service, whereas Level 2 recommendations are recommended in highly secure
environments. In some cases Level 2 settings could also lead to reduced
functionality.

```ruby
# cis-rhel7-level1

control "xccdf_org.cisecurity.benchmarks_rule_1.1.1_Create_Separate_Partition_for_tmp" do
  title "Create Separate Partition for /tmp"
  desc  "The /tmp directory is a world-writable directory used for temporary storage by all users and some applications."
  impact 1.0
  describe mount("/tmp") do
    it { should be_mounted }
  end
end

control "xccdf_org.cisecurity.benchmarks_rule_1.1.2_Set_nodev_option_for_tmp_Partition" do
  title "Set nodev option for /tmp Partition"
  desc  "The nodev mount option specifies that the filesystem cannot contain special devices."
  impact 1.0
  describe mount("/tmp") do
    it { should be_mounted }
  end
  describe mount("/tmp") do
    its("options") { should include "nodev" }
  end
end
```

We'll follow the Center for Internet Security's lead and adopt a layered
security compliance approach consisting of our organization's customization of
the CIS benchmarks for Level 1 and Level 2. At minimum, we'll test against and
enforce CIS Level 1 for our entire fleet.  In areas that require greater
security, we will additionally leverage Level 2.  On systems where hardening
breaks functionality, we can exclude by creating an exception.

Let's begin with adding the inspec tests to our cookbook

```yaml
# .kitchen.yaml
suites:
  - name: default
    verifier:
      inspec_tests:
        - compliance://cis-rhel7-level1
```

Profiles can be hosted from various locations; for details see: https://github.com/chef/kitchen-inspec#use-remote-inspec-profiles

We could also add the inspec tests to the `test/` directory of our cookbook
where we normally place our integration tests. If you own Chef Automate
licenses, you can find the profiles here on your Compliance Server: `/var/opt/chef-compliance/core/runtime/compliance-profiles`.  This would suffice
when adding tests into `test/`:

```yaml
# .kitchen.yml
verifier:
  name: inspec
  format: doc
```

All the example above are intended for running in Local Dev or Pipeline context.
When deployed onto the nodes, the `brewinc_rhel_auditing` cookbook will utilize
a ruby Hash attribute to control which Compliance Profiles are executed:

```ruby
audit = {
  "profiles" => {
    # org / profile name from Chef Compliance
    'base/linux' => true,
    # supermarket url
    'brewinc/ssh-hardening' => {
      # location where inspec will fetch the profile from
      'source' => 'supermarket://hardening/ssh-hardening',
      'key' => 'value',
    },
```
**Important**
You will want to set the attribute above based on each Node's OS family type so
that only the OS appropriate controls are executed and reported.

More examples: https://github.com/chef-cookbooks/audit#configure-node

## Remediation
During Local Development, you'll want to remediate the problems detected by the
inspec scans prior to Deployment.

Building automated compliance testing with remediation into your pipeline adds
significant velocity to your processes.

A simple way to manage what level of hardening gets applied is by utilizing a
node attribute:

```ruby
# brewinc_rhel_hardening::default.rb

# add hardening
case node['hardening_level']
when 1
  include_recipe "brewinc_rhel_hardening::level1"
when 2
  include_recipe "brewinc_rhel_hardening::level2"
end
```

In the future, Chef Compliance may include remediation recipes, but for now it
does not.  You will need to create those yourself.  Some possible resources that
you can quickly leverage can be found on Supermarket: https://supermarket.chef.io/cookbooks?utf8=✓&q=hardening&platforms%5B%5D=


## Allowing Exceptions
You may discover that some Applications were built insecurely and break if
certain hardening is applied.  If that is the case, you can create an array
attribute which lists the CIS Control sections that should not be enforced.

```ruby
default['hardening_exceptions'] = ['1.2.3','1.4.1',...]
```

In the hardening recipes you can utilize guards on each resource.
```ruby
file '/etc/shadow' do
  owner 'root'
  group 'root'
  mode '0600'
  not_if { node['hardening_exceptions'].includes?('1.1.2') }
end
```

**Important:**
Do not create a process by which certain InSpec controls are skipped or ignored.
Doing so only complicates your ability to quickly understand your Compliance
Reports and potentially masks security vulnerabilities that you want to be
aware of. Measure your fleet's state of Compliance in its entirety with
completeness and with accuracy. Minimize the type of buckets used for reports:
"Level1" vs "Level2" is a good example of this.

## Reporting
There are several options for handling Compliance reports generated by the
`audit` cookbook.
Namely reporting can be:
1. Straight to Compliance Server
2. Via Chef Server by means of `chef_gate`
3. To Visibility's data_collector

For more detail: https://github.com/chef-cookbooks/audit#overview

## Creating Custom Compliance profiles
Ultimately, you existing organization's Security practitioners will likely want
to specify custom Compliance controls.  An easy way to accomplish this is with
InSpec's Profile Inheritance.  

```ruby
include_controls 'cis-rhel7-level1' do
  skip_control 'xccdf_org.cisecurity.benchmarks_rule_1.1.2_Set_nodev_option_for_tmp_Partition'

  control 'brewinc-1.1.0' do
    impact 0.0
    ...
  end
end
```
