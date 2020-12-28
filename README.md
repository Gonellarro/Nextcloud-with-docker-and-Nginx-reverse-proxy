# Installation NextCloud with docker using Nginx as reverse proxy



## Requisites



- Ubuntu Server (version 20.20 used)
- Docker installation (snap default installation)
- Docker compose installation
- nginx reverse proxy (used) with letsencrypt and host domain (duckdns.org used)
- Network from nginx reverse proxy from docker-compose (proxy_net used )



## Installation



- In order to make all the installation process, we will stand as root user: `sudo su`
- Create folder dc_nextcloud: `mkdir dc_nextcloud`
- Create docker-compose.yml file into dc_nextcloud folder: `nano docker-compose.yml`. Copy/Paste this code, then save it with Ctrl+X

```yml
version: '3.5'

services:

  db:
    image: mariadb
    container_name: nextcloud-mariadb
    networks:
      - nextcloud_network
    volumes:
      - db:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_ROOT_PASSWORD=toor
      - MYSQL_PASSWORD=mysql
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    restart: unless-stopped

  app:
    image: nextcloud:latest
    container_name: nextcloud-app
    networks:
      - nextcloud_network
    depends_on:
      - db
    volumes:
      - nextcloud:/var/www/html
      - ./app/config:/var/www/html/config
      - ./app/php_conf/nextcloud.ini:/usr/local/etc/php/conf.d/nextcloud.custom.ini
      - ./app/custom_apps:/var/www/html/custom_apps
      - /mnt/data:/var/www/html/data
      #Samba shares from NAS. Only if needed
      - /mnt/samba/Documents:/home/Documents
      - /mnt/samba/Photos:/home/Photos
      - /mnt/samba/Videos:/home/Videos
      - /mnt/samba/Music:/home/Music
      - ./app/themes:/var/www/html/themes
      - /etc/localtime:/etc/localtime:ro
    environment:
      - VIRTUAL_HOST=nextcloud.xxxx.duckdns.org
      - LETSENCRYPT_HOST=nextcloud.xxxx.duckdns.org
      - LETSENCRYPT_EMAIL=xxxxx@gmail.com
      - NEXTCLOUD_OVERWRITEPROTOCOL=https
    restart: unless-stopped

volumes:
  nextcloud:
  db:

networks:
  nextcloud_network:
      external:
         name: proxy_net
```

- Now you can execute `docker-compose up -d`.  At this moment, docker will download and install Nextcloud app and MariaDB data base and leave the system working



**Common errors:**

1. Can't upload files bigger than 2GB. This is due the file restriction of PHP and NGNINX. To solve this we have to modify/create config files into the right folders.

   1. Change *upload_max_filesize* of php:

      - Create file dc_nexcloud/app/config/php_conf/nexcloud.ini and Copy/Paste 

      ```
      upload_max_filesize=2048M
      post_max_size=2058M
      max_execution_time = 200
      memory_limit=1G
      ```

   2. Create custom.ini in the right folder of proxy installation (dc_proxy/proxy/conf.d/custom.ini used) and Copy/Paste

      ```
      client_max_body_size 25m;
      ```

2. Can't grant access from android to Nextcloud. This issue is due the https reverse proxy configuration. We have to change it editing *dc_nextcloudd/app/config/config.php* file

   ```
   'overwrite.cli.url' => 'http://nextcloud.xxxx.duckdns.org',
   'overwriteprotocol' => 'https', 
   ```

   

## Bonus: Acces to remote data shared with SMB

In my case, I needed to acces to my data stored in my home NAS. This data is shared to local network without any kind of access permission because it's into a secure zone. Nevertheless, if we want to share it with the rest of the world, we need to implement a new security layer. Here comes in handy Nextcloud installation.

1. In my case I have 4 folders that I want to be accessible from the Internet:
   - Documents
   - Photos
   - Music
   - Videos
2. This 4 folders have a share folder from the NAS: //192.168.1.x/Documents,... So we want to map this 4 folders at the boot of the VM, so we have to use fstab file
3. This folders have to be accessible to the docker instance that executes Nexcloud, so we have to configure in some way that Docker knows them



### Installation

1. We have to install cifs: `apt-get install cifs-utils`

2. We create a folder where to mount them: `mkdir /mnt/samba`

3. Edit with `nano /etc/fstab` and Copy/Paste. Please, be aware that the owner of the session will be www-data, so we have to specify here. Save the file with Ctrl+X

   ```
   //192.168.1.x/Imatges /mnt/samba/Photos cifs guest,uid=www-data,gid=www-data 0 0
   //192.168.1.x/Documents /mnt/samba/Documents cifs guest,uid=www-data,gid=www-data 0 0
   //192.168.1.x/Musica /mnt/samba/Music cifs guest,uid=www-data,gid=www-data 0 0
   //192.168.1.x/Videos /mnt/samba/Videos cifs guest,uid=www-data,gid=www-data 0 0
   ```

4. Create the volumes into docker-compose.yml, as we saw before

   ```yml
     #Samba shares from NAS. Only if needed
     - /mnt/samba/Documents:/home/Documents
     - /mnt/samba/Imatges:/home/Photos
     - /mnt/samba/Videos:/home/Videos
     - /mnt/samba/Musica:/home/Music
   ```



Now it's ready to be accessed, but Nextcloud has to recognize them as an external storage. Indeed, for Nextcloud this is not its own storage into its repository (/var/www/html/data) , so we need to install the app *external storage support* from the app market. 

1. Go to your user profile (top-right) and go to *+Applications*
2. Search *External storage support* and *Download and activate*
3. Go to your user profile (top-right) and go to *Settings*
4. At the left panel, go to *Administrations/External Storage*
5. Add a new external storage, fill in the blanks: Name, Local, No auth, /home/Documents, user
6. Repeat for every share you want. At this point, I need to highlight, that the folder that Nextcloud access is /home/Documents, that it's inside the docker instance. Remember we are working with docker, so we have to make links between the real data and docker data.


