# howto_proxmox

Gotchas around Proxmox and installed LXCs

## paperless-ngx

Install via <https://community-scripts.github.io/ProxmoxVE/scripts?id=paperless-ngx>

To make it work via Nginx Proxy Manager (NPM) edit `/opt/paperless/paperless.conf` and set `PAPERLESS_URL` and `PAPERLESS_CSRF_TRUSTED_ORIGINS` to desired domains otherwise you'll get either CSRF errors after login or the dashboard won't fully load.

Stop running services:

```shell
systemctl stop paperless-consumer paperless-webserver paperless-scheduler paperless-task-queue
```

Edit config options

```shell
vi /opt/paperless/paperless.conf
PAPERLESS_URL=<https://paperless.example.com>
PAPERLESS_CSRF_TRUSTED_ORIGINS=<https://paperless.example.com,https://paperless-ngx.example.com> # can be set using PAPERLESS_URL
```

Start services

```shell
systemctl start paperless-consumer paperless-webserver paperless-scheduler paperless-task-queue
```

Enable Websockets Support in NPM  
Don't force SSL in NPM or else upload of files breaks!

### backup and restore of data/documents/settings/etc

My installation was running on a Debian 12 and for the life of me I couldn't get it upgraded to Debian 13 using <https://github.com/community-scripts/ProxmoxVE/discussions/7489> hence I decided to do a clean install of the LXC (see above) and just import my data.

Paperless offers an export/import function see <https://docs.paperless-ngx.com/administration/#exporter> and <https://docs.paperless-ngx.com/administration/#importer>  
In the old LXC I just exported the data as follows:

```shell
cd /opt/paperless/src
python3 manage.py document_exporter /home/$USER/ -z
```

Installed a new LXC and copied the zip file over (using `scp`).  
In the new container I had to source the `venv` first to get the script to run:

```shell
cd /opt/paperless
source .venv/bin/activate
cd src/
python3 manage.py document_importer /home/$USER/export-2025-12-14.zip
```
