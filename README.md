# Coding an on-chain roulette game using QRNG

This example project demonstrates how to code a Solidity roulette game that uses API3's QRNG for true randomness. You will use Remix IDE to code and deploy the contract. [Click here to check out the project's Github repo with a proper working frontend]().

[Click here to try out the Roulette]()

Before starting, make sure you have a proper understanding of [Airnode]() and [how it works.]()

[Read more about QRNG and how it works.]()

## Introduction

In a game of roulette, players place their bets on a table with numbers and betting options. The table corresponds to a spinning wheel with numbered pockets, which is spun by the dealer. Once the ball comes to a stop in one of the pockets, the dealer announces the winning number and pays out any winning bets.

Players can bet on a variety of options, including specific numbers or groups of numbers, such as whether the ball will land on an odd or even number or on a red or black pocket.

The roulette that we're going to code will have the following betting options for the users:

- The user can select either the first, second or the third dozen of the numbers on the board.
- The user can select either the first or the second half of the numbers on the board.
- The user can select either the set of all even or odd numbers on the board
- The user can select all the red or black numbers on the board.
- The user can choose any one number he wishes on the board.

If the number after the spin lands on one of the selected numbers, the user wins the bet.

1. Coding the `Roulette` Contract

::: tip Check your Network

Make sure you're on a Testnet before trying to deploy the contracts on-chain!

:::

> The complete contract code can be found [here]()

Head on to [Remix online IDE]() using a browser that you have added Metamask support to. Not all browsers support [MetaMask➚]().

It should load up the Roulette contract.

[Open in Remix➚]()

> ![Remix 1](/src/SS1.png)

The Roulette contract is going to be the main Requester contract that makes request to the QRNG Airnode using the [Request-Response Protocol (RRP)]().

You first start by importing the `RrpRequesterV0`, which is the [Request-Response Protocol (RRP)](). You can then start coding the `Roulette` contract by inheriting  from `RrpRequesterV0`

You then define the following state variables:
- `MIN_BET`: The minimum amount that is required to bet for a spin.
- `spinCount`: An unsigned integer to keep track of the number of times the roulette wheel has been spun.
- `airnode`: The Airnode address of the QRNG Provider.
- `deployer`: An immutable address variable to store the address of the contract deployer.
- `sponsorWallet`: A `payable` address variable to store the `sponsorWallet`, that needs to be funded later to cover the gas costs for the QRNG request fulfillment.
- `endpointId`: A `bytes32` variable to store the unique identifier for the QRNG API endpoint.

The contract also defines an enumeration type called "BetType" with five possible values:

    Color
    Number
    EvenOdd
    Third
    Half

These values represent different types of bets that players can make in the game of Roulette.

```solidity
pragma solidity >=0.8.4;

import "@api3/airnode-protocol/contracts/rrp/requesters/RrpRequesterV0.sol";

contract Roulette is RrpRequesterV0 {
  uint256 public constant MIN_BET = 10000000000000; // .001 ETH
  uint256 spinCount;
  address airnode;
  address immutable deployer;
  address payable sponsorWallet;
  bytes32 endpointId;

  // ~~~~~~~ ENUMS ~~~~~~~

  enum BetType {
    Color,
    Number,
    EvenOdd,
    Third,
	 Half
  }
```

The contract defines several mapping variables to store information about user bets and the results of each spin in the game of Roulette:

```solidity
  mapping(address => bool) public userBetAColor;
  mapping(address => bool) public userBetANumber;
  mapping(address => bool) public userBetEvenOdd;
  mapping(address => bool) public userBetThird;
  mapping(address => bool) public userBetHalf;
  mapping(address => bool) public userToColor;
  mapping(address => bool) public userToEven;

  mapping(address => uint256) public userToCurrentBet;
  mapping(address => uint256) public userToSpinCount;
  mapping(address => uint256) public userToNumber;
  mapping(address => uint256) public userToThird;
  mapping(address => uint256) public userToHalf;

  mapping(bytes32 => bool) expectingRequestWithIdToBeFulfilled;

  mapping(bytes32 => uint256) public requestIdToSpinCount;
  mapping(bytes32 => uint256) public requestIdToResult;

  mapping(uint256 => bool) blackNumber;
  mapping(uint256 => bool) public blackSpin;
  mapping(uint256 => bool) public spinIsComplete;

  mapping(uint256 => BetType) public spinToBetType;
  mapping(uint256 => address) public spinToUser;
  mapping(uint256 => uint256) public spinResult;
```

