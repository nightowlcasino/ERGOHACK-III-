# ERGOHACK-III-

Hi

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
