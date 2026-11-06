# AWS Lambda + DynamoDB + Scheduled S3 Backups

**Project report**

| | |
|---|---|
| **Name** | Asif Istiaque |
| **Project** | S3 / Lambda / DynamoDB / S3 Backup |
| **AWS Region** | US West (Oregon), `us-west-2` |
| **Work done on** | 5 and 6 November 2026 |

---

## 1. What I built

In this project I built a small pipeline on AWS that runs without any server of my own.

It does three things:

1. It gets 10 sample users from a public website (JSONPlaceholder).
2. It saves those users in a DynamoDB table.
3. Every 2 minutes it copies the whole table into a new JSON file in an S3 bucket.

I did everything in the AWS Console, and the code is written in Python. I never put any access key in the code. Each Lambda function gets its permissions from an IAM role.

These are the names I used:

| Part | Name |
|---|---|
| DynamoDB table | `jsonplaceholder-users` (partition key: `id`, type Number) |
| S3 bucket | `asif-lambda-backups-958710` |
| Lambda #1 (import) | `import-jsonplaceholder-users` |
| Lambda #2 (export) | `export-dynamodb-to-s3` |
| Schedule | `dynamodb-backup-every-2min` with `rate(2 minutes)` |
| Lambda memory | 128 MB for both functions |

---

## 2. How it works

```text
   https://jsonplaceholder.typicode.com/users
                     |
                     |  HTTP GET (10 users)
                     v
   [ Lambda #1: import-jsonplaceholder-users ]
                     |
                     |  PutItem
                     v
   [ DynamoDB table: jsonplaceholder-users ]
                     |
                     |  Scan
                     v
   [ Lambda #2: export-dynamodb-to-s3 ] <---- [ EventBridge Scheduler ]
                     |                           rate(2 minutes)
                     |  PutObject
                     v
   [ Private S3 bucket ]  backups/DATE/users-TIME-REQUESTID.json

   Both Lambda functions write their logs to CloudWatch.
```

In simple words:

- **Lambda #1** is the one I start by hand. It calls the public API, gets a list of 10 users, and saves each user in DynamoDB.
- **DynamoDB** keeps the users. Each user is one item, and the `id` number is the key.
- **Lambda #2** reads every item in the table, puts them in one JSON list, and saves that list as a new file in S3.
- **EventBridge Scheduler** is like an alarm clock. Every 2 minutes it starts Lambda #2 for me.
- **S3** keeps all the backup files.
- **CloudWatch** keeps the logs, so I can see what each function did.

---

## 3. Part A: DynamoDB table and S3 bucket

### 3.1 DynamoDB table

I made a table called `jsonplaceholder-users`. The partition key is `id` and its type is **Number**. The capacity mode is **On-demand**, and the table status is **Active**.

![Figure 1: DynamoDB table jsonplaceholder-users is Active with id (Number) as the partition key](screenshots/01-dynamoDbTableCreated.png)

*Figure 1: The new table is Active. The partition key is `id (Number)`. The item count is 0 because I had not run Lambda #1 yet.*

You can see two tables in the list on the left. The first one has a spelling mistake in its name (`jsonlaceholder-users`, the "p" is missing). I noticed this when my first Lambda test gave a `ResourceNotFoundException`, because the name in my code did not match the table. I made a new table with the right name. I deleted both tables at the end (see section 8).

### 3.2 S3 bucket

I made a general purpose bucket called `asif-lambda-backups-958710` in the same region. I kept the default settings, so the bucket is private and Block all public access stays on. There is no need to make a `backups/` folder, because S3 makes the prefix by itself when the first file is saved.

![Figure 2: The S3 bucket was created and is empty](screenshots/02-s3BucketCreated.png)

*Figure 2: The bucket was created successfully. It shows "Objects (0)" because nothing was exported yet.*

---

## 4. Part B: Lambda #1 (import users into DynamoDB)

### 4.1 What the code does

