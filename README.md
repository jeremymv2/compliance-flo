# Chef Compliance Patterns

As part of your organization's Security Compliance efforts, you will want to be able to quickly understand what risks your infrastructure has so that you are able to remediate and eliminate them from the environment.  Chef provides all the components necessary to automate this process.

## The Idea
Let's keep it really simple.

 **Scan -> Harden -> Repeat**

We'll create a Hardening Cookbook for our organization.  It will be OS specific so as to keep the scope small.

We shall christen it `brewinc_rhel_hardening`
It will have all the inspec tests for testing our organization's RHEL 7.x image and all the recipes for ensuring the system adheres to the controls.  It will serve as a `Role Cookbook` wrapping the https://github.com/chef-cookbooks/audit cookbook.

With the role cookbook added to the end of the run_list, every time your fleet converges, Chef executes the hardening recipes, inspec locally scans the systems and reports back on how compliant they are.

## Scanning
Inspec is the toolset that allows you to easily scan your entire infrastructure for risks and compliance issues and report on them.  The awesome part of how inspec is implemented is in how

As you might suspect, the place to get started with **Compliance as Code** is in local development, for instance utilizing: ChefDK's `test-kitchen`.

If you have purchased a Chef Automate license then you can jumpstart with the pre-built profiles bundled on the Compliance Server.

First, let's review how the CIS benchmarks are categorized.  Quite simply, they are grouped by Operating System flavor and then broken out into Level 1 and Level 2.  Level 1 recommendations are settings that should be configured at a minimum on a server or a database, and should cause little or no interruption of service, whereas Level 2 recommendations are recommended in highly secure environments. In some cases Level 2 settings could also lead to reduced functionality.

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

We'll follow the Center for Internet Security's lead and adopt a layered security compliance approach consisting of our organization's customization of the CIS benchmarks for Level 1 and Level 2. At minimum, we'll test against and enforce CIS Level 1 for our entire fleet.  In areas that require greater security, we will additionally leverage Level 2.  On systems where hardening breaks functionality, we can exclude by creating an exception.

Let's begin with adding the inspec tests to our cookbook

```yaml
# .kitchen.yaml
suites:
  - name: default
    verifier:
      inspec_tests:
        - compliance://cis-rhel7-level1
```

Profiles can be hosted in various locations; for details see: https://github.com/chef/kitchen-inspec#use-remote-inspec-profiles

## Remediation
Automate
Build automated compliance testing and remediation into your pipeline

```ruby
# default.rb
case node['hardening_level']
when 1
  include_recipe "brewinc_rhel_hardening::level1"
when 2
  include_recipe "brewinc_rhel_hardening::level2"
end
```

## Allowing Exceptions

```ruby
file '/etc/shadow' do
  owner 'root'
  group 'root'
  mode '0600'
  not_if { node['hardening_exceptions'].includes?('1.1.2') }
end
```

## Reporting
https://github.com/chef-cookbooks/audit#overview
