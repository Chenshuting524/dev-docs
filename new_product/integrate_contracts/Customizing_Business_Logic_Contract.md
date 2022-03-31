<h1 align="center">Guidelines for Developing</h1>

Chains must only be deployed with one CCM contract to implement cross-chain features. For normal running, all the business logic contracts have to interconnect with the CCM contract via the interfaces offered by the CCM contract. See the following for a detailed description or reference to the complete code of the CCM contract.


### Step1. Mapping asset

The business contract should maintain both the **asset mapping** and **business logic contract mapping**. 
With the chain ID mapped to the business logic contract address on the target chain, asset hash is mapped from the source chain to the target chain.

#### Example:

The asset mapping relationship stored in the business logic contract will help complete transaction data. Binding actions also prevent the wrong input from users, leading to the transfer of assets to the incorrect asset contract address.

```solidity
pragma solidity ^0.5.0;

import "./../../libs/ownership/Ownable.sol";

contract LockProxy is Ownable {
    address public managerProxyContract;
    mapping(uint64 => bytes) public proxyHashMap;
    mapping(address => mapping(uint64 => bytes)) public assetHashMap;
    
    // toChainId: the target chain id
    // targetProxyHash: the address of business logic contract on target chain
    function bindProxyHash(uint64 toChainId, bytes memory targetProxyHash) onlyOwner public returns (bool) {
        proxyHashMap[toChainId] = targetProxyHash;
        emit BindProxyEvent(toChainId, targetProxyHash);
        return true;
    }
    
    // fromAssetHash: asset hash on source chain 
    // toAssetHash: asset hash on target chain
    function bindAssetHash(address fromAssetHash, uint64 toChainId, bytes memory toAssetHash) onlyOwner public returns (bool) {
        assetHashMap[fromAssetHash][toChainId] = toAssetHash;
        emit BindAssetEvent(fromAssetHash, toChainId, toAssetHash, getBalanceFor(fromAssetHash));
        return true;
    }
}
```

### Step2. Requesting transaction

Now, you should complete a method to invoke the cross-chain function in the CCM. You can also see specifics in the [source code](https://dev-docs.poly.network/new_chain/side_chain/contracts.html#step3-pushing-transactions). 


````solidity
/*  
 *  @param toChainId              The target chain id
 *  @param toAddress              The address in bytes format to receive the same amount of tokens in the target chain
 *  @param toContract             Target smart contract address in bytes in the target blockchain
 *  @param txData                 Transaction data for target chain, include toAssetHash, toAddress, amount
 *  @return                       true or false 
*/
function crossChain (uint64 toChainId, bytes calldata toContract, bytes calldata method, bytes calldata txData) whenNotPaused external returns (bool)
````

- Function `crossChain`, including information of **target chain ID**, **target address**, **transaction hash** and **transaction data**, is invoked by `Lock` and emits `LockEvent` that contains  **asset contract address** on the source chain, **target chain ID**, **target address,** and **amount of token** to be transferred.
- Then put the hash of `rawParam` into storage to prove the cross-chain transaction.

#### Example:

```solidity
/*  
 *  @param fromAssetHash     The asset address in the current chain
 *  @param toChainId         The target chain id
 *  @param toAddress         The address in bytes format to receive the same amount of tokens in the target chain 
 *  @param amount            The number of tokens to be crossed from Ethereum to the chain with chainId
*/
function lock(address fromAssetHash, uint64 toChainId, bytes memory toAddress, uint256 amount) public payable returns (bool) {
    require(amount != 0, "amount cannot be zero!");
    require(_transferToContract(fromAssetHash, amount), "transfer asset from fromAddress to lock_proxy contract failed!");
        
    bytes memory toAssetHash = assetHashMap[fromAssetHash][toChainId];
    require(toAssetHash.length != 0, "empty illegal toAssetHash");

    TxArgs memory txArgs = TxArgs({
        toAssetHash: toAssetHash,
        toAddress: toAddress,
        amount: amount
    });
    bytes memory txData = _serializeTxArgs(txArgs);
        
    IEthCrossChainManagerProxy eccmp = IEthCrossChainManagerProxy(managerProxyContract);
    address eccmAddr = eccmp.getEthCrossChainManager();
    IEthCrossChainManager eccm = IEthCrossChainManager(eccmAddr);
        
    bytes memory toProxyHash = proxyHashMap[toChainId];
    require(toProxyHash.length != 0, "empty illegal toProxyHash");
    require(eccm.crossChain(toChainId, toProxyHash, "unlock", txData), "EthCrossChainManager crossChain executed error!");

    emit LockEvent(fromAssetHash, _msgSender(), toChainId, toAssetHash, toAddress, amount);
        
    return true;
    
}
```

- By calling this method, the business logic contract will **lock** a certain amount of valuable assets.


### Step3. Verifying and executing

You need to deploy a methods to parse and execut the transaction information transferred by `verifyHeaderAndExecuteTx()`. 
- `verifyHeaderAndExecuteTx()` in CCM contracts determines the **legitimacy** of cross-chain transaction information and resolves the parameters of transaction data from the Poly chain transaction Merkle proof and `crossStateRoot` contained in the block header. 

````solidity
/*  
 *  @param proof                  Poly chain transaction Merkle proof
 *  @param rawHeader              The header containing crossStateRoot to verify the above tx Merkle proof
 *  @param headerProof            The header Merkle proof used to verify rawHeader
 *  @param curRawHeader           Any header in current epoch consensus of Poly chain
 *  @param headerSig              The converted signature variable for solidity derived from Poly chain consensus nodes' signature 
 *                                used to verify the validity of curRawHeader
 *  @return                       true or false
*/
function verifyHeaderAndExecuteTx (bytes memory proof, bytes memory rawHeader, bytes memory headerProof, bytes memory curRawHeader, bytes memory headerSig) whenNotPaused public returns (bool)
````

> [!Note|style:flat|label:Notice]
> Only if the return value of `verifyHeaderAndExecuteTx()`is true will the whole cross-chain transaction be executed successfully. 

Here is an example:

#### Example:

```solidity
/*  
 *  @param argsBs            The argument bytes received by the lock proxy contract on source chain, 
 *                           need to be deserialized based on the way of serialization in the 
 *                           lock proxy contract on source chain.
 *  @param fromContractAddr  The source chain contract address
 *  @param fromChainId       The source chain id
*/
function unlock(bytes memory argsBs, bytes memory fromContractAddr, uint64 fromChainId) onlyManagerContract public returns (bool) {
    TxArgs memory args = _deserializeTxArgs(argsBs);
    require(fromContractAddr.length != 0, "from proxy contract address cannot be empty");
    require(Utils.equalStorage(proxyHashMap[fromChainId], fromContractAddr), "From Proxy contract address error!");
        
    require(args.toAssetHash.length != 0, "toAssetHash cannot be empty");
    address toAssetHash = Utils.bytesToAddress(args.toAssetHash);

    require(args.toAddress.length != 0, "toAddress cannot be empty");
    address toAddress = Utils.bytesToAddress(args.toAddress);

    require(_transferFromContract(toAssetHash, toAddress, args.amount), "transfer asset from lock_proxy contract to toAddress failed!");
        
    emit UnlockEvent(toAssetHash, toAddress, args.amount);
    return true;
}
```


- Call the function `unlock()` to deserialize the transaction data, transfer a certain amount of token to the target address on the target chain, and complete the cross-chain contract invocation.
