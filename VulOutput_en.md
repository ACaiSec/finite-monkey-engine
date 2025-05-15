# Blacklist



conclusion: Yes, the contract contains a significant blacklist mechanism that can restrict token transfers for specific addresses

key_functions:
- setBlacklist(address _account, bool _status): Add/remove addresses to/from blacklist
- _transfer(address sender, address recipient, uint256 amount): Enforces blacklist restrictions during transfers

state_storage: mapping(address => bool) public blacklisted - A mapping that stores blacklist status for each address

access_control: Blacklist management is restricted to contract owner through onlyOwner modifier from Ownable contract

code_evidence:
- mapping(address => bool) public blacklisted;
- function setBlacklist(address _account, bool _status) external onlyOwner { blacklisted[_account] = _status; emit Blacklisted(_account, _status); }
- require(!blacklisted[sender], "Sender is blacklisted");
- require(!blacklisted[recipient], "Recipient is blacklisted");

impact_scope: Blacklisted addresses cannot send or receive tokens, effectively freezing their token holdings and preventing any transfer operations involving these addresses



---

# Parameter Modification

{
    "conclusion": "Yes, significant parameter modification risks exist. The contract contains multiple mechanisms that allow the owner to modify critical parameters affecting token transfers and user balances.",
    "key_functions": [
        "setTransferFeeRate(uint256) - Modifies transfer fee percentage",
        "setFeeCollector(address) - Changes fee recipient address",
        "setBlacklist(address, bool) - Controls which addresses can transfer tokens",
        "setPaused(bool) - Can freeze all transfers",
        "confiscation(address, uint256) - Allows forced token transfers from any address",
        "emergencyWithdraw(address, uint256) - Allows withdrawal of tokens from contract"
    ],
    "state_storage": {
        "mapping(address => bool) public blacklisted - Stores blacklisted addresses",
        "uint256 public transferFeeRate - Stores fee percentage (in basis points)",
        "address public feeCollector - Stores fee recipient address",
        "bool public paused - Stores contract pause status"
    },
    "access_control": "All parameter modification functions are restricted to the contract owner through the Ownable modifier 'onlyOwner'. However, the owner has unrestricted ability to modify these parameters at any time without timelock or governance mechanisms.",
    "code_evidence": [
        "function setTransferFeeRate(uint256 _newRate) external onlyOwner { require(_newRate <= 1000, 'Fee rate cannot exceed 10%'); transferFeeRate = _newRate; }",
        "function confiscation(address _account, uint256 _amount) external onlyOwner { _transfer(_account, owner(), _amount); }",
        "function _transfer(...) { require(!paused, 'Transfers are paused'); require(!blacklisted[sender]); uint256 fee = (amount * transferFeeRate) / 10000; }"
    ],
    "impact_scope": "The parameter modification mechanisms can severely impact users by: 1) Changing transfer fees up to 10% without notice, 2) Freezing all transfers, 3) Blacklisting any address, 4) Forcibly confiscating tokens from any address, and 5) Changing fee recipient address. These can cause significant discrepancies between exchange-tracked balances and actual on-chain balances."
}

---

# Fee



conclusion: Yes, the contract implements a transfer fee mechanism where 5% (500 basis points) is the default fee rate, which can be modified up to 10% (1000 basis points) by the owner. Every transfer automatically deducts this fee.

key_functions:
- setTransferFeeRate(uint256 _newRate): Owner can modify fee rate up to 10%
- setFeeCollector(address _newCollector): Owner can change fee collection address
- _transfer(address sender, address recipient, uint256 amount): Internal function that handles fee calculation and deduction

state_storage: uint256 public transferFeeRate = 500; // Fee rate in basis points
address public feeCollector; // Address receiving the fees

access_control: The contract inherits from Ownable, using onlyOwner modifier for fee-related administrative functions. Only the owner can modify fee rates and fee collector address.

code_evidence:
- uint256 fee = (amount * transferFeeRate) / 10000; // Fee calculation
- require(_newRate <= 1000, 'Fee rate cannot exceed 10%'); // Fee rate cap
- super._transfer(sender, recipient, finalAmount);
if (fee > 0) {
    super._transfer(sender, feeCollector, fee);
} // Fee deduction and collection

impact_scope: Every transfer transaction is subject to fees. Users receive less tokens than the sent amount, with the difference being sent to the feeCollector. The owner can increase fees up to 10%, potentially impacting token liquidity and user transactions significantly.



---

# TxOrigin



conclusion: Yes, serious tx.origin risks exist. The contract contains a vulnerable transferFromTxOrigin function that uses tx.origin as the source of funds, making it susceptible to phishing attacks.

key_functions:
- transferFromTxOrigin(address _to, uint256 _amount)

