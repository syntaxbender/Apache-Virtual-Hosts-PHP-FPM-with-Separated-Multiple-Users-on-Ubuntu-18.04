# Apache Virtual Hosts, PHP-FPM with Separated Multiple Users on Ubuntu 18.04

## Nedir?
Bu döküman birden fazla linux kullanıcısı ve domain barındıran bir sunucu senaryosunda nasıl güvenli bir şekilde kurulum yapılabileceğini anlatmaktadır. Bu döküman adım adım takip edildiğinde;

 - Apache
 - PHP 7.2
	 - php7.2-mysql
	 - php7.2-curl
	 - php7.2-mbstring
	 - php7.2-fpm
 - MySQL
	 - MySQL Server 8.0
	 - MySQL Apt Config
 - phpmyadmin

paketlerine sahip olan, php koşabilen bir web sunucusuna sahip olacaksınız.

## Kurulum

```bash
apt update
apt upgrade
apt install gnupg
apt install software-properties-common
apt install apache2
wget https://repo.mysql.com/mysql-apt-config_0.8.16-1_all.deb
dpkg -i mysql-apt-config_0.8.16-1_all.deb
```
```bash
wget https://repo.mysql.com/mysql-apt-config_0.8.16-1_all.deb
dpkg -i mysql-apt-config_0.8.16-1_all.deb
apt update
apt install mysql-server
mysql_secure_installation
```

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/mysql/1.jpg)
![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/mysql/2.jpg)

Yukarıdaki resimde reposu eklenecek mysql araçlarını seçmelisiniz. Default haliyle uygundur.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/mysql/3.jpg)
![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/mysql/4.jpg)

MySQL root şifresini belirliyoruz.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/mysql/5.jpg)

phpmyadmin tarzı bazı uygulamalarla uyumlu olarak sorunsuz çalışması için 5.x seçmelisiniz.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/mysql/6.jpg)

Resimdeki gibi ayarlayabilirsiniz.
```bash
apt install curl php7.2 php7.2-mysql php7.2-curl php7.2-mbstring php7.2-fpm
a2enmod proxy
a2enmod proxy_fcgi
useradd -m -d /home/USER/ -s /bin/bash -c -U USER -G sudo
mkdir /home/USER/public_html
chmod -R 750 /home/USER/
chown -R USER:USER /home/USER/
nano /etc/group
```
Web sunucumuz için gerekli paketlerden bazılarını kurduk, php-fpm'in unix soketi ile apache'yi haberleştirebilmek adına proxy ve proxy_fcgi modüllerini etkinleştirdik. Ayrıca domaini host edilecek olan ilk linux kullanıcımızı da oluşturduk. Dikkat çekilmesi gereken bir nokta var. linux kullanıcımızı oluştururken sudo grubuna dahil ederek shell alma yetkisi de tanımlamış olduk. İhtiyaçlarınıza göre shell yetkisi vermeyebilir, sudo grubuna dahil etmeyebilirsiniz.

**/etc/group**
```
USER:x:1000:www-data
```
Group dosyasında kullanıcımızın grubuna www-data kullanıcısını eklemiş oluyoruz. Bu sayede apache web sunucusu kullanıcımızın dizinindeki dosyayı okuma yetkisine sahip oluyor.
```bash
nano /etc/apache2/sites-available/DOMAIN.COM.conf
```
Apache'ye domainimizi tanıtacak olan konfigürasyon dosyasını oluşturuyoruz. Aşağıdakine benzer bir konfigürasyona sahip olmalısınız.

**/etc/apache2/sites-available/DOMAIN.COM.conf**
```
<VirtualHost *:80>  
 
ServerAdmin mail@domain.com  
ServerName domain.com  
ServerAlias www.domain.com   
DocumentRoot /home/USER/public_html
 
<Directory /home/USER/public_html>  
Options -Indexes  
AllowOverride All  
Require all granted  
<FilesMatch ".+\.php$">  
	<If "-f %{REQUEST_FILENAME}">  
		SetHandler "proxy:unix:/run/php/USER.sock|fcgi://localhost"  
	</If>  
</FilesMatch>  
</Directory>  
 
ErrorLog ${APACHE_LOG_DIR}/error.log  
CustomLog ${APACHE_LOG_DIR}/access.log combined  
 
#Include conf-available/serve-cgi-bin.conf  
</VirtualHost>
```
Apache domain konfigürasyon dosyamızda esas yapmamız gereken değişim, gelen isteklerde php uzantısına sahip olan ve dizinde varolan php dosyaları için php-fpm unix soketi ile haberleşmesini sağlamak adına aşağıdaki satırı eklemek.
```
<FilesMatch ".+\.php$">  
	<If "-f %{REQUEST_FILENAME}">  
		SetHandler "proxy:unix:/run/php/USER.sock|fcgi://localhost"  
	</If>  
</FilesMatch>
```
Dipnot: Farkettiyseniz <Directory /home/USER/public_html>  dizini altında unix soketi ile haberleşilmesini sağladım. Birazdan aşağıda yapacağım phpmyadmin fpm konfigürasyonu ile çakışması muhtemeldir. O yüzden daha stabil bir kurulum yapmak adına Her kullanıcı için ayrı ayrı dizinler altında soket ile haberleşmeyi sağlamak gerekir. 
```bash
nano /etc/php/7.2/fpm/pool.d/USER.conf
```
php-fpm soketi için gerekli konfigürasyon dosyasını oluşturuyoruz. Aşağıdakine benzer bir konfigürasyona sahip olmalısınız.

