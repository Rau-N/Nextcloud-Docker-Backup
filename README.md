# Nextcloud-Docker-Backup

This repository contains a linux bash script to automate an incremental and consistent backup process for a docker-based nextcloud instance.
It is based on the famous BorgBackup (https://borgbackup.readthedocs.io/en/latest/) program, which enables a secure and space-efficient way to backup data.
The internally used data deduplication technique makes Borg suitable for daily backups since only changes are stored. 
It also supports the encryption of the created backup archives to grant a safe way of storing them on not fully trusted targets, e.g. cloud storages.

## General Information

A complete backup of any normal Nextcloud instance requires to backup the following items:

- The Nextcloud file directory (usually */var/www/nextcloud*) 
- The data directory of Nextcloud (which is either located inside the file directory or placed in a separated directory)
- The Nextcloud database

For a docker-based nextcloud instance, usually it's only recommended to mount, persist and backup the most important data.
In this case, the file directory, containing the application binaries among other things, is usually to exclude, because it will always be resetted after updating the docker image and recreation of the container.

The following docker volumes (bind mounts) show an example of most important data to mount and backup.
It is adapted to the official nextcloud docker image (https://hub.docker.com/_/nextcloud)

- ./nextcloud/config:/var/www/html/config
- ./nextcloud/custom_apps:/var/www/html/custom_apps
- ./nextcloud/data:/var/www/html/data
- ./nextcloud/themes:/var/www/html/themes

Additionally, it is necessary to backup the nextcloud database, which is usually running in a separate docker container.
These five items have to be backed up or snapshotted at the same time, handled like a critical section, to get a consistent and flawlessly restorable backup state.

## Script Backup Algorithm

The script takes care of the whole backup process by performing the following general steps:

1. Enabling the nextcloud maintenance mode inside the nextcloud docker container (occ maintenance:mode --on).
2. Stopping the webserver instance inside the nextcloud docker container (service apache2 stop).
3. Create a database dump from the nextcloud database docker container.
4. Create a backup archive using BorgBackup consisting of the changed data in the bind mount directories and the database dump file.
5. Restarting the webserver instance inside the nextcloud docker container (service apache2 start).
6. Disabling the nextcloud maintenance mode inside the nextcloud docker container (occ maintenance:mode --off).


**Important:**

- Before using this script, you have to check for installed prerequisites (*apt install docker borgbackup moreutils*) and edit a bunch of variables (directories, users, options, etc.) according to your environment.
- All variables which need to be customized are marked with *TODO* in the script's comments.
- The scripts assumes that you configured the bind mount directories all being in the same parent directory, which will be recursively backed up by the script.
- If you separated the bind mount directories across the host's filesystem, you might include multiple variables in the script, each for one bind mount directory and add each variable to the *backup* variable (*backup="$nextcloudBindMountDir $nextcloudDbDumpDir"*).
- There's a default backup retention policy configured which should be changed according your needs to ensure that BorgBackup prunes old backup archives after specific time intervals.
- This script must be run with root privileges.

## Executing Backups

In order to create a backup, simply call the script *NextcloudDockerIncrementalBackup.sh* on the docker host system.
The script will create a borg repository at the defined *backupDestination* directory (this can be an internal directory, an externally mounted drive or a mounted network share).
To automate the backup process it is recommended to execute this script as a root user cronjob on the host system (e.g. *0 1 * * * /usr/local/scripts/NextcloudDockerBackupWithBorg.sh*).

## Accessing and Restoring Backups

To be able to access already created backup archives, you have to mount the borg repository, which resides in the *backupDestination* directory.
Therefore execute the following command:

- borg mount *backupDestination* *mountDirectory*

After successful mounting, you'll be able to access all created backup archives on a file-level base and be able to access every single file from the backed up bind mount directories including the database dump file. In case of a data recovery scenario, just copy the necessary files from a specific archive to another place or immediately replace the original bind mount directory files on the host system and import the database dump back into the nextcloud database container. Don't forget to enable the nextcloud maintenance mode before restoring a backup archive.
After a backup recovery it is important to unmount the borg repository, otherwise it will stay locked and no future backups will be created.
Use the following command to unmount the borg repository:

- borg umount *mountDirectory*
