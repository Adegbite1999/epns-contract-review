 # EPNS STAKING CONTRACT REVIEW 

# Solidity version `^0.6.11`;


Import  openzepellin ERC20 interface standard (https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol)

Import openzepellin safeMath.sol which help guide against overflow and underflow (https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeMath.sol)

Import openzepellin ReentrancyGuard.sol which helps guide against reentrancy attack

# STATE VARIABLES

# Base Multiplier
```
uint128 constant private BASE_MULTIPLIER = uint128(1 * 10 **18);

```

# startTime of stake
```
uint256 public immutable epoch1Start;
```
# duration of stake
```
uint256 public immutable epochDuration;
```
BASE_MULTIPLIER is declared constant while epoch1Start and epochDuration are declared as immutable.

In solidity, state variables can be declared as constant or immutable.In both cases, variables cannot be modified after the contract has been constructed.

For Constant variables, the values has to be fixed at compile-time while immutable variables can still be assigned at construction time.

# mapping that holds the current balance of the user for each token

```
mapping(address => mapping(address => uint256)) private balances;
```

# struct that keeps track of pool size and status of pool(true or false)
```
  struct Pool {
        uint256 size;
        bool set;
    }
```
# struct that stores the pool size for each token. its visibility is marked private i.e it only visible within the contract
```
mapping(address => mapping(uint256 => Pool)) private poolSize;
```
# A checkpoint of the valid balance of a user for an epoch
   ```
    struct Checkpoint {
        uint128 epochId;
        uint128 multiplier;
        uint256 startBalance;
        uint256 newDeposits;
    }
```
# A 2D mapping user address to tokenAddress to Checkpoint[]  
```
mapping(address => mapping(address => Checkpoint[])) private balanceCheckpoints;

```
# keeps track of the balance of a user
```
    mapping(address => uint128) private lastWithdrawEpochId;
```

# The constructor takes in two parameters uint256 _epoch1start, uint256 _epochDuration which initialized the state variable. 

```
constructor (uint256 _epoch1Start, uint256 _epochDuration) public {
epoch1Start = _epoch1Start;
epochDuration = _epochDuration;
}
```


# Deposit function
 ```
function deposit(address tokenAddress, uint256 amount) public nonReentrant {
        require(amount > 0, "Staking: Amount must be > 0");

        IERC20 token = IERC20(tokenAddress);

        balances[msg.sender][tokenAddress] = balances[msg.sender][tokenAddress].add(amount);

        token.transferFrom(msg.sender, address(this), amount);

        uint128 currentEpoch = getCurrentEpoch();
        uint128 currentMultiplier = currentEpochMultiplier();
        uint256 balance = balances[msg.sender][tokenAddress];

        if (!epochIsInitialized(tokenAddress, currentEpoch)) {
            address[] memory tokens = new address[](1);
            tokens[0] = tokenAddress;
            manualEpochInit(tokens, currentEpoch);
        }

        Pool storage pNextEpoch = poolSize[tokenAddress][currentEpoch + 1];
        pNextEpoch.size = token.balanceOf(address(this));
        pNextEpoch.set = true;

        Checkpoint[] storage checkpoints = balanceCheckpoints[msg.sender][tokenAddress];

        uint256 balanceBefore = getEpochUserBalance(msg.sender, tokenAddress, currentEpoch);

        if (checkpoints.length == 0) {
            checkpoints.push(Checkpoint(currentEpoch, currentMultiplier, 0, amount));

            checkpoints.push(Checkpoint(currentEpoch + 1, BASE_MULTIPLIER, amount, 0));
        } else {
            uint256 last = checkpoints.length - 1;

            if (checkpoints[last].epochId < currentEpoch) {
                uint128 multiplier = computeNewMultiplier(
                    getCheckpointBalance(checkpoints[last]),
                    BASE_MULTIPLIER,
                    amount,
                    currentMultiplier
                );
                checkpoints.push(Checkpoint(currentEpoch, multiplier, getCheckpointBalance(checkpoints[last]), amount));
                checkpoints.push(Checkpoint(currentEpoch + 1, BASE_MULTIPLIER, balance, 0));
            }

            else if (checkpoints[last].epochId == currentEpoch) {
                checkpoints[last].multiplier = computeNewMultiplier(
                    getCheckpointBalance(checkpoints[last]),
                    checkpoints[last].multiplier,
                    amount,
                    currentMultiplier
                );
                checkpoints[last].newDeposits = checkpoints[last].newDeposits.add(amount);

                checkpoints.push(Checkpoint(currentEpoch + 1, BASE_MULTIPLIER, balance, 0));
            }

            else {
                if (last >= 1 && checkpoints[last - 1].epochId == currentEpoch) {
                    checkpoints[last - 1].multiplier = computeNewMultiplier(
                        getCheckpointBalance(checkpoints[last - 1]),
                        checkpoints[last - 1].multiplier,
                        amount,
                        currentMultiplier
                    );
                    checkpoints[last - 1].newDeposits = checkpoints[last - 1].newDeposits.add(amount);
                }

                checkpoints[last].startBalance = balance;
            }
        }

        uint256 balanceAfter = getEpochUserBalance(msg.sender, tokenAddress, currentEpoch);

        poolSize[tokenAddress][currentEpoch].size = poolSize[tokenAddress][currentEpoch].size.add(balanceAfter.sub(balanceBefore));

        emit Deposit(msg.sender, tokenAddress, amount);
    }

```
# Deposit function breakdown

