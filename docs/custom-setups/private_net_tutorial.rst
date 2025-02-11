Raiden on Private Network Tutorial
##################################

Introduction
============

This tutorial shows how to run Raiden on a private network, using the ``master`` branch (this is useful when you are working on a pull-request).  This tutorial assumes Ubuntu 18.04.2 LTS and ``bash``.

Creating a Virtual Environment
==============================

In a shell, run

.. code:: bash

 $ sudo apt-get install libncurses5-dev
 $ mkdir priv_chain

If ``mkdir`` fails, choose a different name, or move the existing ``priv_chain`` directory to somewhere else.

.. code:: bash

 $ cd priv_chain
 $ virtualenv -p python3.7 env
 $ source env/bin/activate

You should now be in the virtual environment, where all Python package installations are separately managed.

Now the command prompt should look like:

.. code:: bash

 (env) $


Install Raiden and dependencies
===============================

.. code:: bash

 (env) $ pwd
 <snip>/priv_chain
 (env) $ git clone https://github.com/raiden-network/raiden
 (env) $ cd raiden
 (env) $ pip install pip-tools
 (env) $ make install-dev

Launch a private network
========================

Installing Geth
---------------

Follow `the guide <https://geth.ethereum.org/docs/install-and-build/installing-geth>`__ and install Geth. A command ``geth`` should be available in your shell. This guide assumes version 1.10.6, but other versions might work.

Preparing a genesis config
--------------------------

Prepare a file ``genesis.json`` with the following content.

.. code:: bash

 (env) $ cd ..
 (env) $ pwd
 <snip>/priv_chain
 (env) $ cat genesis.json
 {
  "config": {
    "chainId": 4321,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "ByzantiumBlock": 0,
    "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "ConstantinopleBlock": 0,
    "PetersburgBlock": 0
  },
  "alloc": {},
  "difficulty": "0x1",
  "gasLimit": "0x9880000"
 }

Starting a chain
----------------

With the ``genesis.json`` you can initialize a blockchain.

.. code:: bash

 (env) $ pwd
 <snip>/priv_chain
 (env) $ geth --datadir blkchain1 init genesis.json
 (env) $ geth --rpc --datadir blkchain1 --networkid 4321 --rpcapi "eth,net,web3" console
 <snip>
 > personal.newAccount()
 "0xd4de892c06cf4a0557c7d515f79fd20b8356d6cf"

Copy the shown address somewhere and shutdown geth. And start mining on your own private blockchain
using the following command so that geth creates a stable stream of blocks at regular intervals.

.. code:: bash

 (env) $ geth --rpc --datadir blkchain1 --networkid 4321 --rpcapi "eth,net,web3" --allow-insecure-unlock --mine --miner.threads=1 --miner.etherbase=0xd4de892c06cf4a0557c7d515f79fd20b8356d6cf

In this console ``geth`` should keep running.

In another console navigate to the `priv_chain` folder and
an IPC connection can be established with geth using:

.. code:: bash

 geth attach ipc:blkchain1/geth.ipc

Figure out the contract version
===============================

Open a new console, and load the Python environment.

.. code:: bash

 $ pwd
 <snip>/priv_chain
 $ source env/bin/activate
 (env) $

Figure out the value ``CONTRACTS_VERSION``

.. code:: bash

 (env) $ cd raiden
 (env) $ grep 'CONTRACTS_VERSION = ' -r ../env/lib/python3.7/site-packages/raiden_contracts
 ../env/lib/python3.7/site-packages/raiden_contracts/constants.py:CONTRACTS_VERSION = "0.37.0"

Copy the shown version somewhere.

Define constants
================

The contract version will be used quite often, so let bash remember it.

.. code:: bash

 (env) $ export VERSION="0.37.0"

You will need your private key for the account you created.

.. code:: bash

 (env) $ cd ..
 (env) $ pwd
 <snip>/priv_chain
 (env) $ export PRIV_KEY=./blkchain1/keystore/UTC-<use TAB-completion to fill in>

If the TAB-completion shows more than two files, something has gone wrong. In that case, back up all files and start over.

The biggest 256-bit unsigned int is a useful default as deposit limits and the max number of TokenNetwork contracts.

.. code:: bash

 (env) $ export MAX_UINT256=115792089237316195423570985008687907853269984665640564039457584007913129639935

The RPC connection point is used often.

.. code:: bash

 (env) $ export PROVIDER="http://127.0.0.1:8545"


Deploy contracts
================

Now we can start deploying the Raiden smart contracts on the private chain.

