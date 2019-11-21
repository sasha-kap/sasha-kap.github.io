---
layout: post
title: AWS Workflow on OS X
---

While completing my deep learning project using AWS services, I spent a good amount of time learning the process of moving files between my local machine, S3 buckets, and EC2 instances, setting up the right type of instance to accommodate my volume of data and model training, installing the right version of Python and libraries on the instance, and running Python code on the instance without my script terminating when the SSH connection times out.  I put together this list of steps to refer to in the future, and offer it to others for reference as well.  Where possible, I included links to related AWS documentation articles or other resources.

```html
<ul class="ul custom">
<li>Created a new EC2 instance (Amazon Linux AMI) with a 20GB EBS attached (enough disk space for data to be transferred from my S3 bucket), using AWS Console.</li>
<li>Created and saved keypair .pem file (e.g., aws_20181230.pem) on my local machine in the \Users\{your user name}\.ssh folder.</li>
<li>Navigated to the .ssh folder in my Terminal window and ran “chmod 400 aws_20181230.pem” to set read-only permissions on the file.</li>
<li>Mounted EBS volume on EC2 instance per instructions on [Making an Amazon EBS Volume Available for Use on Linux](https://docs.aws.amazon.com/en_pv/AWSEC2/latest/UserGuide/ebs-using-volumes.html)
  <ul class="ul custom">
  <li>connected to instance using SSH (ssh -i ~/.ssh/aws_20181230.pem ec2-user@{insert Public DNS (IPv4) address of the instance, starting with ec2-xx-xxx-...}.compute-1.amazonaws.com)</li>
  <li>ran “lsblk” command to view available disk drives</li>
  <li>created file system on the device
    <ul class="ul custom">
    <li>ran “sudo file -s /dev/xvd…” to check if there is a file system</li>
    <li>ran “sudo mkfs -t ext4 /dev/xvd…” to create an ext4 file system on the volume</li>
    </ul>
  </li>
  <li>created a mount point directory for the volume
    <ul class="ul custom">
    <li>sudo mkdir s3data</li>
    </ul>
  </li>
  <li>mounted the volume at the location above
    <ul class="ul custom">
    <li>sudo mount /dev/xvd… s3data</li>
    </ul>
  </li>
  <li>set appropriate permissions on the volume mount to be able to write to it
    <ul class="ul custom">
    <li>ran "sudo chmod 777 s3data"</li>
    </ul>
  </li>
  </ul>
</li>
<li>Installed python3 on the instance
  <ul class="ul custom">
  <li>python —version</li>
  <li>sudo yum update</li>
  <li>checked which Python3 packages are installed:
    <ul class="ul custom">
    <li>ran "sudo yum list | grep python3" command</li>
    </ul>
  </li>
  <li>installed python3: sudo yum install python36 -y</li>
  <li>confirmation commands: python3 —version & which python3.6</li>
  </ul>
</li>
<li>Created virtual environment on the instance
  <ul class="ul custom">
  <li>per [How do I create an isolated Python 3.4 environment with Boto 3 with an Amazon EC2 instance that's running Amazon Linux using virtualenv?](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-linux-python34-boto3/)
    <ul class="ul custom">
    <li>under /home/ec2-user directory:
      <ul class="ul custom">
      <li>mkdir venv</li>
      <li>cd venv</li>
      <li>virtualenv -p /usr/bin/python3.6 python36</li>
      <li>activated the environment: source /home/ec2-user/venv/python36/bin/activate</li>
      <li>which python3 (to check that venv python is now the default)</li>
      </ul>
    </li>
    </ul>
  </li>
  </ul>
</li>
</ul>
The rest of the steps are more specific to the computer vision project I was implementing, but still provide some good general workflow guidance:
<ul class="ul custom">
<li>Installed OpenCV (inside the venv folder)
  <ul class="ul custom">
  <li>pip install opencv-python</li>
  <li>checked that it works:
    <ul class="ul custom">
    <li>ran "python3" to start the Python 3 interpreter</li>
    <li>ran "import cv2" #no error</li>
    <li>ran "exit()"</li>
    </ul>
  </li>
  </ul>
</li>
<li>Copied data file from S3 to EC2 (EBS)
  <ul class="ul custom">
  <li>ran "aws configure" on EC2</li>
  <li>per [How-to Copy Data from S3 to EBS](https://n2ws.com/blog/how-to-guides/how-to-copy-data-from-s3-to-ebs)
    <ul class="ul custom">
    <li>aws s3 cp s3://{bucket-name}/{file-to-be-copied} s3data/</li>
    <li>Here, I at first received an error about no space left on device.
      <ul class="ul custom">
      <li>My problem was actually that I was not in the /home directory when running the command above, but for reference, the below commands increase volume size:
        <ul class="ul custom">
        <li>per [Amazon EBS Elastic Volumes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-modify-volume.html)
          <ul class="ul custom">
          <li>aws ec2 modify-volume --region us-east-1 --volume-id vol-0562d5e2f036e1bfb --size 30</li>
          <li>df -h</li>
          <li>sudo resize2fs /dev/xvd…</li>
          </ul>
        </li>
        </ul>
      </li>
      </ul>
    </li>
    </ul>
    <li>checked that data was copied: ls -l s3data/</li>
  </ul>
<li>Copied my script (Video_Processing.py file) from my local machine to the EC2 instance:
  <ul class="ul custom">
  <li>ran this from the Terminal on my local machine:
    <ul class="ul custom">
    <li>scp -i ~/.ssh/aws_20181230.pem Video_Processing.py ec2-user@{insert Public DNS (IPv4) address of the instance, starting with ec2-xx-xxx-...}.compute-1.amazonaws.com:/home/ec2-user</li>
    </ul>
  </li>
  </ul>
</li>
<li>Ran code (inside virtual environment): python3 ….
  <ul class="ul custom">
  <li>Used Linux Screen (per [How To Use Linux Screen](https://linuxize.com/post/how-to-use-linux-screen) and [EC2 ssh broken pipe terminates running process](https://stackoverflow.com/questions/37796392/ec2-ssh-broken-pipe-terminates-running-process?rq=1) so that the Python script would not terminate when AWS SSH connection timed out
    <ul class="ul custom">
    <li>ran “screen” command</li>
    <li>python3 ….py (ran the desired program) (can check on the status from a different Terminal window)</li>
    <li>At the end, Ctrl-a followed by Ctrl-d to detach from the screen session</li>
    </ul>
  </li>
  </ul>
</li>
<li>Copied output from Python script back to the S3 bucket
  <ul class="ul custom">
  <li>From the directory containing output on the EC2 instance, ran: aws s3 sync . s3://{bucket-name}/{name_of_folder_to_hold_output}
    <ul class="ul custom">
    <li>Additional commands for managing objects can found at: [Managing Objects](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-s3-commands.html#using-s3-commands-managing-objects)</li>
    </ul>
  </li>
  </ul>
</li>
<li>Deactivated virtual environment
  <ul class="ul custom">
  <li>ran "deactivate" command</li>
  </ul>
</li>
<li>Exited / logged out of EC2 instance:
  <ul class="ul custom">
  <li>disown (more info at: [Keep SSH Sessions running after disconnection](https://unix.stackexchange.com/questions/479/keep-ssh-sessions-running-after-disconnection))</li>
  <li>exit (just logs out / closes the connection)</li>
  </ul>
</li>
</ul>
```
