Markdown# Oracle Linux 8 Üzerinde Oracle 21c RAC Kurulumu

Bu rehber, "ORACLE 21C RAC KURULUMU (21C RAC INSTALLATION) On Oracle Linux 8 _ LinkedIn.pdf" dokümanı referans alınarak hazırlanmış olup, Oracle Linux 8 işletim sistemi üzerinde **Oracle Database 21c Real Application Clusters (RAC)** kurulum adımlarını detaylı bir şekilde içermektedir.

## Proje Bilgileri
* **Yazar:** Ercan Turgut (IT Manager - Oracle & Linux)
* **Tarih:** 11 Aralık 2024

---

## Giriş ve Mimari
Oracle Real Application Cluster (RAC), kesintisiz bir veri tabanı erişimi için hazırlanmış paylaşımlı disk teknolojisini kullanan birden fazla sunucunun tek bir hizmet için çalıştığı küme yapısıdır. Minimum iki sunucu ile çalışan bu yapı sayesinde üretim ortamları maksimum verimde kesintisiz hizmeti hedefler.

### Gereksinimler
* 2 adet tamamen aynı özelliklere sahip Oracle Linux 8 işletim sistemi (Donanım, yazılım, ağ ayarları birebir eşleşmelidir).
* Her işletim sistemine bağlı 2 adet ethernet kartı.
* Her iki işletim sisteminin de görebileceği paylaşımlı disk mimarisi (Shared Disk).

---

## Kurulum Aşamaları

