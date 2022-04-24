## 金融系统
**包括金融系统的原语合约**

### 0x00 PaymentSplitter.sol
---
**该合约用于实现一个支付分配器，包括`ether`和`token`。**
#### 0. 继承&依赖
##### a. [SafeERC20](./token_erc20.md)
##### b. [Address](./utils.md)
##### c. [Context](./utils.md)

#### 1. 全局变量
```solidity
// 总共的股份数量
uint256 private _totalShares;
// 总共发了多少ether
uint256 private _totalReleased;

// 地址的股份数量
mapping(address => uint256) private _shares;
// 已经发了多少ether
mapping(address => uint256) private _released;
// 收款人的地址
address[] private _payees;

// 每个token总共已经发了多少token
mapping(IERC20 => uint256) private _erc20TotalReleased;
// 每个token向每个地址已经发了多少token
mapping(IERC20 => mapping(address => uint256)) private _erc20Released;
```

### 2. 函数
```solidity
// 构造函数
constructor(address[] memory payees, uint256[] memory shares_) payable {
    // 收款人和股份数组的数量必须相同
    require(payees.length == shares_.length, "PaymentSplitter: payees and shares length mismatch");
    // 收款人数量不为零
    require(payees.length > 0, "PaymentSplitter: no payees");

    // 为每个收款人添加股份
    for (uint256 i = 0; i < payees.length; i++) {
        _addPayee(payees[i], shares_[i]);
    }
}
// 添加收款人的股份
function _addPayee(address account, uint256 shares_) private {
    require(account != address(0), "PaymentSplitter: account is the zero address");
    require(shares_ > 0, "PaymentSplitter: shares are 0");
    require(_shares[account] == 0, "PaymentSplitter: account already has shares");

    _payees.push(account);
    _shares[account] = shares_;
    // 总共的股份数量增加
    _totalShares = _totalShares + shares_;
    // 触发PayeeAdded事件
    emit PayeeAdded(account, shares_);
}

// fallback函数，触发PaymentReceived事件
receive() external payable virtual {
    emit PaymentReceived(_msgSender(), msg.value);
}

// 返回总共的股份数量
function totalShares() public view returns (uint256) {
    return _totalShares;
}

// 返回总共发了多少ether
function totalReleased() public view returns (uint256) {
    return _totalReleased;
}

// 返回token总共已经发了多少token
function totalReleased(IERC20 token) public view returns (uint256) {
    return _erc20TotalReleased[token];
}

// 返回地址持有的股份数量
function shares(address account) public view returns (uint256) {
    return _shares[account];
}

// 返回地址已经发了多少ether
function released(address account) public view returns (uint256) {
    return _released[account];
}

// 返回每个token向地址已经发了多少token
function released(IERC20 token, address account) public view returns (uint256) {
    return _erc20Released[token][account];
}

// 返回index下标收款人的地址
function payee(uint256 index) public view returns (address) {
    return _payees[index];
}

// 按持股比例发钱给收款人
function release(address payable account) public virtual {
    require(_shares[account] > 0, "PaymentSplitter: account has no shares");

    uint256 totalReceived = address(this).balance + totalReleased();
    uint256 payment = _pendingPayment(account, totalReceived, released(account));

    require(payment != 0, "PaymentSplitter: account is not due payment");

    // 该地址已发送的钱增加payment
    _released[account] += payment;
    _totalReleased += payment;

    // 发钱发钱
    Address.sendValue(account, payment);
    // 触发事件
    emit PaymentReleased(account, payment);
}
// 需要发钱的金额，要减去之前已经发给这个地址的金额
function _pendingPayment(
    address account,
    uint256 totalReceived,
    uint256 alreadyReleased
) private view returns (uint256) {
    return (totalReceived * _shares[account]) / _totalShares - alreadyReleased;
}

// 同上，这里是按比例发token
function release(IERC20 token, address account) public virtual {
    require(_shares[account] > 0, "PaymentSplitter: account has no shares");

    uint256 totalReceived = token.balanceOf(address(this)) + totalReleased(token);
    uint256 payment = _pendingPayment(account, totalReceived, released(token, account));

    require(payment != 0, "PaymentSplitter: account is not due payment");

    _erc20Released[token][account] += payment;
    _erc20TotalReleased[token] += payment;

    SafeERC20.safeTransfer(token, account, payment);
    emit ERC20PaymentReleased(token, account, payment);
}
```

### VestingWallet.sol
---
TODO: VestingWallet.sol