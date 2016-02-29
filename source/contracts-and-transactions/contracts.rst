********************************************************************************
Contracts
********************************************************************************

What is a Contract?
================================================================================
A contract is a collection of code (its functions) and data (its state) that resides at a specific address on the Ethereum blockchain. They govern the behaviour of accounts within the Ethereum state. Contract accounts are able to pass messages between themselves as well as doing practically Turing complete computation. Contracts live on the blockchain in a Ethereum-specific binary format called Ethereum Virtual Machine (EVM) bytecode. However, contracts are typically written in some high level language such as `Solidity <https://solidity.readthedocs.org/en/latest/>`_ and then compiled into byte code to be uploaded on the blockchain. Note that other languages also exist, notably Serpent and LLL, which are described further in the `Ethereum High Level Languages`_ section of this documentation.

Write a Contract 
================================================================================
No language would be complete without a Hello World program. Operating within the Ethereum environment, Solidity has no obvious way of "outputting" a string. The closest we can do is to use a log event to place a string into the blockchain:

.. code:: js

	contract HelloWorld
	{
  		event Print(string out);
  		function() { Print("Hello, World!"); }
	}

This contract, if placed on the blockchain and called, would create a log entry on the blockchain of type Print with a parameter "Hello, World!". Easy, eh?

Visit the `Solidity documentation <https://solidity.readthedocs.org/en/latest/>`_ for more examples and guidelines to writing Solidty code.

Compile a Contract 
================================================================================

``geth`` and ``eth++`` supports solidity compilation through a system call to ``solc``, the command line solidity compiler. A faster/easier way to compile Solidity code is to use the `online Solidity realtime compiler <https://chriseth.github.io/browser-solidity/>`_ or the `Mix IDE <https://github.com/ethereum/wiki/wiki/Mix:-The-DApp-IDE>`_ program. More information on solc and compiling Solidity contract code can be found `here <https://solidity.readthedocs.org/en/latest/frequently-asked-questions.html#how-do-i-compile-contracts>`_.

Using solc in geth
--------------------------------------------------------------------------------
If you start up your ``geth`` node, you can check if the solidity compiler is available.
This is what happens, if it is not:

.. code:: bash

    > eth.compile.solidity("")
    eth_compileSolidity method not available: solc (solidity compiler) not found
        at InvalidResponse (<anonymous>:-57465:-25)
        at send (<anonymous>:-115373:-25)
        at solidity (<anonymous>:-104109:-25)
        at <anonymous>:1:1

`After you find a way to install solc <https://github.com/ethereum/go-ethereum/wiki/Building-Ethereum>`_,  make sure it's in the
path. If `eth.getCompilers\(\) <https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethgetcompilers>`__ still does not find solc (returns an empty array), you can set a custom path to the ``solc`` executable on the command line using th ``solc`` flag.

.. code:: console

    geth --datadir ~/homestead/00 --solc /usr/local/bin/solc --natspec

Alternatively, you can set this option at runtime via the console:

.. code:: bash

    > admin.setSolc("/usr/local/bin/solc")
    solc v0.9.32
    Solidity Compiler: /usr/local/bin/solc
    Christian <c@ethdev.com> and Lefteris <lefteris@ethdev.com> (c) 2014-2015
    true

Let's compile a simple contract source:

.. code:: bash

    > source = "contract test { function multiply(uint a) returns(uint d) { return a * 7; } }"

This contract offers a unary method: called with a positive integer
``a``, it returns ``a * 7``.

You are ready to compile solidity code in the ``geth`` JS console using `eth\.compile\.solidity\(\) <https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethcompilesolidity>`_:

.. code:: bash

    > contract = eth.compile.solidity(source).test
    {
      code: '605280600c6000396000f3006000357c010000000000000000000000000000000000000000000000000000000090048063c6888fa114602e57005b60376004356041565b8060005260206000f35b6000600782029050604d565b91905056',
      info: {
        language: 'Solidity',
        languageVersion: '0',
        compilerVersion: '0.9.13',
        abiDefinition: [{
          constant: false,
          inputs: [{
            name: 'a',
            type: 'uint256'
          } ],
          name: 'multiply',
          outputs: [{
            name: 'd',
            type: 'uint256'
          } ],
          type: 'function'
        } ],
        userDoc: {
          methods: {
          }
        },
        developerDoc: {
          methods: {
          }
        },
        source: 'contract test { function multiply(uint a) returns(uint d) { return a * 7; } }'
      }
    }

The compiler is also available via `RPC <https://github.com/ethereum/wiki/wiki/JSON-RPC>`__ and therefore via `web3\.js <https://github.com/ethereum/wiki/wiki/JavaScript API#web3ethcompilesolidity>`__ to any in-browser Ðapp connecting to ``geth`` via RPC/IPC.

The following example shows how you interface ``geth`` via JSON-RPC to
use the compiler.

