# Night Owl Contract Design

This document provides detailed description of the Ergoscript used to support the Night Owl Platform. Please note there are numerous references to proxy contracts which are not included in this document. 


# Requirements

The Night Owl Platform requires the following functionality:

 - A swapping mechanism for users to swap between sigUSD and OWL tokens (to be extended to any stable-token)
 - A liquidity pool for users to provide capital for the house.
 - For custom casino games to be created that can utilise this stored capital.

## Swapping Mechanism

The swapping mechanism described in the requirements can be implemented in Ergoscript. 
We propose the following design:

At launch, Night Owl casino mints a sufficiently large quantity of OWL tokens and stores them in a UTXO box. Users can interact with this box by providing sigUSD and obtaining released OWL tokens at a 1:1 ratio. The box should have a guard script that enforces the following behaviour:

 - The box is used as an input for a transaction alongside some other inputs containing x sigUSD or OWL. 
 - A new box will be created with the same guard script as our input box that contains:
	 - Initial amount of OWLs +- x
	 - Initial amount of sigUSD -+ x

The ErgoScript which achieves this is below:
```scala
{
val contractTokensOwl = SELF.tokens(0)._2
val contractTokensSigUSD = SELF.tokens(1)._2
val owlId = fromBase58("CqK3dmwgkK83qVnHrc8YLpm46t5aDLWNViwrhmtLqPeh")
val sigUsdId = fromBase58("B9XWSU56ob1S5JPrEHPE5PcWg2HE73AsFBPAYLXXWc7v")
val tokenAmount = OUTPUTS(0).R4[Long].get
val tokenType = OUTPUTS(0).R5[Coll[Byte]].get

// Swap SigUSD for OWL
val swapContract = if (tokenType == owlId) {        
allOf(Coll(
OUTPUTS(0).tokens(0)._1 == owlId,
OUTPUTS(0).tokens(0)._2 == tokenAmount,
OUTPUTS(1).tokens(0)._1 == owlId,
OUTPUTS(1).tokens(0)._2 == contractTokensOwl - tokenAmount,
OUTPUTS(1).tokens(1)._1 == sigUsdId,
OUTPUTS(1).tokens(1)._2 == contractTokensSigUSD + tokenAmount,
OUTPUTS(1).propositionBytes == SELF.propositionBytes))
} else {
// Swap OWL for SigUSD
allOf(Coll(
tokenType == sigUsdId,                      
OUTPUTS(0).tokens(0)._1 == sigUsdId,
OUTPUTS(0).tokens(0)._2 == tokenAmount,
OUTPUTS(1).tokens(0)._1 == owlId,
OUTPUTS(1).tokens(0)._2 == contractTokensOwl + tokenAmount,
OUTPUTS(1).tokens(1)._1 == sigUsdId,
OUTPUTS(1).tokens(1)._2 == contractTokensSigUSD - tokenAmount,
OUTPUTS(1).propositionBytes == SELF.propositionBytes))
}

sigmaProp(swapContract)
}
```

 *Remarks*:

 - The user must inform the script of their 'x' in R4, which represents the number of tokens they are offering for the swap. The user must also inform the script of the token type they are offering in R5. 
 - Notice that if the user attempts to deceive the contract by supplying a false R4 or R5 the INPUTS and OUTPUTS token quantity will not be equal and thus the transaction will be illegal.
 - The arbitarily large amount of minted OWLs must be so large that it is impossible for the contract to ever run out of OWLs. 
 - There is only one UTXO containing all the OWLs and held sigUSD.
 - OUTPUTS(0).propositionBytes is not defined and thus can be set as an ErgoMixer address to allow users to gamble with tokens that have been run through the mixer for additional privacy.



## Liquidity Pool
The liquidity pool acts as the casino's source of capital or bank balance, if you like. Any user with OWLs can provide these tokens as liquidity. All winning bets in the casino are paid from the liquidity pool and a proportion of all losing bets and service fees (for player vs player games) are collected by the liquidity pool. The casino games are designed to produce capital for the pool over time, liquidity providers can capture this produced capital when they redeem their liquidity position. 

Thus, the liquidity pool requires the following functionality:

 - Users can provide and redeem liquidity
 - Redemption of liquidity should give the user their proportion of the tokens in the pool
 - The pool's funds can be used to fund casino games
 
We will leave discussion of this last point for the custom games section.
We propose the following design:

