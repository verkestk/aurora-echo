
## Aurora Echo

### Bringing you yesterday's database today!
This tool assists in AWS RDS Aurora database lifecycle management. It is a companion project to [RDS Echo](https://github.com/blacklocus/rds-echo), which deals solely in non-Aurora instances.

### What do?
Use this tool to automatically restore an Aurora database cluster from a snapshot, promote it to live via DNS updates in Route53, and at EOL destroy the managed cluster.

The three different stages here -- new, promote, retire -- can all be used independently of each other and run periodically without regard for timing of the other stages. This is because the commands are idempotent and the lifecycle stages are tracked on the database instances themselves via tags.
Thus each command will only operate on a database that is tagged appropriately and exit cleanly if there is no database in the appropriate stage.

Have multiple development databases? Aurora Echo allows management of unlimited independent lifecycles; just name them differently in configuration and don't worry about them interfering with each other.

## Stages
This is our story: On a regular basis, restore the latest production snapshot to a new development instance, promote
that instance to replace its former self, and then destroy the old instance.

Each `aurora_echo` command progresses a managed instance through a series of stages:

  - (non-existent) --`aurora_echo new`-->     **new**
  - **new**   --`aurora_echo promote`--> **promoted**
    - This also results in any previously **promoted** instance advancing to **retired**
  - **retired**  --`aurora_echo retire`-->  (non-existent)

So in the straightforward case, each command is run in succession after the previous commands stabilize and leave the
DB instance in the "available" state.

All state tracking metadata is stored as AWS resource tags on the cluster and instance.


## Installation
Aurora Echo is a command-line tool.

You will need to set up AWS auth as per the [boto documentation](https://boto3.readthedocs.io/en/latest/guide/quickstart.html#configuration)

TODO link to tagged release
TODO installation instructions
TODO sample usage


## Usage

### `new`
- **What**: Create a new cluster and instance restored from a snapshot.
- **How**: Given a cluster name as input, find the latest snapshot of said cluster and use it as a basis for restoring a new cluster.
- **When**: You may want to run this periodically on a cron job. It includes a safety check to make sure it hasn't restored a cluster in the last (configurable) n hours and aborts if a managed cluster is too new.

#### Configuration
- `-a, --aws_account_number [required]`
  - Your AWS account number
- `-r, --region [required]`
  - e.g. `us-east-1`
- `-n, --managed_name [required]`
  - The name of the cluster and instance you want to create/restore to. This will also go into the tag to track managed instances. e.g. `development`
- `-s, --cluster_snapshot_name [required]`
  - The cluster name of the snapshot you want to restore from, e.g. `production`
- `-sub, --db_subnet_group_name [required]`
  - VPC subnet group to restore into
- `-c, --db_instance_class [required]`
  - Size of the database instance to create, e.g. `db.r3.2xlarge`
- `-e, --engine`
  - Defaults to `aurora`
- `-az, --availability_zone`
  - e.g. `us-east-1c`. If not set, AWS defaults this to the region the parent snapshot is in.
- `-sg, --vpc_security_group_id`
  - Allows multiple inputs (use one option flag per input).
  - The ID of any security groups to assign to the created instance/cluster
- `-t, --tag`
  - Allows multiple inputs (use one option flag per input).
  - Any custom tags to assign to the cluster and instance along with the managed tags automatically assigned by Aurora Echo, e.g. `purple=true`
- `-h, --minimum_age_hours`
  - If an existing managed instance has been created within the last `-h` hours, abort creation of a new instance. Defaults to 20.
- `--help`
  - Show options and exit.

### `promote`
- **What**: Progress a database instance from `new` to `promoted` by updating a record set's DNS entry in Route53 to point to the newly promoted database's endpoint.
- **How**: Look for a managed instance in RDS that is in the stage `new`, and update the supplied record set's DNS entry to its endpoint. Move any appropriate existing instance's stage from `promoted` to `retired`, and update this instance's stage from `new` to `promoted`.
- **When**: You may want to run this periodically on a cron job. It will only operate when an instance is in the `new` stage and has status `available`.

#### Configuration
- `-a, --aws_account_number [required]`
  - Your AWS account number
- `-r, --region [required]`
  - e.g. `us-east-1`
- `-n, --managed_name [required]`
  - The managed name tracking the instance you want to promote. This is the same a the `-managed_name` parameter used in the `new` step.
- `-z, --hosted_zone_id [required]`
  - The ID of the hosted zone containing the DNS record set to be updated
- `-rs, --record_set [required]`
  - Name of the record set to update, e.g. `dev-db.mycompany.com`. Aurora Echo only supports CNAME updates.
- `--ttl`
  - TTL in seconds. Defaults to 60.
- `--help`
  - Show options and exit.


### retire
- **What**: Delete a managed instance and cluster that is in the `retired` stage.
- **How**: Look for a managed instance in RDS that is in the stage `retired` and delete the instance and its containing cluster. There is no option to make a final snapshot. All automated instance/cluster snapshots **will be deleted**.
- **When**: You may want to run this periodically on a cron job. It will only operate when a managed instance is in the `retired` stage.

#### Configuration
- `-a, --aws_account_number [required]`
  - Your AWS account number
- `-r, --region [required]`
  - e.g. `us-east-1`
- `-n, --managed_name [required]`
  - The managed name tracking the instance you want to retire. This is the same a the `-managed_name` parameter used in previous steps.


## Notes!
- This tool creates instances and clusters with today's date attached, such as `development-2016-10-05`. This combined with the previous-instance freshness check will prevent multiple instances from being created in a cluster.

