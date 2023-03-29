**Размещаем свой RPM в своем репозитории**

**Задание**

- создать свой RPM (можно взять свое приложение, либо собрать к примеру апач с определенными опциями);
- создать свой репо и разместить там свой RPM;
- реализовать это все либо в вагранте, либо развернуть у себя через nginx и дать ссылку на репо.

**Выполнение:**

- VPN сервера не имею, поэтому необходимый образ скачиваю с сайта https://app.vagrantup.com/centos/boxes/8

- Добавляю вручную образ:
```
vagrant box add --name centos/8 /home/neon/Downloads/CentOS-8-Vagrant-8.3.2011-20201204.2.x86_64.vagrant-virtualbox.box
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'centos/8' (v0) for provider: 
    box: Unpacking necessary files from: file:///home/neon/Downloads/CentOS-8-Vagrant-8.3.2011-20201204.2.x86_64.vagrant-virtualbox.box
==> box: Successfully added box 'centos/8' (v0) for 'virtualbox'!

root@neon-desktop:/test_vm# vagrant box list
centos/8                        (virtualbox, 0)
```
- правлю под него Vagrantfile (во вложении) и запускаю командой 
```
vagrant up > otus_hw6_repo
```

**Скрипт с выполнением ДЗ, и вывод всех команд в приложенных файлах: [install_repo.sh](https://github.com/hellolightSP/otus_hw6/blob/main/install_repo.sh),
[otus_hw6_repo](https://github.com/hellolightSP/otus_hw6/blob/main/otus_hw6_repo)**


- add mirrors to repo (без изменения зеркала из коробки vagrant/Centos8 не устанавливаются пакеты)

```
sudo sed -i -e "s|mirrorlist=|#mirrorlist=|g" /etc/yum.repos.d/CentOS-*
sudo sed -i -e "s|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g" /etc/yum.repos.d/CentOS-*
```

- install required packages
```
yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils gcc
```

- download nginx src
```
cd /root/
wget https://nginx.org/packages/centos/8/SRPMS/nginx-1.20.2-1.el8.ngx.src.rpm
rpm -i nginx-1.*
```

- download OpenSSL
```
wget https://github.com/openssl/openssl/archive/refs/heads/OpenSSL_1_1_1-stable.zip
unzip OpenSSL_1_1_1-stable.zip
```
- install dependencys (Здесь заранее выложил в свой github repo ```nginx.spec``` с параметром ```--with-openssl=/root/openssl-OpenSSL_1_1_1-stable``` в ```%define BASE_CONFIGURE_ARGS``` т.к. в методичке приложен nginx.spec для версии nginx-1.14, а мы скачиваем пакет nginx-1.20.2-1.el8.ngx.src.rpm)
```
yum-builddep -y /root/rpmbuild/SPECS/nginx.spec
wget https://raw.githubusercontent.com/hellolightSP/otus_hw6/main/nginx.spec
rm -f /root/rpmbuild/SPECS/nginx.spec
mv nginx.spec /root/rpmbuild/SPECS/
```
- rpmbuild nginx
```
rpmbuild -bb /root/rpmbuild/SPECS/nginx.spec
ll /root/rpmbuild/RPMS/x86_64/
```

- install compiled package
```
yum localinstall -y /root/rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm
systemctl start nginx
systemctl status nginx
nginx -V
```

- create repo (Дополнительно добавил в свой репозиторий пакет ansible с зависимостями, которых нет в appstream и baseos репозитории)
```
mkdir /usr/share/nginx/html/repo

cp /root/rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm /usr/share/nginx/html/repo/

wget http://mirror.centos.org/centos/8-stream/AppStream/x86_64/os/Packages/ansible-core-2.12.2-3.el8.x86_64.rpm -O /usr/share/nginx/html/repo/ansible-core-2.12.2-3.el8.x86_64.rpm
wget http://mirror.centos.org/centos/8-stream/AppStream/aarch64/os/Packages/python38-resolvelib-0.5.4-5.el8.noarch.rpm -O /usr/share/nginx/html/repo/python38-resolvelib-0.5.4-5.el8.noarch.rpm
wget http://mirror.centos.org/centos/8-stream/AppStream/x86_64/os/Packages/sshpass-1.09-4.el8.x86_64.rpm -O /usr/share/nginx/html/repo/sshpass-1.09-4.el8.x86_64.rpm
wget https://cbs.centos.org/kojifiles/packages/ansible/5.6.0/2.el8/noarch/ansible-5.6.0-2.el8.noarch.rpm -O /usr/share/nginx/html/repo/ansible-5.6.0-2.el8.noarch.rpm

createrepo /usr/share/nginx/html/repo/

cat <<EOF > /etc/nginx/conf.d/default.conf
server {
    listen       80;
    server_name  localhost;

    location / {
      root /usr/share/nginx/html;
      index index.html index.htm;
      autoindex on;
      }


    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
EOF

nginx -t
nginx -s reload
sleep 10
curl -a http://localhost/repo/
```

- enable repo
```
cat << EOF >> /etc/yum.repos.d/otus.repo
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF

yum repolist enabled | grep otus
yum list | grep otus
yum clean all
yum install ansible -y
```
- for update repo use "createrepo /usr/share/nginx/html/repo/"
