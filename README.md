##### Forked from:
https://aws.amazon.com/blogs/compute/creating-an-enterprise-scheduler-using-aws-lambda-and-tagging/

## Purpose:
Run as a AWS Lamda scheduler to power on/off EC2 instances based on a TAG that defines the schedule. 

## How to Schedule

 To enable the lk-ec2-scheduler, a Lambda function is added and run on a schedule. The function describes EC2 
 instances within the same region, and determines if there are any On or Off operations to be executed. 
 The recommend configuration is to run the function every 20 minutes. By default, the function determines On/Off actions 
 from the last 22 minutes.
 
 Controlling this behavior is done in two parts:

* The Lambda function schedule
* The max_delta parameter

The function schedule controls how often the function is invoked and begins to search for On/Off actions to perform. 

The max_delta parameter controls the time window in which to search for individual schedule tags on resources. For 
example, you may choose to run the function every 20 minutes, and then set max_delta to 22 minutes. It is necessary to 
keep the max_delta value slightly higher than the rate which the function is invoked so as not to miss any On/Off 
actions. We recommend setting it two minutes above the function invocation rate.

## Required IAM permissions

The Lambda function requires permissions to query the resource tags using DescribeInstances, and then to act on them, 
with either a StartInstances or StopInstances API operation call.  This is accomplished via a IAM Role assigned to the
Lambda function. See the IAM Role section below on how to create and assign the role. 


## Create the Lambda function

* Open the Lambda console and choose Create a Lambda function.
* Choose Next to skip a blueprint selection.
* Choose Next to skip creation of a trigger at this time.
* Enter a function name and note it for later use. For example: lk-ec2_scheduler_function.
* For Runtime, choose Python 2.7.
* For Code entry type, choose Edit code inline.
* Paste the function code from the file lk-ec2_scheduler.py.
* For Handler, enter lk-ec2_scheduler.lambda_handler .
* For Role selection, choose Create a custom role.

A window pops up displaying the IAM form for creating a new IAM execution role for Lambda.

#### Create/Add an IAM role
* In the IAM window, for Role name, enter ent_scheduler_execution_role (or similar text)
* Choose View Policy Document, Edit the policy, OK (to confirm that you've read the documentation)
* Replace the contents of the default policy with the contents from the lk-scheduler_role.json file. 
* Choose Allow to save the policy and close the window.
* For Timeout, enter 5 minutes.
* Choose Next.
* Review the configuration and choose Create Function.  

## Add an event schedule

* Choose Triggers, Add trigger.
* Click on the box and choose CloudWatch Events – Schedule.
* For Schedule expression, enter the rate at which you would like the lk-ec2-scheduler to be invoked. Recommended value of "rate(20 minutes)".

For example use `cron(0/20 * * * ? *)` to run the lk-scheduler task every 20 minutes. 

At this point, the lk-ec2-scheduler will run every 20 minutes within the region where the Lambda function is running.  The function finds an instance with the associated tag, it will evaluate the schedule string in the tag and either turned on or off the instance, based on its schedule.

## Enabling the scheduler on a resource
Assigning a schedule to a resource using a tag requires following these guidelines:

* The resource tag must match the tag named in the Lambda function. The default Tag used by the function is: `lk-EntSched`
* The scheduler works on a weekly basis (7 days of the week), and you may specify up to 1 set hour for turning the resource On or Off. For each day that's set in the On or Off section, the same hour is used.
* The current time is interpreted by the function using UTC, therefor the schedule tag should use the UTC time for turning instances both On and Off.
* The scheduler syntax is comprised of 5 tokens separated by a semicolon. Use the following scheduler syntax in the Lambda function:

  `Days-for-On;Hour-for-On;Days-for-Off;Hour-for-off;Optional-Disable;`
  
* The Days values should be noted with the first 3 letters of the day. Accepted values are: mon, tue, wed, thu, fri, sat, sun. Multiple days can be specified with comma separation and order is ignored.
* The Hour value should be noted in 24H format HHMM.
* Optional-Disable states that this tag will be ignored by the function, allowing the user to keep the configuration and have the scheduler skip over this resource temporarily. 

## Example TAG syntax: 

Note: CloudWatch/Lambda Schedule times are in UTC! If you wish to schedule a EC2 instance on at 8 AM PST (local time) you must convert the local time to UTC. 

The default Tag used by the function is: `lk-EntSched`

`mon,tue,wed,thu,fri,sat,sun;0830;mon,tue,wed,thu,fri,sat,sun;1700;` = Resource would be turned on at 8:30 am UTC daily and turned off at 5:00 pm UTC daily.

`mon;0700;fri;1800;`= Resource would be turned on at 7:00 am on Mondays and turned off at 6:00 pm UTC on Fridays.

`wed,thu,fri;2100;thu,fri,sat;0500;`= Resource would be turned on at 9:00 pm UTC on Wednesdays, Thursdays, and Fridays, and turned off at 5:00 am UTC on Thursdays, Fridays, and Saturdays.

`wed,thu,fri;2100;thu,fri,sat;0500;disable;`= Resource would be left untouched and the schedule ignored due to the disable option.

`mon,tue,wed,thu,fri;1600;tue,wed,thu,fri,sat;0100;` = Resource would be turned on at 8 AM PST Mon - Friday and turned off 
Mon-Fri at 5 PM (1700) PST. 


## Assign the lk-EntSched Tag to the instance(s)

1. Determine which instances you need to scheduled
2. Create the scheduler string using the examples above.  
3. Create and assign a new tag named: `lk-EntSched` with the assigned schedule.
4. Monitor the CloudWatch Logs for actions on your scheduled instances.  
