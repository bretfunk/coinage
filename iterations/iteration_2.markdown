# Iteration 2 - Working with the Block Chain

Now that we have a way to generate blocks containing (so far)
fresh coinbase transactions, and mine those blocks to produce
a valid nonce and satisfy the network's Proof-of-Work requirements,
let's look at some of the basic questions we will want to ask of the
information contained in the block chain.

## Checking Balances

An obvious first thing you'd like to know is "how much money do I have
available?" To answer this, we need to remember 2 key things:

1. *A Person* only has money insofar as they hold the private key for
a keypair which has transactions assigned to it in the block chain. Technically
all value in the network is assigned to public keys through transactions, so when
we want to look at a "balance", we'll be looking for all the value assigned to
a given public key.
2. Value on the network is only ever transferred in the form of **Transaction Outputs**.
Determining the amount of money available to a given key is really a question of
determining the value of all Outputs that are assigned to it.

### Balance Algorithm

With these ideas in mind, we can determine the balance available to a given public
key using the following steps:

1. Search the block chain for all Transaction Outputs assigned to the given key
2. Of these outputs, pull out only those that are still *unspent*
3. Add up the amounts of all of these remaining unspent outputs

Remember that Transaction Outputs are simple data structures with this format:

```json
{
  "amount": 5,
  "address": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxpaKTGz1LlgVihe0dGlE\nPsn\/cJk+Zo7uePr8hhjCAj+R0cxjE4Q8xKmVAA3YAxenoo6DShn8CSvR8AvNDgMm\nAdHvKjnZXsyPBBD+BNw5vIrEgQiuuBl7e0P8BfctGq2HHlBJ5i+1zitbmFe\/Mnyr\nVRimxM7q7YGGOtqQ5ZEZRL1NcvS2sR+YxTL5YbCBXUW3FzLUjkmtSEH1bwWADCWj\nhz6IXWqYU0F5pRECVI+ybkdmirTbpZtQPyrND+iclsjnUUSONDLYm27dQnDvtiFc\nIn3PZ3Qxlk9JZ6F77+7OSEJMH3sB6\/JcPZ0xd426U84SyYXLhggrBJMXCwUnzLN6\nuwIDAQAB\n-----END PUBLIC KEY-----\n"
}
```

So we can determine if an output is assigned to a key by comparing the `"address"` field
of the output with the key's PEM representation. And we can simply read the value
of the output from its `"amount"` field.

### Identifying Unspent Transaction Outputs

So how do we know which of the outputs assigned to a key can still
be considered "unspent"?

Outputs of a transaction are spent by inputs to other, subsequent
transactions. When spending an output, an input identifies it using
the hash of the transaction to which it belongs and its numeric index
within the list of outputs contained in that transaction.

Thus when checking the status of an output, we'll need to have these
identifying "coordinates" -- transaction hash and index. Once we have these,
we can again search the block chain to check for any transaction
inputs which claim our output's coordinates as their `source_hash` and `source_index`.

If we find a match, we know that the output in question has been spent.
Otherwise, it is unspent. When checking the balance of a key, you will need
to use this process against each output to determine whether it has been
spent or not.

## Transferring Funds

Now that we can check the balance for a key, the next thing we might like
to do is transfer some of these funds to another address (i.e. make a payment).

We have all the pieces in place required to make this happen, but the process
is somewhat involved, so let's step through it.

### Payment Outline

In broad strokes, the process of setting up a payment transaction looks
like this:

1. Determine the amount to send as well as any transaction fees to include
as incentive to miners
2. Find all Unspent Transaction Outputs assigned to your wallet (key), and select
enough of these to cover *at least* the payment amount + transaction fee (it's ok
if you end up going over, since any excess can be returned to you as change)
3. Generate 1 Transaction Input for each of these *sources*. Remember
that inputs identify the transaction output they spend by providing the source transaction
hash and output index
4. Generate the payment output, which assigns the desired amount to the address
you want to pay
5. If the total value of inputs being included in the transaction is greater than the
sum of your payment amount plus the transaction fee, add an additional *change* output
that assigns the remaining value back to your own public key.
6. Add the transaction hash
7. Sign the transaction inputs

