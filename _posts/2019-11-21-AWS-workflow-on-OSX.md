---
layout: post
title: AWS Workflow on OS X
---

While completing my deep learning project using AWS services, I spent a good amount of time learning the process of moving files between my local machine, S3 buckets, and EC2 instances, setting up the right type of instance to accommodate my volume of data and model training, installing the right version of Python and libraries on the instance, and running Python code on the instance without my script terminating when the SSH connection times out.  I put together this list of steps to refer to in the future, and offer it to others for reference as well.  Where possible, I included links to related AWS documentation articles or other resources.

1. Created a new EC2 instance (Amazon Linux AMI) with a 20GB EBS attached (enough disk space for data to be transferred from my S3 bucket), using AWS Console.
2. Created and saved keypair .pem file (e.g., aws_20181230.pem) on my local machine in the \Users\{your user name}\.ssh folder.
3. Navigated to the .ssh folder in my Terminal window and ran “chmod 400 aws_20181230.pem” to set read-only permissions on the file.
4. Mounted EBS volume on EC2 instance per instructions on [Making an Amazon EBS Volume Available for Use on Linux](https://docs.aws.amazon.com/en_pv/AWSEC2/latest/UserGuide/ebs-using-volumes.html)
   connected to instance using SSH (ssh -i ~/.ssh/aws_20181230.pem ec2-user@{insert Public DNS (IPv4) address of the instance, starting with ec2-xx-xxx-...}.compute-1.amazonaws.com)
  1. ran “lsblk” command to view available disk drives
  2. created file system on the device
    1. ran “sudo file -s /dev/xvd…” to check if there is a file system
    2. ran “sudo mkfs -t ext4 /dev/xvd…” to create an ext4 file system on the volume
  3. created a mount point directory for the volume
    1. sudo mkdir s3data
  4. mounted the volume at the location above
    1. sudo mount /dev/xvd… s3data
  5. set appropriate permissions on the volume mount to be able to write to it
    1. ran "sudo chmod 777 s3data"
5. Installed python3 on the instance
  1. python —version
  2. sudo yum update
  3. checked which Python3 packages are installed:
    1. ran "sudo yum list \| grep python3" command
  4. installed python3: sudo yum install python36 -y
  5. confirmation commands: python3 —version & which python3.6
6. Created virtual environment on the instance
  1. per [How do I create an isolated Python 3.4 environment with Boto 3 with an Amazon EC2 instance that's running Amazon Linux using virtualenv?](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-linux-python34-boto3/)
    1. under /home/ec2-user directory:
      1. mkdir venv
      2. cd venv
      3. virtualenv -p /usr/bin/python3.6 python36
      4. activated the environment: source /home/ec2-user/venv/python36/bin/activate
      5. which python3 (to check that venv python is now the default)

The rest of the steps are more specific to the computer vision project I was implementing, but still provide some good general workflow guidance:

7. Installed OpenCV (inside the venv folder)
  1. pip install opencv-python
  2. checked that it works:
    1. ran "python3" to start the Python 3 interpreter
    2. ran "import cv2" #no error
    3. ran "exit()"
8. Copied data file from S3 to EC2 (EBS)
  1. ran "aws configure" on EC2
  2. per [How-to Copy Data from S3 to EBS](https://n2ws.com/blog/how-to-guides/how-to-copy-data-from-s3-to-ebs)
    1. aws s3 cp s3://{bucket-name}/{file-to-be-copied} s3data/
    2. Here, I at first received an error about no space left on device.
      1. My problem was actually that I was not in the /home directory when running the command above, but for reference, the below commands increase volume size:
        1.  per [Amazon EBS Elastic Volumes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-modify-volume.html)
          1. aws ec2 modify-volume --region us-east-1 --volume-id vol-0562d5e2f036e1bfb --size 30
          2. df -h
          3. sudo resize2fs /dev/xvd…
    3. checked that data was copied: ls -l s3data/
9. Copied my script (Video_Processing.py file) from my local machine to the EC2 instance:
  1. ran this from the Terminal on my local machine:
    1. scp -i ~/.ssh/aws_20181230.pem Video_Processing.py ec2-user@{insert Public DNS (IPv4) address of the instance, starting with ec2-xx-xxx-...}.compute-1.amazonaws.com:/home/ec2-user
10. Ran code (inside virtual environment): python3 ….
  1. Used Linux Screen (per [How To Use Linux Screen](https://linuxize.com/post/how-to-use-linux-screen) and [EC2 ssh broken pipe terminates running process](https://stackoverflow.com/questions/37796392/ec2-ssh-broken-pipe-terminates-running-process?rq=1) so that the Python script would not terminate when AWS SSH connection timed out
    1. ran “screen” command
    2. python3 ….py (ran the desired program) (can check on the status from a different Terminal window)
    3. At the end, Ctrl-a followed by Ctrl-d to detach from the screen session
11. Copied output from Python script back to the S3 bucket
  1. From the directory containing output on the EC2 instance, ran: aws s3 sync . s3://{bucket-name}/{name_of_folder_to_hold_output}
    1. Additional commands for managing objects can found at: [Managing Objects](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-s3-commands.html#using-s3-commands-managing-objects)
12. Deactivated virtual environment
  1. ran "deactivate" command
13. Exited / logged out of EC2 instance:
  1. disown (more info at: [Keep SSH Sessions running after disconnection](https://unix.stackexchange.com/questions/479/keep-ssh-sessions-running-after-disconnection)
  1. exit (just logs out / closes the connection)
