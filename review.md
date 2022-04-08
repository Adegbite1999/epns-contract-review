 # EPNS STAKING CONTRACT REVIEW 

# Solidity version `^0.6.11`;


# State variable


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
# withdrawl tracker
```
    mapping(address => uint128) private lastWithdrawEpochId;
```


# Events
 ```
 event Deposit(address indexed user, address indexed tokenAddress, uint256 amount);
event Withdraw(address indexed user, address indexed tokenAddress, uint256 amount);
event ManualEpochInit(address indexed caller, uint128 indexed epochId, address[] tokens);
event EmergencyWithdraw(address indexed user, address indexed tokenAddress, uint256 amount);

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

        // epoch logic
        uint128 currentEpoch = getCurrentEpoch();
        uint128 currentMultiplier = currentEpochMultiplier();
        uint256 balance = balances[msg.sender][tokenAddress];

        if (!epochIsInitialized(tokenAddress, currentEpoch)) {
            address[] memory tokens = new address[](1);
            tokens[0] = tokenAddress;
            manualEpochInit(tokens, currentEpoch);
        }

        // update the next epoch pool size
        Pool storage pNextEpoch = poolSize[tokenAddress][currentEpoch + 1];
        pNextEpoch.size = token.balanceOf(address(this));
        pNextEpoch.set = true;

        Checkpoint[] storage checkpoints = balanceCheckpoints[msg.sender][tokenAddress];

        uint256 balanceBefore = getEpochUserBalance(msg.sender, tokenAddress, currentEpoch);

        // if there's no checkpoint yet, it means the user didn't have any activity
        // we want to store checkpoints both for the current epoch and next epoch because
        // if a user does a withdraw, the current epoch can also be modified and
        // we don't want to insert another checkpoint in the middle of the array as that could be expensive
        if (checkpoints.length == 0) {
            checkpoints.push(Checkpoint(currentEpoch, currentMultiplier, 0, amount));

            // next epoch => multiplier is 1, epoch deposits is 0
            checkpoints.push(Checkpoint(currentEpoch + 1, BASE_MULTIPLIER, amount, 0));
        } else {
            uint256 last = checkpoints.length - 1;

            // the last action happened in an older epoch (e.g. a deposit in epoch 3, current epoch is >=5)
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
            // the last action happened in the previous epoch
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
            // the last action happened in the current epoch
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


#Helper functions

 * Returns the id of the current epoch derived from block.timestamp

```
    function getCurrentEpoch() public view returns (uint128) {
        if (block.timestamp < epoch1Start) {
            return 0;
        }

        return uint128((block.timestamp - epoch1Start) / epochDuration + 1);
    }
```

* Returns the percentage of time left in the current epoch
     */
     ```
    function currentEpochMultiplier() public view returns (uint128) {
        uint128 currentEpoch = getCurrentEpoch();
        uint256 currentEpochEnd = epoch1Start + currentEpoch * epochDuration;
        uint256 timeLeft = currentEpochEnd - block.timestamp;
        uint128 multiplier = uint128(timeLeft * BASE_MULTIPLIER / epochDuration);

        return multiplier;
    }
    ```


