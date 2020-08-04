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

test ovs in sandbox environment
-------------------------------
./boot.sh
make
make sandbox or tutorial/ovs-sandbox
ctrl+d or exit 


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

