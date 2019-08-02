# servo-ec2asg
Set instance type on instances in AWS Auto-Scaling Groups

This Optune adjust driver represents one or more ASGs as application components with a single setting each: `inst_type`. This setting is an enumerated value, the allowed values for it are all possible EC2 instance types.

When ran, it updates the instances in each ASG to the value provided in the input data structure. Only the type of running instances is changed, the ASG's launch template (or launch configuration, if one was used instead of a launch template) is not modified, therefore any new instances will not have the newly set type, but will be created with the type set in the launch template.

If all instances in the ASG match the given type, nothing is done - they will not be stopped and restarted. Only the instances that have a different type will be updated as follows, one by one or in fixed-size batches as specified by the configuration:

- the instances are placed into Standby mode (removing them from the count of active instances and detaching them from ELB, if one is set for the ASG).
- the instances are stopped
- the instance type is changed to the desired one
- the instances are re-started
- the instances are placed back into in-service mode (adding them to the active instances and re-connecting them to ELB)

NOTE if any instances are found in Standby state when the update is ran, they will be re-activated. If it is necessary to exclude an instance temporarily from the ASG for maintenance and/or examination at a time when this driver may be ran to update the ASG, the instance should be detached completely from the ASG instead of using the Standby mode.

## Configuration

The driver expects a configuration file named `config.yaml` in the current directory, with the following structure:

    ec2asg:
       asg: "name-of-ASG,name-of-ASG2,..."
       batch_size: 1

Only the top-level `ec2asg` key is required, any other keys present are ignored (the config file may be used by other Optune drivers, as well). The `asg` setting is mandatory and defines the ASG(s) to be operated on. It is a string containing a comma-separated list of ASG names. Each ASG will be represented as a 'component' in the application description provided when the driver is ran with `--query`.

The `batch_size` setting is optional and defines the maximum number of instances that may be updated in parallel. The default for it is 1 and it cannot be set to more than 20 (AWS API limitation). NOTE if the batch size exceeds the number of instances in the ASG, all of them will be stopped and restarted simultaneously, possibly leading to a short downtime for the application (as there will be no in-service instances during the update, which could last for 1-2 minutes).

In addition to the driver configuration file, an AWS API configuration file and credentials must be present in the `$HOME/.aws/` directory. As a minimum, `~/.aws/config` should contain the `region` setting, e.g.:

    [default]
    region = us-east-1

If the driver is not ran from an AWS EC2 instance that has appropriate role setting that permits API access without credentials, there should also be a `~/.aws/credentials` file with contents similar to:

    [default]
    aws_access_key_id = AAAAAAAAAAAAAAAAAAAA
    aws_secret_access_key = ssssssssssssssssssssssssssssssssssssssss

You can extract particular group's instance ids for monitoring by a measure driver. To do so, define section `monitoring` using an example below.

```yaml
monitoring:
    inst_ids_asg: test-asg
    ref_inst_ids_asg: ref-test-asg
    all_inst_ids_param: InstanceId
```  
`inst_ids_asg` - which group to use to extract instance ids that would be treated as `canary` instances.
`ref_inst_ids_asg` - which group to use to extract instance ids that would be treated as `reference` or in other words `production` instances.
`all_inst_ids_param` - which parameter from EC2 Describe Instance API (https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html#API_DescribeInstances_ResponseElements) to use.
