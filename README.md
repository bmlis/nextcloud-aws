# Nextcloud with AWS CloudFormation
Not so automatic setup of Nextcloud using free tier of AWS.

## Table of Contents
1. [Setup](#setup)
2. [Restoring backups](#restoring-backups)
3. [To do](#to-do)


## Setup
### 1. VPC
```
> cp vpc/stack-parameters.example.json vpc/stack-parameters.json
> ./create-stack vpc
```
Above will create simple VPC with two public subnets.

### 2. Nextcloud buckets
Project is set up with three buckets - nextcloud instalation's backups, database backups and nextcloud installation's main storage. At this point, you need to come up with your domain name, proably something similar to `nextcloud.example.com`. Copy stack parameters file, fill up the domain name and create the stack.
```
> cp buckets/stack-parameters.example.json buckets/stack-parameters.json
> ./create-stack buckets
```

### 3. Cluster
Unfortunately, the way things are currently set up, you **will** need to ssh into your instance to tinker up a little bit. In order for us to do that, you need to create key pair in EC2 console on your AWS console.

Copy stack parameters, fill up vpc and subnet ids (check in the console). Rember to change the `KeyName` to name of the key you created, by default it's `nextcloud`. You will also need to provide names of the buckets you created at previous step. We need domain name for creating a user that will be used to access data in nextcloud, cron jobs for backups, and finally for nginx configuration.
```
> cp cluster/stack-parameters.example.json cluster/stack-parameters.json
> ./create-stack cluster
```

### 4. Setting up the domain and SSL
First up, we need to set up our domain so that it points to EC2 instance. In EC2 AWS console find your instance and copy it's dns name. Go to your provider's website and create CNAME for your domain that points to mentioned dns name.

Next, we need to get our certificate, proving that we own that domain in the meantime. In order to do that I've create a simple nginx configuration that just listens for a challenge from letsencrypt's certbot. SSH to your instance, run nginx and launch certbot:
```
> docker run --rm \
      -d \
      -v /data/letsencrypt:/data/letsencrypt \
      -v /etc/nginx/certificateCreator.conf:/etc/nginx/conf.d/default.conf \
      -p 80:80 \
      nginx:alpine

> docker run -it --rm \
      -v /etc/letsencrypt:/etc/letsencrypt \
      -v /data/letsencrypt:/data/letsencrypt \
      certbot/certbot \
      certonly \
      --email <your-email> \
      --agree-tos \
      --webroot --webroot-path=/data/letsencrypt \
      -d nextcloud.example.com
> docker stop <nginx-container-id>
```
Disclaimer - you may need to wait for your DNS name to propagate, certbot has 5 requests/hour limit after which you'll get soft banned.

### 5. Services
In order to setup services, we need to fill up quite a lot of parameters, mostly passwords and usernames for db and nextcloud access. After that simply create the stack, you're almost done.
```
> cp services/stack-parameters.example.json services/stack-parameters.json
> # fill up stack parameters
> ./create-stack services
```

### 6. Setting up S3 as primary storage
Before setting up the S3 go and log in to your nextcloud instance first. I've noticed that if you set up objectstore before logging in installation goes bonkers and spits out 500's like there is no tomorrow.

You need to ssh into your instance and edit the `/data/nextcloud/config/config.php`. Before that, go to AWS IAM console and create access keys for storage user. Next, enter the maintenence mode:
```
docker exec --user www-data -it <nextcloud-container-id> php occ maintenance:mode --on
```
and add following to the `config.php` file:
```
  'objectstore' =>
  array (
    'class' => 'OC\\Files\\ObjectStore\\S3',
    'arguments' =>
    array (
      'bucket' => '<storage-bucket-name>',
      'key' => '<storage-user-key>',
      'secret' => '<storage-user-secret>',
      'use_ssl' => true,
      'region' => '<region-you-deployed-to>',
      'use_path_style' => true,
    ),
  ),
```
Finally, turn off the maintenance mode:
```
docker exec --user www-data -it <nextcloud-container-id> php occ maintenance:mode --off
```
I also strongly recommend adding encryption app to your installation before uploading anything. You'll need to set it up for the storage after installation, refer to nextcloud docs for more info.


## Restoring backups
1. Start services (nextcloud and database)
2. Download backup files:
```
aws s3 cp s3://<your-nextcloud-backups>/25_12_18.tar.gz .
aws s3 cp s3://<your-nextcloud-db-backups>/nextcloud_sqlbkp_20181225.bak .
```
3. Turn on maintenance mode
```
docker exec -it <nextcloud-container-id> php occ maintenance:mode --off
```
4. Untar the backup and copy it to nextcloud data folder:
```
tar -xvf 25_12_18.tar.gz
cp -R ./data/nextcloud /data/nextcloud
```
If later you have problems with internal errors even though the `php occ maintenance:repair` looks ok, chown the /data/nextcloud to `www-data` user:
```
docker exec -it <nextcloud-container-id> sh -c "chown -R www-data:www-data /var/www/html"
```
5. Restore db backup:
```
docker cp ./nextcloud_sqlbkp_20181225.bak <mariadb-container-id>:/root/
docker exec <mariadb-container-id> sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD" < /root/nextcloud_sqlbkp_20181225.bak'

```

## To do
- Renewal of certificate
