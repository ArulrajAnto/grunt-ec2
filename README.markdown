# grunt-ec2

> Grunt tasks to create, terminate, and deploy to AWS EC2 instances

Abstracts away [**aws-cli**](https://github.com/aws/aws-cli) allowing you to easily launch, terminate, and deploy to AWS EC2 instances.

Note: This is a _very_, _**very**_ opinionated package. You're invited to fork it and produce your own flow, and definitely encouraged to create pull requests with your awesome improvements.

# Features

This is pretty feature packed

- Launch EC2 instances and set them up with a single task. Look ma', no hands!
- Shut them down from the console. No need to look up an id or anything.
- Use individual SSH key-pairs for each different instance, for increased security
- Deploy with a single Grunt task, using `rsync` for speed
- Use `pm2` to deploy and do hot code swaps!
- Works after reboots, too

# Installation

```shell
npm install --save-dev grunt-ec2
```

For every task in this document, you'll need to set up the AWS configuration for the project. You'll also need to have a **Security Group** set up on AWS. Make sure to enable rules for inbound SSH (port 22) and HTTP (port 80) traffic.

The first time around, you'll [**need to get**](http://www.pip-installer.org/en/latest/installing.html) `pip` to be able to deploy.

Once you installed `pip`, you can install the `awscli` tools.

```shell
pip install awscli --upgrade
```

# Setup

```js
grunt.initConfig({
    pkg: grunt.file.readJSON('package.json'),
    ec2: {
        "AWS_ACCESS_KEY_ID": "<redacted>",
        "AWS_SECRET_ACCESS_KEY": "<redacted>",
        "AWS_SECURITY_GROUP_NAME": "something"
    }
});

grunt.loadNpmTasks('grunt-ec2');
```

You'll need to get an access key pair for AWS, as well as create a security group on AWS by hand. Creating security groups through the CLI is not supported by this package yet.

The `package.json` entry is used to take the `version` number when deploying.

# Configuration

If you're confident enough, you can use the tool with just those options. Here is the full set of options and their defaults.

### AWS_DEFAULT_REGION

Passed to the CLI directly, defaults to `"us-east-1"`

### AWS_IMAGE_ID

Used when creating a new instance with the `ec2_create_instance` task. Defaults to the `"ami-c30360aa"` [Ubuntu AMI](http://cloud-images.ubuntu.com/releases/raring/release-20130423/ "Ubuntu 13.04 (Raring Ringtail)").

### AWS_INSTANCE_TYPE

The magnitude for our instance. Defaults to `"t1.micro"`. Used when creating instances.

### AWS_SECURITY_GROUP_NAME

The security group used for new instances. You'll have to create this one yourself.

### AWS_SSH_USER

The user used to SSH into the instance when setting it up for the first time, after creating it.

### AWS_RSYNC_USER

The user to SSH into the instance when deploying through `rsync`.

### SSH_KEYS_FOLDER

The relative path to a folder where you want to use with tasks that create SSH key-pairs. It doesn't need to exist, `mkdir -p` will take care of that. This defaults to a folder inside this package, which is pretty lame if you want to look at the key-pairs yourself. Although you _shouldn't need to_, I've got you covered.

### PROJECT_ID

Just an identifier for your project, in case you're hosting multiple ones, for some stupid reason, in the same instance. Defaults to `ec2`. This is used when creating folders inside the instance.

### RSYNC_IGNORE

Relative path to an rsync exclusion patterns file. These are used to exclude files from being uploaded to the server during `rsync` on deploys. Defaults to ignoring `.git` and `node_modules`.

```
# vcs files
.git

# will `npm install --production` on the server
node_modules
```

### NODE_SCRIPT

The path to your script. Defaults to `app.js`, as in `node app.js`. Relative to your `cwd`.

# Tasks

Although this package exposes quite a few different tasks, here are the ones you'll want to be using directly.

## Launch an EC2 instance `ec2_launch:name`

Launches an instance and sets it up.

- Creates an SSH key-pair
- Uploads the public key to AWS
- Creates an AWS EC2 instance
- Tags the instance with the friendly name you provided
- Pings the instance until it warms up a DNS and provides SSH access (typically takes a minute)
- Sets up the instance, installing Node, `npm`, and `pm2`

#### Example:

```shell
grunt ec2_launch:teddy
```

## Shutdown an EC2 instance `ec2_shutdown:name`

Terminates an instance and deletes related objects

- Looks up the id of an instance tagged `name`
- Terminates the AWS EC2 instance
- Deletes the key-pair associated with the instance

#### Example:

```shell
grunt ec2_shutdown:teddy
```

## List running EC2 instances `ec2_list`

Returns a JSON list of running EC2 instances. Defaults to filtering by `running` state. You can use `ec2_list:all` to remove the filter, or pick another `instance-state-name` to filter by.

## Get an SSH connection command for an instance `ec2_ssh:name`

Gives you a command you can copy and paste to connect to an EC2 instance through SSH. Useful to get down and dirty.

```shell
grunt ec2_ssh:teddy
```

## Deploy to an EC2 instance `ec2_deploy`

Deploys to a running EC2 instance using `rsync` over SSH.

- Connects to the instance through SSH
- Uploads `cwd` to an `rsync` folder such as `/srv/rsync/example/latest`
- Only transmits changed files, similar to how `git` operates
- Using `pkg.version`, creates a folder with the newest version, like `/srv/apps/example/v/0.6.5`
- Creates a link from `/srv/apps/example/v/0.6.5` to `/srv/apps/example/current`
- Either starts the application, or reloads it with zero downtime, using `pm2`

Example:

```shell
grunt ec2_deploy:teddy
```