### Required Information

In order to generate a payment transaction, we'll need to have access to
a few pieces of data:

* The Block Chain -- We need to be able to gather source outputs from
the chain in order to use them as inputs to our payment transaction
* Paying Keypair -- We need the private key in order to sign our inputs,
proving our ability to spend them. Additionally we may need the public key
to use as a *change address* if we end up sourcing more outputs than are needed
* Receiving Address -- This will be the `address` for the actual payment output in our
transaction
* Amount -- We need to know how much to pay!

### Transaction Fees

We haven't talked in depth about transaction fees up to now. In short,
transaction fees are provided as an incentive for miners to include
our transaction as part of a block they are mining -- sort of a "tip"
that the miner receives as a small bonus for mining a block that contains
your transaction. Whenever a miner includes a transaction in a block, they are gain any fees attached
to that transaction.

So how do fees work at a technical level? Interestingly, the fees
are not explicitly stated. Rather, any difference between the combined
value of all inputs and the combined value of all outputs in a transaction
is assumed to be a transaction fee.

This partly explains why accounting for change in our transactions is so
important -- any "leftovers" that we don't account for are assumed to
be a transaction fee and will be captured by the miner.

So the "input value" of transaction fees is implied, but surely the
output value must appear somewhere. It does -- in fact, transaction
fees are simply incorporated into the block's coinbase transaction.

Previously we described the coinbase as a special transaction containing
0 inputs and 1 output whose value equals the current block reward amount.
It turns out there's an additional component -- the coinbase output will
also include the sum of all transaction fees included within the transaction.
If the current miner reward is set at `25`, it's likely that most coinbase
transactions will have a value slightly higher than `25`, since they will
often incorporate one or more transaction fees.

__Why have implicit transaction fees?__

This system for assigning fees may seem a little strange at first, but
if we consider the distributed design of the system we can see that it's actually
quite necessary. The problem is that at the time we generate a transaction,
we don't necessarily know which miner will end up finding the block
which adds it to the block chain. Thus it's not possible at that time to generate
a valid transaction output assigning the appropriate fee to the miner's address.

Using implicit transaction fees gives us a flexible system that allows fees
to be specified at the time the transaction is generated but captured later
when the transaction is actually added to a block.

## Payment Walkthrough

Let's look at a more detailed example by walking through this process with a hypothetical
block chain. To demonstrate, we'll imagine we had a block chain represented by this JSON
structure:

```json
[{
  "transactions": [
    {
      "hash": "a98f3d",
      "timestamp": 1450584386520,
      "outputs": [{"address": "Public-Key-A", "amount": 25 }],
      "inputs": []
    }
  ],
  "header": {
    "hash": "00000ba50a4",
    "nonce": 286743,
    "timestamp": 1450584386,
    "target": "0000100000000000000000000000000000000000000000000000000000000000",
    "transactions_hash": "some-hash",
    "parent_hash": "0000000000000000000000000000000000000000000000000000000000000000"
  }
}]
```

Our chain here contains a single block which contains a single transaction (the coinbase).
**Note** that for this example block I'll be filling in many of the fields with placeholder values
(such as the hashes and public keys) in order to keep things simple. While the structure
is accurate, the individual fields may not be.

### Checking Balance

Let's start with a quick balance check. We'll refer to the public/private keypair contained
in our wallet as `Wallet-A`, so in order to find the corresponding balance, we need
to find all Unspent Transaction Outputs addressed to `Public-Key-A`.

Searching through the chain we find 1 output -- the output contained in transaction `a98f3d`
at index `0` that is assigned to our key. Looking through the inputs in the chain, we find there
are none, so we can safely assume that our output is unspent -- our wallet contains a
balance of `25` coins.

### Making a Payment

Now let's suppose we wanted to send a payment of 15 coins to the address `Public-Key-B`, and
that we want to include a transaction fee of 1 coin. Our total payment amount will thus be 16 coins,
so in order to fund this transaction we'll need to come up with one or more unspent transaction outputs
whose value add up to at least 16.