.. code:: bash

 (env) $ pwd
 <snip>/priv_chain
 (env) $ python -m raiden_contracts.deploy raiden --rpc-provider $PROVIDER --private-key $PRIV_KEY --gas-price 10 --gas-limit 6000000 --contracts-version $VERSION --max-token-networks $MAX_UINT256
 {
     "SecretRegistry": "0x6436d3B7205F18044a320403b1Cd0FfFd7e5D998",
     "TokenNetworkRegistry": "0xC5e4a9189ac801077317CD6BCFA643677897D15B"
 }

We will use these addresses later, so let's remember them.

.. code:: bash

 (env) $ export TokenNetworkRegistry="0xC5e4a9189ac801077317CD6BCFA643677897D15B"
 (env) $ export SecretRegistry="0x6436d3B7205F18044a320403b1Cd0FfFd7e5D998"

Before we deploy the other contracts, we need a token contract for service payments.

.. code:: bash

 (env) $ python -m raiden_contracts.deploy token --rpc-provider $PROVIDER --private-key $PRIV_KEY --gas-price 10 --gas-limit 6000000 --token-supply 10000000000 --token-name ServiceToken --token-decimals 18 --token-symbol SVT --contracts-version $VERSION
 {
    "CustomToken": "0xC5e9F7407359d1492d515C303A3aeDB434D3f0e1"
 }

We use the address of this token to deploy service contracts.

First note down the address that you generated earlier on the geth console using:

.. code:: bash

 > eth.accounts
 ["0xb97d3df9f9d112e1fe8f95f9608ffbd52e07a995"]

Then deploy the service contracts

.. code:: bash

 (env) $ export ServiceRegistryController="0xb97d3df9f9d112e1fe8f95f9608ffbd52e07a995"
 (env) $ export SERVICE_TOKEN="0xC5e9F7407359d1492d515C303A3aeDB434D3f0e1"
 (env) $ python -m raiden_contracts.deploy services --rpc-provider $PROVIDER --private-key $PRIV_KEY --gas-price 10 --gas-limit 6000000 --token-address $SERVICE_TOKEN --user-deposit-whole-limit $MAX_UINT256 --service-deposit-bump-numerator 5 --service-deposit-bump-denominator 4 --service-deposit-decay-constant 100000000 --initial-service-deposit-price 100000000000 --service-deposit-min-price 1000 --service-registration-duration 234000000 --contracts-version $VERSION --token-network-registry-address $TokenNetworkRegistry --service-registry-controller $ServiceRegistryController

From the output, we remember the address of the UserDeposit contract.

.. code:: bash

 (env) $ export UserDeposit="0x50E5f50b98a78615163E89A65fD60D551933CaE2"


We deploy another Token contract that's going to be transferred on Raiden network.

.. code:: bash

 (env) $ python -m raiden_contracts.deploy token --rpc-provider $PROVIDER --private-key $PRIV_KEY --gas-price 10 --gas-limit 6000000 --token-supply 10000000000 --token-name Token --token-decimals 18 --token-symbol TKN --contracts-version $VERSION
 {
     "CustomToken": "0x818cBB172D1a1b769acaA94e80e4c71ba40bdc79"
 }

We register this token to the TokenNetworkRegistry.

.. code:: bash

 (env) $ export TOKEN="0x818cBB172D1a1b769acaA94e80e4c71ba40bdc79"
 (env) $ python -m raiden_contracts.deploy register --rpc-provider $PROVIDER --private-key $PRIV_KEY --gas-price 10 --gas-limit 6000000 --token-address $TOKEN --token-network-registry-address $TokenNetworkRegistry --contracts-version $VERSION --channel-participant-deposit-limit 10000000 --token-network-deposit-limit 1000000000

Start Raiden Client
===================

In ``geth`` console, figure out the deployer's address.

.. code:: bash

 > web3.toChecksumAddress(eth.accounts[0])
 "0x35ebA3Dc57D2A66D378638B19A7CEb194dc29eb6"


Find the relevant contract addresses.

.. code:: bash

 (env) $ export DeployerAddress="0x35ebA3Dc57D2A66D378638B19A7CEb194dc29eb6"

Store the password associated with the private key.

.. code:: bash

 (env) $ echo "password" > passwd_file

And you can start the Raiden client:

.. code:: bash

   (env) $ raiden --datadir exchange-a  --keystore-path   ./blkchain1/keystore/ --network-id 4321  --accept-disclaimer --address $DeployerAddress --rpc --api-address 0.0.0.0:5001 --web-ui  --environment-type development  --console --no-sync-check --accept-disclaimer --user-deposit-contract-address $UserDeposit --routing-mode private --password-file passwd_file
