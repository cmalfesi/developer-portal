# DAO APIs

## Introduction
The DAO contract primarily provides the following functionalities:
- Creation and cancellation of campaigns by authorized personnel
- Voting for campaigns
- Claiming staker rewards for participation in voting for campaigns

For stakers and pool masters, the actions you will be interested in are:
1. Viewing Campaign Details
2. Voting for a campaign
3. Claim Rewards

 The voting formula used for deciding the winning options of a campaign may also be of interest.

## Creation and cancellation of campaigns
These actions can only be carried out by the `campaignCreator`.

## Campaign Types
There are currently 3 different types of campaigns:
1. `GENERAL`: A generic campaign. 
2. `NETWORK_FEE`: The fee amount (in basis points) to be charged for every trade.
3. `FEE_HANDLER_BRR`: Proportion of fees to be **burnt**, given to stakers as **rewards**, and given to reserves as **rebates** for liquidity contribution.

## Voting Formula
The formula was designed to be simple enough such that it can be implemented on-chain, and is gas efficient for calculations. Hence, the "1 token 1 vote" idea is used for its simplicity. A quorum is also defined, in order for a winning option to be considered valid.

The quorum needed is linearly inversely proportional to the percentage of votes received for that campaign. In other words, fewer votes received, stricter quorum. 

In the event that there is more than 1 winning option (ie. more than 1 option sharing the same no. of votes), the campaign is considered to not have a winner.

### Quick Primer
A line can be expressed as `Y = tX + C`, where `t` is the gradient, and `C` shifts the line vertically up/down.

### Variables
- `winningOption`: Campaign option that received the most votes
- `totalVotes`: Total amount of votes received for a campaign
- `votedPercentage`: Percentage of votes (out of the total KNC supply at the time of the campaign)
- `minThreshold`: Minimum percentage (out of all campaign votes) needed for the winning option to be valid

### Explanation
1. We first find `winningOption` with a simple iteration through the votes received by each option.
2. Calculate `votedPercentage`: `totalVotes` / total KNC supply at time of campaign
3. `minThreshold`: `-t * votedPercentage + C`. `t` and `C` were determined by `campaignCreator` upon campaign initialisation (to allow for flexibility). Rearranging this, we get `minThreshold = C - t * votedPercentage`
4. If enough people have voted such that `t * votedPercentage > C` (ie. `minThreshold < 0`), then the quorum condition has been met, and we can consider `winningOption` to be valid.
5. Otherwise, we have to check that `winningOption >= minThreshold` before considering it to be a valid winning option.

## Public DAO Variables
`MAX_EPOCH_CAMPS`: Maximum number of campaigns that can be created in an epoch by `campaignCreator`
`MAX_CAMP_OPTIONS`: Maximum number of options that a campaign can have
`MIN_CAMP_DURATION_BLOCKS`: Minimum duration for a campaign
`kncToken`: The KNC token contract
`staking`: The staking contract
`feeHandler`: The contract that stores the network fees (in ETH) and distrubution amount information
`numberCampaigns`: No. of campaigns in total, used to generate a unique ID for each campaign

## Getting Current Epoch Number
Obtain the current epoch number of the staking contract

---
function **`getCurrentEpochNumber`**() public view returns (uint)

#### Example
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper
let currentEpochNum = await StakingContract.getCurrentEpochNumber().call();
```

## `NETWORK_FEE` Campaign Information
While this section is specific to the `NETWORK_FEE` campaign type, the other methods in the [Reading Campaign Information]() are of relevance.

### `latestNetworkFeeResult`
Network fee (in basis points) charged for trades.

#### Example
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper
let result = await DAOContract.methods.latestNetworkFeeResult().call();
```

### `getLatestNetworkFeeData`

### `getLatestNetworkFeeDataWithCache`

### `networkFeeCamp`
`networkFeeCamp[epoch] = uint`
`NETWORK_FEE` campaign ID number for a given epoch.