The contract also defines several error messages and events:

```solidity
  error HouseBalanceTooLow();
  error NoBet();
  error ReturnFailed();
  error SpinNotComplete();
  error TransferToDeployerWalletFailed();
  error TransferToSponsorWalletFailed();

  // ~~~~~~~ EVENTS ~~~~~~~

  event RequestedUint256(bytes32 requestId);
  event ReceivedUint256(bytes32 indexed requestId, uint256 response);
  event SpinComplete(bytes32 indexed requestId, uint256 indexed spinNumber, uint256 qrngResult);
  event WinningNumber(uint256 indexed spinNumber, uint256 winningNumber);
```

The constructor function will take the `_airnodeRrpAddress` during deployment of the contract.
You also need to set the `deployer` variable to the address of the user who deployed the contract `(msg.sender)`.

It also sets certain numbers as black by setting their corresponding values in the `blackNumber` mapping to `true`. These numbers are 2, 4, 6, 8, 10, 11, 13, 15, 17, 20, 22, 24, 26, 28, 29, 31, 33, and 35. These are the numbers on a roulette wheel that are colored black.

```solidity
  constructor(address _airnodeRrp) RrpRequesterV0(_airnodeRrp) {
    deployer = msg.sender;
    blackNumber[2] = true;
    blackNumber[4] = true;
    blackNumber[6] = true;
    blackNumber[8] = true;
    blackNumber[10] = true;
    blackNumber[11] = true;
    blackNumber[13] = true;
    blackNumber[15] = true;
    blackNumber[17] = true;
    blackNumber[20] = true;
    blackNumber[22] = true;
    blackNumber[24] = true;
    blackNumber[26] = true;
    blackNumber[28] = true;
    blackNumber[29] = true;
    blackNumber[31] = true;
    blackNumber[33] = true;
    blackNumber[35] = true;
  }
```

The `_spinRouletteWheel()` is an internal function that makes a request for a random number to use as the result of a roulette spin. It calls the `airnodeRrp.makeFullRequest()` function of the `AirnodeRrpV0.sol` protocol contract which adds the request to its storage and emits a `requestId`. It takes a `_spinCount` parameter that represents the unique identifier for the spin.

The function sets the `expectingRequestWithIdToBeFulfilled` mapping with the `requestId` key to true. This is used to track whether the request has been fulfilled.

It sets the `requestIdToSpinCount` mapping with the `requestId` key to the `_spinCount` parameter. This is used to map the request ID to the specific spin count.

It emits a `RequestedUint256` event with the `requestId` parameter to indicate that a request has been made for a random number.

The off-chain QRNG Airnode gathers the request and performs a callback to the contract with the random number. The `fulfillUint256()` is a callback function that is called by `AirnodeRrp` when a response is received.

`_qrngUint256` stores the random number which is further passed to the `_spinComplete()` with the `requestId`

Finally, the function emits a `ReceivedUint256` event with the received `requestId` and the decoded `_qrngUint256`.

```solidity
  function _spinRouletteWheel(uint256 _spinCount) internal {
    require(!spinIsComplete[_spinCount], "spin already complete");
    require(_spinCount == userToSpinCount[msg.sender], "!= msg.sender spinCount");
    bytes32 requestId = airnodeRrp.makeFullRequest(
      airnode,
      endpointId,
      address(this),
      sponsorWallet,
      address(this),
      this.fulfillUint256.selector,
      ""
    );
    expectingRequestWithIdToBeFulfilled[requestId] = true;
    requestIdToSpinCount[requestId] = _spinCount;
    emit RequestedUint256(requestId);
  }

    /** @dev AirnodeRrp will call back with a response
   *** if no response returned (0) user will have bet returned (see check functions) */
  function fulfillUint256(bytes32 requestId, bytes calldata data) external onlyAirnodeRrp {
    require(expectingRequestWithIdToBeFulfilled[requestId], "Unexpected Request ID");
    expectingRequestWithIdToBeFulfilled[requestId] = false;
    uint256 _qrngUint256 = abi.decode(data, (uint256));
    requestIdToResult[requestId] = _qrngUint256;
    _spinComplete(requestId, _qrngUint256);
    emit ReceivedUint256(requestId, _qrngUint256);
  }
```

