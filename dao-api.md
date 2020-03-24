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
`winningOption`: Campaign option that received the most votes
`totalVotes`: Total amount of votes received for a campaign
`votedPercentage`: Percentage of votes (out of the total KNC supply at the time of the campaign)
`minThreshold`: Minimum percentage (out of all campaign votes) needed for the winning option to be valid

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

## Reading Campaign Information




### Deposit
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

### Withdrawing KNC
The user can withdraw KNC (in token wei) from the staking contract at any point in time. 

---
function **`withdraw`**(uint amount) public
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `amount` | uint | KNC twei to be withdrawn |
---

#### Example
Withdraw 1000 KNC

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

const BN = web3.utils.BN;
let tokenAmount = new BN(10).pow(new BN(21)); // 1000 KNC in twei

txData = StakingContract.methods.withdraw(tokenAmount).encodeABI();

txReceipt = await web3.eth.sendTransaction({
  from: USER_WALLET_ADDRESS, //obtained from web3 interface
  to: STAKING_CONTRACT_ADDRESS,
  data: txData
});
```

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

## Reading Staking Data
There are primarily 3 parameters of interest (apart from getting the current epoch number):
|       Parameter      |                 Description                   |
| ---------------------|:---------------------------------------------:|
| `stake` | KNC amount staked by a staker |
| `delegatedStake` | KNC amount delegated to a staker |
| `delegatedAddress` / `dAddr` | Who the staker delegated his stake to |

We can classify the APIs for reading staking data in 4 broad sections:
- Reward percentage calculation for pool masters
- Getting the above 3 parameters for the past epochs and the current epoch
- Getting the above 3 parameters for the next epoch

## Section 1: Reward calculation for pool masters
### Staker Data Of An Epoch
Obtains a staker's information for a specified epoch. Used for calculating reward percentage by pool masters (and the DAO contract). Kindly refer to [this example](faqs.md#2-how-do-i-make-use-of-the-getstakerdataforpastepoch-function-to-calculate-the-stake-and-reward-distribution-for-my-pool-members) for a walkthrough on reward calculation.

---
function **`getStakerDataForPastEpoch`**(address staker, uint epoch) public view returns (uint _stake, uint _delegatedStake, address _delegatedAddress)

**Inputs**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `staker` | address | Staker's wallet address |
| `epoch` | uint | epoch number |

**Returns:**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `_stake` | uint | `staker` stake amount |
| `_delegatedStake` | uint | Stake amount delegated to `staker` by other stakers |
| `_delegatedAddress` | address | Wallet address `staker` delegated his stake to |
---
**Notes:**
- Delegated stakes to `staker` are not forwarded to `delegatedAddress`. `staker` is still responsible for voting on behalf of all stakes delegated to him.

#### Example
Obtain staker's information (of address `0x12340000000000000000000000000000deadbeef`) at epoch 5.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address
let epoch = new BN(5);

let result = await StakingContract.methods.getStakerDataForPastEpoch(staker, epoch).call();
```

## Section 2: Staking info of past and current epochs
### Staker's KNC stake
Obtains a staker's KNC stake for a specified epoch (up to the next epoch)

---
function **getStake**(address staker, uint epoch) public view returns (uint)

**Inputs**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `staker` | address | Staker's wallet address |
| `epoch` | uint | epoch number |

**Returns:**\
`staker` KNC stake at `epoch`
---
**Note:**
`getStake(staker, N+1) == getLatestStakeBalance(staker)` where `N` is the current epoch number

#### Example
Obtain staker's KNC stake (of address `0x12340000000000000000000000000000deadbeef`) at epoch 5.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address
let epoch = new BN(5);

let result = await StakingContract.methods.getStake(staker, epoch).call();
```

### Staker's delegated stake
Obtains staking amount delegated to an address for a specified epoch (up to the next epoch)

---
function **getDelegatedStake**(address staker, uint epoch) public view returns (uint)

**Inputs**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `staker` | address | Staker's wallet address |
| `epoch` | uint | epoch number |

**Returns:**\
Delegated stake amount to `staker` at `epoch`
---
**Note:**
`getDelegatedStake(staker, N+1) == getLatestDelegatedStake(staker)` where `N` is the current epoch number

#### Example
Obtain stake amount delegated to address `0x12340000000000000000000000000000deadbeef` at epoch 5.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address
let epoch = new BN(5);

let result = await StakingContract.methods.getDelegatedStake(staker, epoch).call();
```

### Staker's delegated address
Obtains the pool master's address of `staker` at a specified epoch (up to the next epoch)

---
function **getDelegatedAddress**(address staker, uint epoch) public view returns (address)

**Inputs**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `staker` | address | Staker's wallet address |
| `epoch` | uint | epoch number |

**Returns:**\
`staker` pool master address
---
**Notes:**
- `getDelegatedAddress(staker, N+1) == getLatestDelegatedAddress(staker)` where `N` is the current epoch number
- If user is not a staker, null address is returned
- If user did not delegate to anyone, `staker` address is returned

#### Example
Get `0x12340000000000000000000000000000deadbeef` pool master's address at epoch 5.
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address
let epoch = new BN(5);

let result = await StakingContract.methods.getDelegatedAddress(staker, epoch).call();
```

## Section 3: Staking info of the next epoch
### Staker's KNC stake
Obtains a staker's KNC stake for the next epoch

---
function **getLatestStakeBalance**(address staker) public view returns (uint)

**Inputs**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `staker` | address | Staker's wallet address |

**Returns:**\
`staker` KNC stake for the next epoch.
---
**Note:**
`getLatestStakeBalance(staker) == getStake(staker, N+1)` where `N` is the current epoch number

#### Example
Obtain staker's KNC stake (of address `0x12340000000000000000000000000000deadbeef`) for the next epoch.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address

let result = await StakingContract.methods.getLatestStakeBalance(staker).call();
```

### Staker's delegated stake
Obtains staking amount delegated to an address for the next epoch

---
function **getDelegatedStake**(address staker) public view returns (uint)

**Inputs**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `staker` | address | Staker's wallet address |

**Returns:**\
Delegated stake amount to `staker` for the next epoch.
---
**Note:**
`getLatestDelegatedStake(staker) == getDelegatedStake(staker, N+1)` where `N` is the current epoch number

#### Example
Obtain stake amount delegated to address `0x12340000000000000000000000000000deadbeef` for the next epoch.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address

let result = await StakingContract.methods.getLatestDelegatedStake(staker).call();
```

### Staker's delegated address
Obtains the pool master's address of `staker` for the next epoch

---
function **getLatestDelegatedAddress**(address staker, uint epoch) public view returns (address)

**Inputs**
| Parameter | Type | Description |
| ---------- |:-------:|:-------------------:|
| `staker` | address | Staker's wallet address |

**Returns:**\
`staker` pool master address
---
**Notes:**
- `getLatestDelegatedAddress(staker) == getDelegatedAddress(staker, N+1)` where `N` is the current epoch number
- If user is not a staker, null address is returned
- If user did not delegate to anyone, `staker` address is returned

#### Example
Get `0x12340000000000000000000000000000deadbeef` pool master's address for the next epoch.
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper

let staker = "0x12340000000000000000000000000000deadbeef" //staker's address

let result = await StakingContract.methods.getLatestDelegatedAddress(staker).call();
```
