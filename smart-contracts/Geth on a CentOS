#!/bin/bash
#
# This script installs and runs Geth on CentOS 
# and deploy smart contracts on a private Ethereum blockchain.

######### user settings ############

set -xve # turn trace on, exit on first error 

NB_OF_CORES=$(grep -c ^processor /proc/cpuinfo)

CURL_VERSION=7.59.0
#CMAKE_VERSION=3.11.0   # for latest, see: https://cmake.org/files/LatestRelease/
CMAKE_VERSION=$(curl --silent "https://cmake.org/files/LatestRelease/" | grep cmake | tail -1 | grep -Po '(?<=cmake-)\d+.\d+.\d+' | tail -1)
GCC_VERSION=7.3.0       # for latest, see: https://gcc.gnu.org/releases.html 
BOOST_SUBVERSION=66     # for latest, see: https://sourceforge.net/projects/boost/files/boost
GCC_PREFIX_DIR="/usr"

NODE_VERSION=8 # 8 or 9

# See Geth options: https://github.com/ethereum/go-ethereum/wiki/Command-Line-Options
GETH_BIN_PATH="\${HOME}/go-ethereum/build/bin/"
GETH_ROOT_PATH="${HOME}/Private_Chain"
GETH_DATA_DIR="${GETH_ROOT_PATH}/chaindata"
GETH_LOG_PATH="${GETH_ROOT_PATH}/geth.log"
GETH_WORKSPACE="${HOME}/ethereum.workspace"  # location of smart contract code

GETH_PASSWORD_FILE="${GETH_ROOT_PATH}/passwordfile"

GETH_PORT="30301"
GETH_ETHER_BASE_INDEX=0   # the account index among all created accounts (0 for first eth. address, 1 for 2nd, .. ) 

GETH_VERBOSITY_LEVEL=5  # '5' for full details

GETH_NETWORK_ID="1900"
GETH_IDENTITY="private_chain"
GETH_MAX_PEERS="25"

GETH_GAS_PRICE="0"
GETH_TARGET_GAS_LIMIT="9999999"

GETH_IPC_PATH="${GETH_ROOT_PATH}/geth.ipc"
GETH_IPC_API="admin,db,eth,debug,miner,net,shh,txpool,personal,web3"

# Set allowed RPC APIs 
# 'personal' and 'admin' handle account managing operations (change credentials, unlock, create, etc.)
# 'db'   handles the local copy of the block chain
# 'eth'  is an object use to manage transaction related operations (making transactions, retrieving receipts, contract deployment, etc.)
# 'net'  is a network manager module (shows the number of connected peers, allows disconnecting a peer, etc.)
# 'web3' is a JS framework used to communicate with geth through JSON objects received by a client
 
# !! WARNING !!  any peer with remote access to the server can interact with the client and if an account is unlocked 
# an attacker can empty all of its ethers through a transaction sent as a JSON object to the port geth is listening on. 
GETH_RPC_API="personal,admin,db,eth,net,web3,miner,shh,txpool,debug"  
GETH_RPC_ADDR="0.0.0.0"  # If  two nodes are on the same device, use 127.0.0.1. If both nodes are on the same WLAN, use the private IP address of the device.
GETH_RPC_PORT=8545       # default, but can be 8080
GETH_RPCCORS_DOMAIN="*"  # specify a cross domain command line argument to get around the 'same-origin-policy' most browsers implement

GETH_WS_API="personal,admin,db,eth,net,web3,miner,shh,txpool,debug"  
GETH_WS_ADDR="0.0.0.0"
GETH_WS_PORT=8546
GETH_WS_ORIGINS="*"

GENESIS_FILE_PATH="${GETH_ROOT_PATH}/genesis.json"
GENESIS_DIFFICULTY="0x400"
GENESIS_GASLIMIT="0x2fefd8"
GENESIS_NONCE="0x0000000000000042"

ADDR1_BALANCE="0x1337000000000000000000"
ADDR2_BALANCE="0x1937000000000000000000"


######### DO NOT EDIT FROM HERE ############

# This function removes the '$2' last extensions of filename '$1'
# $1: the filename
# $2: the number of extensions to remove
function removeExt { local __a=$1; for (( c=1; c<=$2; c++ )); do __a=${__a%.*}; done; echo "$__a"; }