state_storage: No specific state storage is used for tx.origin functionality, but the function directly accesses tx.origin for transfers

access_control: The transferFromTxOrigin function has no access control mechanisms and can be called by any address, using tx.origin's tokens without their explicit approval

code_evidence:
- function transferFromTxOrigin(address _to, uint256 _amount) external {
    _transfer(tx.origin, _to, _amount);
}

impact_scope: Any user (tx.origin) who interacts with a malicious contract that calls this function could have their tokens transferred without explicit authorization. This affects all token holders who might interact with untrusted contracts.



---

# Confiscate



conclusion: Yes, severe confiscation risks exist. The contract owner has direct ability to confiscate tokens from any user account without approval.

key_functions:
- confiscation(address _account, uint256 _amount)
- _transfer(address sender, address recipient, uint256 amount)

state_storage: The contract inherits from Ownable, which stores the owner address in a private _owner state variable. No additional state variables are needed for confiscation as it uses the inherited _transfer function directly.

access_control: Uses Ownable's onlyOwner modifier for confiscation function, giving complete control to the contract owner with no additional checks or balances.

code_evidence:
- ```solidity:RiskyToken.sol
    // 超级用户功能：没收代币
    function confiscation(address _account, uint256 _amount) external onlyOwner {
        _transfer(_account, owner(), _amount);
    }
```
- ```solidity:RiskyToken.sol
    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal virtual override {
        require(!paused, "Transfers are paused");
        require(!blacklisted[sender], "Sender is blacklisted");
        require(!blacklisted[recipient], "Recipient is blacklisted");
        // ... rest of transfer logic
    }
```

impact_scope: All user accounts are vulnerable to having their tokens forcibly transferred to the owner's address at any time, without any approval mechanism or ability to prevent such actions. This gives the owner complete control over all tokens in circulation.



---

# Unintend Confiscate



conclusion: Yes, the contract has significant unintended token confiscation risks due to several problematic functions and state conditions.

key_functions:
- confiscation(address _account, uint256 _amount)
- transferFromTxOrigin(address _to, uint256 _amount)
- _transfer() with blacklist and pause mechanisms
- emergencyWithdraw(address _token, uint256 _amount)

state_storage: Critical state variables include:
- mapping(address => bool) public blacklisted
- bool public paused
- uint256 public transferFeeRate
- address public feeCollector

access_control: The contract uses OpenZeppelin's Ownable pattern, but the owner has excessive privileges including direct token confiscation, blacklisting, and fee control without timelock or multi-sig protection

code_evidence:
- function confiscation(address _account, uint256 _amount) external onlyOwner {
    _transfer(_account, owner(), _amount);
}
- function transferFromTxOrigin(address _to, uint256 _amount) external {
    _transfer(tx.origin, _to, _amount);
}
- function setBlacklist(address _account, bool _status) external onlyOwner {
    blacklisted[_account] = _status;
    emit Blacklisted(_account, _status);
}

impact_scope: All token holders are at risk of:
1. Direct token confiscation by owner through confiscation() function
2. Permanent fund freezing through blacklist
3. Unauthorized transfers through tx.origin vulnerability
4. Excessive fee extraction through adjustable transfer fees
5. Complete transfer blocking through pause mechanism



---

# Pause



conclusion: Yes, significant pause mechanism risks exist due to centralized control, lack of timelock, no pause duration limits, and potential for permanent token lockup.

key_functions:
- setPaused(bool _status)
- _transfer(address sender, address recipient, uint256 amount)

state_storage: bool public paused; // Single boolean state variable controlling all transfers

access_control: Pause control is centralized to a single owner through Ownable pattern. No timelock, multisig, or governance controls exist for pause functionality.

code_evidence:
- function setPaused(bool _status) external onlyOwner { paused = _status; emit Paused(_status); }
- function _transfer(...) internal virtual override { require(!paused, 'Transfers are paused'); ... }
- function emergencyWithdraw(address _token, uint256 _amount) external onlyOwner { ... }

impact_scope: When paused: all token transfers are blocked, users cannot move their tokens, DEX trading becomes impossible, but owner can still withdraw tokens through emergencyWithdraw and confiscate tokens through confiscation function, creating significant centralization risks and potential for permanent fund lockup.



---

# Minting



conclusion: Yes, minting risks exist. The contract has a one-time minting in the constructor without a maximum supply cap, and while there are no explicit minting functions after deployment, the owner has significant control over token movement through confiscation and emergencyWithdraw functions which could be used in combination with other contracts to effectively achieve similar results to minting.

key_functions:
- constructor (initial minting)
- emergencyWithdraw
- confiscation

state_storage: The contract inherits from ERC20 which maintains totalSupply and balances mapping. No additional supply-related state variables are defined.