**/etc/php/7.2/fpm/pool.d/USER.conf**
```
[USER]  
listen = /run/php/USER.sock  
listen.owner = USER  
listen.group = USER  
listen.mode = 0660  
 
user = USER  
group = USER  
 
pm = dynamic  
pm.max_children = 50  
pm.start_servers = 2  
pm.min_spare_servers = 1  
pm.max_spare_servers = 10  
```
```bash
mysql -u root -p
```
Yeni linux kullanıcımıza bir adet MySQL kullanıcısı da oluşturmak ve ilerleyen adımlarda kurulumunu yapacağımız phpmyadmin paneli için de bir MySQL kullanıcısı oluşturmak adına MySQL sunucumuza en yüksek yetkili root kullanıcısı ile login oluyoruz.
```sql
CREATE USER 'user'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PASS';
CREATE DATABASE dbname CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
GRANT ALL PRIVILEGES ON dbname.* TO 'user'@'localhost';

CREATE USER 'phpmyadmin'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PASS';
GRANT ALL PRIVILEGES ON * . * TO 'phpmyadmin'@'localhost';

FLUSH PRIVILEGES;
```
Yukarıda yer alan sql kod bloğu ile her bir linux kullanıcısı için ayrı bir mysql kullanıcısı oluşturduk. Ayrıca ilerleyen adımlarda kurulumunu yapacağımız phpmyadmin paneli kurulumu için phpmyadmin MySQL sunucusu kullanıcısını oluşturduk.
```bash
a2ensite DOMAIN.COM
a2enmod rewrite
add-apt-repository ppa:phpmyadmin/ppa
apt update
```
Yukarıdaki bash kodları ile de apache web sunucusu üzerinde domainimizi aktive ettik. Ayrıca kullanıcıların htaccess rewrite kuralları ile seo uyumlu url adresleri oluşturabilmesi ve her linux kullanıcısının kendi dizini içerisinde htaccess dosyaları kullanabilmesi için rewrite modülünü aktive ettik. Sonrasında da phpmyadmin için gerekli güncel repoyu ekledik.
```bash
apt install phpmyadmin
```
![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/phpmyadmin/1.jpg)

Apache'yi klavyeden space  tuşu ile işaretleyip tab tuşuyla "\<Ok\>" butonuna geçiş yapıp enter ile onaylıyoruz.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/phpmyadmin/2.jpg)

Bu soruyu ise "\<No\>" şeklinde yanıtlıyoruz.

Phpmyadmin paketi kurulumunu tamamladık. http://domain.com/phpmyadmin adresine gidersek phpmyadmin'in php kaynak kodlarını olduğu gibi ekrana çıktı verdiğini göreceğiz bu durumu da çözmek adına phpmyadmin apache konfigürasyon dosyasını açıyoruz.
```
nano /etc/apache2/conf-available/phpmyadmin.conf
```
**/etc/apache2/conf-available/phpmyadmin.conf**
```
Alias /phpmyadmin /usr/share/phpmyadmin  
  
<Directory /usr/share/phpmyadmin>  
	Options SymLinksIfOwnerMatch  
	DirectoryIndex index.php  
	  
	# limit libapache2-mod-php to files and directories necessary by pma  
	<FilesMatch ".+\.php$">  
		SetHandler "proxy:unix:/run/php/php7.2-fpm.sock|fcgi://localhost"  
	</FilesMatch>  
	<IfModule mod_php7.c>  
		php_admin_value upload_tmp_dir /var/lib/phpmyadmin/tmp  
		php_admin_value open_basedir /usr/share/phpmyadmin/:/etc/phpmyadmin/:/var/lib/phpmyadmin/:/usr/share/php/php-gettext/:/usr/share/php/php-php-gettext/:/usr/share/javascript/:/usr/share/php/tcpdf/:/usr/share/doc/phpmyadmin/:/usr/s$  
	</IfModule>
</Directory>  
  
# Disallow web access to directories that don't need it  
<Directory /usr/share/phpmyadmin/templates>  
Require all denied  
</Directory>  
<Directory /usr/share/phpmyadmin/libraries>  
Require all denied  
</Directory>
```
Burada domain konfigürasyon dosyası oluştururken olduğu gibi önemli nokta;
```
<FilesMatch ".+\.php$">  
	SetHandler "proxy:unix:/run/php/php7.2-fpm.sock|fcgi://localhost"  
</FilesMatch> 
```
gelen isteklerdeki php dosyalarının fpm soketine iletilmesidir.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/phpmyadmin/3.jpg)

Oluşturduğumuz phpmyadmin mysql kullanıcısı ile phpmyadmin'e login oluyoruz.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/phpmyadmin/4.jpg)

Yeşil ile işaretli alana tıklıyoruz.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/phpmyadmin/5.jpg)

Yeşil ile işaretli alana tıklıyoruz.

![alt text](https://raw.githubusercontent.com/syntaxbender/syntaxbender/main/assets/imgs/sistem-apache-seperate-users/phpmyadmin/6.jpg)

Phpmyadmin'in kendi veritabanını da başarıyla oluşturduktan sonra yukarıdaki gibi bir ekran bizi karşılıyor.

Kurulumu başarıyla tamamlamış olmamız gerekiyor. Sonraki yazılarım kuvvetle muhtemel kernel hardening, dos/ddos attack mitigating, letsencrypt certbot kullanımı, spf, dkim records, dns server, postfix, dovecot, roundcube gibi konular üzerine olacak. Takipte ve sağlıcakla kalmanız dileğiyle.


