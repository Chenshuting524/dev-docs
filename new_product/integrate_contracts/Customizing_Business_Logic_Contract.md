<h1 align="center">Develop for Customized Contracts</h1>

Business logic contracts can be customized according to your cross-chain requirements, but the following three important logics must be fulfilled to ensure cross-chain functionality in Poly Network ecosystem.


### Step1. Mapping relationship

Business contracts need to maintain two mapping relationships between chains, one is **asset mapping relationship**, and the other is **business logic contract mapping relationship**. 

#### Example:

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
- The `assetHashMap` is a mapping structure used to store the asset mapping relationship between chains. It takes the source asset hash (`fromAssetHash`) and the target chain ID (`toChainId`) as the key, and the target asset hash (`toAssetHash`) as the value. The function `bindAssetHash` is used to write this relationship in the contract store.
- The `proxyHashMap` is a mapping structure used to store the business logic contract mapping relationship between chains, which helps the CCM contract to find the correct contract on the target chain. It takes the target chain ID (`toChainId`) as the key, and the target business logic contract hash (`targetProxyHash`) as the value. And the function `bindProxyHash` is used to write this relationship in the contract store.

### Step2. Initiating transaction on source chain

A method is required to invoke the `crossChain` function in the CCM, i.e., to initiate a cross-chain transaction. The source code of `crossChain` is [here](https://dev-docs.poly.network/new_chain/side_chain/contracts.html#step3-pushing-transactions). 


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
- The function `lock` is used to invoke the function `crossChain` in CCM contract, whose parameters include `toChainId`, `toContract`, `method` and `txData`.  The `toChainId`, `toContract` meant the chain ID and the business logic contract on the target chain. The `method` is the function called on the target chain. Besides, the `lock` also needs to pack the transaction data, like  `toAssetHash`, `toAddress`, `amount`, into `txData`, so that the target chain method（mentioned `unlock`） can deserialize it.
- By calling this method, the business logic contract will **lock** a certain amount of valuable assets. And a `CrossChainEvent` will be emitted in CCM contract in order to be catched by relayer to complete the remaining processes.
- The `LockEvent` is necessary for concatenation between source trasactions and target transactions.


### Step3. Executing on target chain

A methods is required to parse and execut the transaction information transferred by `verifyHeaderAndExecuteTx` in CCM contract. The `verifyHeaderAndExecuteTx` function verifies the **legality** of the cross-chain transaction information, and passes the parsed transaction data from Poly chain to the business logic contract. The source code of `verifyHeaderAndExecuteTx` is [here](https://dev-docs.poly.network/new_chain/side_chain/contracts.html#step4-Verifying & executing). 

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

- The customized method shoud be conformed to the format called by `verifyHeaderAndExecuteTx`, see following:
````solidity
 // The returnData will be bytes32, the last byte must be 01;
(success, returnData) = _toContract.call(abi.encodePacked(bytes4(keccak256(abi.encodePacked(_method, "(bytes,bytes,uint64)"))), abi.encode(_args, _fromContractAddr, _fromChainId)));

````


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

- The mapping relationship of business logic contracts needs to be checked in `proxyHashMap`. 
- The function `unlock` is used to deserialize and excute the transaction data `argsBs`, i.e., to transfer a certain amount of token to the target address on the target chain. 
- For safety, the function `unlock` only can be called by the [CCM contract].In this case, the modifier `onlyManagerContract` restricts the calling authority by obtaining the CCM contract address of CCM in [CCMP contract]. While the function `setManagerProxy` is uesd to set the CCMP contract address. See following:

```solidity
    modifier onlyManagerContract() {
        IEthCrossChainManagerProxy ieccmp = IEthCrossChainManagerProxy(managerProxyContract);
        require(_msgSender() == ieccmp.getEthCrossChainManager(), "msgSender is not EthCrossChainManagerContract");
        _;
    }
    
    function setManagerProxy(address ethCCMProxyAddr) onlyOwner public {
        managerProxyContract = ethCCMProxyAddr;
        emit SetManagerProxyEvent(managerProxyContract);
    }
```
