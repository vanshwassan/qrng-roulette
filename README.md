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

[Remix 1](/src/1.png)

