<h1 align="center">Importing Tokens</h1>

To import tokens, there are four steps to follow which are listed below.


### Step1. Asset deployment
- Deploy mapping contracts of the corresponding asset on the target chain using the [template](https://github.com/polynetwork/eth-contracts/blob/master/contracts/core/assets/erc20_template/ERC20Template.sol). 
- Ensure the total amount of contracts stay consistent with the amount on the source chain.

### Step2. Assets transfer
- Transfer the initialized assets to the target chain [proxy contract](../../Core_Smart_Contract/Contract/LockProxy.md).

### Step3. Information submission
- [Download](https://dev-docs.poly.network/new_product/integrate_assets/resources/token_import_template.xlsx) the asset template and fill in the information according to an example in the template.
- [Submit](https://docs.google.com/forms/d/e/1FAIpQLSfIOje4k_t4rjzli-ItMqTTckPZ7MqtoTzP2Bjr666wy4Up1g/viewform) the .xlsx file to poly.
- Contact Poly Network via <a class="fab fa-discord" href= "https://discord.com/invite/y6MuEnq"></a>

### Step4. Test 
- Wait until the operators successfully add your contract address to the whitelist.
- Click [here](https://bridge.poly.network/testnet) and follow the [User Manual](../../Core_Smart_Contract/User_Manuals/Token_Transaction.md) to test the token cross-chain transfer.
