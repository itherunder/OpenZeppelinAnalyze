## 工具
**用于实现功能的一些工具库合约**

### 0x00 Address.sol
---
**地址相关的工具库合约，library合约，无依赖**

#### 0. 函数
```solidity
// 判断地址是否为合约地址
function isContract(address account) internal view returns (bool) {
    // This method relies on extcodesize/address.code.length, which returns 0
    // for contracts in construction, since the code is only stored at the end
    // of the constructor execution.
    // 以前还需要extcodesize 内联实现，现在不需要了，详情查看https://github.com/ethereum/solidity/blob/develop/Changelog.md
    return account.code.length > 0;
}

// 发送以太
function sendValue(address payable recipient, uint256 amount) internal {
    require(address(this).balance >= amount, "Address: insufficient balance");

    // 可以把所有的gas 都发过去执行，而transfer 只有2300 gas可以供给调用的合约使用
    (bool success, ) = recipient.call{value: amount}("");
    // 如果报错，直接revert
    require(success, "Address: unable to send value, recipient may have reverted");
}

// 函数调用，封装调用合约的函数
function functionCall(address target, bytes memory data) internal returns (bytes memory) {
    return functionCall(target, data, "Address: low-level call failed");
}
// 继续封装，加了个错误信息参数
function functionCall(
    address target,
    bytes memory data,
    string memory errorMessage
) internal returns (bytes memory) {
    return functionCallWithValue(target, data, 0, errorMessage);
}
// 继续封装，加了个value参数，可以发送以太
function functionCallWithValue(
    address target,
    bytes memory data,
    uint256 value,
    string memory errorMessage
) internal returns (bytes memory) {
    // 如果没有足够的以太，直接revert
    require(address(this).balance >= value, "Address: insufficient balance for call");
    // 调用的必须是合约地址
    require(isContract(target), "Address: call to non-contract");

    // 用底层call，发送以太并携带执行参数data
    (bool success, bytes memory returndata) = target.call{value: value}(data);
    // 验证结果
    return verifyCallResult(success, returndata, errorMessage);
}
// 验证结果
function verifyCallResult(
    bool success,
    bytes memory returndata,
    string memory errorMessage
) internal pure returns (bytes memory) {
    if (success) {
        return returndata;
    } else {
        // Look for revert reason and bubble it up if present
        if (returndata.length > 0) {
            // The easiest way to bubble the revert reason is using memory via assembly

            // 利用内联汇编revert并且将调用合约的结果放到内存中
            assembly {
                let returndata_size := mload(returndata)
                revert(add(32, returndata), returndata_size)
            }
        } else {
            // 如果没有返回内容的话，直接revert错误信息
            revert(errorMessage);
        }
    }
}

// 调用上面的functionCallWithValue，携带异常信息
function functionCallWithValue(
    address target,
    bytes memory data,
    uint256 value
) internal returns (bytes memory) {
    return functionCallWithValue(target, data, value, "Address: low-level call with value failed");
}

// 封装staticcall 函数，以静态调用的方式调用合约的函数
function functionStaticCall(address target, bytes memory data) internal view returns (bytes memory) {
    return functionStaticCall(target, data, "Address: low-level static call failed");
}
// 继续封装，加了个错误信息参数
function functionStaticCall(
    address target,
    bytes memory data,
    string memory errorMessage
) internal view returns (bytes memory) {
    // 由于staticcall不允许所调用的合约状态发生改变，因此也不能发送以太
    require(isContract(target), "Address: static call to non-contract");

    // staticcall，和call 一样，不过不允许target 合约修改它自身的状态
    (bool success, bytes memory returndata) = target.staticcall(data);
    // 验证结果
    return verifyCallResult(success, returndata, errorMessage);
}

// 封装delegatecall 函数，以delegatecall 调用的方式调用合约的函数
function functionDelegateCall(address target, bytes memory data) internal returns (bytes memory) {
    return functionDelegateCall(target, data, "Address: low-level delegate call failed");
}
// 继续封装，加了个错误信息参数
function functionDelegateCall(
    address target,
    bytes memory data,
    string memory errorMessage
) internal returns (bytes memory) {
    require(isContract(target), "Address: delegate call to non-contract");

    // delegatecall，仅使用target 合约的代码，其他状态、内存都是使用调用者的，目的是为了使用library 合约的代码，因此当代码中使用了library 时都会使用到这个delegatecall 调用
    (bool success, bytes memory returndata) = target.delegatecall(data);
    // 验证结果
    return verifyCallResult(success, returndata, errorMessage);
}
```