### 1. Host Dosyasının Ayarlanması
Her iki sunucuda da `/etc/hosts` dosyasını aşağıdaki gibi düzenleyin:
```bash
vi /etc/hosts
Plaintext#Public
10.5.x.x racl.localdomain racl
10.5.x.x rac2.localdomain rac2
#Private
10.222.x.x rac1-priv.localdomain rac1-priv
10.222.x.x rac2-priv.localdomain rac2-priv
#Virtual
10.5.x.x rac1-vip.localdomain rac1-vip
10.5.x.x rac2-vip.localdomain rac2-vip
#SCAN IP Adress
10.5.x.x rac-scan.localdomain rac-scan
10.5.x.x rac-scan.localdomain rac-scan
10.5.x.x rac-scan.localdomain rac-scan
2. SELinux KapatılmasıBashvi /etc/selinux/config
# Aşağıdaki satırı güncelleyin:
SELINUX=disabled
3. Firewall (Güvenlik Duvarı) KapatılmasıBashsystemctl stop firewalld
systemctl disable firewalld
4. NTP (Chronyd) YapılandırmasıNode1 ve Node2 üzerinde servisi etkinleştirin ve yeniden başlatın:  Bashsystemctl enable chronyd.service
systemctl restart chronyd.service
5. Paket Yönetimi: RPM KurulumlarıBashdnf update -y
dnf install -y bc binutils compat-openssl10 elfutils-libelf glibc glibc-devel ksh libaio libXrender libX11 libXau libXi libxtst libgcc libnsl libstdc++ libxcb libibverbs make policycoreutils policycoreutils-python-utils smartmontools sysstat unixODBC
6. Kullanıcı ve Grupların OluşturulmasıBashgroupadd -gid 54321 oinstall
groupadd -gid 54322 dba
groupadd -gid 54323 asmdba
groupadd -gid 54324 asmoper
groupadd -gid 54325 asmadmin
groupadd -gid 54326 oper
groupadd -gid 54327 backupdba
groupadd -gid 54328 dgdba
groupadd -gid 54329 kmdba
groupadd -gid 54330 racdba

# Oracle kullanıcısı
useradd --uid 54321 -gid oinstall -groups dba,oper,asmdba,racdba,backupdba,dgdba,kmdba oracle
passwd oracle

# Grid kullanıcısı
useradd --uid 54322 -gid oinstall -groups dba,asmadmin,asmdba,asmoper,racdba grid
passwd grid
7. Otomatik Preinstall Paketi KurulumuBashdnf install -y oracle-database-preinstall-21c
8. ASMLIB Kurulumu ve Servis BaşlatmaBashdnf install -y kmod-oracleasm
dnf install oracleasm-support
systemctl start oracleasm.service
systemctl enable oracleasm.service
9. Kernel Ayarı ve RebootEğer mevcut kernel sürümü uyumsuzsa desteklenen kararlı sürüme çekip sunucuları yeniden başlatın:  Bashgrubby --set-default /boot/vmlinuz-4.18.0-477.27.1.el8_8.x86_64
reboot
10. Sistem Limitlerinin KontrolüAşağıdaki dosyadan limit ayarlarını kontrol edin:  Bashcat /etc/security/limits.d/oracle-database-preinstall-21c.conf
11. ASM Yetkilendirme İşlemleriBash/etc/init.d/oracleasm configure -i
# Default user to own the driver interface []: grid
# Default group to own the driver interface []: asmadmin

# Durum kontrolü ve başlatma
oracleasm status
oracleasm init
12. Disklerin Yapılandırılması (Sadece Node1 üzerinde)fdisk /dev/sdX komutlarıyla diskleri partition (n -> p -> 1 -> w) yaptıktan sonra ASM disklerini oluşturun:  Bashoracleasm createdisk DATA1 /dev/sdc1
oracleasm createdisk DATA2 /dev/sdf1
oracleasm createdisk DATA3 /dev/sde1
oracleasm createdisk DATA4 /dev/sdd1
oracleasm createdisk FRA1 /dev/sdg1
oracleasm createdisk FRA2 /dev/sdh1

# Kontrol etmek için:
oracleasm scandisks
oracleasm listdisks
13. SSH Key-Based Authentication (Şifresiz Geçiş)oracle ve grid kullanıcıları için her iki sunucuda karşılıklı şifresiz SSH bağlantısını tanımlayın:  Bashmkdir -p ~/.ssh
chmod 700 ~/.ssh
/usr/bin/ssh-keygen -t rsa

# Doğrulama komutları:
ssh rac1 date
ssh rac2 date
ssh rac1.localdomain date
ssh rac2.localdomain date
exec /usr/bin/ssh-agent $SHELL
/usr/bin/ssh-add
14. Dizinlerin Oluşturulması ve YetkilendirmeBashmkdir -p /u01/app/21.0.0/grid
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle/product/21.8.0/db_1

chown -R grid:oinstall /u01
chown -R oracle:oinstall /u01/app/oracle
chmod -R 775 /u01/
15. Çevresel Değişkenler (.bash_profile)oracle kullanıcısı için .db_home dosyası içeriği:  Plaintextexport ORACLE_HOSTNAME=rac1.localdomain
export ORACLE_UNONAME=orc11
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/21.0.0/db_1
export GRID_HOME=/u01/app/21.0.0/grid
export CRS_HOME=$GRID_HOME
export ORACLE_SID=orc11
export ORACLE_TERM=xterm
export PATH=/usr/sbin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
grid kullanıcısı için .grid dosyası içeriği:  Plaintextexport ORACLE_SID=+ASM1
export ORACLE_BASE=/u01/app/grid
export ORACLE_HOME=/u01/app/21.0.0/grid
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export TMP=/tmp
export TMPDIR=/tmp
export PATH=$PATH:$HOME/.local/bin:$HOME/bin:$ORACLE_HOME/bin
Grid Infrastructure Kurulum Aşamasıcvuqdisk paketini root kullanıcısı ile tüm node'lara yükleyin:  Bash   cd /u01/app/21.0.0/grid/cv/rpm
   rpm -Uvh cvuqdisk-1.0.18-1.rpm
   scp cvuqdisk-1.0.18-1.rpm rac2:/tmp
Kümenin Grid kurulumuna hazır olup olmadığını Cluvfy ile kontrol edin:  Bash   ./runcluvfy.sh stage pre crsinst -n rac1,rac2 -verbose
grid kullanıcısıyla /gridSetup.sh komutunu çalıştırarak grafik arayüzü başlatın.  Sihirbaz Adımları (GUI):Step 1: Configure Oracle Grid Infrastructure for a New Cluster seçin.  Step 2: Configure an Oracle Standalone Cluster seçeneğini seçin.  Step 3: Cluster Name ve Scan Name (Port: 1521) bilgilerini girin.  Cluster Node Information: İkinci sunucuyu (rac2 ve rac2-vip) listeye ekleyin, SSH bağlantı testini gerçekleştirin.  Network Interface Usage: Network kartlarınızın Public ve Private rollerini belirleyin.  Storage Option: Use Oracle Flex ASM for storage seçeneğini seçin.  GIMR Option: Do Not use a GIMR database diyerek devam edin.  Create ASM Disk Group: DATA disk grubunu oluşturun, Redundancy ayarını NORMAL seçin ve PROVISIONED durumdaki diskleri ekleyin.  Specify ASM Password: SYS ve ASMSNMP kullanıcıları için şifre belirleyin.  Failure Isolation: Varsayılan IPMI seçeneğini (Do not use...) kabul edin.  Management Options: EM (Enterprise Manager) seçimini kaldırıp ilerleyin.  Privileged Operating System Groups: asmadmin, asmdba, asmoper gruplarını eşleştirin.  Root script execution: Root scriptlerinin otomatik yürütülmesi için root şifrenizi tanımlayabilirsiniz.  Prerequisite Checks: Donanım eksiklikleri veya uyarılar gelirse IGNORE ALL seçeneği ile geçiş yapabilirsiniz.  Kurulum bittikten sonra grid kullanıcısıyla asmca komutunu çalıştırarak FRA disk grubunu da ekleyin.  Oracle Database 21c KurulumuYazılım dosyalarını /u01/app/oracle/product/21.0.0/db_1 dizinine kopyaladıktan sonra oracle kullanıcısıyla ./runInstaller komutunu çalıştırın.  Sihirbaz Adımları (GUI):Configuration Option: Set Up Software Only seçeneğini seçin.  Database Installation Option: Oracle Real Application Clusters database installation seçeneğini işaretleyin.  Nodes Selection: Her iki düğümün de (rac1, rac2) seçili olduğundan emin olun.  Database Edition: Enterprise Edition seçeneğini seçin.  Privileged Operating System Groups: İlgili işletim sistemi gruplarını (dba, oper, racdba vb.) atayın.  Database Oluşturma (DBCA)Yazılım kurulumu bittikten sonra oracle kullanıcısıyla dbca komutunu çalıştırın.  Sihirbaz Adımları (GUI):Database Operation: Create a database seçin.  Database Creation Mode: Advanced configuration seçeneğini işaretleyin[cite: 1].Deployment Type: Database type olarak Oracle Real Application Cluster (RAC) database, şablon olarak General Purpose or Transaction Processing seçin[cite: 1].List of Nodes: Her iki node'un da seçili olduğunu doğrulayın[cite: 1].Database Identification: Global Database Name (orcl) alanını doldurun[cite: 1].Storage Option: Storage type olarak Automatic Storage Management (ASM), konum olarak +DATA/{DB_UNIQUE_NAME} seçin[cite: 1].Fast Recovery Option: FRA ve Archivelog yapılandırmasını isteğe göre belirleyin[cite: 1].Configuration Options: Dil ayarlarından karakter setini TR8MSWIN1254 olarak seçin[cite: 1].Management Options: EM Database Express portunu 5500 olarak bırakın[cite: 1].User Credentials: SYS ve SYSTEM kullanıcıları için şifreleri belirleyin[cite: 1].Prerequisite Checks: Çıkan uyarıları IGNORE ALL diyerek geçin ve kurulumu tamamlayın[cite: 1].