# This function installs package dependencies for Geth misc. dev tools
installDependencies()
{
 # install Chrome
 sudo bash -c 'cat << EOF > /etc/yum.repos.d/google-chrome.repo
[google-chrome]
name=google-chrome
baseurl=http://dl.google.com/linux/chrome/rpm/stable/x86_64
enabled=1
gpgcheck=1
gpgkey=https://dl.google.com/linux/linux_signing_key.pub
EOF'

sudo yum install -y google-chrome-stable

sudo yum install -y jq # to parse JSON from blockchain client

# for support ssl on curl
sudo yum install -y openssl openssl-devel && \
sudo yum install -y zlib zlib-devel && \
wget https://curl.haxx.se/download/curl-${CURL_VERSION}.tar.gz && \
tar -xzvf curl-${CURL_VERSION}.tar.gz && \
cd curl-${CURL_VERSION}/ && \
./configure --with-ssl --prefix=/usr && \
make && \
sudo make install && \
sudo mv /lib64/libcurl.so.4 /lib64/libcurl.so.4.bak && \
sudo cp curl-${CURL_VERSION}/lib/.libs/libcurl.so* /lib64/ && \
cd .. && rm -rf curl* && \
curl --version

# install cmake 3.x from source (required to build solc)
sudo yum remove cmake -y  `# remove default cmake version 2.x if installed` && \
wget https://cmake.org/files/LatestRelease/cmake-${CMAKE_VERSION}.tar.gz && \
tar -zxvf cmake-${CMAKE_VERSION}.tar.gz && \
cd cmake-${CMAKE_VERSION} && \
./bootstrap --system-curl `#./configure` && \
make && \
sudo make install && \
cd .. && rm -rf cmake-${CMAKE_VERSION}* `# cleanup` && \
source ~/.bash_profile  `# update current shell to path to cmake` && \
cmake --version


# install GCC
# Ref: https://github.com/FezVrasta/ark-server-tools/wiki/Install-of-required-versions-of-glibc-and-gcc-on-RHEL-CentOS
# sudo yum install -y glibc-devel.i686 glibc-i686   # may be needed if default GCC package was removed

ARCHIVE=gcc-${GCC_VERSION}.tar.gz && curl ftp://ftp.mirrorservice.org/sites/sourceware.org/pub/gcc/releases/gcc-${GCC_VERSION}/${ARCHIVE} | tar xvz && \
ARCHIVE=`removeExt ${ARCHIVE} 1` && rm -f ${ARCHIVE} && \
cd gcc-${GCC_VERSION} && \
./contrib/download_prerequisites && \
mkdir build && cd build && \
../configure --disable-multilib --enable-languages=c,c++ --prefix=$GCC_PREFIX_DIR && \
make -j ${NB_OF_CORES} && \
sudo yum remove -y gcc `# remove old GCC` && \
sudo make install && \
cd ../.. && rm -rf gcc-${GCC_VERSION}
which gcc
gcc --version

# CentOS-specific alternative option
# sudo yum install -y centos-release-scl
# sudo yum install -y devtoolset-7-gcc*
# scl enable devtoolset-7 bash
# which gcc
# gcc --version

# install Boost from source 
sudo yum install -y python-devel  # for Python headers otherwise fail to compile.
ARCHIVE=boost_1_${BOOST_SUBVERSION}_0.tar.gz && wget https://sourceforge.net/projects/boost/files/boost/1.${BOOST_SUBVERSION}.0/${ARCHIVE} && \
tar zxvf ${ARCHIVE} && \
ARCHIVE=`removeExt ${ARCHIVE} 1` && rm -f ${ARCHIVE} && \
ARCHIVE=`removeExt ${ARCHIVE} 1` && cd ${ARCHIVE} && \
./bootstrap.sh --prefix=/usr/local && \
sudo ./b2 install --with=all && \
cd .. && sudo rm -rf ${ARCHIVE}*  # sudo needed as b2 is sudoed, cleanup

# install solc from Github source repository s
git clone https://github.com/ethereum/solidity && \
cd solidity && \
mkdir build && cd build && \
cmake .. && \
make all && \
sudo make install && \
cd ../.. && rm -rf solidity # cleanup 


# using solc
# Ref: https://www.mobilefish.com/developer/blockchain/blockchain_quickguide_using_solc.html
cd solidity/std  && \
solc --overwrite -o . --optimize --bin --ast --asm --abi mortal.sol  `# compile a sample smart contract` && \
ls mortal* && \
cd ../..


 # install solcjs
 # ref: https://www.rosehosting.com/blog/how-to-install-node-js-and-npm-on-centos-7/
 sudo yum install -y make epel-release
 curl --silent --location https://rpm.nodesource.com/setup_${NODE_VERSION}.x | sudo bash -
 sudo yum install -y nodejs
 node -v
 npm -v
 sudo npm install -g solc # install solcjs, NOT solc
 solcjs --version

 # dependencies for Geth 
 sudo yum install -y golang gmp-devel
}

 
# This function builds Geth from source and updates the user's PATH accordingly   
installGeth()
 {
  git clone https://github.com/ethereum/go-ethereum
   
  cd go-ethereum/ 
  make all
  echo "export PATH=\$PATH:${GETH_BIN_PATH}" >> ~/.bash_profile && source ~/.bash_profile
  
  geth version
}


# This function creates Ethereum accounts (EOA)  
createExternallyOwnAccount()
 {
  geth account new --datadir ${GETH_DATA_DIR} --password ${GETH_ROOT_PATH}/passwordfile |awk -F '[{}]' '{print $2}'
 }


