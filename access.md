## 权限控制
**用于控制权限的合约样例。**
### 0x00 AccessControl.sol
---
**基础的权限控制合约**
#### 0. 继承&依赖
##### a. IAccessControl.sol:
```solidity
interface IAccessControl {
    event RoleAdminChanged(bytes32 indexed role, bytes32 indexed previousAdminRole, bytes32 indexed newAdminRole);
    event RoleGranted(bytes32 indexed role, address indexed account, address indexed sender);
    event RoleRevoked(bytes32 indexed role, address indexed account, address indexed sender);
    function hasRole(bytes32 role, address account) external view returns (bool);
    function getRoleAdmin(bytes32 role) external view returns (bytes32);
    function grantRole(bytes32 role, address account) external;
    function revokeRole(bytes32 role, address account) external;
    function renounceRole(bytes32 role, address account) external;
}
```
##### b. [Context, Strings](utils.md), [ERC165](utils_introspection.md)

#### 1. 全局变量
```solidity
// 角色结构体
struct RoleData {
    // 该角色包含的地址，即哪些地址是该角色的成员
    mapping(address => bool) members;
    // 该类角色的管理角色
    bytes32 adminRole;
}

// 合约的所有角色，用字典保存，`角色名=>角色数据`
mapping(bytes32 => RoleData) private _roles;

// 角色默认的管理角色
bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
```
#### 2. 修饰器
```solidity
// 用于保证某些函数只能由某一类角色执行
modifier onlyRole(bytes32 role) {
    _checkRole(role);
    _;
}
// 检查msg.sender是否是某个角色
function _checkRole(bytes32 role) internal view virtual {
    _checkRole(role, _msgSender());
}
// 检查msg.sender是否是某个角色
function _checkRole(bytes32 role, address account) internal view virtual {
    if (!hasRole(role, account)) {
        revert(
            string(
                abi.encodePacked(
                    "AccessControl: account ",
                    Strings.toHexString(uint160(account), 20),
                    " is missing role ",
                    Strings.toHexString(uint256(role), 32)
                )
            )
        );
    }
}
// 检查`role`角色中是否包含`account`地址，是则返回true，否则返回false，由members中的值决定
function hasRole(bytes32 role, address account) public view virtual override returns (bool) {
    return _roles[role].members[account];
}
```
#### 3. 函数
```solidity
// 获取角色的管理角色
function getRoleAdmin(bytes32 role) public view virtual override returns (bytes32) {
    return _roles[role].adminRole;
}

// 设置角色，只能由该角色的管理角色执行
function grantRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
    _grantRole(role, account);
}
// 设置`account`的角色为`role`，如果`account`已经有角色，直接修改为`true`即可，并触发事件`RoleGranted`
function _grantRole(bytes32 role, address account) internal virtual {
    if (!hasRole(role, account)) {
        _roles[role].members[account] = true;
        emit RoleGranted(role, account, _msgSender());
    }
}

// 取消角色，只能由该角色的管理角色执行
function revokeRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
    _revokeRole(role, account);
}
// 取消角色，如果`account`已经有角色，直接修改为`false`即可，并触发事件`RoleRevoked`
function _revokeRole(bytes32 role, address account) internal virtual {
    if (hasRole(role, account)) {
        _roles[role].members[account] = false;
        emit RoleRevoked(role, account, _msgSender());
    }
}

// 自行退出角色，只能由自己执行
function renounceRole(bytes32 role, address account) public virtual override {
    require(account == _msgSender(), "AccessControl: can only renounce roles for self");

    _revokeRole(role, account);
}

// 设置角色，任意执行，可用作最终的合约管理员设置角色，`样例合约中未使用`
function _setupRole(bytes32 role, address account) internal virtual {
    _grantRole(role, account);
}

// 用于设置角色的管理角色，任意执行，可用作最终的合约管理员设置角色，`样例合约中未使用`，触发事件`RoleAdminChanged`
function _setRoleAdmin(bytes32 role, bytes32 adminRole) internal virtual {
    bytes32 previousAdminRole = getRoleAdmin(role);
    _roles[role].adminRole = adminRole;
    emit RoleAdminChanged(role, previousAdminRole, adminRole);
}
```

