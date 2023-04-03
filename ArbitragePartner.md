Cron-Fi Arbitrage Partner Integration 1.0
================================================================================


Summary
--------------------------------------------------------------------------------

Arbitrage Partner integration requires a shareable public Ethereum blockchain
address controlled by the partner and a deployed Arbitrage List Contract. The
Arbitrage List Contract contains a list of the Arbitrage Partner's arbitrageurs
and conforms to the Arbitrage List Interface. The system is described in further
detail below.


Arbitrage Partner Address
--------------------------------------------------------------------------------

The public address of an Ethereum Blockchain wallet, controlled by the Arbitrage
Partner. This address is shared with arbitrageurs and passed into the Cron-Fi
TWAMM pool's swap function as user data. It is used to decode the current
address of the Arbitrage Partner's Arbitrage List Contract, which is then used
to confirm the swap function caller is permitted to swap at the reduced fee.


Arbitrageur List Contract
--------------------------------------------------------------------------------

The Arbitrageur List Contract is a contract deployed to the Ethereum blockchain
mainnet by the Arbitrage Partner. It contains a list of the Arbitrage Partner's
arbitrageur addresses. The list can be updated in place, or updated wholesale by
deploying an entirely new list and then calling the Cron-Fi TWAMM pool's update
arbitrage list method (which can only be called by the Arbitrage Partner
Address).

The method to update the arbitrage list has the following definition and is the
only method uniquely callable by the Arbitrage Partner:

  function updateArbitrageList() 
           external
           senderIsArbitragePartner
           nonReentrant returns (address);

While this method must be defined, it's use is optional and a configurable
Arbitrage List Contract as provided in the example below is entirely possible.
The flexibility is provided to meet the needs of different partner requirements
and scale.


Example Partner Swap Function Call (Preliminary)
--------------------------------------------------------------------------------
```solidity
    uint256 amountIn = ???;                // The amount of tokenIn being sold
                                           // to the pool.
    address tokenIn = ???;                 // The ERC20 contract address of one
                                           // of the pool's two tokens to be
                                           // sold to the pool in exchange for
                                           // the other token.
    address arbitrageurAddress = ???;
    address arbitrageParnerAddress = ???;

    bytes32 poolId = ICronV1Pool(_pool).POOL_ID();
    (IERC20[] memory tokens, , ) = vault.getPoolTokens(poolId);
    IAsset[] memory assets = _convertERC20sToAssets(tokens);

    // Approve tokens to spend from this contract in the vault:
    //
    IERC20 token = (tokenIn == address(tokens[0])) ? tokens[0] : tokens[1];
    token.approve(address(vault), amountIn);

    uint256 amountOut = IVault(vault).swap(
      IVault.SingleSwap(
        poolId,
        IVault.SwapKind.GIVEN_IN,
        (tokenIn == address(tokens[0])) ? assets[0] : assets[1],
        (tokenIn == address(tokens[0])) ? assets[1] : assets[0],
        amountIn,
        abi.encode(
          ICronV1Pool.SwapType.PartnerSwap,
          uint256(arbitrageParnerAddress) 
        )
      ),
      IVault.FundManagement(
        arbitrageurAddress,
        false,
        payable (arbitrageurAddress),
        false
      ),
      0,
      block.timestamp + 1000
    );

...

  function _convertERC20sToAssets(IERC20[] memory tokens)
           internal
           pure
           returns (IAsset[] memory assets) {
    // solhint-disable-next-line no-inline-assembly
    assembly {
      assets := tokens
    }
  }
```

Arbitrage List Interface  (Preliminary)
--------------------------------------------------------------------------------

```solidity
// (c) Copyright 2023, Bad Pumpkin Inc. All Rights Reserved
//
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.7.6;

/// @notice Interface for managing list of addresses permitted to perform
///         preferred rate arbitrage swaps on Cron-Fi TWAMM V1.0.
///
interface IArbitrageurList {
  /// @param sender is the address that called the function changing list owner
  ///               permissions.
  /// @param listOwner is the address to change list owner permissions on.
  /// @param permission is true if the address specified in listOwner is granted
  ///                   list owner permissions. Is false otherwise.
  ///
  event ListOwnerPermissions(address indexed sender,
                             address indexed listOwner,
                             bool indexed permission);

  /// @param sender is the address that called the function changing arbitrageur
  ///               permissions.
  /// @param arbitrageurs is a list of addresses to change arbitrage permissions
  ///                     on.
  /// @param permission is true if the addresses specified in arbitrageurs is
  ///                   granted arbitrage permissions. Is false otherwise.
  ///
  event ArbitrageurPermissions(address indexed sender,
                               address[] arbitrageurs,
                               bool indexed permission);

  /// @param sender is the address that called the function changing the next
  ///               list address.
  /// @param nextListAddress is the address the return value of the nextList
  ///                        function is set to.
  ///
  event NextList(address indexed sender, address indexed nextListAddress);

  /// @notice Returns true if the provide address is permitted the preferred
  ///         arbitrage rate in the partner swap method of a Cron-Fi TWAMM pool.
  ///         Returns false otherwise.
  /// @param _address the address to check for arbitrage rate permissions.
  ///
  function isArbitrageur(address _address) external returns (bool);

  /// @notice Returns the address of the next contract implementing the next
  ///         list of arbitrageurs.  If the return value is the NULL address,
  ///         address(0), then the TWAMM contract's update list method will keep
  ///         the existing address it is storing to check for arbitrage
  ///         permissions.
  ///
  function nextList() external returns (address);
}
```

Example Arbitrage List Contract
--------------------------------------------------------------------------------

