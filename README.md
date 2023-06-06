# Project deprecated kindly refer to [Eth-Arbitrage 2.0](https://github.com/Devilla/eth-arbitrage-2)

This project will perform an arbitrage of LUSD/ETH pair. This will be done by 2 steps
1. Swap ETH for LUSD at uniswap 
2. Redeem LUSD for ETH at liquity

# Design

The project has two main components , one is the node server and another is the smart contract. 

The node server is responsible for keeping a track of the prices of ETH at chainlink and at uniswap. When we notice a price difference where there is a possiblity of a profit from arbitrage it will trigger the smart contract to perform the arbitrage between uniswap and liquity smart contracts.

### How is the amount of eth to be used for the arbitrage calculated ?

We have three restrictions in determining the ETH that can be used for the arbitrage. 
1) The ETH present in the arbitragers wallet
2) The redeemable amount at liquity 
3) The uniswap reserves in the LUSD/ETH pair

We will first take the minimum of 
```
min( <ETH present in Wallet> , <12.5% of the reserve pool of uniswap pair> )  // We will not use a bigger percentage of the uniswap pool because it will impact the prices heavily
```
Once the minimum is determined we will then check at liquity if we can redeem LUSD the amount of ETH swapped. If not , we will use the redeemableLUSD at liquity to determine the ETH used for swap.

###### Note: The smaller the percent of the uniswap pool we can use the better prices we can get, I have taken 12.5% of the pool for now because the pool reserves are small.

### How will it determine if there is an arbitrage opportunity ?

We can determine the arbitrage opportunity by the following formula.
```
profit = ETH_USED * (uniswapPrice/chainLinkPrice) * (1-redemptionFeePercentage) * (1-uinswap_liquidity_provider_fee) - GasCost
```

If we find that the profit is positive we will then invoke the smart contract.

### What mechanism will be used to make sure the server can be run 24/7 ?

To run the server 24/7 we will need a poller mechanism which will continously monitor the prices. To do this , this project will subscribe to the new block headers generated in the ethereum chain. This is done considering uniswap prices are updated after every block is mined.

# Deployed contract on Goerli testnet
![image](https://user-images.githubusercontent.com/15603274/233756332-55b737ab-4fd2-484a-88c7-99b0538de61c.png)

# How to start

Run the following commands
```
> npm
> cp node_modules/@liquity/lib-ethers/deployments/default/goerli.json node_modules/@liquity/lib-ethers/dist/deployments/dev.json 
```

After this add your private keys and wallet addresses to .env file.

#### To start the server:
```
npm start
```
Once the server is started you will need to ping `localhost:3000` to start polling for prices.

###### Note: We use redeemLUSD function to determine the input values for the smart contract inputs, to run the redeemLUSD we need LUSD present in our wallet. So keep a amount of LUSD equivalent to the ETH in your wallet or the amount being arbitraged for. 

#### Sample `.env` file
```
PRIVATE_KEY='PVT_KEY_FROM_WALLET'
WSS_PROVIDER='wss://goerli.infura.io/ws/v3/API_KEY'
HTTPS_PROVIDER='https://goerli.infura.io/v3/API_KEY'
ACCOUNT_ADDRESS='WALLET_ADDRESS_OF_THE_PRIVATE_KEY'
```

# Author

[Dev Yadav](https://github.com/Devilla)



This project demonstrates a basic Hardhat use case. It comes with a sample contract, a test for that contract, and a script that deploys that contract.

Try running some of the following tasks:

```shell
npx hardhat help
npx hardhat test
REPORT_GAS=true npx hardhat test
npx hardhat node
npx hardhat run scripts/deploy.js
```