.. code:: bash

    ./geth --datadir ~/eth/ --loglevel 6 --logtostderr=true --rpc --rpcport 8100 --rpccorsdomain '*' --mine console  2>> ~/eth/eth.log
    curl -X POST --data '{"jsonrpc":"2.0","method":"eth_compileSolidity","params":["contract test { function multiply(uint a) returns(uint d) { return a * 7; } }"],"id":1}' http://127.0.0.1:8100

The compiler output for one source will give you contract objects each representing a single contract. The actual return value of ``eth.compile.solidity`` is a map of contract name -- contract object pairs. Since our contract's name is ``test``, ``eth.compile.solidity(source).test`` will give you the contract object for the test contract containing the following fields:

-  ``code``: the compiled EVM code
-  ``info``: the rest of the metainfo the compiler outputs
-  ``source``: the source code
-  ``language``: contract language (Solidity, Serpent, LLL)
-  ``languageVersion``: contract language version
-  ``compilerVersion``: compiler version
-  ``abiDefinition``: `Application Binary Interface Definition <https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`__
-  ``userDoc``: `NatSpec Doc <https://github.com/ethereum/wiki/wiki/Ethereum-Natural-Specification-Format>`__
-  ``developerDoc``: `NatSpec Doc <https://github.com/ethereum/wiki/wiki/Ethereum-Natural-Specification-Format>`__

The immediate structuring of the compiler output (into ``code`` and ``info``) reflects the two very different **paths of deployment**. The compiled EVM code is sent off to the blockchain with a contract creation transaction while the rest (info) will ideally live on the decentralised cloud as publicly verifiable metadata complementing the code on the blockchain.

If your source contains multiple contracts, the output will contain an entry for each contact, the corresponding contract info object can be retrieved with the name of the contract as attribute name. You can try this by inspecting the most current GlobalRegistrar code:

.. code:: js

    contracts = eth.compile.solidity(globalRegistrarSrc)

Create and Deploy a Contract
================================================================================
Before you begin this section, make sure you have both an unlocked account as well as some funds.
You will now create a contract on the blockchain by `sending a transaction <https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethsendtransaction>`__ to the empty address with the EVM code from the previous section as data. Again, note that this can be accomplished much easier using the `online Solidity realtime compiler <https://chriseth.github.io/browser-solidity/>`_ or the `Mix IDE <https://github.com/ethereum/wiki/wiki/Mix:-The-DApp-IDE>`_ program.

.. code:: js

    primaryAddress = eth.accounts[0]
    MyContract = eth.contract(abi);
    contract = MyContract.new(arg1, arg2, ...,{from: primaryAddress, data: evmCodeFromPreviousSection})

All binary data is serialised in hexadecimal form. Hex strings always have a hex prefix ``0x``.

Note that ``arg1, arg2, ...`` are the arguments for the contract constructor, in case it accepts any.

Also note that this step requires you to pay for execution. Your balance on the account (that you put as sender in the ``from`` field) will be reduced according to the gas rules of the VM once your transaction makes it into a block. After some time, your transaction should appear included in a block confirming that the state it brought about is a consensus. Your contract now lives on the blockchain.

The asynchronous way of doing the same looks like this:

.. code:: js

    MyContract.new([arg1, arg2, ...,]{from: primaryAccount, data: evmCode}, function(err, contract) {
      if (!err && contract.address)
        console.log(contract.address); 
    });

Interacting with a Contract
================================================================================
`eth.contract\(\) <https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethcontract>`_ can be used to define a contract *class* that will comply with the contract interface as described in its `ABI definition <https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`_.

.. code:: js

    var Multiply7 = eth.contract(contract.info.abiDefinition);
    var myMultiply7 = Multiply7.at(address);

Now all the function calls specified in the ABI are made available on the contract instance. You can just call those methods on the contract instance and chain ``sendTransaction(3, {from: address})`` or ``call(3)`` to it. The difference between the two is that ``call`` performs a "dry run" locally, on your computer, while ``sendTransaction`` would actually submit your transaction for inclusion in the block chain and the results of its execution will eventually become part of the global consensus. In other words, use ``call``, if you are interested only in the return value and use ``sendTransaction`` if you only care about "side effects" on the state of the contract. Using ``call`` does not cost any gas, so it is better to use call when you need to retrieve a contract state and not modify the state.

In the example above, there are no side effects, therefore ``sendTransaction`` only burns gas and increases the entropy of the universe. ``call`` is demonstrated below:

.. code:: js

    myMultiply7.multiply.call(6)
    42

Contract Metadata
================================================================================
Now suppose this contract is not yours, and you would like documentation or to look at the source code. This is made possible by making available the contract info bundle and registering it on the blockchain or through a third party service, such as `EtherChain <https://www.etherchain.org/contracts>`_. The ``admin`` API provides convenience methods to fetch this bundle for any contract that chose to register.

