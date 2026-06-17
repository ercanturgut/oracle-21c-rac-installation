2. SELinux Kapatılması
Bash
vi /etc/selinux/config
# Aşağıdaki satırı güncelleyin:
SELINUX=disabled
3. Firewall (Güvenlik Duvarı) Kapatılması
Bash
systemctl stop firewalld
systemctl disable firewalld
4. NTP (Chronyd) Yapılandırması
Node1 ve Node2 üzerinde servisi etkinleştirin ve yeniden başlatın:

Bash
systemctl enable chronyd.service
systemctl restart chronyd.service
5. Paket Yönetimi: RPM Kurulumları
Bash
dnf update -y
dnf install -y bc binutils compat-openssl10 elfutils-libelf glibc glibc-devel ksh libaio libXrender libX11 libXau libXi libXtst libgcc libnsl libstdc++ libxcb libibverbs make policycoreutils policycoreutils-python-utils smartmontools sysstat unixODBC
6. Kullanıcı ve Grupların Oluşturulması
Bash
groupadd -gid 54321 oinstall
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
7. Otomatic Preinstall Paketi Kurulumu
Bash
dnf install -y oracle-database-preinstall-21c
8. ASMLIB Kurulumu ve Servis Başlatma
Bash
dnf install -y kmod-oracleasm
dnf install oracleasm-support

systemctl start oracleasm.service
systemctl enable oracleasm.service
9. Kernel Sabitleme ve Reboot (Gerekliyse)
Mevcut kernel uyumsuzluklarında (Örn: OEL 8.8) kernel sürümünü ayarlayıp sunucuları yeniden başlatın:

Bash
grubby --set-default /boot/vmlinuz-4.18.0-477.27.1.el8_8.x86_64
reboot
10. Sistem Limitlerinin Kontrolü
/etc/security/limits.d/oracle-database-preinstall-21c.conf dosyasının içeriğini doğrulayın.

11. ASM Yetkilendirme İşlemleri
Bash
/etc/init.d/oracleasm configure -i
# Default user: grid
# Default group: asmadmin

# Durum kontrolü ve başlatma
oracleasm status
oracleasm init
12. Disklerin Yapılandırılması (Sadece Node1 üzerinde)
fdisk /dev/sdX komutlarıyla diskleri partition (n -> p -> 1 -> w) yaptıktan sonra ASM disklerini oluşturun:

Bash
oracleasm createdisk DATA1 /dev/sdc1
oracleasm createdisk DATA2 /dev/sdf1
oracleasm createdisk DATA3 /dev/sde1
oracleasm createdisk DATA4 /dev/sdd1
oracleasm createdisk FRA1 /dev/sdg1
oracleasm createdisk FRA2 /dev/sdh1

# Kontrol için:
oracleasm scandisks
oracleasm listdisks
13. SSH Key-Based Authentication (Kullanıcı Eşdeğerliği)
oracle ve grid kullanıcıları için her iki sunucuda karşılıklı şifresiz SSH bağlantısını tanımlayın:

Bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
/usr/bin/ssh-keygen -t rsa

# Key'leri karşılıklı authorized_keys dosyasına scp ile aktarın.
# Doğrulama için:
ssh rac1 date
ssh rac2 date
14. Dizinlerin Oluşturulması ve Yetkilendirme
Bash
mkdir -p /u01/app/21.0.0/grid
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle/product/21.8.0/db_1

chown -R grid:oinstall /u01
chown -R oracle:oinstall /u01/app/oracle
chmod -R 775 /u01/
15. Çevresel Değişkenler (.bash_profile)
oracle kullanıcısı .bash_profile içeriğine .db_home dosyasını source edin.
.db_home içeriği:

Plaintext
export ORACLE_HOSTNAME=rac1.localdomain
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/21.0.0/db_1
export GRID_HOME=/u01/app/21.0.0/grid
export CRS_HOME=$GRID_HOME
export ORACLE_SID=orcl1
export PATH=/usr/sbin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
grid kullanıcısı .bash_profile içeriğine .grid dosyasını source edin.
.grid içeriği:

Plaintext
export ORACLE_SID=+ASM1
export ORACLE_BASE=/u01/app/grid
export ORACLE_HOME=/u01/app/21.0.0/grid
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export PATH=$PATH:$HOME/.local/bin:$HOME/bin:$ORACLE_HOME/bin
Arayüz (GUI) Kurulum Adımları Giriş
Grid Altyapısı (Grid Infrastructure) Kurulumu
cvuqdisk paketini her iki node'a da kurun (/u01/app/21.0.0/grid/cv/rpm).

Ön kontrolü çalıştırın: ./runcluvfy.sh stage pre crsinst -n rac1,rac2 -verbose

/gridSetup.sh komutu ile kurulum sihirbazını başlatın.

Step 1: Configure Oracle Grid Infrastructure for a New Cluster

Step 2: Configure an Oracle Standalone Cluster

Step 3: Cluster Name ve Scan Name bilgilerini girin.

Node Seçimi: İkinci sunucuyu (rac2) ekleyin ve SSH bağlantısını test edin.

Network: Public ve Private network arayüzlerini (ens37 vb.) seçin.

Storage: Use Oracle Flex ASM for storage seçeneğini seçin.

GIMR: Do Not use a GIMR database seçeneğini işaretleyin.

Disk Grubu: DATA isimli disk grubunu oluşturup ilgili diskleri (PROVISIONED) seçin. Redundancy: NORMAL.

Şifreler: SYS ve ASMSNMP şifrelerini belirleyin.

Root Script: Kurulum sonundaki root scriptlerinin otomatik çalışması için root kimlik bilgilerini girebilirsiniz.

Prerequisite Checks: Eksik veya uyarı veren kaynaklar (Örn: RAM/Swap uyarısı) varsa gerekirse IGNORE ALL diyerek geçebilirsiniz.

FRA Disk Grubu Ekleme
Grid kurulumu bittikten sonra grid kullanıcısıyla asmca komutunu çalıştırarak FRA disk grubunu (NORMAL yedeklilikte) oluşturun.

Oracle Database Software & DB Kurulumu
oracle kullanıcısıyla /runInstaller komutunu çalıştırın.

Set Up Software Only seçeneğiyle sadece yazılımı her iki node için kurun.

Kurulum tamamlandıktan sonra dbca (Database Configuration Assistant) komutunu çalıştırın:

Create a database -> Advanced configuration seçin.

Deployment tipi olarak Oracle Real Application Cluster (RAC) database seçin.

Veritabanı adını (orcl) girin ve Storage kısmında ASM (+DATA) seçeneğini işaretleyin.

Character Set olarak TR8MSWIN1254 (veya tercihen UTF8) seçin.

İşlem tamamlandığında Oracle 21c RAC veritabanınız hazır hale gelecektir.