#### Example
Get the `NETWORK_FEE` campaign ID number for epoch `5`.
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let epochNum = new BN(5);
let result = await DAOContract.methods.networkFeeCamp(epochNum).call();
```






## `FEE_HANDLER_BRR` Campaign Information
While this section is specific to the `FEE_HANDLER_BRR` campaign type, the other methods in the [Reading Campaign Information]() are of relevance.

### `latestBrrResult`
Encoded BRR data (to save on storage). Call `latestBrrResultDecoded` for the more readable and understandable version.

### `latestBrrResultDecoded`
Returns the latest BRR data, current epoch number and block number for which BRR data expires.

---
function **`latestBrrResultDecoded`**() public view returns (uint burnInBps, uint rewardInBps, uint rebateInBps, uint epoch, uint expiryBlockNumber)

**Returns:**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `burnInBps` | `uint` | Ratio (in basis points) of fees collected for burning KNC |
| `rewardInBps` | `uint` | Ratio (in basis points) of fees collected for staker rewards |
| `rebateInBps` | `uint` | Ratio (in basis points) of fees collected for reserve rebates |
| `epoch` | `uint` | Current epoch of Staking and DAO contracts |
| `expiryBlockNumber` | `uint` | Block number for which BRR becomes invalid |
---
**Notes:**
- `epoch` has the same value as calling [`getCurrentEpochNumber()`]()
- Unlike the `getLatestBRRData()` function below, this is a `view` function, and is meant to be used for reading the BRR data

#### Example
```js
```

### `getLatestBRRData`
In addition to reading the latest BRR data, it also concludes a BRR campaign if necessary, thereby caching the latest BRR data as well.

---
function **getLatestBRRData()** public returns (uint burnInBps, uint rewardInBps, uint rebateInBps, uint epoch, uint expiryBlockNumber)

**Returns:**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `burnInBps` | `uint` | Ratio (in basis points) of fees collected for burning KNC |
| `rewardInBps` | `uint` | Ratio (in basis points) of fees collected for staker rewards |
| `rebateInBps` | `uint` | Ratio (in basis points) of fees collected for reserve rebates |
| `epoch` | `uint` | Current epoch of Staking and DAO contracts |
| `expiryBlockNumber` | `uint` | Block number for which BRR becomes invalid |
---
**Notes:**
- `epoch` has the same value as calling [`getCurrentEpochNumber()`]()
- Unlike the `latestBrrResultDecoded()` function above, this function caches the BRR data too.

#### Example
```js

```

### `brrCampaign`











## Reading Campaign Information
### `campExists`
`campExists[campaignID] = bool`
To determine a campaign exists given a campaign ID. Returns `true` if it does, `false` if it does not.

#### Example
See if campaign ID `5` exists.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let campaignID = new BN(5);
let result = await DAOContract.methods.campExists(campaignID).call();
```

### `numberVotes`
`numberVotes[stakerAddress][epoch] = uint`
To determine the number of campaigns that a staker has voted for at a given epoch number.

#### Example
Get number of campaigns staker has voted for in epoch `5`.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address
let epochNum = new BN(5);
let result = await DAOContract.methods.numberVotes(staker, epochNum).call();
```

### `hasClaimedReward`
`hasClaimedReward[stakerAddress][epoch] = bool`
To determine if a staker has claimed his reward for a given epoch. Returns `true` if he has claimed, `false` if he has not.

#### Example
See if staker has claimed his reward for epoch `5`.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address
let epochNum = new BN(5);
let result = await DAOContract.methods.hasClaimedReward(staker, epochNum).call();
```

### `stakerVotedOption`
`stakerVotedOption[stakerAddress][campID] = uint`
To obtain the option number a staker voted on, for a campaign.

#### Example
See which option staker has voted on, for campaign ID `5`.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address
let campaignID = new BN(5);
let result = await DAOContract.methods.stakerVotedOption(staker, campaignID).call();
```


<!-- ### Deposit
The first step for any user is to deposit KNC into the staking contract (in token wei).

---
function **`deposit`**(uint amount) public
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `amount` | uint | KNC twei to be deposited |
---
Note: The user must have given an allowance to the staking contract, ie. made the following function call
`KNC.approve(stakingContractaddress, someAllowance)`

#### Example
Deposit 1000 KNC

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

const BN = web3.utils.BN;
let tokenAmount = new BN(10).pow(new BN(21)); // 1000 KNC in twei

txData = StakingContract.methods.deposit(tokenAmount).encodeABI();

txReceipt = await web3.eth.sendTransaction({
  from: USER_WALLET_ADDRESS, //obtained from web3 interface
  to: STAKING_CONTRACT_ADDRESS,
  data: txData
});
```

### Delegate
Once the user has staked some KNC, he can delegate his **entire** KNC stake to a pool master (with address `dAddr`), who will vote on his behalf. Note that we do not support partial stake delegation. Also, users can only have a maximum of 1 pool master.

---
function **`delegate`**(address dAddr) public
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `dAddr` | address | Pool master's wallet address |
---

#### Example
User delegates his stake to a pool master (of address `0x12340000000000000000000000000000deadbeef`) who will vote on the user's behalf.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let dAddr = "0x12340000000000000000000000000000deadbeef" //pool master's address

txData = StakingContract.methods.delegate(dAddr).encodeABI();

txReceipt = await web3.eth.sendTransaction({
  from: USER_WALLET_ADDRESS, //obtained from web3 interface
  to: STAKING_CONTRACT_ADDRESS,
  data: txData
});
```
``` -->
