## 跨链合约
**用于实现跨链的合约样例。**
### 0x00 CrossChainEnabled.sol
该合约为虚合约，类似于`C++`中的抽象类，因此仅包含函数声明，不包含实现
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
该合约仅声明了两个[自定义异常](https://blog.soliditylang.org/2021/04/21/custom-errors/)，需要编译器版本在`V0.8.4`以上
```solidity
error NotCrossChainCall();
error InvalidCrossChainSender(address actual, address expected);
```