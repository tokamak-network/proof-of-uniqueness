# Tokamak Sybil Resistence
## Overview
The goal is to create a register for Ethereuem addresses, where each address on the register is assigned a ‘score’ in a sybil-resistent way. This means that although someone can create an unlimited number of Ethereum addresses and add them to the register, the total ‘score’ of the addresses will still be limited. These scores can then be used as a measure of uniqueness. Applications that require protection against sybil attacks can use this register. 

The project uses a mechanism similar to a web-of-trust. Adding an address to the register requires a deposit (which can be retrieved by removing address from the register). Then any two registered address can vouch for each other by both ‘staking’ some of their deposit on each other. This gives the data of a weighted graph, where the vertices are the addresses and the edges are the pairs of addressses that have vouched for each other (the weight of an edge is the amount that was staked by the pair on each other).

From this weighted graph, a score is calculated for each address. The score is a measure of how well connected a particular address is and is robust against manipulation and collusion. The algorithm to compute the score is computationally expensive so it is not possible to run the scoring algorithm in the L1 contract. Instead, the contract requires the updated scores to be submitted to it along with a proof that the score updates are correct. The contract will verify this proof and then update the state. To incentivize people to compute the scores for the contract, it will award a small amount of TON for a correct proof. 

Applications can integrate with this project to obtain sybil-resistence, for example for an airdrop or for a voting application. Users can also use this register in a privacy preserving way by creating a proof that they own an address in the register with a score above a certain threshold.

## Scoring algorithm
The way the scoring algorithm works is by starting with a set (denoted by a calligraphic A) of subsets of the nodes (in practice, these will be the subsets that are smaller than a certain threshold). Then to compute the score for a node, we look at each of these subsets that contains the node and compute the sum of the weights of the links that are leaving this subset and divide it by the number of elements in the subset. The score is then defined to be minimum of these values. More precisely, the scoring function **_f_** is defined by:![eq1](https://github.com/tokamak-network/proof-of-uniqueness/blob/main/img1.png?raw=true) 

where 

![graph](https://github.com/tokamak-network/proof-of-uniqueness/blob/main/img3.png?raw=true)

and

![graph](https://github.com/tokamak-network/proof-of-uniqueness/blob/main/img2.png?raw=true)


### Example:

![graph](https://github.com/tokamak-network/proof-of-uniqueness/blob/main/graph-example.png?raw=true)


In this example, let us suppose the set of subsets we are using is all subsets with three or less nodes. Then, to compute the score for node 5, we look at all subsets with three or less elements, that contain node 5:

```{5},{0,5},{1,5},{2,5},{3,5},{4,5},{5,6},{0,1,5},{0,2,5},{0,3,5},{0,4,5},{0,6,5},{1,2,5},{1,3,5},{1,4,5},{1,5,6},{2,3,5},{2,4,5},{2,5,6},{3,4,5},{3,5,6},{4,5,6}```

Then for each of these subsets, we compute the sum of the weights of the links leaving the subset. For example, for the subset ```{1,3,5}``` the sum of the links leaving it is ```7+1+1+1+2=12```. Then we divide this by the number of elements in the subset we get ```12/3=4```. If we do this for each subset we get a list of values, and then if we take the minimum of this list of values we get the score for node 5.

### Scoring circuit
This algorithm is implemented in the file [scoring_algorithm.circom](https://github.com/tokamak-network/proof-of-uniqueness/blob/main/scoring_algorithm.circom). It is implemented as a circom circuit so that the scores can be computed off chain and the proof sent to convince the L1 contract that the scores have been computed correctly.


To test this code first install circom and snarkjs: https://docs.circom.io/getting-started/installation/.

The first step is to run ```circom scoring_circuit.circom --r1cs --wasm --sym --c```. This compiles the circuit and generates the constraints for the circuit. It also generates a directory ```scoring_circuit_js``` that contains the Wasm code and other files needed to generate the witness. We then need to provide the data of a weighted graph for the circuit to run on. The data for the example given above is available in the file ```test_input1.json```, save this file in the ```scoring_circuit_js``` directory. (The data of the set of subsets is encoded as a ```0/1``` matrix where the subsets are given by the columns).

Then next step is to calculate values for all wires from the input wires: ```node generate_witness.js scoring_circuit.wasm test_input1.json witness.wtns```

To create a proof, we need to use a trusted setup. This is done using the powers of tau ceremony which can be run using ```snarkjs``` as follows (see https://docs.circom.io/getting-started/proving-circuits/):

```snarkjs powersoftau new bn128 15 pot15_0000.ptau -v```

```snarkjs powersoftau contribute pot15_0000.ptau pot15_0001.ptau --name="First contribution" -v```

```snarkjs powersoftau prepare phase2 pot15_0001.ptau pot15_final.ptau -v``` (This step in the setup ceremony takes about 10 mins).

```snarkjs plonk setup scoring_circuit.r1cs pot15_final.ptau scoring_circuit_final.zkey```

Next we export the verification key for our circuit: ```snarkjs zkey export verificationkey scoring_circuit_final.zkey verification_key.json```

And now finally we create a Plonk proof for the witness: ```snarkjs plonk prove scoring_circuit_final.zkey witness.wtns proof.json public.json```  

This step creates a file ```public.json``` containing the values for the scores for the nodes, and also ```proof.json``` proving that these scores have been calculated correctly.

To verify the proof run the command: ```snarkjs plonk verify verification_key.json public.json proof.json```  

## Contract design

The contract will keep a record of the id, owner, deposit and score for each node, as well as how much each node has staked on each other node. This data will be organized into a Merkle tree with a subtree for each leaf. For this tree will use the MiMC hash function.


![graph](https://github.com/tokamak-network/proof-of-uniqueness/blob/main/img4.jpg?raw=true)

Each leaf in the main accounts tree stores the ETH address associated with the account, how much TON that account has deposited into the rollup, the “uniqueness” score for that account, and the root hash of a subtree of link data. The link subtree stores the amount that has been staked by that account on a particular other account in the system, (using the account’s id in the account tree). For example if the first account has staked 0.5 TON on the account with id 3, then the third leaf of the subtree for the first leaf of the account tree will store the value 0.5. 

The amounts staked on a particular account are used in the scoring algorithm to compute it’s uniqueness score. To prevent this from being manipulatable, we need to ensure that the total amount that an account can stake on other accounts is limited by how much TON they have deposited into the rollup. Hence we make the rule that a valid instance of the state tree must satisfy

![graph](https://github.com/tokamak-network/proof-of-uniqueness/blob/main/img5.png?raw=true)

for each leaf of the account tree.

The contract will accept transactions that update this tree, and store them in a queue. Then anyone can create a batch of these transactions, compute the update of the state tree and provide a proof of correctness. At any time, there is only one possibility for the next block, namely the next N txns in the queue (if the size of the queue is less than N, then the contract wont accept a proposed block). This means we dont need any leader selection process, its just the first valid proof from any user which creates the next block. 

## Ownership proofs
Given the current state root, a user can make a proof of ownership of a leaf with score above a certain threshold. A different proof circuit is needed depending on the threshold. Also, any contract that wishes to intergrate with this system and check ownership proofs would need its own verifier contract and also a mapping to record nullifiers to ensure each user can only prove once.
