At AppNeta, we provide software instrumentation, open source libraries and network hardware to instrument your network and software to provide information to you about your Full Stack (tm).  On the software side, we do this by emitting tracing information back to our application, where we process, analyze and consolidate it to provide you with precise information about the health of your stack.  We're a big fan of AWS, and have used spot pricing and AutoScaling groups to make our trace processing pool efficient and, more importantly, inexpensive to run.

### Problems We've Encountered ###

However, auto-scaling groups are not a panacea to allievieating scaling concerns.  Spot instances can be in heavy demand during certain periods, and bidding for them can be fierce, to say the least.  One strategy that we have seen other users of spot instances employ is heavily over-bidding on spot instances.  For example, we've seen spot prices for m3.2xlarge instances go as high as $7.00 for a four-day period.  The normal on-demand price for these instance types is $1.00.  Although this can ensure spot instance requests can be fulfilled, it's economically infeasable, to say the least. 

Depending on how your autoscaling group is configured as well, being blocked from fulfilling spot instance requests can cause other problems.  We recently modified our autoscaling groups to split across three availability zones (AZ).  However, we noticed that in one AZ, the spot price had jumped well above our request spot price.  Since autoscaling groups, by default, attempt to balance the instances evenly across the zones you have defined, this resulted in the autoscaling group terminating instances in one of the other AZs as well.  This effectively decreased our processing capability to a third of its normal capacity.  While this was technically enough capacity to handle our trace stream without problems, we would now have to manually manage our instance counts to handle possible incoming spikes.

### Our solution ###

We knew that we wanted to keep a minimum number of instances up regardless of the availability of spot instances.  However, we obviously want to favor using spot instances for the cost factor.  So, our implementation follows one basic rule:  if a spot instance terminates, bring up a on-demand instance to cover it.  We do this with a combination of SNS, SQS and boto.

We define a SQS queue and SNS topic to provide the bridge to provide the bridge between our autoscaling group and the daemon that will monitor it.

```bash
ubuntu@host $ aws --region=us-east-1 sqs create-queue --queue-name spot-scale-demand
{
    "QueueUrl": "https://queue.amazonaws.com/1111111111/spot-scale-demand"
}
```

```bash
SNS_TOPIC='spot-autoscale-notifications'
SQS_QUEUE='spot-autoscale-notifications'

SNS_ARN=$(sns-create-topic $SNS_TOPIC)
SQS_ARN="arn:aws:sqs:us-east-1:1111111111:${SQS_QUEUE}"
sns-add-permission $SNS_ARN --label "ETLDemandBackupGroup" --action-name Publish,Subscribe,Receive --aws-account-id 11111111111 
sns-subscribe $SNS_ARN --protocol sqs --endpoint "$SQS_ARN"
```

We set up our autoscaling groups.  We start off with having the demand group be empty, since we want to favor having our processing done on spot instances.
```bash
as-create-launch-config spot-lc --image-id ami-12345678 --instance-type m3.2xlarge \
    --group etl --key appneta --user-data-file=prod_data.txt \
    --spot-price 1.00
as-create-launch-config demand-lc --image-id ami-12345678 --instance-type m3.2xlarge \
    --group etl --key appneta --user-data-file=prod_data.txt
as-create-auto-scaling-group spot-etl-20131116 --launch-configuration spot-lc \
    --availability-zones us-east-1a,us-east-1b --default-cooldown 600 --min-size 1 --max-size 3
as-create-auto-scaling-group demand-etl-20131116 --launch-configuration demand-lc \
    --availability-zones us-east-1a,us-east-1b --default-cooldown 600 --min-size 0 --max-size 3
```
I'm leaving off scaling policies and metric alarms, since they'll be specific to your processing and workload.

We define notification configurations for each autoscaling group, and tell the autoscaling group to send a message to the SNS topic we defined whenever an instance is launched or terminated in the group.  We also have another notification configuration for sending an email to our operations team when these scaling actions happen so that we have a secondary history.
```bash
for group in spot-etl-20131116 demand-etl-20131116; do
    for notification in spot-autoscale-notifications ops-alerts-email ; do
        as-put-notification-configuration $group -t "arn:aws:sns:us-east-1:111111111:${notification}" \
            -n autoscaling:EC2_INSTANCE_LAUNCH, autoscaling:EC2_INSTANCE_TERMINATE
    done
done
```

