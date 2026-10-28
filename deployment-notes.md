# Deployment Notes: PrestaShop on AWS

## About These Notes

These notes summarise the main configuration steps from my completed deployment. They were reconstructed from the project records rather than exported as a complete terminal history.

I used documentation, research and AI guidance during the assessment. Addresses and credentials are replaced with placeholders.

## 1. Connect to the EC2 Instance

I connected to the Ubuntu server using SSH and my private key.

```powershell
ssh -i "<PATH_TO_PRIVATE_KEY>" ubuntu@<EC2_PUBLIC_IP>
```

The private key is not included in this repository.

## 2. Install the Web Server and PHP Dependencies

The application server used Apache and PHP 8.3 with the extensions required for the installation.

```bash
sudo apt update
sudo apt install apache2 unzip curl mysql-client \
  php8.3 libapache2-mod-php8.3 php8.3-cli php8.3-mysql \
  php8.3-curl php8.3-gd php8.3-intl php8.3-mbstring \
  php8.3-xml php8.3-zip php8.3-bcmath
```

The MySQL client allowed me to connect to RDS. The database server itself ran separately on RDS.

## 3. Adjust PHP Settings

I configured the following settings in a dedicated Apache PHP configuration file:

```ini
memory_limit = 512M
max_execution_time = 300
upload_max_filesize = 32M
post_max_size = 32M
max_input_vars = 5000
allow_url_fopen = On
```

I restarted Apache after applying the settings.

These were installation settings for this lab, not a complete production hardening configuration.

## 4. Add Swap Space

I added 2 GB of swap space to support installation on the small EC2 instance.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

I also added an entry to `/etc/fstab` so the swap configuration would persist:

```text
/swapfile none swap sw 0 0
```

Swap supplements memory; it does not replace adequate RAM.

## 5. Prepare the Application Files

I downloaded the PrestaShop 9.2.0 Classic distribution and extracted the application into:

```text
/var/www/prestashop
```

The downloaded distribution contained a nested application archive, which also needed to be extracted.

I assigned the application files to the Apache service account:

```bash
sudo chown -R www-data:www-data /var/www/prestashop
```

This supported installation. A production deployment would need a more detailed review of writable directories and file ownership.

## 6. Configure Apache

I created `/etc/apache2/sites-available/prestashop.conf`.

The configuration below uses a placeholder for the original server address:

```apache
<VirtualHost *:80>
    ServerName <EC2_PUBLIC_IP>
    DocumentRoot /var/www/prestashop

    <Directory /var/www/prestashop>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/prestashop_error.log
    CustomLog ${APACHE_LOG_DIR}/prestashop_access.log combined
</VirtualHost>
```

I enabled URL rewriting and the application site, disabled the default site, and checked the configuration:

```bash
sudo a2enmod rewrite
sudo a2ensite prestashop.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

The configuration test returned `Syntax OK`.

The website used HTTP. HTTPS was not configured during this assessment.

## 7. Configure Database Connectivity

The RDS database had public access disabled.

Its security group allowed inbound MySQL traffic on TCP port 3306 from the security group attached to the application server.

I downloaded the AWS RDS certificate bundle:

```bash
curl -o /home/ubuntu/global-bundle.pem \
  https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

I tested the connection from EC2 using certificate and hostname verification:

```bash
mysql -h <RDS_ENDPOINT> -P 3306 -u dbadmin -p \
  --ssl-mode=VERIFY_IDENTITY \
  --ssl-ca=/home/ubuntu/global-bundle.pem
```

The `-p` option prompts for the password rather than including it in the command.

This test verified the command-line connection. It did not independently verify TLS for the PrestaShop application's database connection.

## 8. Create the Application Database and User

I created a database and a dedicated application user.

The SQL below shows the structure of the configuration with placeholders:

```sql
CREATE DATABASE prestashop
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'prestashop_user'@'<EC2_PRIVATE_IP>'
  IDENTIFIED BY '<APPLICATION_DATABASE_PASSWORD>';

GRANT ALL PRIVILEGES ON prestashop.*
  TO 'prestashop_user'@'<EC2_PRIVATE_IP>';
```

The application user had permissions on the PrestaShop database rather than administrator access across the database server.

The password placeholders are not real credentials.

## 9. Complete Installation

I opened the browser installer, entered the RDS connection details and completed the PrestaShop installation.

After installation, I removed the installer directory:

```bash
sudo rm -rf /var/www/prestashop/install
```

This was a targeted removal of the installation directory after successful setup.

## 10. Verify the Deployment

I checked that:

- The storefront loaded with demo products.
- The administration dashboard was accessible.
- The storefront returned an HTTP 200 response.
- The installer directory had been removed.
- RDS public access was disabled.
- The database access rule referenced the application security group.

Example commands for the checks:

```bash
sudo apache2ctl configtest
sudo systemctl status apache2 --no-pager
test ! -d /var/www/prestashop/install && echo "Installer removed"
curl -I http://<EC2_PUBLIC_IP>
```

## Troubleshooting and Lessons Learned

### Certificate bundle

The database connection initially failed because the required certificate bundle was unavailable. Downloading the AWS RDS certificate bundle allowed me to validate the connection.

### Application database credentials

I corrected the application database user's password configuration and checked its permissions before continuing.

### Memory constraints

The small instance required additional swap space during setup. This highlighted the importance of understanding resource requirements.

### Administrator access

I used the PrestaShop command-line password reset tool to restore administrator access:

```bash
cd /var/www/prestashop
sudo -u www-data php bin/console \
  prestashop:employee:change-password <ADMIN_EMAIL> --env=prod
```

## Further Improvements

- Configure HTTPS.
- Verify TLS for application-to-database traffic.
- Review file permissions and credential storage.
- Configure centralised logging and alerts.
- Test backup restoration.
- Review patching and resource costs.

## Evidence

See the [project README](README.md#project-screenshots) for screenshots of the storefront, private database configuration and database access rule.