A sufficiently large quantity of LP tokens are minted at the launch of Night Owl. These tokens are stored in a box guarded by the liquidity pool contract. These LP tokens have no function or utility other than to act as a measure of a user's share in the liquidity pool. When a user provides OWL tokens to the pool they receive LP tokens in a 1:1 ratio. When a user redeems their LP tokens they receive:

Users LP Tokens/Total LP Tokens sold * Total Owl Tokens in LP

The contract describing this is as follow:

```scala
{ // LIQUIDITY CONTRACT
val contractTokensOwl = SELF.tokens(0)._2
val contractTokensLP = SELF.tokens(1)._2
val owlId = fromBase58("CqK3dmwgkK83qVnHrc8YLpm46t5aDLWNViwrhmtLqPeh")
val lpId = fromBase58("B9XWSU56ob1S5JPrEHPE5PcWg2HE73AsFBPAYLXXWc7v")
val maxLpValue = 10000000000000000L // some sufficiently large value
val tokenAmount = OUTPUTS(0).R4[Long].get
val tokenType = OUTPUTS(0).R5[Coll[Byte]].get

// OWL for LP
val liquditySwap = if (tokenType == owlId) { 
allOf(Coll(
OUTPUTS(0).tokens(0)._1 == lpId,
OUTPUTS(0).tokens(0)._2 == tokenAmount,
OUTPUTS(1).tokens(0)._1 == owlId,
OUTPUTS(1).tokens(0)._2 == contractTokensOwl + tokenAmount,
OUTPUTS(1).tokens(1)._1 == lpId,
OUTPUTS(1).tokens(1)._2 == contractTokensLP - tokenAmount,
OUTPUTS(1).propositionBytes == SELF.propositionBytes))
} else {
// LP for Owl
allOf(Coll(
tokenType == lpId,
OUTPUTS(0).tokens(0)._1 == owlId,
OUTPUTS(0).tokens(0)._2 == tokenAmount /(maxLpValue - contractTokensLP) * contractTokensOwl,
OUTPUTS(1).tokens(0)._1 == owlId,
OUTPUTS(1).tokens(0)._2 == contractTokensOwl - (tokenAmount /(maxLpValue - contractTokensLP) * contractTokensOwl),
OUTPUTS(1).tokens(1)._1 == lpId,
OUTPUTS(1).tokens(1)._2 == contractTokensLP + tokenAmount,
OUTPUTS(1).propositionBytes == SELF.propositionBytes))
}
sigmaProp(liquditySwap)
}
```

 *Remarks*:

 - Similar to the swap contract, the user proposes the token amount to be swapped in R4 and the token type in R5 and once again cannot cheat the contract due to conservation of tokens
 - The sufficiently large amount of minted LPs is used to reference the number of circulating LPs and thus give an accurate pool share proportion
 - There will be only one UTXO that contains LP tokens. 
 - Division by 0 is impossible since a LP:OWL means there is always some circulating LP. 

## Custom Casino Games
Night Owl allows for any game to be created and launched on the casino through community voting.  To Design the liquidity pool contract (hereafter referred to as the house contract) to be compatible with custom games we present the following design:

The House Contract is used to match player bets on the casino. The spending of this contract needs to be controlled. 
To allow any game to be developed on the platform the house contract's spending condition must be **customisable**.
We have achieved this through the use of Game tokens. Every Game created on the platform will use the same game token. If a game token is present in a transaction, the house contracts funds can be spent. A game token is stored in a custom game contract that limits the spending of house contract funds as well as describing all the rules of the game. The creation of the contracts can be voted on by the community. 

To the existing LP spending paths, the House Contracts will have two additional spending paths.
- One that allows the House Contract to be spent in a Casino game to match a player's bet. 
This spending path checks if the transaction contains a box with the Game NFT.
(see casinoBet)
- A second spending path allows payout winnings of the House Contract to be collected to a single box. (see collector)

Full House Contract:

