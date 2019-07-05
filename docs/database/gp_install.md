## Greenplum安装

- 安装OS层面需要的库 (CentOS 7 only)

```bash
# enable network, onboot = on
vi /etc/sysconfig/network-scripts/ifcfg-XXX

# basics
yum install net-tools.x86_64 -y
yum install wget -y
yum install git -y
yum install vim -y

# update yum source
cd /etc/yum.repos.d
mv CentOS-Base.repo CentOS-Base.repo.bak
wget http://mirrors.163.com/.help/CentOS7-Base-163.repo
```

- 新增需要的环境变量

```bash
export INSTALLS=$HOME/usr
export WORKSPACES=$HOME/workspaces

export XERCES_INSTALL=$INSTALLS/A_install_gp_xerces
export LD_LIBRARY_PATH=$XERCES_INSTALL/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$XERCES_INSTALL/lib:$LIBRARY_PATH
export C_INCLUDE_PATH=$XERCES_INSTALL/include:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=$XERCES_INSTALL/include:$CPLUS_INCLUDE_PATH
export CMAKE_LIBRARY_PATH=$XERCES_INSTALL/lib:$CMAKE_LIBRARY_PATH
export CMAKE_INCLUDE_PATH=$XERCES_INSTALL/include:$CMAKE_INCLUDE_PATH
export PATH=$XERCES_INSTALL/bin:$PATH

export ORCA_SRC_FOLDERNAME=gporca
export ORCA_SRC=$WORKSPACES/$ORCA_SRC_FOLDERNAME
export ORCA_INSTALL=$INSTALLS/A_install_gporca
export LD_LIBRARY_PATH=$ORCA_INSTALL/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$ORCA_INSTALL/lib:$LIBRARY_PATH
export C_INCLUDE_PATH=$ORCA_INSTALL/include:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=$ORCA_INSTALL/include:$CPLUS_INCLUDE_PATH

export CFLAGS='-O0 -g'
export GP_SRC_FOLDERNAME=gpdb
export GP_SRC=$WORKSPACES/$GP_SRC_FOLDERNAME
export GP_INSTALL=$INSTALLS/A_install_gpdb
export GP_DATA_FOLDER=$INSTALLS/A_install_data
export MASTER_DATA_DIRECTORY=$GP_DATA_FOLDER/gpseg-1

source $GP_INSTALL/greenplum_path.sh
```

- 下载源代码

```bash
export CMAKE_VERSION="3.13.3"
export NINJA_VERSION="1.9.0"
export XERCES_VERSION="3.1.2-p1"
#export ORCA_VERSION="3.21.1"
#export GPDB_VERSION="5.16.0"
export ORCA_VERSION="3.39.0"
export GPDB_VERSION="6.0.0-beta.3"

# download dependencies
mkdir $HOME/usr
cd $HOME/usr
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
wget https://github.com/ninja-build/ninja/archive/v$NINJA_VERSION.tar.gz -O ninja-$NINJA_VERSION.tar.gz
wget https://github.com/Kitware/CMake/releases/download/v$CMAKE_VERSION/cmake-$CMAKE_VERSION.tar.gz -O cmake-$CMAKE_VERSION.tar.gz
wget https://github.com/greenplum-db/gp-xerces/archive/v$XERCES_VERSION.tar.gz -O gp-xerces-$XERCES_VERSION.tar.gz

# download orca and gpdb
cd $WORKSPACES
wget https://github.com/greenplum-db/gporca/archive/v$ORCA_VERSION.tar.gz -O gporca-$ORCA_VERSION.tar.gz
wget https://github.com/greenplum-db/gpdb/archive/$GPDB_VERSION.tar.gz -O gpdb-$GPDB_VERSION.tar.gz
tar zxvf gporca-$ORCA_VERSION.tar.gz
mv gporca-$ORCA_VERSION $ORCA_SRC_FOLDERNAME
tar zxvf gpdb-$GPDB_VERSION.tar.gz
mv gpdb-$GPDB_VERSION $GP_SRC_FOLDERNAME

# clone orca and gpdb
cd $WORKSPACES
git clone https://github.com/greenplum-db/gporca.git
git clone https://github.com/greenplum-db/gpdb.git
mv gporca $ORCA_SRC_FOLDERNAME
mv gpdb $GP_SRC_FOLDERNAME
```

- 安装依赖 (CentOS 7)