Fortunately for us we have an appropriate output immediately available -- the previously mentioned output
at transaction `a98f3d` and index `0`.

### Adding Inputs

Let's start building our example transaction by including this as an input:

```json
{
  "inputs": [{"source-hash": "a98f3d", "source-index": 0}]
}
```

Now we have started a transaction with 1 input bringing an available value of 25.

### Adding Payment Output

The next step is to add the output assigning the payment amount (15) to
the receiving address (`Public-Key-B`). Adding this would look like:

```json
{
  "inputs": [{"source-hash": "a98f3d", "source-index": 0}],
  "outputs": [{"amount": 15, "address": "Public-Key-B"}]
}
```

### Including Change

So far we have pulled in 25 coins worth of inputs to our transaction but only
included outputs spending 15 coins, leaving a difference of 10. Remember -- any
difference between output value and input value is assumed to be a transaction
fee.

But we wanted to provide a fee of 1 coin, not 10. To fix this, we need to transfer
the difference *back to ourselves* in the form of "change". Fortunately, returning
change doesn't actually require any special structures or techniques. We simply apply
the standard process for including outputs in a transaction, since change is
simply another output of the appropriate value that goes back to the *sending key*.

We can calculate the change amount by the following formula:

```
total input value - total output value - desired transaction fee
```

For our example, this will yield: `25 - 15 - 1 = 9`

So, let's add another output to our transaction paying this amount back
to our key (`Public-Key-A`):

```json
{
  "inputs": [{"source-hash": "a98f3d", "source-index": 0}],
  "outputs": [{"amount": 15, "address": "Public-Key-B"},
	          {"amount": 9, "address": "Public-Key-A"}]
}
```

Now this is looking better -- our transaction has total inputs of 25 and
total outputs of 24, leaving the intended 1 coin as an implicit transaction
fee.

### Signing Inputs

Now that we have the expected inputs and outputs of our transaction in place,
we can perform the vital step of signing the inputs. This process is described
in detail in [Iteration 0](https://github.com/worace/coinage/blob/master/iterations/iteration_0.markdown#signing-transaction-inputs),
but for now we'll use a placeholder signature.

The important thing to remember is that this signature must be provided using
the private key associated with Public-Key-A, since that is the address
to which the output that our input references is assigned.

```json
{
  "inputs": [{"source-hash": "a98f3d",
              "source-index": 0,
              "signature": "Signed-With-Private-Key-A"}],
  "outputs": [{"amount": 15, "address": "Public-Key-B"},
	          {"amount": 9, "address": "Public-Key-A"}]
}
```

### Filling in Transaction Details

Now let's fill in a few more pieces of transaction info.

First, we'll add a simple timestamp (remember transactions are
timestamped to the millisecond):

```json
{
  "inputs": [{"source-hash": "a98f3d",
              "source-index": 0,
              "signature": "Signed-With-Private-Key-A"}],
  "outputs": [{"amount": 15, "address": "Public-Key-B"},
              {"amount": 9, "address": "Public-Key-A"}],
  "timestamp": 1452028966891
}
```

Finally, we'll fingerprint the transaction with a hash of all its contents,
which will help us detect if anything in the transaction was to change.

