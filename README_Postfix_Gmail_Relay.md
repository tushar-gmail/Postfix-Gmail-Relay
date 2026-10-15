# Configure Postfix on Ubuntu to Send Email Through Gmail SMTP

This guide covers installing Postfix on Ubuntu, configuring it to relay outbound email through Gmail SMTP, testing delivery, and troubleshooting a stuck mail queue.

> **Important**
> - Replace example hostnames, domains, and email addresses with your own values.
> - Use a Google **App Password** rather than your normal Google account password.
> - Never commit SMTP credentials or the `/etc/postfix/sasl_passwd` file to GitHub.
> - For an AWS EC2 instance, confirm outbound connectivity to Gmail SMTP on TCP port `587`. AWS network controls or provider restrictions may affect mail delivery.

## 1. Set the hostname and update packages

```bash
sudo hostnamectl set-hostname mail.example.com
echo '127.0.0.1 mail.example.com' | sudo tee -a /etc/hosts
sudo apt update
```

Replace `mail.example.com` with the hostname you intend to use.

![Hostname and initial setup commands](screenshots/screenshot-01.png)


## 2. Install Postfix and mail utilities

Preconfigure Postfix as an Internet Site:

```bash
echo "postfix postfix/main_mailer_type select Internet Site" | sudo debconf-set-selections
echo "postfix postfix/mailname string mail.example.com" | sudo debconf-set-selections
sudo apt install -y postfix mailutils
```

## 3. Configure Postfix

Open the main configuration file:

```bash
sudo nano /etc/postfix/main.cf
```

Verify or adjust these settings for your environment:

```ini
myhostname = mail.example.com
mydestination = $myhostname, example.com, localhost.localdomain, localhost
inet_interfaces = all
```

If the server is only sending mail and should not accept inbound SMTP connections, consider limiting the listening interface rather than using `inet_interfaces = all`. Make sure `mydestination` matches the domains this server is meant to deliver locally.

## 4. Firewall ports (optional; mainly for inbound mail)

The source guide lists these UFW rules:

```bash
sudo ufw allow 25/tcp
sudo ufw allow 587/tcp
sudo ufw allow 465/tcp
```

Only open inbound ports that your design actually requires. These rules do **not** enable outbound Gmail relay by themselves.

![UFW firewall rule commands](screenshots/screenshot-03.png)


## 5. Restart and test Postfix

```bash
sudo systemctl restart postfix
sudo systemctl status postfix
```

![Postfix service status](screenshots/screenshot-04.png)

Send a basic test message:

```bash
echo "Test email body" | mail -s "Test from Postfix" you@example.com
```

Check the queue and logs if delivery does not complete:

```bash
mailq
sudo tail -n 100 /var/log/mail.log
```

## 6. Install SASL authentication modules

Postfix needs SASL modules to authenticate to Gmail:

```bash
sudo apt update
sudo apt install -y libsasl2-modules
```

## 7. Generate a Google App Password

1. Open your Google Account security settings.
2. Enable **2-Step Verification**, if it is not already enabled.
3. Find **App passwords** in the Google Account security settings, if the option is available.
4. Create an app password with a descriptive name, such as `Postfix Ubuntu`.
5. Copy the generated password. Google may display it with spaces; use the password as directed by Google.

Availability of App Passwords depends on the Google account's security configuration and policies.

## 8. Create the Postfix credentials file

Create the credentials file:

```bash
sudo nano /etc/postfix/sasl_passwd
```

Add this line, replacing the placeholders with your Gmail address and Google App Password:

```text
[smtp.gmail.com]:587 your_email@gmail.com:your_app_password
```

Secure the file and generate Postfix's lookup database:

```bash
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
```

The `postmap` command should create `/etc/postfix/sasl_passwd.db`. Keep both files outside your Git repository.

![Creating and securing the SASL password map](screenshots/screenshot-09.png)


## 9. Configure Gmail as the SMTP relay

Edit the configuration:

```bash
sudo nano /etc/postfix/main.cf
```

Add or update the following settings:

```ini
# Gmail SMTP Relay Configuration
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```

![Gmail SMTP relay settings in main.cf](screenshots/screenshot-11.png)


Restart Postfix:

```bash
sudo systemctl restart postfix
sudo systemctl status postfix
```

## 10. Send a test email through Gmail

```bash
echo "This email is relayed through Gmail!" | mail -s "Postfix Gmail Test" destination_email@example.com
```

Check delivery status:

```bash
mailq
sudo tail -n 100 /var/log/mail.log
```

![Postfix mail queue output](screenshots/screenshot-13.png)

![Test messages visible in the inbox](screenshots/screenshot-17.png)


## 11. Troubleshooting

### A. IPv6 network unreachable

The source guide showed a queue error similar to:

```text
connect to smtp.gmail.com[IPv6-address]:587: Network is unreachable
```

This indicates that the host tried an IPv6 address but did not have working IPv6 connectivity to that destination.

![Example of the IPv6 network unreachable error](screenshots/screenshot-10.png)


To configure Postfix to use IPv4, edit:

```bash
sudo nano /etc/postfix/main.cf
```

Add or update this setting:

```ini
inet_protocols = ipv4
```

Then restart Postfix and retry queued messages:

```bash
sudo systemctl restart postfix
sudo postqueue -f
mailq
```

### B. `local data error while talking to smtp.gmail.com`

The source guide showed errors like:

```text
local data error while talking to smtp.gmail.com
```

The guide identifies a likely cause as Postfix being unable to find or open its SASL password lookup database (`sasl_passwd.db`). Check that the database exists and rebuild it:

![Postfix queue error during Gmail relay troubleshooting](screenshots/screenshot-14.png)


```bash
sudo ls -l /etc/postfix/sasl_passwd*
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
sudo postqueue -f
mailq
```

If the error persists, inspect the mail log for the detailed SASL or lookup error:

```bash
sudo tail -n 100 /var/log/mail.log
```

Also verify that `smtp_sasl_password_maps` in `/etc/postfix/main.cf` is exactly:

```ini
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
```

Do not publish the contents of `sasl_passwd` or paste your App Password into logs, screenshots, or GitHub issues.

## 12. Useful commands

| Purpose | Command |
|---|---|
| Check service status | `sudo systemctl status postfix` |
| Restart Postfix | `sudo systemctl restart postfix` |
| View queued messages | `mailq` |
| Retry queued messages | `sudo postqueue -f` |
| View recent mail logs | `sudo tail -n 100 /var/log/mail.log` |
| Rebuild SASL credential map | `sudo postmap /etc/postfix/sasl_passwd` |

## 13. Suggested GitHub repository layout

```text
postfix-gmail-relay/
├── README.md
└── .gitignore
```

Suggested `.gitignore`:

```gitignore
# Never commit local credentials or private configuration
sasl_passwd
sasl_passwd.db
.env
*.pem
```

Before pushing, check that no real passwords, private keys, or personal email credentials are included.

## References

- [Postfix documentation](https://www.postfix.org/documentation.html)
- [Google Account Help: Sign in with app passwords](https://support.google.com/accounts/answer/185833)