```scala
{ // FULL HOUSE CONTRACT
// LIQUIDITY CONTRACT
val contractTokensOwl = SELF.tokens(0)._2
val contractTokensLP = SELF.tokens(1)._2
val owlId = fromBase58("CqK3dmwgkK83qVnHrc8YLpm46t5aDLWNViwrhmtLqPeh")
val lpId = fromBase58("B9XWSU56ob1S5JPrEHPE5PcWg2HE73AsFBPAYLXXWc7v")
val maxLpValue = 1000000L
val tokenAmount = OUTPUTS(0).R4[Long].get
val tokenType = OUTPUTS(0).R5[Coll[Byte]].get

// OWL for LP
val dualSwap = if (tokenType == owlId) { 
allOf(Coll(
OUTPUTS(0).tokens(0)._1 == lpId,
OUTPUTS(0).tokens(0)._2 == tokenAmount,
OUTPUTS(1).tokens(0)._1 == owlId,
OUTPUTS(1).tokens(0)._2 == contractTokensOwl + tokenAmount,
OUTPUTS(1).tokens(1)._1 == lpId,
OUTPUTS(1).tokens(1)._2 == contractTokensLP - tokenAmount,
OUTPUTS(1).propositionBytes == SELF.propositionBytes))
} else {
// swap LP for Owl
allOf(Coll(
tokenType == lpId,
OUTPUTS(0).tokens(0)._1 == owlId,
OUTPUTS(0).tokens(0)._2 == tokenAmount /(maxLpValue - contractTokensLP) * contractTokensOwl,
OUTPUTS(1).tokens(0)._1 == owlId,
OUTPUTS(1).tokens(0)._2 == contractTokensOwl - (tokenAmount /(maxLpValue - contractTokensLP) * contractTokensOwl),
OUTPUTS(1).tokens(1)._1 == lpId,
OUTPUTS(1).tokens(1)._2 == contractTokensLP + tokenAmount,
OUTPUTS(1).propositionBytes == SELF.propositionBytes))
}

// CASINO MATCH BET
val gameNFT = fromBase58("CqK3dmwgkK83qVnHrc8YLpm46t5aDLWNViwrhmtLqPeh")
val casinoBet = INPUTS(0).tokens(0)._1 == gameNFT

val collectorChecks = allOf(Coll(
OUTPUTS.size == 2,
OUTPUTS(0).propositionBytes == SELF.propositionBytes))
val minerGetOwls = OUTPUTS(1).tokens.exists({(t: (Coll[Byte], Long)) => t._1 == owlId})
val minerGetLp = OUTPUTS(1).tokens.exists({(t: (Coll[Byte], Long)) => t._1 == lpId})

val collector = collectorChecks && !(minerGetOwls || minerGetLp)

sigmaProp(dualSwap || casinoBet || collector)
} 
```

 *Remarks*:
- We would suggest to build custom game contracts in at least two transactions so that the House contract funds can be used immediately rather than wait for game result. 
	- One transaction that extracts the matching bet amount from the House Contract and returns the remaining tokens back to a single house contract box.
	- A second transaction to determine the outcome of the game.
- Games will produce many boxes guarded by the house contract with minor amounts of OWL tokens. These boxes can be seen as shards of the larger main box. In order for a liqudity provider to receive the full rewards from all OWLs under the house contract guard box, collection should be performed before liqudity redemption. However, collection could be costly in terms of mining fees and computation and hence it is suggested liqudity redemption be issued through a periodic proxy contract that is called after a periodic collection (for example every 5 minutes)
 
 **Example Game Design: Simplified Roulette**
 
 A simplified Roulette game is designed to show how the platform's game token can be implemented. As mentioned, the game will have two transactions. 
 - Matching player's bet transaction
 - Game Result transaction

In this Roulette game a player can only bet on the colors: red, black or green. The payout will be higher if a player wins with a green bet.

The following contract is a box which stores a singular game token. It defines the amount that is needed from the House Contract to match the player's bet as well as payout paths to developers, marketing or bounty contracts (for simplicity this example does not consider these paths). 
Besides the House Contract and this game token box, an additional player proxy contract is present as an input, which stores the player's bet in Owls, payment address and guess in the Roulette game (proxy contract used to standardise INPUTS size)