The following example is an example of a possible arbitrage list contract--it
allows the deployer to add and remove arbitrageurs and configure the list as
needed. Other possibilities are possible and encouraged to suite the needs of
the Arbitrage Partner.

```solidity
// (c) Copyright 2023, Bad Pumpkin Inc. All Rights Reserved
//
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.7.6;

import { IArbitrageurList } from "../interfaces/IArbitrageurList.sol";
import { ReentrancyGuard } from "../balancer-core-v2/lib/openzeppelin/ReentrancyGuard.sol";
 
address constant NULL_ADDR = address(0);

/// @notice Abstract contract for managing list of addresses permitted to
///         perform preferred rate arbitrage swaps on Cron-Fi TWAMM V1.0.
///
/// @dev    In Cron-Fi TWAMM V1.0 pools, the partner swap (preferred rate
///         arbitrage swap) may only be successfully called by an address that
///         returns true when isArbitrageur in a contract derived from this one
///         (the caller must also specify the address of the arbitrage partner
///         to facilitate a call to isArbitrageur in the correct contract).
///
/// @dev    Two mechanisms are provided for updating the arbitrageur list, they
///         are:
///             - The setArbitrageurs method, which allows a list of addresses
///               to be given or removed arbitrage permission.
///             - The nextList mechanism. In order to use this mechanism, a new
///               contract deriving from this contract with new arbitrage
///               addresses specified must be deployed. A listOwner then sets
///               the nextList address to the newly deployed contract address
///               with the setNextList method. Finally, the arbPartner address
///               in the corresponding Cron-Fi TWAMM contract will then call
///               updateArbitrageList to retrieve the new arbitrageur list
///               contract address from this contract instance. Note that all
///               previous arbitraguer contracts in the TWAMM contract using the
///               updated list are ignored.
///
/// @dev    Note that this is a bare-bones implementation without conveniences
///         like a list to inspect all current arbitraguer addresses at once
///         (emitted events can be consulted and aggregated off-chain for this
///         purpose), however users are encouraged to modify the contract as
///         they wish as long as the following methods continue to function as
///         specified:
///             - isArbitrageur
///
contract ArbitrageurListExample is IArbitrageurList, ReentrancyGuard {
  mapping(address => bool) private listOwners;
  mapping(address => bool) private permittedAddressMap;
  address public override(IArbitrageurList) nextList;

  modifier senderIsListOwner() {
    require(listOwners[msg.sender], "Sender must be listOwner");
    _;
  }

  /// @notice Constructs this contract with next contract and the specified list
  ///         of addresses permitted to arbitrage.
  /// @param _arbitrageurs is a list of addresses to give arbitrage permission
  ///                      to on contract instantiation.
  ///
  constructor(address[] memory _arbitrageurs) {
    bool permitted = true;

    listOwners[msg.sender] = permitted;
    emit ListOwnerPermissions(msg.sender, msg.sender, permitted);

    setArbitrageurs(_arbitrageurs, permitted);
    emit ArbitrageurPermissions(msg.sender, _arbitrageurs, permitted);

    nextList = NULL_ADDR;
  }

  /// @notice Sets whether or not a specified address is a list owner.
  /// @param _address is the address to give or remove list owner priviliges
  ///                 from.
  /// @param _permitted if true, gives the specified address list owner
  ///                   priviliges. If false removes list owner priviliges.
  ///
  function setListOwner(address _address, bool _permitted)
           public
           nonReentrant
           senderIsListOwner {
    listOwners[_address] = _permitted;

    emit ListOwnerPermissions(msg.sender, _address, _permitted);
  }

  /// @notice Sets whether the specified list of addresses is permitted to
  ///         arbitrage Cron-Fi TWAMM pools at a preffered rate or not.
  /// @param _arbitrageurs is a list of addresses to add or remove arbitrage
  ///                      permission from.
  /// @param _permitted specifies if the list of addresses contained in
  ///                   _arbitrageurs will be given arbitrage permission when
  ///                   set to true. When false, arbitrage permission is removed
  ///                   from the specified addresses.
  ///
  function setArbitrageurs(address[] memory _arbitrageurs, bool _permitted)
           public
           nonReentrant
           senderIsListOwner {
    uint256 length = _arbitrageurs.length;
    for (uint256 index = 0; index < length; index++) {
      permittedAddressMap[_arbitrageurs[index]] = _permitted;
    }

    emit ArbitrageurPermissions(msg.sender, _arbitrageurs, _permitted);
  }

  /// @notice Sets the next contract address to use for arbitraguer permissions.
  ///         Requires that the contract be instantiated and that a call to
  ///         updateArbitrageList is made by the arbitrage partner list on the
  ///         corresponding TWAMM pool.
  /// @param _address is the address of the instantiated contract deriving from
  ///                 this contract to use for address arbitrage permissions.
  ///
  function setNextList(address _address) public nonReentrant senderIsListOwner {
    nextList = _address;

    emit NextList(msg.sender, _address);
  }

  /// @notice Returns true if specified address has list owner permissions.
  /// @param _address is the address to check for list owner permissions.
  ///
  function isListOwner(address _address) public view returns (bool) {
    return listOwners[_address];
  }

  /// @notice Returns true if the provide address is permitted the preferred
  ///         arbitrage rate in the partner swap method of a Cron-Fi TWAMM pool.
  ///         Returns false otherwise.
  /// @param _address the address to check for arbitrage rate permissions.
  ///
  function isArbitrageur(address _address) 
           public
           view
           override(IArbitrageurList)
           returns (bool) {
    return permittedAddressMap[_address];
  }
}
```