# This function generates a 'Genesis.json' file, initializes Geth and creates Ethereum external accounts (EOA)  
#  !! Warning !!: deletes config files, logs, and blockchain data from previous runs
initGeth()
{
rm -rf ${GETH_ROOT_PATH}
mkdir -p ${GETH_DATA_DIR}

cat << EOF > ${GENESIS_FILE_PATH}
{
  "coinbase"   : "0x0000000000000000000000000000000000000001",
  "difficulty" : "0x600",
  "extraData"  : "",
  "gasLimit"   : "0x2fefa8",
  "nonce"      : "0x0000010000000042",
  "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp"  : "0x00",
  "alloc": {},
  "config": {
        "chainId": 15,
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
    }
}
EOF

# updates properties: difficulty, gas limit, and nonce  
sed -i.${DATE}.old '/difficulty/s|"0x[0-9a-fA-F]*"|"'${GENESIS_DIFFICULTY}'"|g;/gasLimit/s|"0x[0-9a-fA-F]*"|"'${GENESIS_GASLIMIT}'"|g;/nonce/s|"0x[0-9a-fA-F]*"|"'${GENESIS_NONCE}'"|g' ${GENESIS_FILE_PATH}

geth \
       --identity    "${GETH_IDENTITY}" \
       --nodiscover \
       --networkid     ${GETH_NETWORK_ID} \
       --datadir     ${GETH_DATA_DIR} \
       init ${GENESIS_FILE_PATH}

if [ -e ${GETH_PASSWORD_FILE} ]
then
    echo "password file: ${GETH_PASSWORD_FILE} already exists, do nothing"
else
    echo "Creating password file: ${GETH_PASSWORD_FILE}"
    cat << EOF > ${GETH_PASSWORD_FILE}
passer
EOF
fi

ADDR1=$(createExternallyOwnAccount)
echo $ADDR1

ADDR2=$(createExternallyOwnAccount)
echo $ADDR2

sed -i.${DATE}.old '/alloc/s|{}|{'\\n'          "'$ADDR1'":{ "balance": "'${ADDR1_BALANCE}'" },'\\n'          "'$ADDR2'":{ "balance": "'${ADDR2_BALANCE}'" }'\\n'         }|g' ${GENESIS_FILE_PATH}

#geth removedb --datadir ${GETH_DATA_DIR}
echo "Removing datadir db"
rm -rf ${GETH_DATA_DIR}/geth/*

geth \
     --identity    "${GETH_IDENTITY}" \
     --nodiscover \
     --networkid   ${GETH_NETWORK_ID} \
       --datadir   ${GETH_DATA_DIR} \
       init        ${GENESIS_FILE_PATH}

}


# This function kills previous Geth instances if any and start a new one  
startGeth() 
 {
  # kill previous geth instances if any 
  DATE=`date +%Y%m%d_%H%M%S`

  if test -n `ps -ef | grep 'geth' | grep -v grep | awk '{print $2}'`  # test if string length != 0 
   then
    echo "Geth is alreay running, trying to stop the process.."
    killall -q --signal SIGINT geth && sleep 10  # kill previous process if any 
    rm -f ${GETH_IPC_PATH}                       # clear IPC as this may cause problems
    if [ -e ${GETH_LOG_PATH} ]
    then
      mv ${GETH_LOG_PATH} ${GETH_LOG_PATH}.old.${DATE}
    fi
     
   fi

  echo "Starting Geth with verbosity ${GETH_VERBOSITY_LEVEL}... "

  (nohup geth \
       --datadir     ${GETH_DATA_DIR} \
       --mine \
       --nodiscover \
       --nat "any" \
       --identity       ${GETH_IDENTITY} \
       --networkid      ${GETH_NETWORK_ID} \
       --port           ${GETH_PORT} \
       --verbosity      ${GETH_VERBOSITY_LEVEL} \
       --ipcpath        ${GETH_IPC_PATH} \
       --rpc --rpcaddr  ${GETH_RPC_ADDR} --rpcport ${GETH_RPC_PORT} --rpccorsdomain "${GETH_RPCCORS_DOMAIN}" --rpcapi   "${GETH_RPC_API}" \
       --ws   --wsaddr  ${GETH_WS_ADDR}  --wsport  ${GETH_WS_PORT}  --wsorigins "${GETH_WS_ORIGINS}"         --wsapi "${GETH_WS_API}" \
       --maxpeers       ${GETH_MAX_PEERS} \
       --etherbase      ${GETH_ETHER_BASE_INDEX} \
       --gasprice       ${GETH_GAS_PRICE} \
       --targetgaslimit ${GETH_TARGET_GAS_LIMIT} \
       2>> ${GETH_LOG_PATH}) & 
 }  


# script root function
main()
 {
  #installDependencies
  #installGeth
  initGeth
  startGeth
  echo "Done."
 }

# script entry point
main "$@"