Lambda #1 reads the table name from the environment variable `TABLE_NAME`. Then it:

1. Calls `https://jsonplaceholder.typicode.com/users` using Python's built-in `urllib`.
2. Checks that it got a list of users.
3. Saves every user into DynamoDB with `put_item`. The nested `address` and `company` parts are saved as they are.
4. Prints a log line for each saved user and one final `SUCCESS` line.

### 4.2 Permission

By default a new Lambda can only write logs. It cannot touch DynamoDB. So I added a small inline policy to its role. It allows one action, `dynamodb:PutItem`, and only on my table:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteDemoUsers",
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/jsonplaceholder-users"
    }
  ]
}
```

(`REGION` and `ACCOUNT_ID` are placeholders here. I used my real values in the console.)

After I saved the policy, the role shows it next to the basic logging policy.

![Figure 3: IAM role of Lambda #1 with the inline policy for PutItem](screenshots/17-iam-lambda1-policy.png)

*Figure 3: The execution role of `import-jsonplaceholder-users`. It has the basic logging policy and my inline policy that allows only `dynamodb:PutItem` on the table.*

### 4.3 Upload the code

I set `TABLE_NAME=jsonplaceholder-users` as an environment variable and uploaded `lambda-import-users.zip`. The handler is `lambda_function.lambda_handler`. The console says the function was updated and deployed.

![Figure 4: Lambda #1 code uploaded and deployed](screenshots/03-lamda-zip.png)

*Figure 4: The code of `import-jsonplaceholder-users` after uploading the ZIP. The green message says "Successfully updated the function".*

### 4.4 Test run

I made a test event called `ImportUsersTest` with `{}` as the body, and clicked Test. It worked. The function took 441.73 ms and used 93 MB of the 128 MB I gave it. The log output shows each user being saved.

![Figure 5: Successful test of Lambda #1](screenshots/04-lambda1-test-success.png)

*Figure 5: The test passed. The log shows "Saved user 1: Leanne Graham", "Saved user 2: Ervin Howell", and so on.*

### 4.5 CloudWatch logs

In CloudWatch, the log group `/aws/lambda/import-jsonplaceholder-users` has all the messages from the code, from user 1 to user 10, and then the final line `SUCCESS: Saved 10 users to DynamoDB table jsonplaceholder-users`.

![Figure 6: CloudWatch log events for Lambda #1](screenshots/05-lambda-cloudwatch-logs.png)

*Figure 6: The log has two runs of the function. The first one ended at 15:51:28 and the second started at 15:51:46 (UTC). Both saved 10 users.*

### 4.6 Data in DynamoDB

I opened **Explore table items** and ran a scan. It returned exactly 10 items. Each item has `id`, `address`, `company`, `email`, `name`, `phone`, `username` and `website`.

![Figure 7: 10 items in the DynamoDB table](screenshots/06-dynamoDb-items.png)

*Figure 7: "Items returned: 10". The scan was done after Lambda #1 had run two times (see Figure 6), and the table still has only 10 items. I explain why in question 2 below.*

---

## 5. Part C: Lambda #2 (export DynamoDB to S3)

### 5.1 What the code does

Lambda #2 uses three environment variables:

| Variable | Value |
|---|---|
| `TABLE_NAME` | `jsonplaceholder-users` |
| `BUCKET_NAME` | `asif-lambda-backups-958710` |
| `BACKUP_PREFIX` | `backups` |

Then it:

1. Scans the table. If the table is big, it keeps reading page after page until there is nothing left.
2. Sorts the items by `id`.
3. Turns DynamoDB's `Decimal` numbers into normal JSON numbers.
4. Saves everything as one JSON file in S3 with a new name each time:
   `backups/YYYY-MM-DD/users-HH-MM-SS-microseconds-REQUESTID.json`

### 5.2 Permission

This function needs more than Lambda #1, but still only two things. It can read the table with `Scan`, and it can write files into the `backups/` part of my bucket with `PutObject`. It cannot delete anything and it cannot change the table.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadDemoUsers",
      "Effect": "Allow",
      "Action": "dynamodb:Scan",
      "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/jsonplaceholder-users"
    },
    {
      "Sid": "WriteTimestampedJsonBackups",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/backups/*"
    }
  ]
}
```