* The deposit function requires that the amount to stake is greater than zero, add amount to balance. And does  transferFrom(_from, _to, _amount), which give permission to the contract `address(this)` to spend the specified `amount` from the balance of `msg.sender`.

* if epoch is not initialized , initialize it with the `manualEpochInit` function. 

* it updates the poolSize and set nextEpoch size to token balance of contract and set status to true.

# Checkpoint[] storage checkpoints = balanceCheckpoints[msg.sender][tokenAddress];

* declear a stroage pointer to access balanceCheckpoints

*  if checkpoints length is equal to zero, it push the struct checkPoint into array and initialize the structs value.

* else if checkpoints length is not equal to zero, it gets the last checkPoint in checkpoints array 

* else if the last checkpoint epochId is less than currentEpoch.
`checkpoints[last].epochId < currentEpoch`
- push Checkpoint struct and initialize it

*  else if the last checkpoint is equal to currentEpoch ( tracks  last action happened in the previous epoch
), compute multiplier, initialize new deposit value and push Checkpoint struct with `currentEpoch + 1`, `BASE_MULTIPLIER`, `balance`, `0`

* if `last >= 1 && checkpoints[last - 1].epochId == currentEpoch

- 


 
# Withdraw Function

    function withdraw(address tokenAddress, uint256 amount) public nonReentrant {
        require(balances[msg.sender][tokenAddress] >= amount, "Staking: balance too small");

        balances[msg.sender][tokenAddress] = balances[msg.sender][tokenAddress].sub(amount);

        IERC20 token = IERC20(tokenAddress);
        token.transfer(msg.sender, amount);

        // epoch logic
        uint128 currentEpoch = getCurrentEpoch();

        lastWithdrawEpochId[tokenAddress] = currentEpoch;

        if (!epochIsInitialized(tokenAddress, currentEpoch)) {
            address[] memory tokens = new address[](1);
            tokens[0] = tokenAddress;
            manualEpochInit(tokens, currentEpoch);
        }

        // update the pool size of the next epoch to its current balance
        Pool storage pNextEpoch = poolSize[tokenAddress][currentEpoch + 1];
        pNextEpoch.size = token.balanceOf(address(this));
        pNextEpoch.set = true;

        Checkpoint[] storage checkpoints = balanceCheckpoints[msg.sender][tokenAddress];
        uint256 last = checkpoints.length - 1;

        // note: it's impossible to have a withdraw and no checkpoints because the checkpoints[last] will be out of bound and revert

        // there was a deposit in an older epoch (more than 1 behind [eg: previous 0, now 5]) but no other action since then
        if (checkpoints[last].epochId < currentEpoch) {
            checkpoints.push(Checkpoint(currentEpoch, BASE_MULTIPLIER, balances[msg.sender][tokenAddress], 0));

            poolSize[tokenAddress][currentEpoch].size = poolSize[tokenAddress][currentEpoch].size.sub(amount);
        }
        // there was a deposit in the `epochId - 1` epoch => we have a checkpoint for the current epoch
        else if (checkpoints[last].epochId == currentEpoch) {
            checkpoints[last].startBalance = balances[msg.sender][tokenAddress];
            checkpoints[last].newDeposits = 0;
            checkpoints[last].multiplier = BASE_MULTIPLIER;

            poolSize[tokenAddress][currentEpoch].size = poolSize[tokenAddress][currentEpoch].size.sub(amount);
        }
        // there was a deposit in the current epoch
        else {
            Checkpoint storage currentEpochCheckpoint = checkpoints[last - 1];

            uint256 balanceBefore = getCheckpointEffectiveBalance(currentEpochCheckpoint);

            // in case of withdraw, we have 2 branches:
            // 1. the user withdraws less than he added in the current epoch
            // 2. the user withdraws more than he added in the current epoch (including 0)
            if (amount < currentEpochCheckpoint.newDeposits) {
                uint128 avgDepositMultiplier = uint128(
                    balanceBefore.sub(currentEpochCheckpoint.startBalance).mul(BASE_MULTIPLIER).div(currentEpochCheckpoint.newDeposits)
                );

                currentEpochCheckpoint.newDeposits = currentEpochCheckpoint.newDeposits.sub(amount);

                currentEpochCheckpoint.multiplier = computeNewMultiplier(
                    currentEpochCheckpoint.startBalance,
                    BASE_MULTIPLIER,
                    currentEpochCheckpoint.newDeposits,
                    avgDepositMultiplier
                );
            } else {
                currentEpochCheckpoint.startBalance = currentEpochCheckpoint.startBalance.sub(
                    amount.sub(currentEpochCheckpoint.newDeposits)
                );
                currentEpochCheckpoint.newDeposits = 0;
                currentEpochCheckpoint.multiplier = BASE_MULTIPLIER;
            }

            uint256 balanceAfter = getCheckpointEffectiveBalance(currentEpochCheckpoint);

            poolSize[tokenAddress][currentEpoch].size = poolSize[tokenAddress][currentEpoch].size.sub(balanceBefore.sub(balanceAfter));

            checkpoints[last].startBalance = balances[msg.sender][tokenAddress];
        }

        emit Withdraw(msg.sender, tokenAddress, amount);
    }

# Withdraw function breakdown

* The withdraw function require that balance is greater than or equal amount to withdraw
- Then transfer the amount to `msg.sender`

*   update the pool size of the next epoch to its current balance

```
Pool storage pNextEpoch = poolSize[tokenAddress][currentEpoch + 1];
pNextEpoch.size = token.balanceOf(address(this));
pNextEpoch.set = true;
```
* checks if there's a deposit in the `epochId - 1`;

```
(checkpoints[last].epochId == currentEpoch) {
checkpoints[last].startBalance = balances[msg.sender][tokenAddress];
checkpoints[last].newDeposits = 0;
checkpoints[last].multiplier = BASE_MULTIPLIER;

poolSize[tokenAddress][currentEpoch].size = poolSize[tokenAddress][currentEpoch].size.sub(amount);
        }

