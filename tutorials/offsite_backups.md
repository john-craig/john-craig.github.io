---
title: Setting up Offsite Backups on Low-Power Hosts
---
# Tutorial: Setting up Offsite Backups on Low-Power Hosts
## Introduction
How can one set up secure off-site backups to a hosting solution while using a host with storage and processing constraints, such as a Raspberry Pi 4?

In this tutorial we will take a look at how to use Unix pipes to create compressed, encrypted off-site backups to an s3-compatible storage service, such as BackBlaze.

## Instructions
**Pre-Requisites**
For this tutotial, we will be using `tar`, `gnupg`, and `s3cmd`. These should all be installed and available in your PATH. 

**Compression**
The first step in our process should be compressing the file. This is because the following steps-- encrypting the backups and transferring them-- are both made easier when working with a single file.

For this we can simply use `tar`. If your backup is located at `/srv/backup`, the first part of our command looks like so:

```
tar -czf - /srv/backup
```

Note the `-` as the destination argument. That will be used to send the output of the command to stdout, so that it can be sent to the next command in our pipe.

**Encryption**
For encryption we will use `gnupg`. Our first step will be to generate a key pair to use for the encryption. 

To do this we will use the command `gpg --full-gen-key`. After issuing this command, you will be given a series of prompts. These should be left as the default values, unless you know what you are doing. 

When asked to provide a "Real name" for the user ID of the key, specify a suitable title, such as `offline-backup`. The "Email address" and "Comment" prompts may be filled in as you chose.

Once we have generated our key, we are ready to introduce the next command in our pipe:
```
gpg --encrypt --recipient offsite-backup
```

Because we have not specified arguments for input and output, this command will implicitly read from stdin and output to stdout.

**Transferring**
For the transfer, we will be using `s3cmd`. To start off with, you will need to configure the credentials of the backup provider you are using. This can be initiated using `s3cmd --configure`, which will show you a series of prompts.

Most providers have dedicated pages containing instructions for how to set up credentials for their services with `s3cmd`, or with a similar tool such as the AWS CLI. These are left as an exercise for the reader.

Once these credentials have been configured, we are ready for the final part of our command. Supposing we have created a bucket with the name `my-offsite-backups`, our command would look like:
```
s3cmd --multipart-chunk-size-mb=500 put - s3://my-offsite-backups/offsite-backup.tar.gz.gpg
```

In this case, the `-` means that the input is being recieved from stdin. The `--multipart-chunk-size-mb=500` option specifies the size of the each chunk of the upload that is buffered before being sent. In my personal testing, a larger size has meant faster uploads over the long term.

**Putting it Together**
Now we can assemble our final command:
```
tar -czf - /srv/backup | gpg --encrypt --recipient offsite-backup | s3cmd --multipart-chunk-size-mb=500 put - s3://my-offsite-backups/offsite-backup.tar.gz.gpg
```

**Undoing It**
Of course, what good are encrypted offsite backups if they can't be retrieved? If we wanted to recover the backed-up files, the command would look like the following:
```
s3cmd get s3://my-offsite-backups/offsite-backup.tar.gz.gpg - | gpg --decrypt | tar -xzf - -C /srv/backup
```

## Conclusion
Hopefully this tutorial gives you the means to make your data safer and stored more reliably while not having to compromise on security. Have a nice day. :\)