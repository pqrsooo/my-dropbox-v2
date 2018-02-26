# myDropbox V2

[![standard-readme compliant](https://img.shields.io/badge/readme%20style-standard-brightgreen.svg?style=flat-square)](https://github.com/RichardLitt/standard-readme)

## Table of Contents

- [Quickstart](#quickstart)
	- [Amazon Web Service](#amazon-web-service)
  - [Your Local Machine](#your-local-machine)
- [Design](#design)
  - [File Hierarchy in S3](#file-hierarchy-in-s3)
  - [A User Table in Amazon DynamoDB (myDropboxUsers)](#a-user-table-in-amazon-dynamodb-mydropboxusers)
  - [A File Table in Amazon DynamoDB (myDropboxFiles)](#a-file-table-in-amazon-dynamodb-mydropboxfiles)
- [License](#license)

## Quickstart

### Amazon Web Service

1. Create a table in Amazon DynamoDB named `myDropboxUsers` and set `username` attribute as a primary partition key. An example instance of `myDropboxUsers` table is shown as follows:
    
    | username | password | uid |
    |----------------|-------------------------------------------------------------------------------------------|----------------------------------|
    | bob@mail.com | ao8NoBvvr9grT66kmO7lVNA3RnNLbYrZjOaghnUfs58=$ZOuMkNhtu8nMkJRS/BpJ5ybAmHBXS3JuQQaUazVhNB0= | 97C712AA60976209AE6D1C741B1338D3 |
    | alice@mail.com | NAc+HYrDudwQaZcRg817PRxqjF/3GEo0kHayPOwL5ZU=$QFtw0X5aHxPmItmB6l+DwMM7pAo0/qN7xplbFawUxHc= | 7BAE794AE414A192DA370A24B80CD55A |
    | john@mail.com | u9xLkE4bgNAAs16SCJAiHazrW6agDCPu8UacX6sYnAU=$tm2cWwCaiboKG6oM1pwdbuoexkwiwYH0Kl65gcuOaXM= | 4CF38BD6AC1546139696F852BB3625CA |

2. Create a table in Amazon DynamoDB named `myDropboxFiles` and set `key_name` attribute as a primary partition key. An example instance of `myDropboxFiles` table is shown as follows:

    | key_name | version_id | owner | shared_by | file_size | last_modified_time |
    |-----------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------|-----------|--------------------|
    | 4CF3…/star.eps | yZTgZfau3R2bdPVFTVevz2WkgoRHKP_R | 4CF38BD6AC1546139696F852BB3625CA | {“7BAE794AE414A192DA370A24B80CD55A"} | 3179 | 1519644167000 |
    | 7BAE…/gcp-icons.zip | XDos_hNudKjfyUHeFdMvMJNLS47RR5cL | 7BAE794AE414A192DA370A24B80CD55A | {"4CF38BD6AC1546139696F852BB3625CA", "97C712AA60976209AE6D1C741B1338D3"} | 787303 | 1519664332000 |
    | 7BAE…/Toey-t2 (1024Hz).json | K8AoGIL7BOQLsnCm4P1dvkop.966rQOW | 7BAE794AE414A192DA370A24B80CD55A |  | 1281 | 1519664369000 |

3. In Amazon S3, create a new bucket for myDropbox's file storage. As a bucket's name has to be globally unique across Amazon Web Service, it cannot be as same as mine. So, name it whatever you want but make sure that a **versioning property** is enabled.

4. Create an IAM User with the following policies:
    - `AmazonS3FullAccess`
    - `AmazonDynamoDBFullAccess`
    
    After creating, you will get a key ID and secret.

5. Create or update your local AWS `credential` file with the key ID and secret you have got from the previous step.
    ```
    [default]
    aws_access_key_id = [AWS_ACCESS_KEY_ID]
    aws_secret_access_key = [AWS_SECRET_ACCESS_KEY]
    ```

### Your Local Machine

1. Clone this repository.
2. Import this project to your preferable Java IDE.
3. In `src/myDropbox_v2_5730329521/myDropbox_v2_5730329521.java` (line 48), change a value of `bucketName` variable to be your S3 bucket's name.
4. Build and run.

## Design

### File Hierarchy in S3
The Amazon S3 data model is a flat structure. There is no hierarchy of subfolders; however, you can infer logical hierarchy using key name prefixes and delimiters as the Amazon S3 console does.

So, to separate user’s files, every object’s key name is prefixed with a UID of the user it belongs to.
This allows users to store their files in the same S3 bucket without collision even two users uploaded files with the same name.

![File Hierarchy in S3](readme/images/file-hierarchy-in-s3.png)

### A User Table in Amazon DynamoDB (myDropboxUsers)

![Instance of myDropboxUsers Table](readme/images/my-dropbox-users-table.png)

In Amazon DynamoDB aspect, this table consists of 3 attributes: `username` (String) as a primary partition key, `password` (String), and `uid` (String).

A user’s UID is generated from his/her username with the MD5 hashing algorithm. However, there is still little chance that two different strings can have exact MD5 hash value. To avoid this collision, we re-generated new hash value from the previous one till it is unique across all users in the database.

For a security’s purpose, a password is encrypted by the salted PBKDF2 hashing algorithm and is kept in the form of `salt$hashedPassword`.

As there is no update, reading this table is eventually consistent.

### A File Table in Amazon DynamoDB (myDropboxFiles)

![Instance of myDropboxFiles Table](readme/images/my-dropbox-files-table.png)

There are 6 attributes: `key_name` (String), `version_id` (String), `owner` (String), `shared_by` (StringSet), `file_size` (Number), and `last_modified_time` (Number).

After a file is successfully uploaded to S3, its key name along with metadata are also added to myDropboxFiles table. However, if a file is successfully uploaded to S3, but an addition of the file record to myDropboxFiles table is unsuccessful, a rolling back mechanism is triggered by removing that file in S3 as it has never be uploaded.

When a file is shared with another user, UID of that user is added to the `shared_by` set.

All commands use eventually consistent reads, except `get` command which uses strongly consistent reads to make sure that a user will always receive a file with the latest version.

## License

[MIT](LICENSE) © Parinthorn Saithong