For the last part of our setup, let's be nice to our operations team and turn on Cloudwatch metrics on the autoscaling groups.
```bash
for group in spot-etl-20131116 demand-etl-20131116; do
    as-enable-metrics-collection $group -g 1Minute \
        -m GroupMinSize,GroupMaxSize,GroupPendingInstances,GroupInServiceInstances,GroupTotalInstances,GroupTerminatingInstances,GroupDesiredCapacity
done

```

Now we'll look into a overview of the daemon itself.  We're primarily a Python shop, and rely heavily on the great boto library for most of our AWS interfaction, just like Amazon.  Our basic program flow looks like:

```
while (true):
  read message from SQS queue
  if it's a TERMINATE message and from a spot group:
    find the corresponding demand group
    adjust the demand group instance count by 1  
```

I won't go into how to set up your environment with the correct keys to allow boto access to your AWS account.  Even better, create an IAM role and assign it to the instance you run the daemon on.

Let's read the message off the queue:

```python
MAX_RETRIES = 5

def update_groups_from_queue(queue=None):
    """ Connect to SQS and update config based on notifications """
    env = os.environ
    if ('AWS_ACCESS_KEY_ID' not in env or 'AWS_SECRET_ACCESS_KEY' not in env):
        raise Exception("Environment must have AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY set.")
    _process_queue(queue)


def _process_queue(queue):
    retries = 0
    delay = 5

    sqs = boto.connect_sqs()
    q = sqs.create_queue(queue)
    q.set_message_class(RawMessage)

    while True:
        try:
            rs = q.get_messages()
            for msg in rs:
                if _process_message(msg):
                    q.delete_message(msg)

            time.sleep(10)
            # if we make it here, we're processing correctly, we want to watch for consecutive
            # errors
            retries = 0
            delay = 5
        except SQSError as sqse:
            retries += 1
            log.info("Error in SQS, attempting to retry: retries: %d, exception: %r" % (retries, sqse))
            if retries >= MAX_RETRIES:
                log.error("Maximum SQS retries hit, giving up.")
                raise sqse
            else:
                time.sleep(delay)
                delay *= 1.2     # exponential backoff, similar to tracelon's @retry
```
There's nothing overly complicated here; it's just your basic processing loop.  We pass in the queue name (in our case, spot-scale-demand) from the command line using argparse and feed the name to update_groups_from_queue.  We process the message and sleep for 10 seconds.  We only delete the message if we successfully processed it, so we can investigate broken messages after the fact.  If we encounter an error reading from the queue, we sleep with an exponential backoff, and try again, up to 5 times.  Past that, we assume that our SQS endpoint is having problems, and it'll be likely we're up and monitoring things :)

Now we process the message to determine if we need to scale up our demand instances.
```python
def _process_message(msg):
    """ Process the SQS message. """
    try:
        m = json.loads(msg.get_body())
    except ValueError:
        # Unexpected message (non-json). Throw it away.
        log.error("Could not decode: %s" % (msg.get_body()))
        return True

    # There are 2 different message types in this queue: AWS generated (a result of autoscale activities)
    # and ones we send explicitly.

    msg_type = m['Type']
    if msg_type == 'Notification':
        # From AWS autoscale:
        payload = json.loads(m['Message'])
        spot_group = payload.get('AutoScalingGroupName', '')
        cause = payload.get('Cause', '')

        if not _is_spot_group(spot_group):
            log.info("Received AWS notification for non-spot group %s, ignoring." % spot_group)
            return True

        # usually we'll want to look for 'was taken out of service in response to a system health-check.'
        # if user initiated, will be 'user health-check', which can be triggered via
        # as-set-instance-health i-______ --status Unhealthy.
        # We could ignore this, but it makes testing much easier.
        if not re.search(r'was taken out of service in response to a (system|user) health-check.', cause):
            log.info("Received AWS notification for spot group %s, but not due to health check termination, ignoring." % spot_group)
            return True

        if event == 'autoscaling:EC2_INSTANCE_TERMINATE':
            _adjust_demand_group(spot_group, 1)
        else:
            log.info("Ignoring notification: %s", payload)
```
We grab the message body and load into a JSON object.  The main attributes we care about are:
  * the notification type
  * the name of the autoscaling group
  * the cause of the autoscaling event