```

*  check if  there is a deposit in the current epoch

```
Checkpoint storage currentEpochCheckpoint = checkpoints[last - 1];

uint256 balanceBefore = getCheckpointEffectiveBalance(currentEpochCheckpoint);
```
#  In case of withdraw, there are two scenario:
1. the user withdraws less than he added in the current epoch
2. the user withdraws more than he added in the current epoch (including 0)



# HELPER FUNCTIONS

# Returns the id of the current epoch derived from block.timestamp

```
    function getCurrentEpoch() public view returns (uint128) {
        if (block.timestamp < epoch1Start) {
            return 0;
        }

        return uint128((block.timestamp - epoch1Start) / epochDuration + 1);
    }
```

# Returns the percentage of time left in the current epoch
     
    function currentEpochMultiplier() public view returns (uint128) {
        uint128 currentEpoch = getCurrentEpoch();
        uint256 currentEpochEnd = epoch1Start + currentEpoch * epochDuration;
        uint256 timeLeft = currentEpochEnd - block.timestamp;
        uint128 multiplier = uint128(timeLeft * BASE_MULTIPLIER / epochDuration);

        return multiplier;
    }


# manualEpochInit

* checks if pool has been initialzed, if `epochId == 0` set `pool.size = 0` and `p.set = true` else `initialize the pool`
  
```
function manualEpochInit(address[] memory tokens, uint128 epochId) public {
    require(epochId <= getCurrentEpoch(), "can't init a future epoch");

    for (uint i = 0; i < tokens.length; i++) {
        Pool storage p = poolSize[tokens[i]][epochId];

        if (epochId == 0) {
            p.size = uint256(0);
            p.set = true;
        } else {
            require(!epochIsInitialized(tokens[i], epochId), "Staking: epoch already initialized");
            require(epochIsInitialized(tokens[i], epochId - 1), "Staking: previous epoch not initialized");

            p.size = poolSize[tokens[i]][epochId - 1].size;
            p.set = true;
        }
    }

    emit ManualEpochInit(msg.sender, epochId, tokens);
}


```

# Emergency Withdrawal

* Require that at least 10 epoch has past, if true, set balance to zero and transfer totalBalance

```
    function emergencyWithdraw(address tokenAddress) public {
    require((getCurrentEpoch() - lastWithdrawEpochId[tokenAddress]) >= 10, "At least 10 epochs must pass without success");

    uint256 totalUserBalance = balances[msg.sender][tokenAddress];
    require(totalUserBalance > 0, "Amount must be > 0");

    balances[msg.sender][tokenAddress] = 0;

    IERC20 token = IERC20(tokenAddress);
    token.transfer(msg.sender, totalUserBalance);

    emit EmergencyWithdraw(msg.sender, tokenAddress, totalUserBalance);
    }


