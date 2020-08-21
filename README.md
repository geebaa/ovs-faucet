# ovs-faucet
experiment with ovs + faucet

Steps below are on 4-aug-2020 on ubuntu 19.10
---------------------------------------------

ovs from source
----------------
git clone https://github.com/openvswitch/ovs.git
cd ovs/
sudo apt-get install -y autoconf
sudo apt-get install -y libssl-dev
sudo apt-get install -y libcap-ng-utils
sudo apt-get install -y unbound
sudo apt install libtool-bin

if kernel module is required to be build from source
----------------------------------------------------
./configure --with-linux=/lib/modules/$(uname -r)/build
this translates to using the current installed kernel for example "/lib/modules/5.3.0-64-generic/build/"

test ovs in sandbox environment
-------------------------------
./boot.sh
make
make sandbox or tutorial/ovs-sandbox
ctrl+d or exit 

where are the builds?
----------------------
kernel module (if configured) - datapath/linux/openvswitch.ko
user space openvswitch -- vswitchd/ovs-vswitchd

Install the built openvswitch in the running system
---------------------------------------------------
sudo make install

where to check installation?
----------------------------
ls -l /usr/local/sbin/ovs-vswitchd
should give the time when this file was created for verification

Install the kernel module built
-------------------------------
make modules_install
seeing a lot of ssl errors ? like 

"At main.c:160:
- SSL error:02001002:system library:fopen:No such file or directory: ../crypto/bio/bss_file.c:72"

This is because the kernel is configured with "CONFIG_MODULE_SIG=y" and therefore "make modules_install"
is trying to sign the openvswitch.ko  and unable to do so since there are no certificates installed to sign with.
nevertheless this is just a warning and proceed to the next step to load

Load the kernel module
----------------------
sudo /sbin/modprobe openvswitch

how to check if kernel module load suceeded ?
---------------------------------------------
/sbin/lsmod | grep "openvswitch" 
should not be empty.. would show up something like below

openvswitch           180224  0
nf_nat                 40960  1 openvswitch
nf_conntrack          139264  2 nf_nat,openvswitch
nf_defrag_ipv6         24576  2 nf_conntrack,openvswitch
udp_tunnel             16384  1 openvswitch
libcrc32c              16384  3 nf_conntrack,nf_nat,openvswitch

how to start openvswitch
------------------------
openvswitch consists of 2 code components ovsdb-server and ovs-vswitchd
it has a utility script ovs-ctl to start the entire system in proper order
because openvswitch was built from source and installed , one will have to
pay attention to the folder structure
first export the path to the proper folder

export PATH=$PATH:/usr/local/share/openvswitch/scripts
then a check with  "which ovs-ctl" should result in 

"/usr/local/share/openvswitch/scripts/ovs-ctl"

now a execution of "ovs-ctl start" throws up an error as below

 * /usr/local/etc/openvswitch/conf.db does not exist
ovsdb-tool: I/O error: /usr/local/etc/openvswitch/conf.db: failed to lock lockfile (Resource temporarily unavailable)
 * Creating empty database /usr/local/etc/openvswitch/conf.db

though it claims to create "/usr/local/etc/openvswitch/conf.db" but it would not since
it has no permission to do so.

manually create the required file using
ovsdb-tool create /usr/local/etc/openvswitch/conf.db     vswitchd/vswitch.ovsschema

now try starting using "ovs-ctl start" . it would fail again this time for permissions as below

install: cannot create directory '/usr/local/var/log': Permission denied
install: cannot create directory '/usr/local/var/run': Permission denied
nice: cannot set niceness: Permission denied
 * Starting ovsdb-server

start using sudo as below
 sudo /usr/local/share/openvswitch/scripts/ovs-ctl start
 
 and you should see this start fine with below messages
 
  * Starting ovsdb-server
 * system ID not configured, please use --system-id
 * Configuring Open vSwitch system IDs
 * Starting ovs-vswitchd
 * Enabling remote OVSDB managers

how to check if everything is running ?
--------------------------------------------------
use the ovs-ctl status command and see the output below

ovsdb-server is running with pid 26969
ovs-vswitchd is running with pid 26990

you should also find the logs for the above daemons as below

/usr/local/var/log/openvswitch/ovs-vswitchd.log
/usr/local/var/log/openvswitch/ovsdb-server.log

what version are the daemons ?
------------------------------
use the command "ovs-ctl version"
ovsdb-server (Open vSwitch) 2.14.90
ovs-vswitchd (Open vSwitch) 2.14.90


faucet from source with docker
------------------------------
git clone https://github.com/faucetsdn/faucet.git
cd faucet/
latest_tag=$(git describe --tags $(git rev-list --tags --max-count=1))
git checkout $latest_tag
sudo docker build -t faucet/faucet -f Dockerfile.faucet .

run faucet
----------
mkdir inst
sudo docker run -d --name faucet --restart=always -v $(pwd)/inst/:/etc/faucet/ -v $(pwd)/inst/:/var/log/faucet/ -p 6653:6653 -p 9302:9302 faucet/faucet
cat inst/faucet.log
docker exec faucet pkill -HUP -f faucet.faucet
sudo docker exec faucet pkill -HUP -f faucet.faucet