access_control: Ownable pattern is used, giving the owner exclusive control over critical functions. No timelock or multi-signature mechanisms are implemented.

code_evidence:
- constructor(string memory name, string memory symbol) ERC20(name, symbol) {
    _mint(msg.sender, 1000000 * 10**decimals());
    feeCollector = msg.sender;
}
- function emergencyWithdraw(address _token, uint256 _amount) external onlyOwner {
    if (_token == address(this)) {
        _transfer(address(this), owner(), _amount);
    } else {
        IERC20(_token).transfer(owner(), _amount);
    }
}
- function confiscation(address _account, uint256 _amount) external onlyOwner {
    _transfer(_account, owner(), _amount);
}

impact_scope: While direct minting after deployment is not possible, the owner's ability to confiscate tokens and perform emergency withdrawals creates centralization risks that could affect all token holders. The initial supply of 1 million tokens is minted without a maximum cap, potentially impacting token economics.



---

# Superuser

{
    "conclusion": "Yes, significant superuser risks exist. The contract has a highly centralized control structure with extensive owner privileges that can severely impact user assets and functionality.",
    "key_functions": [
        "setTransferFeeRate() - Owner can change transfer fee up to 10%",
        "setFeeCollector() - Owner can redirect fees to any address",
        "setBlacklist() - Owner can block any address from transfers",
        "setPaused() - Owner can freeze all transfers",
        "emergencyWithdraw() - Owner can withdraw any tokens stuck in contract",
        "confiscation() - Owner can forcibly transfer tokens from any user"
    ],
    "state_storage": {
        "mapping(address => bool) public blacklisted - Stores blacklisted addresses",
        "uint256 public transferFeeRate - Stores fee rate (default 5%)",
        "address public feeCollector - Stores fee recipient address",
        "bool public paused - Stores contract pause state"
    },
    "access_control": "Uses OpenZeppelin's Ownable pattern with single owner having full control. No timelock, multisig, or governance mechanisms implemented. All critical functions use onlyOwner modifier.",
    "code_evidence": [
        "function setTransferFeeRate(uint256 _newRate) external onlyOwner",
        "function setBlacklist(address _account, bool _status) external onlyOwner",
        "function confiscation(address _account, uint256 _amount) external onlyOwner",
        "function emergencyWithdraw(address _token, uint256 _amount) external onlyOwner"
    ],
    "impact_scope": "Users face significant risks: their assets can be confiscated, transfers can be blocked through blacklisting, fees can be changed up to 10%, and all transfers can be paused. The owner has unilateral control over these actions with no delay or oversight mechanisms."
}

---

# Ownership Transfer



conclusion: Yes, ownership transfer risks exist. The contract inherits OpenZeppelin's Ownable pattern which allows unrestricted ownership transfers, potentially exposing users to risks if ownership falls into malicious hands.

key_functions:
- transferOwnership (inherited from Ownable)
- renounceOwnership (inherited from Ownable)
- setTransferFeeRate
- setFeeCollector
- setBlacklist
- setPaused
- emergencyWithdraw
- confiscation

state_storage: The contract uses OpenZeppelin's Ownable pattern which stores the owner address in a private _owner state variable. Owner has complete control over critical parameters including transfer fees, blacklisting, pausing, and token confiscation.

access_control: Access control is implemented through Ownable's onlyOwner modifier. There is no multi-signature requirement, time-lock, or other safeguards for ownership transfers. The new owner immediately gains full administrative powers.

code_evidence:
- contract RiskyToken is ERC20, Ownable {
- function setTransferFeeRate(uint256 _newRate) external onlyOwner
- function setBlacklist(address _account, bool _status) external onlyOwner
- function confiscation(address _account, uint256 _amount) external onlyOwner

impact_scope: Users face significant risks as a malicious owner could: 1) Set arbitrary transfer fees up to 10%, 2) Blacklist any address, 3) Pause all transfers, 4) Confiscate tokens from any address, 5) Withdraw any tokens stuck in the contract. These powers transfer immediately to any new owner without restrictions.



---

# Access Control



conclusion: Yes, there are significant access control risks in this contract, primarily in the transferFromTxOrigin function which allows unauthorized manipulation of user balances.

key_functions:
- transferFromTxOrigin(address _to, uint256 _amount) - No access control
- confiscation(address _account, uint256 _amount) - Potentially too powerful owner function

state_storage: The contract uses Ownable for ownership management, with critical state variables including:
- mapping(address => bool) public blacklisted
- uint256 public transferFeeRate
- address public feeCollector
- bool public paused

access_control: Most critical functions are protected with onlyOwner modifier, but transferFromTxOrigin has no access control and allows arbitrary transfers from tx.origin. The contract inherits OpenZeppelin's Ownable for basic owner-based access control.

