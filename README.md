# Automatically mount WebDAV on boot

## Information and prerequisites
- Note that systemd `.mount` files can also be used.
- We assume that :
  - You want to connect on the WebDAV FS using HTTPS (and not HTTP).
  - The WebDAV certificate is signed by a CA and you own the CA certificate.
  - The `Common Name` value used in the certificate is correct.
  - The WebDAV access is secured by a password.
- The procedure below has been tested on a clean Ubuntu 20.04 VM.

## Configuration
Install the needed packages. The question `Should unprivileged users be allowed to mount WebDAV resources?` may be asked. Answer `no`.
```
sudo apt install davfs2
```
Save the sample file.
```
sudo mv /etc/davfs2/secrets /etc/davfs2/secrets.save
```
Create a `secrets` file to store credentials.
```
sudo vim /etc/davfs2/secrets
```
Paste the following inside. Replace the username (`<user>`) and password (`<password>`) values. They are the credentials used to access the WebDAV FS.
```
/mnt/webdav <user> <password>
```
Set the correct permissions.
```
sudo chmod 600 /etc/davfs2/secrets
```
Create the directory that we specified.
```
sudo mkdir /mnt/webdav
```
Put the CA cert (which signed the WebDAV certificate) in `/etc/davfs2/certs/`. Edit the davfs2 configuration file.
```
sudo vim /etc/davfs2/davfs2.conf
```
Change the `trust_ca_cert` line to the following (replace the `your_ca.crt` value).
```
trust_ca_cert     /etc/davfs2/certs/your_ca.crt
```
Create a new systemd service file.
```
sudo vim /etc/systemd/system/webdav.service
```
Paste the following (adapt the content). Replace the WebDAV URL (`https://server-name/webdav`) by yours. Replace the `user_name` value by **YOUR** username.
```
[Unit]
Description=WebDAV
After=network-online.target
Wants=network-online.target

[Service]
ExecStartPre=/usr/bin/sleep 5
ExecStart=/usr/bin/mount -t davfs https://server-name/webdav /mnt/webdav -o uid=user_name,gid=user_name
ExecStopPre=/usr/bin/sync /mnt/webdav
ExecStop=/usr/bin/fusermount -u /mnt/webdav/
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```
Reload the systemd files and enable the new service.
```
sudo systemctl daemon-reload
sudo systemctl enable /etc/systemd/system/webdav.service
```

## Test that everything is working
Start the systemd service.
```
sudo systemctl start webdav.service
```
Print the status. Everything should be OK.
```
sudo systemctl start webdav.service
```
Print the `/mnt/webdav` directory content. You should see your files.
```
ll /mnt/webdav/
```

## Debug / questions
- If your WebDAV service is accessible through a VPN tunnel, you may want to start this WebDAV systemd service only after the tunnel has been established. Then, for example if you use a Wireguard VPN (and that it is managed by systemd), replace the line `After=network-online.target` by `After=network-online.target wg-quick@wg0`.