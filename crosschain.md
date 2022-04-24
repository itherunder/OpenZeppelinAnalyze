## 跨链合约
**用于实现跨链的合约样例。**
### 0x00 CrossChainEnabled.sol
---
**该合约为虚合约，类似于`C++`中的抽象类，因此仅包含函数声明，不包含实现**
#### 0. 继承&依赖
##### a. [errors](#0x01-errorssol)

#### 1. 修饰器&函数
```solidity
modifier onlyCrossChain() {
    // 用于检查合约是否是跨链合约
    if (!_isCrossChain()) revert NotCrossChainCall();
    _;
}
// 是否为跨链合约，需要继承合约实现
function _isCrossChain() internal view virtual returns (bool);

modifier onlyCrossChainSender(address expected) {
    // 判断合约跨链调用者是否为expected地址
    address actual = _crossChainSender();
    if (expected != actual) revert InvalidCrossChainSender(actual, expected);
    _;
}
// 获取跨链合约的sender，需要继承合约实现
function _crossChainSender() internal view virtual returns (address);
```

### 0x01 errors.sol
---
**该合约仅声明了两个[自定义异常](https://blog.soliditylang.org/2021/04/21/custom-errors/)，需要编译器版本在`V0.8.4`以上**
#### 0. 异常声明
```solidity
error NotCrossChainCall();
error InvalidCrossChainSender(address actual, address expected);
```

#### 1. testing
经过对下面这个合约的测试（注释另一个函数后部署调用测得执行合约所花费的gas），确实自定义`error`的方式会便宜些
```solidity
pragma solidity ^0.8.4;

error SoSmall();

contract C {
    function T(uint x) public returns(uint) { // 21530
        require(x > 10, "SoSmall");
        return x * 10;
    }
    function T(uint x) public returns(uint) { // 21470
        if (x <= 10) {
            revert SoSmall();
        }
        return x * 10;
    }
}
```