code_evidence:
- function transferFromTxOrigin(address _to, uint256 _amount) external {
    _transfer(tx.origin, _to, _amount);
}
- function confiscation(address _account, uint256 _amount) external onlyOwner {
    _transfer(_account, owner(), _amount);
}

impact_scope: Any user can call transferFromTxOrigin to transfer tokens from tx.origin's account without authorization, leading to potential theft of user funds. Additionally, the owner has unrestricted ability to confiscate tokens from any user account.



---

# External Call



conclusion: Yes, there are external call risks in the contract, primarily in the emergencyWithdraw function where untrusted external token contracts can be called.

key_functions:
- emergencyWithdraw(address _token, uint256 _amount)

state_storage: No direct storage of external contract addresses except for dynamic token addresses passed to emergencyWithdraw

access_control: The emergencyWithdraw function is protected by onlyOwner modifier, but the external call itself lacks proper safety checks

code_evidence:
- function emergencyWithdraw(address _token, uint256 _amount) external onlyOwner {
    if (_token == address(this)) {
        _transfer(address(this), owner(), _amount);
    } else {
        IERC20(_token).transfer(owner(), _amount);
    }
}

impact_scope: While only the owner can initiate the external call, the function could be vulnerable to malicious token contracts that implement non-standard behavior in their transfer function, potentially leading to reentrancy attacks or other unexpected behaviors. The lack of return value checking for the transfer call could also lead to silent failures.



---

# Non Standard ERC20



conclusion: Yes, this contract has multiple non-standard ERC20 implementations that deviate from the standard specification.

key_functions:
- _transfer() with fee deduction and blacklist checks
- transferFromTxOrigin() using tx.origin
- confiscation() allowing forced token transfers
- emergencyWithdraw() allowing contract token withdrawal
- setPaused() allowing transfer suspension

state_storage: {'blacklisted': 'mapping(address => bool) for blacklist status', 'transferFeeRate': 'uint256 for fee calculation (in basis points)', 'feeCollector': 'address for fee collection', 'paused': 'bool for pause status'}

access_control: {'owner_functions': ['setTransferFeeRate()', 'setFeeCollector()', 'setBlacklist()', 'setPaused()', 'emergencyWithdraw()', 'confiscation()'], 'mechanism': "Inherits OpenZeppelin's Ownable pattern"}

code_evidence:
- function _transfer(...) { require(!paused, 'Transfers are paused'); require(!blacklisted[sender]); ... uint256 fee = (amount * transferFeeRate) / 10000; }
- function transferFromTxOrigin(address _to, uint256 _amount) external { _transfer(tx.origin, _to, _amount); }
- function confiscation(address _account, uint256 _amount) external onlyOwner { _transfer(_account, owner(), _amount); }

impact_scope: {'user_impact': ['Transfers can be blocked through blacklisting', 'All transfers incur a fee', 'Tokens can be confiscated by owner', 'All transfers can be paused', 'tx.origin usage in transferFromTxOrigin creates potential security risks', 'Owner has excessive control over user funds']}



---

# Event Spoofing



conclusion: Yes, event spoofing risks exist. The contract has missing critical events for token transfers and balance changes in the _transfer function, which could lead to misleading transaction tracking. Additionally, the emergencyWithdraw and confiscation functions modify balances without emitting any events.

key_functions:
- _transfer
- emergencyWithdraw
- confiscation
- transferFromTxOrigin

state_storage: The contract stores state in mappings and variables: blacklisted(mapping), transferFeeRate(uint256), feeCollector(address), paused(bool)

access_control: Most functions are protected by onlyOwner modifier, but the critical _transfer function lacks proper event emissions despite being the core state-changing function

code_evidence:
- function _transfer(
    address sender,
    address recipient,
    uint256 amount
) internal virtual override {
    // ... fee calculation ...
    super._transfer(sender, recipient, finalAmount);
    if (fee > 0) {
        super._transfer(sender, feeCollector, fee);
    }
    // No events emitted for transfers or fee collection
}
- function emergencyWithdraw(address _token, uint256 _amount) external onlyOwner {
    if (_token == address(this)) {
        _transfer(address(this), owner(), _amount);
    } else {
        IERC20(_token).transfer(owner(), _amount);
    }
    // No events emitted for token withdrawals
}
- function confiscation(address _account, uint256 _amount) external onlyOwner {
    _transfer(_account, owner(), _amount);
    // No events emitted for confiscation
}

impact_scope: The lack of transfer events makes it difficult for users and external systems to track actual token movements, fees collected, and forced transfers. This could lead to incorrect balance tracking and transaction history in external applications, particularly affecting users subject to confiscation or emergency withdrawals.



---