One of the outputs of this transaction is a Roulette Result box (guarded by a roulette result contract) with the game info and the player's and House' bets is created. The Roulette Result Contract will determine the outcome of the game.
The House Contract and the Roulette Game token box are also re-created by the transaction. 

 Roulette Game NFT guard box design:
 ```scala 
 // ROULETTE GAME NFT GUARD BOX
{
val gameNFT = fromBase58("Avp3dmwgkK83qVnHrc8YLpm46t5aDLWNViwrhmtLqPe2") // Token recognised by house contract
val lpId = fromBase58("B9XWSU56ob1S5JPrEHPE5PcWg2HE73AsFBPAYLXXWc7v")
val gameContract = fromBase64("abcdef") // Contract that describes the outcome of the game
val miningAddress = fromBase64("abcdef") // Enter Mining Ergotree here
val owlId = fromBase58("CqK3dmwgkK83qVnHrc8YLpm46t5aDLWNViwrhmtLqPeh")
val houseContract = fromBase64("abcd") // House contract ErgoTree
val betMultiplier = if(INPUTS(2).R4[Long].get == 2) {35} else {1} 

allOf(Coll(
INPUTS.size == 3,
INPUTS(0).propositionBytes == SELF.propositionBytes, 
INPUTS(0).tokens(0)._1 == gameNFT,
INPUTS(0).tokens(0)._2 == 1,
INPUTS(1).tokens(0)._1 == owlId,
INPUTS(1).tokens(1)._1 == lpId,
INPUTS(1).propositionBytes == houseContract, // House Matching Bet
INPUTS(2).propositionBytes != houseContract, // User Bet
INPUTS(2).tokens(0)._1 == owlId,
OUTPUTS.size == 4,
OUTPUTS(0).propositionBytes == gameContract, 
OUTPUTS(0).tokens(0)._1 == owlId,
OUTPUTS(0).tokens(0)._2 == (betMultiplier + 1) * INPUTS(2).tokens(0)._2, // Stores all the Owl tokens
OUTPUTS(0).R4[Long] == INPUTS(2).R4[Long], // Store User Guess (0 for red, 1 for black, 2 for green)
OUTPUTS(0).R5[Coll[Byte]] == INPUTS(2).R5[Coll[Byte]], // Store User payout address
OUTPUTS(1).propositionBytes == houseContract,
OUTPUTS(1).tokens(0)._1 == owlId,
OUTPUTS(1).tokens(1)._1 == lpId,
OUTPUTS(1).tokens(0)._2 == INPUTS(1).tokens(0)._2 - betMultiplier * INPUTS(2).tokens(0)._2,
OUTPUTS(1).tokens(1)._2 == INPUTS(1).tokens(1)._2,
OUTPUTS(2).tokens(0)._1 == gameNFT, // Recycle Token
OUTPUTS(2).tokens(0)._2 == 1,
OUTPUTS(2).propositionBytes == SELF.propositionBytes,
OUTPUTS(3).propositionBytes == miningAddress))
} 
``` 
 
 *Remarks*:
 - Game token is recyled immediately for reuse
 
 
 **The Roulette Result Contract Box** 
 
 The Roulette Result Contract Box containing the game info and both bets will be sent to a payout box containing the payout for the winner of the game.
 It generates a random number and checks with the modulo operator if the player's guess results in a win.
 If it does the payout box assigned to the player's payment address, if not it will be assigned to the House Contract address.
 
 Spending Result Box design:
 ```scala 
 { // ROULETTE RESULT CONTRACT
val index = HEIGHT - SELF.creationInfo._1 - 2
val randomNumber = byteArrayToLong(CONTEXT.headers(index).id)

val miningAddress = fromBase64("abcdef") // Enter Mining Ergotree here
val userGuess = SELF.R4[Long].get
val userPaymentAddress = SELF.R5[Coll[Byte]].get
val houseContract = fromBase64("abcd")
val userWinDouble = randomNumber % 2 == userGuess && randomNumber % 37 != 0
val userWinGreen = userGuess == 2 && randomNumber % 37 == 0
val tokensValid = allOf(Coll(
OUTPUTS.size == 2,
OUTPUTS(0).tokens(0)._1 == fromBase58("CqK3dmwgkK83qVnHrc8YLpm46t5aDLWNViwrhmtLqPeh"),
OUTPUTS(0).tokens(0)._2 == SELF.tokens(0)._2,
OUTPUTS(1).propositionBytes == miningAddress))

val paymentProp = if(userWinDouble || userWinGreen) {
OUTPUTS(0).propositionBytes == userPaymentAddress
} else {
OUTPUTS(0).propositionBytes == houseContract}
sigmaProp(tokensValid && paymentProp)
}
```
 
 
 *Remarks*:
 - The random number generation is used as for demonstration purposes only. More sophisticated RNG methods are required for more dynamic games. 
 
  **Final Comments**
  
It is important to notice that the house contract can only be spent if a game token is present as an INPUT. In the case of our roulette example this game token was guarded by a script that imposed spending limitations on the house contract, if these limitations were not present the house contract could be drained. So what about other game tokens? How are they guarded? At launch Night Owl will mint a sufficiently large number of game tokens and store them in a box which has a guard script that employs some community voting mechanism for its spending (yet to be defined). The game tokens will only be sent to boxes guarded by a script which imposes strict limiations on the spending of house funds that allign with the game being delivered. In this way, the house contract can have custom spending paths based on the design of these game nft guard boxes!