We ignore the request if the event came from a group that's not a spot group (more on this next), and also ignore the event unless it's a health check event.  This allows up to manually adjust the capacity of the spot group without firing off launch commands to the demand autoscaling group.  Last, don't do anything unless the event was from an instance termination.

Now we can implement the logic of when to spin up a demand instance.  Now, please don't laugh, because we have some very simple logic right now :)

```python
def _is_spot_group(group_name):
    """" Returns true if the group's name refers to a spot group. """
    # pretty simple, all our spot groups will have spot in the name
    return '-spot-' in group_name
```
Yes, that's how we mark our autoscaling groups as spot groups.  Pretty simple, huh?

We want to make sure that we only spin up instances in the demand group that corresponds to our spot group.
```
def _find_demand_scaling_group(spot_group):
    """ Find the corresponding on-demand group for the given spot group.
        For now, just rely on them being named similarly.
        Returns the boto autoscale group object, or None if no group found. """
    autoscale = boto.connect_autoscale()
    as_groups = autoscale.get_all_groups()

    demand_group = re.sub(r'spot-', r'', spot_group)
    demand_group = re.sub(r'-\d+$', r'', demand_group)
    log.debug("Given spot group %s, find demand group similar to %s" % (spot_group, demand_group))

    result = [ group for group in as_groups
               if demand_group in group.name and not _is_spot_group(group.name) ]
    return sorted(result, key=lambda group: group.name).pop() if result else None

```
Basically, we name the groups the same, append 'spot-' on the spot autoscaling group, and append the date when we deploy the groups to the names of both.  This could be anything you want, like serial numbers, build numbers, and version numbers (similar to how Netflix defines their groups in Asgard).

Finally, we adjust the capacity of the demand autoscaling group.
```
def _adjust_group(group, adjustment):
    """ Change the number of instances in the given boto group by the given amount. """
    try:
        current_capacity = group.desired_capacity
        desired_capacity = current_capacity + adjustment
        if desired_capacity < group.min_size or desired_capacity > group.max_size:
            log.info("Demand group count already at bound, adjust via AWS if necessary.")
            return
        group.desired_capacity = desired_capacity
        group.min_size = desired_capacity
        group.update()
        log.info("Adjusted instance count of ASG %s from %d to %d." %
                 (group.name, current_capacity, desired_capacity))
    except Exception as e:
        log.exception(e)


def _adjust_demand_group(spot_group, adjustment):
    """ Change the number of instances in on-demand ASG paired with the given spot group name
        by the given amount. """

    try:
        demand_group = _find_demand_scaling_group(spot_group)
        if demand_group:
            _adjust_group(demand_group, adjustment)
        else:
            log.error("No demand group found similar to %s." % spot_group)
    except Exception as e:
        log.exception(e)
```
Again, there's nothing very complicated here.  We don't attempt to set the capacity of the demand group below its minimum or maximum.

And that's it!  Throw all this in a python script and run it.  We're big fans of supervisor here, so we set up a supervisor configuration for our script to ensure the daemon stays up.

### TODO: get screen caps from issue to show the scaling working ###

### Improvements ###

So far, this daemon is working out pretty well for us, but we still plan on improving it.
  * We're researching now moving to an autoscaling group per availability zone for our processing nodes to allow even more fine-grained control over how many instances are in each zone, instead of being at the mercy of having the same count in each AZ as it stands now.
  * The main logic of our algorithm is definitely very simple right now.  The astute reader will notice that we never decrease the number of instances in our demand group.  This is good for ensuring that our processing pool never loses a lot of capacity, but it also means our operations team eventually has to adjust the demand group back down to acceptable levels.  My next steps will definitely involve checking the current spot instance requests and scaling the demand group back down once the requests have been fulfilled and the spot group instance count is stable.

If any of this sounds interesting to you, make sure to check us out at dev.appneta.com!