.. code:: js

    // get the contract info for contract address to do manual verification
    var info = admin.getContractInfo(address) // lookup, fetch, decode
    var source = info.source;
    var abiDef = info.abiDefinition

In the previous sections we explained how you create a contract on the blockchain. Now we deal with the rest of the compiler output, the **contract metadata** or contract info. The idea is that

-  contract info is uploaded somewhere identifiable by a ``url`` which
   is publicly accessible
-  anyone can find out what the ``url`` is only knowing the contracts
   address

These requirements are achieved very simply by using a 2 step blockchain registry. The first step registers the contract code (hash) with a content hash in a contract called ``HashReg``. The second step registers a url with the content hash in the ``UrlHint`` contract. These `simple registry contracts <https://github.com/ethereum/go-ethereum/blob/develop/common/registrar/contracts.go>`__ were part of the Frontier release and have carried on into Homestead.

By using this scheme, it is sufficient to know a contract's address to look up the url and fetch the actual contract metadata info bundle.

So if you are a conscientious contract creator, the steps are the following:

1. Deploy the contract itself to the blockchain
2. Get the contract info json file.
3. Deploy contract info json file to any url of your choice
4. Register codehash ->content hash -> url

The JS API makes this process very easy by providing helpers. Call ``admin.register`` to extract info from the contract, write out its json serialisation in the given file, calculates the content hash of the file and finally registers this content hash to the contract's code hash. Once you deployed that file to any url, you can use ``admin.registerUrl`` to register the url with your content hash on the blockchain as well. (Note that in case a fixed content addressed model is used as document store, the url-hint is no longer necessary.)

.. code:: js

    source = "contract test { function multiply(uint a) returns(uint d) { return a * 7; } }"
    // compile with solc
    contract = eth.compile.solidity(source).test
    // create contract object
    var MyContract = eth.contract(contract.info.abiDefinition)
    // extracts info from contract, save the json serialisation in the given file, 
    contenthash = admin.saveInfo(contract.info, "~/dapps/shared/contracts/test/info.json")
    // send off the contract to the blockchain
    MyContract.new({from: primaryAccount, data: contract.code}, function(error, contract){
      if(!error && contract.address) {
        // calculates the content hash and registers it with the code hash in `HashReg`
        // it uses address to send the transaction. 
        // returns the content hash that we use to register a url
        admin.register(primaryAccount, contract.address, contenthash)
        // here you deploy ~/dapps/shared/contracts/test/info.json to a url
        admin.registerUrl(primaryAccount, hash, url)
      }
    });

Testing Contracts and Transactions
================================================================================

Often you need to resort to a low level strategy of testing and debugging contracts and transactions. This section introduces some debug tools and practices you can use. In order to test contracts and transactions without real-word consequences, you best test it on a private blockchain. This can be achieved with configuring an alternative network id (select a unique integer) and/or disable peers. It is recommended practice that for testing you use an alternative data directory and ports so that you never even accidentally clash with your live running node (assuming that runs using the defaults. Starting your ``geth`` with in VM debug mode with profiling and highest logging verbosity level is recommended:

.. code:: bash

    geth --datadir ~/dapps/testing/00/ --port 30310 --rpcport 8110 --networkid 4567890 --nodiscover --maxpeers 0 --vmdebug --verbosity 6 --pprof --pprofport 6110 console 2>> ~/dapp/testint/00/00.log

Before you can submit any transactions, you need set up your private test chain. See `this section <https://ethereum-homestead.readthedocs.org/en/latest/developing-on-ethereum/test-networks.html>`_.

.. code:: js

    // create account. will prompt for password
    personal.newAccount("mypassword");
    // name your primary account, will often use it
    primary = eth.accounts[0];
    // check your balance (denominated in ether)
    balance = web3.fromWei(eth.getBalance(primary), "ether");

.. code:: js

    // assume an existing unlocked primary account
    primary = eth.accounts[0];

    // mine 10 blocks to generate ether 

    // starting miner
    miner.start(8);
    // sleep for 10 blocks.
    admin.sleepBlocks(10);
    // then stop mining (just not to burn heat in vain)
    miner.stop()  ;
    balance = web3.fromWei(eth.getBalance(primary), "ether");

After you create transactions, you can force process them with the following lines:

.. code:: js

    miner.start(1);
    admin.sleepBlocks(1);
    miner.stop()  ;

You can check your pending transactions with:

.. code:: js

    // shows transaction pool
    txpool.status
    // number of pending txs
    eth.getBlockTransactionCount("pending");
    // print all pending txs
    eth.getBlock("pending", true).transactions

If you submitted contract creation transaction, you can check if the desired code actually got inserted in the current blockchain:

.. code:: js

    txhash = eth.sendTansaction({from:primary, data: code})
    //... mining
    contractaddress = eth.getTransactionReceipt(txhash);
    eth.getCode(contractaddress)



web3.js
================================================================================

RPC
================================================================================