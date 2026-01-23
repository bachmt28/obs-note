## 1. Thực hiện nâng cấp JAVA lên 17 trên các server
## 2. Bản cũ
```bash
#cat /etc/environment 
OWS_JAVA_HOME=/java/jdk1.8.0_202
JAVA_HOME=/java/jdk1.8.0_202


#cat /etc/profile
JAVA_HOME=/java/jdk1.8.0_202
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME

$ ls -l /usr
total 184
dr-xr-xr-x.   2 root root 36864 Feb 20  2024 bin
drwxr-xr-x.   2 root root     6 Jun 21  2021 games
drwxr-xr-x.  43 root root  8192 Feb 20  2024 include
lrwxrwxrwx    1 root root    16 Oct  3 16:06 jdk -> /java/jdk1.8.0_202

```
## 3. Bản mới sẽ nâng cấp
```bash
#cat /etc/environment 
OWS_JAVA_HOME=/opt/jdk-17.0.16
JAVA_HOME=/opt/jdk-17.0.16


#cat /etc/profile
JAVA_HOME=/opt/jdk-17.0.16
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME

$ ls -l /usr
total 184
dr-xr-xr-x.   2 root root 36864 Feb 20  2024 bin
drwxr-xr-x.   2 root root     6 Jun 21  2021 games
drwxr-xr-x.  43 root root  8192 Feb 20  2024 include
lrwxrwxrwx    1 root root    16 Oct  3 16:06 jdk -> /opt/jdk-17.0.16

```
## 4. Chuẩn bị bộ cài và cài
```bash
#Donwload java hoặc vào các server MT nâng cấp tại thư mục /opt để copy lên
jdk-17.0.16_linux-x64_bin.tar.gz
```
Giải nén
```sh
cd /opt
gunzip -c jdk-17.0.16_linux-x64_bin.tar.gz | tar -xvf -
chown -R bin:bin jdk-17.0.16
chmod -R 755 jdk-17.0.16
```
4.1 Sửa biến môi trường `/etc/environment` về thành
```conf
OWS_JAVA_HOME=/opt/jdk-17.0.16
JAVA_HOME=/opt/jdk-17.0.16
```
4.2 Sửa biến môi trường `/etc/profile` về thành
```sh
JAVA_HOME=/opt/jdk-17.0.16
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME
```
4.3 Sửa link trong /usr thành
```sh
cd /usr
rm -rf jdk
ln -s  /opt/jdk-17.0.16 jdk
```
4.4 Rem biến môi trường trong /root/.bash_profile lại
```sh
# cat .bash_profile 
# .bash_profile
# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs
PATH=$PATH:$HOME/bin
export PATH
#export JAVA_HOME=/opt/jdk-17.0.12
#export PATH=$JAVA_HOME/bin:$PATH
```