```bash
# python
yum install python-devel.x86_64 -y
python $HOME/usr/get-pip.py
#yum -y install epel-release

# gcc, g++
yum install gcc -y
yum install gcc-c++ -y

# ninja
cd $HOME/usr
tar zxvf ninja-$NINJA_VERSION.tar.gz
cd ninja-$NINJA_VERSION
./configure.py --bootstrap
cp ninja /usr/bin/

# dependencies
yum install readline-devel.x86_64 -y
yum install zlib-devel.x86_64 -y
yum install krb5-devel.x86_64 -y
pip install gssapi
yum install apr-devel.x86_64 -y
yum install libevent-devel.x86_64 -y
yum install libxml2-devel.x86_64 -y
yum install libcurl-devel.x86_64 -y
yum install bzip2-devel.x86_64 -y
yum install bison -y
yum install flex -y
yum install perl-ExtUtils-Embed -y
yum install openssl-devel.x86_64 -y
pip install paramiko
pip install psutil
pip install lockfile
yum install libcurl-devel.x86_64 --enablerepo="city*" -y # no need for CentOS

# dependencies (Greenplum 6 only)
yum install libzstd-devel.x86_64 -y

# dependencies (make check)
yum install perl-Env
yum install 'perl(Data::Dumper)'
```

- 安装依赖 (Ubuntu)

```bash
# python
sudo apt install python -y
sudo apt install python-dev -y
sudo python $HOME/usr/get-pip.py

# gcc, g++, make
sudo apt install gcc -y
sudo apt install g++ -y
sudo apt install make -y

# ninja
cd $HOME/usr
tar zxvf ninja-$NINJA_VERSION.tar.gz
cd ninja-$NINJA_VERSION
./configure.py --bootstrap
sudo cp ninja /usr/bin/

# dependencies
sudo apt install libreadline-dev -y
sudo apt install zlib1g-dev -y
sudo apt install libkrb5-dev -y
sudo pip install gssapi
sudo apt install libapr1-dev -y
sudo apt install libevent-dev -y
sudo apt install libxml2-dev -y
sudo apt install libcurl4-openssl-dev -y
sudo apt install libbz2-dev -y
sudo apt install bison -y
sudo apt install flex -y
sudo apt install libperl-dev -y
sudo apt install libssl-dev -y
sudo pip install paramiko
sudo pip install psutil
sudo pip install lockfile

# dependencies (Greenplum 6 only)
sudo apt install libzstd-dev -y
```

- 安装cmake, gp-xerces, ORCA, GP

```bash
# cmake
cd $HOME/usr
tar zxvf cmake-$CMAKE_VERSION.tar.gz
cd cmake-$CMAKE_VERSION
./bootstrap
make
make install

# gp-xerces (user)
cd $HOME/usr
tar zxvf gp-xerces-$XERCES_VERSION.tar.gz
cd gp-xerces-$XERCES_VERSION
mkdir build
cd build
../configure --prefix=$XERCES_INSTALL
make
make install

# orca (user)
cd $ORCA_SRC
cmake -GNinja -H. -Bbuild -DCMAKE_INSTALL_PREFIX=$ORCA_INSTALL
ninja install -C build

# gp (user)
cd $GP_SRC
./configure --with-perl --with-python --with-libxml --with-gssapi --prefix=$GP_INSTALL --enable-debug --enable-cassert
make -j8
make -j8 install
cat $WORKSPACES/gp_path_exports.txt >> $GP_INSTALL/greenplum_path.sh

# clear gssapt (no need for CentOS)
sudo pip uninstall gssapi
```

- 调整环境参数

```bash
# add hostname to hosts
# sudo echo "127.0.0.1 ${HOSTNAME}" >> /etc/hosts
vim /etc/hosts

# modify $GP_INSTALL/greenplum_path.sh, add environment variables
vim $GP_INSTALL/greenplum_path.sh

# open file limit
vim /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535

# sem(信号量)
vim /etc/sysctl.conf
kernel.sem = 250 512000 100 2048
```

- 新建GP实例

```bash
mkdir $GP_DATA_FOLDER

source $GP_INSTALL/greenplum_path.sh
cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config $GP_DATA_FOLDER/gpinitsystem_config
sed -i "s/MASTER_HOSTNAME=mdw/MASTER_HOSTNAME=${HOSTNAME}/g" $GP_DATA_FOLDER/gpinitsystem_config
sed -i "s#MASTER_DIRECTORY=/data/master#MASTER_DIRECTORY=$GP_DATA_FOLDER#g" $GP_DATA_FOLDER/gpinitsystem_config
sed -i "s#/data1/primary /data1/primary /data1/primary /data2/primary /data2/primary /data2/primary#$GP_DATA_FOLDER $GP_DATA_FOLDER $GP_DATA_FOLDER#g" $GP_DATA_FOLDER/gpinitsystem_config

echo "${HOSTNAME}" > $GP_DATA_FOLDER/hostfile_exkeys
echo "${HOSTNAME}" > $GP_DATA_FOLDER/hostfile_segments

gpssh-exkeys -f $GP_DATA_FOLDER/hostfile_exkeys
gpssh -f $GP_DATA_FOLDER/hostfile_exkeys -e ls -l $GPHOME

rm -Rf $GP_DATA_FOLDER/gpseg*
gpinitsystem -c $GP_DATA_FOLDER/gpinitsystem_config -h $GP_DATA_FOLDER/hostfile_segments
```

- make check

```bash
make installcheck-world
```