```

# getEpochBalance

```
    function getEpochUserBalance(address user, address token, uint128 epochId) public view returns (uint256) {
        Checkpoint[] storage checkpoints = balanceCheckpoints[user][token];

        // if there are no checkpoints, it means the user never deposited any tokens, so the balance is 0
        if (checkpoints.length == 0 || epochId < checkpoints[0].epochId) {
            return 0;
        }

        uint min = 0;
        uint max = checkpoints.length - 1;

        // shortcut for blocks newer than the latest checkpoint == current balance
        if (epochId >= checkpoints[max].epochId) {
            return getCheckpointEffectiveBalance(checkpoints[max]);
        }

        // binary search of the value in the array
        while (max > min) {
            uint mid = (max + min + 1) / 2;
            if (checkpoints[mid].epochId <= epochId) {
                min = mid;
            } else {
                max = mid - 1;
            }
        }

        return getCheckpointEffectiveBalance(checkpoints[min]);
    }


```


# Returns the amount of `token` that the `user` has currently staked

    function balanceOf(address user, address token) public view returns (uint256) {
        return balances[user][token];
    }

    
# Returns the id of the current epoch derived from block.timestamp
* A view function which checks if  `block.timestamp` is less than `epoch1Start` and returns zero
* if the condition above fails, it returns current inddex of the epoch
    
         function getCurrentEpoch() public view returns (uint128) {
        if (block.timestamp < epoch1Start) {
            return 0;
        }

        return uint128((block.timestamp - epoch1Start) / epochDuration + 1);
    }

# getEpochPoolSize   

* if epoch is initialized , returns the poolSize
* if epoch is not initialzed, return zero
* epoch 0 is initialized => there was an action at some point but none that initialized the epochId  which means the current pool size is equal to the current balance of token held by the staking contract


```

function getEpochPoolSize(address tokenAddress, uint128 epochId) public view returns (uint256) {
        if (epochIsInitialized(tokenAddress, epochId)) {
            return poolSize[tokenAddress][epochId].size;
        }

        // epochId not initialized and epoch 0 not initialized => there was never any action on this pool
        if (!epochIsInitialized(tokenAddress, 0)) {
            return 0;
        }

        IERC20 token = IERC20(tokenAddress);
        return token.balanceOf(address(this));
    }

```

# Returns multiplier
*  `  uint128 currentEpoch = getCurrentEpoch();`: gets current epoch
* Calculate the CurrentEpochEnd by adding `epoch1Start` + `currentEpoch` * `epochDuration`
* Calculate timeLeft by substrating `currentEpoch` from `block.timestamp`
* Calculate multiplier via ` uint128 currentEpoch = getCurrentEpoch();` 

```  
    function currentEpochMultiplier() public view returns (uint128) {
        uint128 currentEpoch = getCurrentEpoch();
        uint256 currentEpochEnd = epoch1Start + currentEpoch * epochDuration;
        uint256 timeLeft = currentEpochEnd - block.timestamp;
        uint128 multiplier = uint128(timeLeft * BASE_MULTIPLIER / epochDuration);

        return multiplier;
    }

```
    
# compute new multiplier based on prevBalance and prevMultiplier, amount, currentMultiplier

    function computeNewMultiplier(uint256 prevBalance, uint128 prevMultiplier, uint256 amount, uint128 currentMultiplier) public pure returns (uint128) {
        uint256 prevAmount = prevBalance.mul(prevMultiplier).div(BASE_MULTIPLIER);
        uint256 addAmount = amount.mul(currentMultiplier).div(BASE_MULTIPLIER);
        uint128 newMultiplier = uint128(prevAmount.add(addAmount).mul(BASE_MULTIPLIER).div(prevBalance.add(amount)));

        return newMultiplier;
    }


# Check the status of an Epoch
* epochIsInitialized accepts three paramter `token `

    function epochIsInitialized(address token, uint128 epochId) public view returns (bool) {
        return poolSize[token][epochId].set;
    }

# Returns the balance of the checkPoint of an Epoch


    function getCheckpointBalance(Checkpoint memory c) internal pure returns (uint256) {
        return c.startBalance.add(c.newDeposits);
    }


```
    function getCheckpointEffectiveBalance(Checkpoint memory c) internal pure returns (uint256) {
        return getCheckpointBalance(c).mul(c.multiplier).div(BASE_MULTIPLIER);
    }

```

# Events
 ```
event Deposit(address indexed user, address indexed tokenAddress, uint256 amount);
event Withdraw(address indexed user, address indexed tokenAddress, uint256 amount);
event ManualEpochInit(address indexed caller, uint128 indexed epochId, address[] tokens);
event EmergencyWithdraw(address indexed user, address indexed tokenAddress, uint256 amount);

 ```