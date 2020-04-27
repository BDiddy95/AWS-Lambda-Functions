
Assume you wish to take a backup of one of your DynamoDB tables each day to an S3 bucket. A simple way to achieve this is to use an Amazon CloudWatch Events rule to trigger an AWS Lambda function daily.

Setting this up requires configuring an IAM role, setting a CloudWatch rule, and creating a Lambda function.

Create Function.
Author from scratch.
Name: [InsertName]
Runtime: Python 3.7
Role: Create a custom role
View policy document.
Edit.

Paste in this policy

(Do not include the border created from equal signs)

=========================================================================================
{
  "Version": "2012-10-17",
  "Statement": [{
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Action": [
        "dynamodb:CreateBackup",
        "dynamodb:DeleteBackup",
        "dynamodb:ListBackups"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
=========================================================================================

Navigate to the Lambda dashboard

Paste this function in the code editor
(Do not include the border created from equal signs)

===========================================================================================
from datetime import datetime

import boto3

MAX_BACKUPS = 3  # maximum number of backups to retain

dynamo = boto3.client('dynamodb')


def lambda_handler(event, context):
    if 'TableName' not in event:
        raise Exception("No table name specified.")
    table_name = event['TableName']

    create_backup(table_name)
    delete_old_backups(table_name)


def create_backup(table_name):
    print("Backing up table:", table_name)
    backup_name = table_name + '-' + datetime.now().strftime('%Y%m%d%H%M%S')

    response = dynamo.create_backup(
        TableName=table_name, BackupName=backup_name)

    print(f"Created backup {response['BackupDetails']['BackupName']}")


def delete_old_backups(table_name):
    print("Deleting old backups for table:", table_name)

    backups = dynamo.list_backups(TableName=table_name)

    backup_count = len(backups['BackupSummaries'])
    print('Total backup count:', backup_count)

    if backup_count <= MAX_BACKUPS:
        print("No stale backups. Exiting.")
        return

    # Backups in date descending order (newest to oldest)
    sorted_list = sorted(backups['BackupSummaries'],
                         key=lambda k: k['BackupCreationDateTime'], reverse=True)

    old_backups = sorted_list[MAX_BACKUPS:]

    print(f'Old backups: {old_backups}')

    for backup in old_backups:
        arn = backup['BackupArn']
        print("ARN to delete: " + arn)
        deleted_arn = dynamo.delete_backup(BackupArn=arn)
        backup_name = deleted_arn['BackupDescription']['BackupDetails']['BackupName']
        status = deleted_arn['BackupDescription']['BackupDetails']['BackupStatus']
        print(f'BackupName: {backup_name}, Status: {status}')

    return


if __name__ == "__main__":
    event = {"TableName": "Movies"}
    lambda_handler(event, {})
============================================================================================

On the Configure test event page, type an event name, and set the content to {"TableName": "Person"}, and choose Create.

Choose Test. It should show a successful execution result.
Open the DynamoDB console, and choose Backups. The Backups page shows you the backup that you took using the Lambda function.

Schedule event to run at the desired interval (e.g., every 1 minute), triggering the Lambda function.

Wait for the CloudWatch rule to trigger the next backup job you have scheduled.
Verify the scheduled backup job ran using CloudWatch Logs.
Verify the backup file exists in the list of DynamoDB backups.
Verify old backups were deleted for subsequent runs. The Lambda function will retain the three most recent backups.