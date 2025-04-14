// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title TRON Staking Contract
 * @dev This contract allows users to stake native TRX and TRC20 USDT to earn rewards based on fixed-term staking plans.
 *      Users can stake multiple times with separate entries for TRX and USDT.
 *      The contract supports configurable staking durations, reward percentages, and fee deduction.
 *      It also provides a collection of view functions to retrieve detailed staking information.
 */

// Minimal interface for TRC20 (ERC20) token, for USDT.
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract TronStaking {
    /// @notice Address of the TRC20 USDT token contract.
    IERC20 public usdtToken;

    /// @notice Owner address (contract deployer by default) with special permissions.
    address public owner;

    /// @notice Staking fee percentage. (Set to 5% by default; can be changed.)
    uint256 public feePercent = 5;

    /// @notice Reward percentage for 1-month plan (e.g., 100 means 100% reward).
    uint256 public rewardPercent1Month = 100;
    /// @notice Reward percentage for 2-month plan.
    uint256 public rewardPercent2Months = 200;
    /// @notice Reward percentage for 3-month plan.
    uint256 public rewardPercent3Months = 300;

    /// @notice Duration in seconds for the 1-month plan. (Default: 30 days.)
    uint256 public duration1Month = 30 days;
    /// @notice Duration in seconds for the 2-month plan. (Default: 60 days.)
    uint256 public duration2Months = 60 days;
    /// @notice Duration in seconds for the 3-month plan. (Default: 90 days.)
    uint256 public duration3Months = 90 days;

    /// @dev Internal accounting for accumulated fee (in TRX) that the owner may withdraw.
    uint256 private accumulatedTrxFees;
    /// @dev Internal accounting for accumulated fee (in USDT) that the owner may withdraw.
    uint256 private accumulatedUsdtFees;

    /// @notice Structure that records details of a user's single stake.
    struct Stake {
        uint256 amount;         // Amount staked
        bool isTRX;             // Token type: true for TRX, false for USDT
        uint256 stakeTime;      // Timestamp when stake was created
        uint256 withdrawTime;   // Timestamp when stake was withdrawn (0 if not yet withdrawn)
        uint256 planMonths;     // Selected staking duration in months (1, 2, or 3)
        bool withdrawn;         // Whether the stake has been withdrawn
        uint256 withdrawAmount; // Amount actually withdrawn (after reward and fee deduction)
    }

    /// @notice Mapping from user address to a dynamic array of stakes.
    mapping(address => Stake[]) public stakes;

    // --- Global Stakes Tracking for "All Active Stakes" Queries ---

    /// @notice Structure to link a stake with its owner and index.
    struct GlobalStake {
        address user;
        uint256 index;
    }
    /// @notice Global array that holds every stake added (for off-chain global queries).
    GlobalStake[] public allStakes;

    // --- Helper Structs for View Functions ---
    
    /// @notice Returns a stake along with its index in the user’s stake array.
    struct StakeWithIndex {
        uint256 index;
        Stake stake;
    }

    /// @notice Returns global stake details including owner, index, and stake info.
    struct GlobalStakeDetail {
        address user;
        uint256 index;
        Stake stake;
    }

    // --- Events ---

    /// @notice Emitted when a user creates a new stake.
    event Staked(address indexed user, bool indexed isTRX, uint256 amount, uint256 planMonths, uint256 stakeIndex);
    /// @notice Emitted when a user withdraws from a stake.
    event Withdrawn(
        address indexed user,
        bool indexed isTRX,
        uint256 stakeIndex,
        uint256 amountWithdrawn,
        uint256 rewardEarned,
        uint256 feeDeducted,
        bool earlyWithdraw
    );
    /// @notice Emitted when the owner withdraws accumulated fees.
    event OwnerWithdrawn(address indexed owner, uint256 trxAmount, uint256 usdtAmount);

    // --- Modifiers ---

    /// @dev Modifier to restrict access to only the owner.
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    /// @dev Reentrancy guard modifier.
    bool private locked;
    modifier nonReentrant() {
        require(!locked, "Reentrant call");
        locked = true;
        _;
        locked = false;
    }

    // --- Constructor ---

    /**
     * @param _usdtTokenAddress The TRC20 USDT token contract address.
     */
    constructor(address _usdtTokenAddress) {
        require(_usdtTokenAddress != address(0), "USDT token address cannot be zero");
        owner = msg.sender;
        usdtToken = IERC20(_usdtTokenAddress);
    }

    // --- Staking Functions ---

    /**
     * @notice Stake TRX (native coin) for a specified plan.
     * @param planMonths The staking duration in months (must be 1, 2, or 3).
     * 
     * The function is payable; send the TRX amount along with this call.
     */
    function stakeTRX(uint256 planMonths) external payable nonReentrant {
        require(msg.value > 0, "Staking amount must be greater than 0");
        require(planMonths == 1 || planMonths == 2 || planMonths == 3, "Invalid plan selected");

        Stake memory newStake = Stake({
            amount: msg.value,
            isTRX: true,
            stakeTime: block.timestamp,
            withdrawTime: 0,
            planMonths: planMonths,
            withdrawn: false,
            withdrawAmount: 0
        });
        stakes[msg.sender].push(newStake);
        uint256 stakeIndex = stakes[msg.sender].length - 1;

        // Record global stake for off-chain queries.
        allStakes.push(GlobalStake({user: msg.sender, index: stakeIndex}));

        emit Staked(msg.sender, true, msg.value, planMonths, stakeIndex);
    }

    /**
     * @notice Stake USDT (TRC20 token) for a specified plan.
     * @param amount The amount of USDT to stake (ensure prior approval).
     * @param planMonths The staking duration in months (must be 1, 2, or 3).
     */
    function stakeUSDT(uint256 amount, uint256 planMonths) external nonReentrant {
        require(amount > 0, "Staking amount must be greater than 0");
        require(planMonths == 1 || planMonths == 2 || planMonths == 3, "Invalid plan selected");

        bool success = usdtToken.transferFrom(msg.sender, address(this), amount);
        require(success, "USDT transfer failed. Check allowance and balance.");

        Stake memory newStake = Stake({
            amount: amount,
            isTRX: false,
            stakeTime: block.timestamp,
            withdrawTime: 0,
            planMonths: planMonths,
            withdrawn: false,
            withdrawAmount: 0
        });
        stakes[msg.sender].push(newStake);
        uint256 stakeIndex = stakes[msg.sender].length - 1;

        allStakes.push(GlobalStake({user: msg.sender, index: stakeIndex}));

        emit Staked(msg.sender, false, amount, planMonths, stakeIndex);
    }

    /**
     * @notice Withdraw a specific stake after or before the staking period.
     * @param stakeIndex The index of the stake to withdraw.
     * 
     * If the stake period is complete, the user receives their stake plus the hardcoded reward, minus fees.
     * If withdrawn early, the user forfeits the reward (only receiving the stake minus fees).
     */
    function withdrawStake(uint256 stakeIndex) external nonReentrant {
        require(stakeIndex < stakes[msg.sender].length, "Invalid stake index");
        Stake storage userStake = stakes[msg.sender][stakeIndex];
        require(!userStake.withdrawn, "Stake already withdrawn");

        uint256 planMonths = userStake.planMonths;
        uint256 stakingDuration;
        uint256 rewardPercent;
        if (planMonths == 1) {
            stakingDuration = duration1Month;
            rewardPercent = rewardPercent1Month;
        } else if (planMonths == 2) {
            stakingDuration = duration2Months;
            rewardPercent = rewardPercent2Months;
        } else {
            stakingDuration = duration3Months;
            rewardPercent = rewardPercent3Months;
        }

        bool isMatured = block.timestamp >= userStake.stakeTime + stakingDuration;
        uint256 reward = 0;
        if (isMatured) {
            // Reward is a fixed percentage of the staked amount.
            reward = (userStake.amount * rewardPercent) / 100;
        }
        uint256 grossAmount = userStake.amount + reward;
        uint256 feeAmount = (grossAmount * feePercent) / 100;
        uint256 payout = grossAmount - feeAmount;

        userStake.withdrawn = true;
        userStake.withdrawTime = block.timestamp;
        userStake.withdrawAmount = payout;

        if (userStake.isTRX) {
            require(address(this).balance >= payout, "Insufficient TRX balance in contract");
            payable(msg.sender).transfer(payout);
            accumulatedTrxFees += feeAmount;
        } else {
            uint256 contractUsdtBalance = usdtToken.balanceOf(address(this));
            require(contractUsdtBalance >= payout, "Insufficient USDT balance in contract");
            bool sent = usdtToken.transfer(msg.sender, payout);
            require(sent, "USDT transfer failed");
            accumulatedUsdtFees += feeAmount;
        }

        emit Withdrawn(msg.sender, userStake.isTRX, stakeIndex, payout, reward, feeAmount, !isMatured);
    }

    // --- Owner Functions ---

    /**
     * @notice Withdraw accumulated fees (both TRX and USDT) to the owner's address.
     */
    function withdrawFees() external onlyOwner {
        uint256 trxToWithdraw = accumulatedTrxFees;
        uint256 usdtToWithdraw = accumulatedUsdtFees;
        require(trxToWithdraw > 0 || usdtToWithdraw > 0, "No fees available for withdrawal");

        accumulatedTrxFees = 0;
        accumulatedUsdtFees = 0;

        if (trxToWithdraw > 0) {
            require(address(this).balance >= trxToWithdraw, "Insufficient TRX balance");
            payable(owner).transfer(trxToWithdraw);
        }
        if (usdtToWithdraw > 0) {
            uint256 contractUsdtBalance = usdtToken.balanceOf(address(this));
            require(contractUsdtBalance >= usdtToWithdraw, "Insufficient USDT balance");
            bool success = usdtToken.transfer(owner, usdtToWithdraw);
            require(success, "USDT transfer failed");
        }

        emit OwnerWithdrawn(msg.sender, trxToWithdraw, usdtToWithdraw);
    }

    /**
     * @notice Emergency function for the owner to withdraw all funds (TRX and USDT) from the contract.
     * @dev Use with caution as it may include user staked funds.
     */
    function emergencyWithdrawAll() external onlyOwner {
        // Loop through all global stakes and mark any still-active stake as withdrawn.
        uint256 len = allStakes.length;
        for (uint256 i = 0; i < len; i++) {
            GlobalStake storage gStake = allStakes[i];
            Stake storage s = stakes[gStake.user][gStake.index];
            if (!s.withdrawn) {
                s.withdrawn = true;
                s.withdrawTime = block.timestamp;
                s.withdrawAmount = 0; // No payout because funds are being withdrawn by the owner.
            }
        }

        // Withdraw all TRX and USDT from the contract.
        uint256 trxBalance = address(this).balance;
        if (trxBalance > 0) {
            payable(owner).transfer(trxBalance);
        }
        uint256 usdtBalance = usdtToken.balanceOf(address(this));
        if (usdtBalance > 0) {
            bool success = usdtToken.transfer(owner, usdtBalance);
            require(success, "USDT transfer failed");
        }
        accumulatedTrxFees = 0;
        accumulatedUsdtFees = 0;

        emit OwnerWithdrawn(msg.sender, trxBalance, usdtBalance);
    }

    /**
     * @notice Transfer contract ownership to a new address.
     * @param newOwner The new owner's address.
     */
    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "New owner cannot be the zero address");
        owner = newOwner;
    }
    
    /**
     * @notice Change the USDT token contract address.
     * @param newTokenAddress The new TRC20 token contract address.
     * Only the owner can call this function.
     */
    function setUsdtToken(address newTokenAddress) external onlyOwner {
        require(newTokenAddress != address(0), "Invalid token address");
        usdtToken = IERC20(newTokenAddress);
    }

    // --- Additional View Functions for Staking Data Retrieval ---

    /**
     * @notice Retrieve all stakes of the caller.
     * @return An array of all stakes for msg.sender.
     */
    function getUserStakes() external view returns (Stake[] memory) {
        return stakes[msg.sender];
    }
    
    /**
     * @notice Retrieve all stakes for a specified user.
     * @param user The address of the user.
     * @return An array of all stakes for the specified user.
     */
    function getUserAllStakes(address user) external view returns (Stake[] memory) {
        return stakes[user];
    }

    /**
     * @notice Retrieve only TRX stakes of the caller along with their index IDs.
     * @return An array of stakes (with index) where the token staked is TRX.
     */
    function getUserTrxStakes() external view returns (StakeWithIndex[] memory) {
        Stake[] storage userStakes = stakes[msg.sender];
        uint256 count = 0;
        for (uint256 i = 0; i < userStakes.length; i++) {
            if (userStakes[i].isTRX) {
                count++;
            }
        }
        StakeWithIndex[] memory result = new StakeWithIndex[](count);
        uint256 j = 0;
        for (uint256 i = 0; i < userStakes.length; i++) {
            if (userStakes[i].isTRX) {
                result[j] = StakeWithIndex(i, userStakes[i]);
                j++;
            }
        }
        return result;
    }

    /**
     * @notice Retrieve only USDT stakes of the caller along with their index IDs.
     * @return An array of stakes (with index) where the token staked is USDT.
     */
    function getUserUsdtStakes() external view returns (StakeWithIndex[] memory) {
        Stake[] storage userStakes = stakes[msg.sender];
        uint256 count = 0;
        for (uint256 i = 0; i < userStakes.length; i++) {
            if (!userStakes[i].isTRX) {
                count++;
            }
        }
        StakeWithIndex[] memory result = new StakeWithIndex[](count);
        uint256 j = 0;
        for (uint256 i = 0; i < userStakes.length; i++) {
            if (!userStakes[i].isTRX) {
                result[j] = StakeWithIndex(i, userStakes[i]);
                j++;
            }
        }
        return result;
    }

    /**
     * @notice Retrieve all active (not withdrawn) stakes of the caller along with their index IDs.
     * @return An array of active stakes (with index) for msg.sender.
     */
    function getActiveUserStakes() external view returns (StakeWithIndex[] memory) {
        Stake[] storage userStakes = stakes[msg.sender];
        uint256 count = 0;
        for (uint256 i = 0; i < userStakes.length; i++) {
            if (!userStakes[i].withdrawn) {
                count++;
            }
        }
        StakeWithIndex[] memory result = new StakeWithIndex[](count);
        uint256 j = 0;
        for (uint256 i = 0; i < userStakes.length; i++) {
            if (!userStakes[i].withdrawn) {
                result[j] = StakeWithIndex(i, userStakes[i]);
                j++;
            }
        }
        return result;
    }

    /**
     * @notice Retrieve the contract's current balances.
     * @return trxBalance The native TRX balance of the contract.
     * @return usdtBalance The USDT balance held by the contract.
     */
    function getContractBalance() external view returns (uint256 trxBalance, uint256 usdtBalance) {
        trxBalance = address(this).balance;
        usdtBalance = usdtToken.balanceOf(address(this));
    }

    /**
     * @notice Retrieve the contract's USDT token balance.
     * @return The USDT token balance held by this contract.
     */
    function getContractUsdtBalance() external view returns (uint256) {
        return usdtToken.balanceOf(address(this));
    }
    
    /**
     * @notice Retrieve the contract's native TRX balance.
     * @return The TRX balance (in sun) held by this contract.
     */
    function getContractTrxBalance() external view returns (uint256) {
        return address(this).balance;
    }
    
    /**
     * @notice Retrieve the total accumulated fees (not yet withdrawn by the owner).
     * @return totalFeesTrx Accumulated TRX fees.
     * @return totalFeesUsdt Accumulated USDT fees.
     */
    function getTotalFees() external view returns (uint256 totalFeesTrx, uint256 totalFeesUsdt) {
        totalFeesTrx = accumulatedTrxFees;
        totalFeesUsdt = accumulatedUsdtFees;
    }

    /**
     * @notice Retrieve the total active (non-withdrawn) staked amounts split by token type.
     * @return totalActiveTrx Total active TRX staked in the contract.
     * @return totalActiveUsdt Total active USDT staked in the contract.
     */
    function getTotalActiveStakedAmount() external view returns (uint256 totalActiveTrx, uint256 totalActiveUsdt) {
        uint256 len = allStakes.length;
        for (uint256 i = 0; i < len; i++) {
            GlobalStake storage gStake = allStakes[i];
            Stake storage s = stakes[gStake.user][gStake.index];
            if (!s.withdrawn) {
                if (s.isTRX) {
                    totalActiveTrx += s.amount;
                } else {
                    totalActiveUsdt += s.amount;
                }
            }
        }
    }

    /**
     * @notice Retrieve all active (non-withdrawn) stakes across all users.
     * @return An array of GlobalStakeDetail structs, each containing the owner, stake index, and stake details.
     */
    function getAllActiveStakes() external view returns (GlobalStakeDetail[] memory) {
        uint256 count = 0;
        uint256 len = allStakes.length;
        for (uint256 i = 0; i < len; i++) {
            GlobalStake storage gStake = allStakes[i];
            Stake storage s = stakes[gStake.user][gStake.index];
            if (!s.withdrawn) {
                count++;
            }
        }
        GlobalStakeDetail[] memory result = new GlobalStakeDetail[](count);
        uint256 j = 0;
        for (uint256 i = 0; i < len; i++) {
            GlobalStake storage gStake = allStakes[i];
            Stake storage s = stakes[gStake.user][gStake.index];
            if (!s.withdrawn) {
                result[j] = GlobalStakeDetail({user: gStake.user, index: gStake.index, stake: s});
                j++;
            }
        }
        return result;
    }

    // --- Fallback Functions ---

    /**
     * @dev Fallback function for unmatched function calls with data.
     */
    fallback() external payable {
        revert("Unsupported operation");
    }

    /**
     * @dev Receive function to accept plain TRX transfers.
     */
    receive() external payable {}
}
