# Serverless-Architecture-assignment
Serverless-Architecture-assignment

Uploading all the Boto3 Scripts for all the Task Created.
The PDF document has full information along with screenshot for the given task. Please go through and Code is updated here

> TASK-1 Code is here:
import boto3
def lambda_handler(event, context):
    # Create an EC2 client
    ec2 = boto3.client("ec2")

    # ***** Stopping the instances using tags *****

    # Define the tag filter
    tag_filter_stop = [{'Name': 'tag:State', 'Values': ['Auto_Stop']}]

    # Get a list of all instances matching the tag filter
    response1 = ec2.describe_instances(Filters=tag_filter_stop)

    # Check if there are any reservations in the response
    if 'Reservations' in response1:
        for reservation in response1['Reservations']:
            for instance in reservation['Instances']:
                instance_id = instance['InstanceId']
                ec2.stop_instances(InstanceIds=[instance_id])
                print(f"Stopped instance with ID: {instance_id}")
             # Print a success message
            print("Successfully stopped all instances matching the tag filter")
    else:
        print("No instances matching the tag filter found for stopping")

    # ***** Starting the instances using tags *****

    # Define the tag filter
    tag_filter_start = [{'Name': 'tag:State', 'Values': ['Auto_Start']}]

    # Get a list of all instances matching the tag filter
    response2 = ec2.describe_instances(Filters=tag_filter_start)

    # Check if there are any reservations in the response
    if 'Reservations' in response2:
        for reservation in response2['Reservations']:
            for instance in reservation['Instances']:
                instance_id = instance['InstanceId']
                ec2.start_instances(InstanceIds=[instance_id])
                print(f"Started instance with ID: {instance_id}")
            # Print a success message
            print("Successfully started all instances matching the tag filter")
    else:
        print("No instances matching the tag filter found for starting")


> TASK-2 Code is Updated Here

import boto3
from datetime import datetime, timedelta, timezone

def lambda_handler(event, context):
    # Replace 'your-bucket-name' with your S3 bucket name
    bucket_name = 'task-18-tilak'
    
    # Replace the number of days as needed
    days_to_keep = 1
    
    s3 = boto3.client('s3')
    
    # Calculate the date threshold for file deletion and make it offset-aware
    threshold_date = datetime.now(timezone.utc) - timedelta(days=days_to_keep)
    
    # List objects in the bucket
    objects = s3.list_objects_v2(Bucket=bucket_name)['Contents']
    deleted_file_names = []
    
    # Iterate through objects and delete if older than the threshold date
    for obj in objects:
        last_modified = obj['LastModified'].astimezone(timezone.utc)
        if last_modified < threshold_date:
            s3.delete_object(Bucket=bucket_name, Key=obj['Key'])
            deleted_file_names.append(obj['Key'])
            print(f"Deleted: {obj['Key']}")
            
    # Save the list of deleted file names to a text file
    deleted_file_names = "\n".join(deleted_file_names)
    s3.put_object(Bucket=bucket_name, Key='deleted_files.txt', Body=deleted_file_names)
    
    return {
        'statusCode': 200,
        'body': 'Old files deleted and list of deleted files saved successfully'
    }

> TASK-18 Code is Here

import boto3
import time
import datetime

def lambda_handler(event, context):
    # Define your EC2 client
    ec2_client = boto3.client('ec2')
    
    # Specify your snapshot and instance properties
    #snapshot_id = 'your-snapshot-id'  # Replace with your snapshot ID
    instance_type = 't3.micro'
    key_name = 'tilak-keypair'
    security_group_ids = ['sg-03a0a98fe163a64bb']
    subnet_id = 'subnet-0ea185273ead71a27'
    instance_name = 'MyEC2Instance'

    # Get the latest snapshot for the root volume in your region and account
    snapshots = ec2_client.describe_snapshots(
        Filters=[
            {'Name': 'volume-id', 'Values': ['vol-0968b088a5ed88dd2']},
            {'Name': 'status', 'Values': ['completed']}
        ],
        OwnerIds=['295397358094'],
    )['Snapshots']

    # Sort the snapshots by their start time in descending order to get the latest
    snapshots.sort(key=lambda x: x['StartTime'], reverse=True)
    
    if not snapshots:
        raise Exception("No valid snapshots found for the root volume.")

    latest_snapshot_id = snapshots[0]['SnapshotId']

    # Create a custom AMI from the desired snapshot
    AMI_Name = datetime.datetime.now().strftime("%Y-%m-%d-%H-%M-%S")
    response = ec2_client.create_image(
        InstanceId='i-0635c371d626497df',  # Replace with your instance ID
        Name=AMI_Name,
        Description='Custom AMI based on a specific snapshot',
        NoReboot=True,  # You can specify whether to reboot the instance or not
        BlockDeviceMappings=[
            {
                'DeviceName': '/dev/sda1',
                'Ebs': {
                    'SnapshotId': latest_snapshot_id,
                    'VolumeSize': 8,  # Specify the size of the root volume
                    'DeleteOnTermination': True
                }
            }
        ]
    )
    
    custom_ami_id = response['ImageId']
    # Wait for the custom AMI to be in the "available" state
    while True:
        ami = ec2_client.describe_images(ImageIds=[custom_ami_id])['Images'][0]
        if ami['State'] == 'available':
            break
        time.sleep(5)  # Sleep for 5 seconds before checking again

    # Launch an EC2 instance using the custom AMI
    response = ec2_client.run_instances(
        ImageId=custom_ami_id,
        InstanceType=instance_type,
        KeyName=key_name,
        MaxCount=1,
        MinCount=1,
        SecurityGroupIds=security_group_ids,
        SubnetId=subnet_id,
        TagSpecifications=[
            {
                'ResourceType': 'instance',
                'Tags': [
                    {'Key': 'Name', 'Value': instance_name},
                ]
            }
        ]
    )

    instance_id = response['Instances'][0]['InstanceId']

    return {
        'statusCode': 200,
        'body': f'EC2 instance {instance_id} created using custom AMI {custom_ami_id}.'
    }


> Task-19 Code here

Code: import boto3
import time
import datetime
 
def lambda_handler(event, context):
    # Create an EC2 client
    ec2 = boto3.client('ec2')
 
    # Replace 'your_instance_id' with the ID of the EC2 instance you want to check
    instance_id = 'i-0239585eb7553e292'
 
    try:
        # Use the describe_instances() method to get information about the instance
        response = ec2.describe_instances(InstanceIds=[instance_id])
 
        if response['Reservations']:
            instance = response['Reservations'][0]['Instances'][0]
            instance_state = instance['State']['Name']
            result = f'Instance {instance_id} is {instance_state}'
        else:
            result = f'Instance {instance_id} not found.'
    except Exception as e:
        result = f'Error: {str(e)}'
    
    # Create a unique file name with a timestamp
    timestamp = datetime.datetime.now().strftime("%Y-%m-%d-%H-%M-%S")
    file_name = f"ec2_instance_status_{timestamp}.txt"
 
    # Save the result in a file
    with open(f"/tmp/{file_name}", "w") as file:
        file.write(result)
 
    # Upload the file to an S3 bucket
    try:
        s3 = boto3.client('s3')
        bucket_name = 'task-18-tilak'  # Replace with your S3 bucket name
        s3.upload_file(f"/tmp/{file_name}", bucket_name, file_name)
        result += f' - File uploaded to {bucket_name}/{file_name}'
    except Exception as e:
        result += f' - Error uploading to S3: {str(e)}'
 
    return result