### 0x01 AccessControlCrossChain.sol
---
**可跨链的权限控制合约**
#### 0. 继承&依赖
##### a. [AccessControl](#0x00-accesscontrolsol)
##### b. [CrossChainEnabled](crosschain.md)

#### 1. 全局变量
```solidity
// 用于投影跨链的角色
bytes32 public constant CROSSCHAIN_ALIAS = keccak256("CROSSCHAIN_ALIAS");
```

#### 2. 函数
```solidity
// 重写_checkRole函数，判断sender/crossChainSender是否为角色/跨链角色
function _checkRole(bytes32 role) internal view virtual override {
    if (_isCrossChain()) {
        _checkRole(_crossChainRoleAlias(role), _crossChainSender());
    } else {
        super._checkRole(role);
    }
}

// 返回该角色的跨链角色
function _crossChainRoleAlias(bytes32 role) internal pure virtual returns (bytes32) {
    return role ^ CROSSCHAIN_ALIAS;
}
```

### 0x02 AccessControlEnumerable.sol
---
**可枚举的权限控制合约**
#### 0. 继承&依赖
##### a. IAccessControlEnumerable.sol
```solidity
interface IAccessControlEnumerable is IAccessControl {
    function getRoleMember(bytes32 role, uint256 index) external view returns (address);
    function getRoleMemberCount(bytes32 role) external view returns (uint256);
}
```
##### b. [AccessControl](#0x00-accesscontrolsol)
##### c. [EnumerableSet](utils_structs.md)

#### 1. 全局变量
```solidity
// 将角色设置为可枚举的
using EnumerableSet for EnumerableSet.AddressSet;
mapping(bytes32 => EnumerableSet.AddressSet) private _roleMembers;
```

#### 2. 函数
```solidity
// 获取角色的第index个成员
function getRoleMember(bytes32 role, uint256 index) public view virtual override returns (address) {
    return _roleMembers[role].at(index);
}

// 获取角色的成员数量
function getRoleMemberCount(bytes32 role) public view virtual override returns (uint256) {
    return _roleMembers[role].length();
}

// 设置角色的成员
function _grantRole(bytes32 role, address account) internal virtual override {
    super._grantRole(role, account);
    // 相较AccessControl增加了改行代码，用于设置角色的成员为可枚举的
    _roleMembers[role].add(account);
}

// 取消角色的成员
function _revokeRole(bytes32 role, address account) internal virtual override {
    super._revokeRole(role, account);
    // 相较AccessControl增加了改行代码，使得角色的成员为可枚举的
    _roleMembers[role].remove(account);
}
```

### 0x03 Ownable.sol
---
**所有权合约**
#### 0. 继承&依赖
##### a. [Context](utils.md)

#### 1. 全局变量
```solidity
// 私有变量，用于存储所有者
address private _owner;
```

#### 2. 修饰器
```solidity
// 只能由所有者执行
modifier onlyOwner() {
    require(owner() == _msgSender(), "Ownable: caller is not the owner");
    _;
}
```

#### 3. 函数
```solidity
// 构造函数，设置所有者为合约调用者
constructor() {
    _transferOwnership(_msgSender());
}
// 内部函数，转移所有权并触发所有者变更事件`OwnershipTransferred`
function _transferOwnership(address newOwner) internal virtual {
    address oldOwner = _owner;
    _owner = newOwner;
    emit OwnershipTransferred(oldOwner, newOwner);
}

// 获取所有者
function owner() public view virtual returns (address) {
    return _owner;
}

// 放弃所有权，直接设置所有权为0地址
function renounceOwnership() public virtual onlyOwner {
    _transferOwnership(address(0));
}

// 转移所有权，只能由当前所有者执行且不能转移给0地址
function transferOwnership(address newOwner) public virtual onlyOwner {
    require(newOwner != address(0), "Ownable: new owner is the zero address");
    _transferOwnership(newOwner);
}
```