Here is the role of Lambda #2 after I added the policy.

![Figure 8: IAM role of Lambda #2 with the inline policy for Scan and PutObject](screenshots/18-iam-lambda2-policy.png)

*Figure 8: The execution role of `export-dynamodb-to-s3`. It has the basic logging policy and my inline policy that allows `dynamodb:Scan` on the table and `s3:PutObject` on `backups/*` in the bucket.*

### 5.3 Upload the code

I uploaded `lambda-backup-to-s3.zip` to the function `export-dynamodb-to-s3` and deployed it.

![Figure 9: Lambda #2 code uploaded and deployed](screenshots/07-lambda-backuo-to-s3.png)

*Figure 9: The code of `export-dynamodb-to-s3`. The top of the file lists the environment variables and the IAM actions it needs.*

### 5.4 Test run

I made a test event called `ExportUsersTest` with `{}` and ran it. The result:

- `statusCode`: 200
- `exported_count`: 10
- `scan_pages`: 1
- the file was saved at `s3://asif-lambda-backups-958710/backups/2026-11-05/users-16-38-34-024727-7b2f9dfc-5e74-4029-9e46-1de17a8ba204.json`

The log also says `Read 10 items over 1 DynamoDB scan page(s)` and then `SUCCESS: Exported 10 users`.

![Figure 10: Successful manual test of Lambda #2](screenshots/08-lambda2Backup-test_success.png)

*Figure 10: The manual test passed. The response shows 10 exported users and the S3 address of the new file.*

### 5.5 Looking inside a backup file

I downloaded one of the files from S3 and opened it. It is a JSON list with 10 users, sorted by `id`. Each user has `name`, `username`, `email`, `phone`, `website`, and the nested `address` and `company` parts.

![Figure 11: Content of one exported JSON file](screenshots/19-s3-json-content.png)

*Figure 11: Inside one backup file. It looks the same as the data in DynamoDB.*

---

## 6. Part D: Run it automatically every 2 minutes

### 6.1 The schedule

In EventBridge Scheduler I made a schedule with these settings:

- Name: `dynamodb-backup-every-2min`
- Group: `default`
- Type: recurring, rate-based, `rate(2 minutes)`
- Target: AWS Lambda Invoke, function `export-dynamodb-to-s3`
- Payload: `{}`
- Role: a new role made just for this schedule

![Figure 12: EventBridge schedule is Enabled with rate(2 minutes)](screenshots/09-eventbridge-schedule.png)

*Figure 12: The schedule detail page. Status is Enabled and the rate is `rate(2 minutes)`.*

### 6.2 Proof that it ran by itself

After a few minutes I opened the S3 bucket. In the folder `backups/2026-11-05/` there were 6 files, each about 5.5 KB.

- The first file (22:38:35) is from my manual test in Figure 10.
- The other five (22:51:58, 22:53:49, 22:55:49, 22:57:49 and 22:59:49) were made by the schedule. After the first one, they are exactly 2 minutes apart.

![Figure 13: Six JSON files in S3 under backups/2026-11-05/](screenshots/10-s3-auto-export.png)

*Figure 13: Every file has a different name, because the name has the time and the request ID in it. The S3 console shows time in UTC+06:00.*

The CloudWatch log for Lambda #2 tells the same story. Each run has its own request ID, and each one reads 10 items and exports 10 users.

![Figure 14: CloudWatch log events for Lambda #2](screenshots/13-lambda2_cloudwatch_logs.png)

*Figure 14: Runs at 16:51:56, 16:53:48 and 16:55:48 (UTC) with the lines "Reading DynamoDB table", "Read 10 items over 1 DynamoDB scan page(s)" and "SUCCESS: Exported 10 users to s3://...". CloudWatch shows UTC, and 16:51 UTC is the same time as 22:51 in Bangladesh.*

---

## 7. Logs and metrics

### 7.1 Logs

I used CloudWatch Logs for both functions (Figure 6 and Figure 14). Every run has two kinds of lines:

- `START`, `END` and `REPORT` lines that Lambda writes by itself. `REPORT` shows how long the run took and how much memory it used.
- The messages from my own code, like `Saved user 1: Leanne Graham` and `SUCCESS: Exported 10 users`.

### 7.2 Metrics

In **Monitor → Metrics** of each function I looked at `Invocations`, `Errors` and `Duration`.

![Figure 15: Metrics of Lambda #1](screenshots/20-lambda1-metrics.png)

*Figure 15: Metrics of `import-jsonplaceholder-users`. This function only ran when I started it by hand.*

![Figure 16: Metrics of Lambda #2](screenshots/21-lambda2-metrics.png)

*Figure 16: Metrics of `export-dynamodb-to-s3`. The Invocations graph goes up with every run of the schedule.*

Logs and metrics are not the same thing. I explain the difference in question 4 below.

---

## 8. Part E: Clean up

The schedule makes a new file every 2 minutes. That is about 720 files a day, so I did not want to leave it on. As soon as I had my proof, I turned everything off and deleted it.

1. **Disabled the schedule.** The console says the schedule has been disabled and the status is Disabled.

   ![Figure 17: Schedule disabled](screenshots/11-eventbridge-disabled.png)

   *Figure 17: `dynamodb-backup-every-2min` is Disabled.*

2. **Deleted the schedule.** The list is empty now.

   ![Figure 18: Schedule deleted](screenshots/12-eventbridge_cleanup.png)

   *Figure 18: "Schedule dynamodb-backup-every-2min has been deleted successfully". Schedules (0).*

3. **Deleted the DynamoDB tables.** Both the correct table and the one with the spelling mistake are gone.

   ![Figure 19: DynamoDB tables deleted](screenshots/14-dynamoDb-table-ckeanup.png)

   *Figure 19: The delete request for `jsonplaceholder-users` and `jsonlaceholder-users` was accepted. Tables (0).*

4. **Deleted the two Lambda functions.** The console says "Successfully deleted 2 functions". The one function still in the list, `fetch-jsonplaceholder-users`, is from my earlier practice and is not part of this assignment.

   ![Figure 20: Lambda functions deleted](screenshots/15-lamda-clean-up.png)

   *Figure 20: Both assignment functions are deleted.*

5. **Emptied the S3 bucket.** 9 files (49.6 KB) were deleted and 0 failed.

   ![Figure 21: S3 bucket emptied](screenshots/16-s3-bucket-cleanup.png)

   *Figure 21: "Successfully emptied bucket". 9 objects deleted, 0 failed.*

---

## 9. Problems I ran into

- **Wrong table name.** I first made a table called `jsonlaceholder-users` by mistake. My Lambda looked for `jsonplaceholder-users` and gave `ResourceNotFoundException: Requested resource not found`. A table name cannot be changed, so I made a new table with the right name. Lesson: the name in the code, the environment variable and the table must match letter by letter.
- **Item count showed 0.** Right after I made the table (Figure 1) the item count was 0. DynamoDB only updates this number about every six hours, so for a quick check it is better to use **Explore table items**, like in Figure 7.
- **Two ZIP files.** The assignment folder has a big outer ZIP and two small ZIPs inside `lambda_zips/`. Only the small ones go to Lambda.

---

## 10. Questions

### 1. What is the difference between Lambda's execution role and Scheduler's execution role?

They are two different roles for two different jobs.

- **Lambda's execution role** is the identity my function uses while it is running. It decides what my code is allowed to do. For Lambda #1 that is writing logs and `PutItem` on the table. For Lambda #2 it is writing logs, `Scan` on the table and `PutObject` on the bucket.
- **Scheduler's execution role** is the identity EventBridge Scheduler uses to start my function. It only needs the permission to invoke the Lambda function (`lambda:InvokeFunction`). Scheduler does not run my code and does not touch DynamoDB or S3.

So one role says "what can the function do", and the other says "who is allowed to wake the function up".

### 2. Why does running Lambda #1 twice not give 20 users in DynamoDB?

Because `PutItem` does not add a new row each time. In DynamoDB the `id` is the key, and each key can exist only once. If I write an item with an `id` that is already in the table, DynamoDB replaces the old item with the new one. User 1 is overwritten with user 1 again, so the table stays at 10 items. Figure 6 shows two runs, and Figure 7 shows 10 items.

### 3. Why do Lambda #2 backups have unique timestamps and request IDs?

So no file is overwritten. In S3, if two files have the same name, the new one silently replaces the old one. The file name has the UTC date, the time down to microseconds, and the Lambda request ID, so every run makes a brand new file. This way I keep a history of backups, and I can look at what the table had at a certain time. Figure 13 shows six files with six different names.

### 4. What is the difference between `print()` logs and Invocations / Errors metrics?

- **Logs** are the detailed story of one run. My `print()` lines show what happened, for example `Saved user 3: Clementine Bauch`. Lambda also adds `START`, `END` and `REPORT` lines. I read logs when I want to know why something happened.
- **Metrics** are numbers over time, like `Invocations` (how many times the function ran), `Errors` (how many runs failed) and `Duration` (how long they took). I look at metrics when I want to see the big picture, for example "did the schedule run every 2 minutes, and did anything fail?".

The names of the users are only in the logs. They are not metrics.

### 5. What if I delete the DynamoDB table but keep the JSON files in S3? Can the indexes and settings be restored from the JSON alone?

I could get the **data** back. The JSON files have all the items, so I could make a new table and write the items in again with a small script. But the table's **settings** would not come back by themselves, because the JSON only has items. These things are not in the files:

- indexes (for example global secondary indexes)
- capacity mode and capacity settings
- alarms
- point-in-time recovery setting
- access policies and tags

I would have to remember them and set them up again by hand. Also, the file is only a copy from one moment, so anything changed after that moment is lost.

### 6. Why is an export every 2 minutes usually a poor production backup strategy for large DynamoDB tables?

- **It costs a lot.** Every export scans the whole table, and every item that is read uses read capacity and money. Even my tiny 10-item scan in Figure 7 used 2 read units. A big table would use a huge amount, 30 times an hour.
- **It can fail.** A big table may not finish inside the Lambda time limit or may run out of memory.
- **It is not a real snapshot.** A scan reads items little by little. If the data changes while it is reading, the file can have a mix of old and new values.
- **Too many files.** It makes about 720 files a day, about 21,600 in 30 days. That means more S3 storage and request costs, and it is hard to find the right backup.
- **It saves only items.** It does not save indexes or settings (see question 5).

For real systems, AWS has better tools: **Point-in-Time Recovery (PITR)** and the managed **DynamoDB Export to S3**. They take proper backups, do not slow down the table the way a scan does, and are made for restoring. The backup time and how long to keep backups should be chosen from how much data loss and downtime the business can accept (RPO and RTO).

---

## 11. Conclusion

I built the full pipeline and it worked from start to end:

- one API call gave 10 users,
- Lambda #1 saved them in DynamoDB,
- Lambda #2 exported the table as a JSON file to S3, first by hand and then by itself every 2 minutes,
- CloudWatch showed every run in the logs,
- and I deleted all the resources at the end so there are no extra charges.

The most useful things I learned were how IAM roles give a function only the permissions it needs, why `PutItem` replaces items with the same key, and why a simple scan-and-save export is good for learning but not a good backup plan for a real production database.
