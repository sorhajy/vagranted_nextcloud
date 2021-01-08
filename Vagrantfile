# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_EXPERIMENTAL'] = "disks"

Vagrant.configure("2") do |config|
  config.vm.box = "debian/stretch64"
  config.vm.box_check_update = false
  config.vm.network "forwarded_port", guest: 80, host: 80
  config.vm.network "forwarded_port", guest: 443, host: 443

  config.vm.disk :disk, size: "500GB", name: "Data"

  config.vm.provision :file, source: 'Nginx/nextcloud.conf', destination: "/tmp/nginx_nextcloud.conf"
  config.vm.provision :file, source: 'Apache/nextcloud.conf', destination: "/tmp/apache_nextcloud.conf"

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.cpus = 2
    vb.memory = "1024"

    # Configure hardware virtualisation
    vb.customize ["modifyvm", :id, "--hwvirtex", "on"]
    vb.customize ["modifyvm", :id, "--vtxvpid", "on"]
    vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
  end

  # Edit ENV here:
  config.vm.provision "shell", env: {"db_name" => "nextcloud", "db_user" => "user", "db_password" => "password", "db_host" => "localhost"}, inline: <<-SHELL
    apt-get update
    apt-get install -y parted
    parted /dev/sdb mktable msdos --align=opt
    mkfs.ext4 /dev/sdb -F

    apt-get install -y apt-transport-https lsb-release ca-certificates curl
    wget -q -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
    sh -c 'echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list'
    apt-get update
    apt-get install -y apache2

    apt-get install -y  openssl
    openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout /etc/ssl/private/selfsigned.key -out /etc/ssl/certs/selfsigned.crt -subj '/CN=ND/O=ND/C=ND'

    apt-get install -y default-mysql-server default-mysql-client
    echo "CREATE DATABASE $db_name;" | sudo su -c "mysql"
    echo "GRANT ALL PRIVILEGES ON $db_name.* TO '$db_user'@'$db_host' IDENTIFIED BY '$db_password';" | sudo su -c "mysql"

    apt-get install -y php7.4 php7.4-fpm php7.4-gd php7.4-sqlite3 php7.4-curl php7.4-zip php7.4-xml php7.4-mbstring php7.4-mysql php7.4-bz2 php7.4-intl php7.4-smbclient php7.4-imap php7.4-gmp

    a2enconf php7.4-fpm
    a2enmod proxy_fcgi setenvif
    service apache2 restart

    mkdir -p /var/www/nextcloud
    mount /dev/sdb /var/www/nextcloud
    mkdir -p /var/www/nextcloud/data
    cd /tmp
    curl -s https://download.nextcloud.com/server/releases/latest.tar.bz2 -o /tmp/nextcloud_latest.tar.bz2
    bzip2 -d nextcloud_latest.tar.bz2
    tar -xf nextcloud_latest.tar
    mv /tmp/nextcloud/* /var/www/nextcloud
    chown -R www-data:www-data /var/www/nextcloud/
    chmod 750 /var/www/nextcloud/data

    mv /tmp/apache_nextcloud.conf /etc/apache2/sites-available/nextcloud.conf
    a2ensite nextcloud.conf
    systemctl reload apache2

    sleep 1
    curl -s 127.0.0.1/nextcloud/index.php > /dev/null
    sleep 1

    sed "s|0 => 'localhost',|0 => '*',|g"  -i /var/www/nextcloud/config/config.php
    chmod 0644 /var/www/nextcloud/config/config.php
    sed "s|post_max_size = 8M|post_max_size = 102400M|g" -i /etc/php/*/fpm/php.ini
    sed "s|upload_max_filesize = 2M|upload_max_filesize = 102400M|g" -i /etc/php/*/fpm/php.ini
    systemctl restart php7.4-fpm.service

    systemctl disable apache2
    systemctl stop apache2

    apt-get install -y nginx-full
    systemctl enable nginx
    systemctl start nginx

    rm /etc/nginx/sites-enabled/default
    mv /tmp/nginx_nextcloud.conf /etc/nginx/sites-available/nextcloud
    ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
    systemctl restart nginx

    systemctl restart php7.4-fpm

  SHELL
end