(Transaction Hashing process is described in [Iteration 0](https://github.com/worace/coinage/blob/master/iterations/iteration_0.markdown#hashing-transactions))

```json
{
  "inputs": [{"source-hash": "a98f3d",
              "source-index": 0,
              "signature": "Signed-With-Private-Key-A"}],
  "outputs": [{"amount": 15, "address": "Public-Key-B"},
              {"amount": 9, "address": "Public-Key-A"}],
  "timestamp": 1452028966891,
  "hash": "02d8c33"
}
```

At this point we have a valid, fleshed-out transaction. We could add it to a
block that we are attempting to mine or send it over the network to other
miners in the hope that they will add it to a block of their own.

## Adding to the Block Chain

Let's continue looking at our example payment and see what might happen
when it gets included into a block. For this example, we'll assume
the owner of Wallet A generated that transaction to transfer funds
to Wallet B. Then, A distributed the transaction over the network and
it was picked up by a third user, C, who successfully included the transaction
in a block and mined it.

Our hypothetical chain might now look like this:

```json
[
{
  "transactions": [
    {
      "hash": "a98f3d",
      "timestamp": 1450584386520,
      "outputs": [{"address": "Public-Key-A", "amount": 25 }],
      "inputs": []
    },
  ],
  "header": {
    "hash": "00000ba50a4",
    "nonce": 286743,
    "timestamp": 1450584386,
    "target": "0000100000000000000000000000000000000000000000000000000000000000",
    "transactions_hash": "some-hash",
    "parent_hash": "0000000000000000000000000000000000000000000000000000000000000000"
  }
},
{
  "transactions": [
    {
      "hash": "4f912a4",
      "timestamp": 1450584386520,
      "outputs": [{"address": "Public-Key-C", "amount": 26 }],
      "inputs": []
    },
    {
     "inputs": [{"source-hash": "a98f3d",
                "source-index": 0,
                "signature": "Signed-With-Private-Key-A"}],
     "outputs": [{"amount": 15, "address": "Public-Key-B"},
                 {"amount": 9, "address": "Public-Key-A"}],
     "timestamp": 1452028966891,
     "hash": "02d8c33"
    }
  ],
  "header": {
    "hash": "000009cd382",
    "nonce": 286743,
    "timestamp": 1450584386,
    "target": "0000100000000000000000000000000000000000000000000000000000000000",
    "transactions_hash": "some-hash",
    "parent_hash": "00000ba50a4"
  }
}]
```

This structure is starting to get complicated, so let's break it down.

To start with, the first block is still the same (blocks in the chain are
immutable -- we can only add more blocks without changing the original ones).
So that block contains 1 transaction which is a coinbase paying 25 coins to Public-Key-A.

On top of that block, Miner C has now added another block. This block is more interesting
since it contains 2 transactions. The first transaction is (as always) a coinbase. But this
coinbase is a little different -- it assigns **25 coins** to Public-Key-C. Why? Because the coinbase
value is equal to the reward amount (25) plus any transaction fees (in this case the 1 coin that the
owner of Key A included with their payment).

Additionally, the second block contains a second transaction -- the actual payment that we
generated above. As we saw, this transaction consumes a single input and transfers 15 coins
in one output to Public-Key-B and 9 coins in a second output to Public-Key-A (the change).
This makes up a total output of 24 coins, leaving 1 coin available as a fee reward to the miner.

### Updated Balances

So let's look at the balances for our 3 wallets at this point.

__Wallet A__

Has 2 outputs: Transaction `a98f3d` at Index `0` (Worth 25) and Transaction `02d8c33` at Index 1 (Worth 9).

However the `a98f3d-0` input was consumed by the first input to Transaction `02d8c33`,
so it can be disregarded. This leaves Wallet A with a current balance of **9**.

__Wallet B__

Has 1 output: Transaction `02d8c33` at Index `0` (Worth 15). There are no subsequent
inputs that spend this transaction output, so it counts toward B's current balance of **15**

__Wallet C__

Has 1 output: Transaction `4f912a4` at Index `0` (a coinbase, worth 26). There are no
subsequent inputs that spend this output, so C's current balance is **26**.

## Takeaways

The main takeaways from this process are:

* A wallet's balance is the result of adding up all outputs assigned to its public key
and removing any that are spent by other transaction inputs.
* The Fee attached to a transaction is implicit and is calculated as the difference between
the value of all inputs and the value of all outputs
* If you need to make change out of a transaction (because the inputs you have sourced are larger
than the amount + txn fee you want to pay), you should include this as an additional
transaction output which pays the appropriate amount back to your public key
* Miners capture transaction fees by including them in their coinbase transaction. So the
value of a coinbase output will always be the current mining reward plus the value of
any fees in the transaction
