# Nextcloud with AWS CloudFormation
Not so automatic setup of Nextcloud using free tier of AWS.

## Setup
1. Create vpc stack with `./create-stack vpc`
2. Create buckets for the installation with './create-stack buckets'
3. Create key pair in ec2 console for accessing your instance, add it to `cluster/stack-parameters.json` anc create cluster stack `./create-stack cluster`
4. Add CNAME for your domain, that points to DNS address of your instance
5. SSH into your instance and fetch certificate for your domain:

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
      -d nextcloud.your.domain
> docker stop <nginx-container-id>
```

6. Create stack for nextcloud and nginx services `./create-stack services`
7. Setup s3 bucket as primary storage. You need to ssh into your instance and edit the `/data/nextcloud/config/config.php` (remember to use maintenance mode). Add following:
```
  'objectstore' =>
  array (
    'class' => 'OC\\Files\\ObjectStore\\S3',
    'arguments' =>
    array (
      'bucket' => '<storage-bucket-name>',
      'key' => '<storage-user-key>',
      'secret' => <storage-user-secret>,
      'use_ssl' => true,
      'region' => '<region-you-deployed-to>',
      'use_path_style' => true,
    ),
  ),
```


TODO:
- s3 as primary storage
- remember to remove references from files

##Restoring backups:
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
