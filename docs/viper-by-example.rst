###################
Viper by Example
###################

.. index:: voting, ballot


******
Voting
******

As an introductory example of a smart contract written in Viper, we will begin
with a voting contract of moderate complexity. As we dive into the contract,
it is important to remember that all Viper syntax is valid Python3 syntax,
however not all Python3 functionality is available in Viper.

In this contract, we will implement a system for participants to vote on a list
of proposals. The chairperson of the contract will be able to give each
participant the right to vote and each participant may choose to vote or
delegate his/her vote to another voter. Finally, a winning proposal will be
determined upon calling the ``winning_proposals()`` method, which iterates through
all the proposals and returns the one with the greatest number of votes.

::

  voters: public({
      weight: num,
      voted: bool,
      delegate: address,
      vote: num
  }[address])

  proposals: public({
      name: bytes32,
      vote_count: num
  }[num])

  voter_count: public(num)
  chairperson: public(address)

  def __init__(_proposalNames: bytes32[2]):
      self.chairperson = msg.sender
      self.voter_count = 0
      for i in range(2):
          self.proposals[i] = {
              name: _proposalNames[i],
              vote_count: 0
          }

  def give_right_to_vote(voter: address):
      assert msg.sender == self.chairperson
      assert not self.voters[voter].voted
      assert self.voters[voter].weight == 0
      self.voters[voter].weight = 1
      self.voter_count += 1

  def delegate(_to: address):
      to = _to
      assert not self.voters[msg.sender].voted
      assert not msg.sender == to
      for i in range(self.voter_count, self.voter_count+1):
          if self.voters[to].delegate:
              assert self.voters[to].delegate != msg.sender
              to = self.voters[to].delegate
      self.voters[msg.sender].voted = True
      self.voters[msg.sender].delegate = to
      if self.voters[to].voted:
          self.proposals[self.voters[to].vote].vote_count += self.voters[msg.sender].weight
      else:
          self.voters[to].weight += self.voters[msg.sender].weight

  def vote(proposal: num):
      assert not self.voters[msg.sender].voted
      self.voters[msg.sender].voted = True
      self.voters[msg.sender].vote = proposal
      self.proposals[proposal].vote_count += self.voters[msg.sender].weight

  @constant
  def winning_proposal() -> num:
      winning_vote_count = 0
      for i in range(5):
          if self.proposals[i].vote_count > winning_vote_count:
              winning_vote_count = self.proposals[i].vote_count
              winning_proposal = i
      return winning_proposal

  @constant
  def winner_name() -> bytes32:
      return self.proposals[self.winning_proposal()].name


As we can see, this is contract of moderate length which we will dissect
section by section. Let’s begin!


::

  voters: public({
      weight: num,
      voted: bool,
      delegate: address,
      vote: num
  }[address])

  proposals: public({
      name: bytes32,
      vote_count: num
  }[num])

  voter_count: public(num)
  chairperson: public(address)

The variable ``voters`` is initialized as a dictionary where the key is
the voter’s public address and the value is a struct describing the
voter’s properties: ``weight``, ``voted``, ``delegate``, and ``vote``, along
with their respective datatypes. You may notice the ``voters`` declaration being
passed into the ``public`` function; this allows the variable to be accessible to
calls external to the contract. Initializing the struct without the  ``public``
function defaults to a private declaration and thus only accessible to methods
within the same contract. The ``public`` function additionally creates a
‘getter’ function for variable, accessible with
``self.get_voters(some_voter_num)``.

Similarly, the ``proposals`` variable is initialized as a ``public`` dictionary
with num as the key’s datatype and a struct to represent each proposal
with the properties ``name`` and ``vote_count``. We can access any value
by key’ing in with a num just as one would with an index in an array.
However, be aware that key’ing in with an un-set key will return a value of
0 rather than nil.

Then, ``voter_count`` and ``chairperson`` are initialized as ``public`` with
their respective datatypes.

Let’s move onto the constructor.

::

  # Setup global variables
  def __init__(_proposalNames: bytes32[2]):
      self.chairperson = msg.sender
      self.voter_count = 0
      for i in range(2):
          self.proposals[i] = {
              name: _proposalNames[i],
              vote_count: 0
          }


When calling any method within a contract, we are provided with a built-in
variable ``msg`` and we can access the public address of any method caller with
``msg.sender``. In the constructor, we hard-coded the contract to accept an
array argument of exactly two proposal names of type ``bytes32`` for the contracts
initialization. Because upon initialization, the ``__init__()`` method is called
by the contract creator, we have access to the contract creator’s address with
``msg.sender`` and store it in the contract variable ``self.chairperson``. We
also initialize the contract variable ``self.voter_count`` to zero to initially
represent the number of votes allowed. This value will be incremented as each
participant in the contract is given the right to vote by the method
``give_right_to_vote()``, which we will explore next. We loop through the two
proposals from the argument and insert them into ``proposals`` dictionary with
their respective index in the original array as its key.

Now that the initial setup is done, lets take a look at the functionality.

::

  # Give `voter` the right to vote on this ballot.
  # May only be called by `chairperson`.
  def give_right_to_vote(voter: address):
      # Throws if sender is not chairpers
      assert msg.sender == self.chairperson
      # Throws if voter has already voted
      assert not self.voters[voter].voted
      # Throws if voters voting weight isn't 0
      assert self.voters[voter].weight == 0
      self.voters[voter].weight = 1
      self.voter_count += 1


We need a way to control who has the ability to vote. The method
``give_right_to_vote()`` is a method callable by only the chairperson by taking
a voter address and granting it a right to vote by incrementing the voter's
``weight`` property. We sequentially check for 3 conditions using ``assert`` which
takes any boolean statement. If all ``assert`` statements pass, we continue
to the next lines; otherwise, the method will throw an error.
The ``assert not`` function will check for falsy boolean values -
in this case, we want to know that the voter has not already voted. To represent
voting power, we will set their ``weight`` to ``1`` and we will keep track of the
total number of voters by incrementing ``voter_count``.


::

  # Delegate your vote to the voter `to`.
  def delegate(_to: address):
      to = _to
      # Throws if sender has already voted
      assert not self.voters[msg.sender].voted
      # Throws if sender tries to delegate their vote to themselves
      assert not msg.sender == to
      # loop can delegate votes up to the current voter count
      for i in range(self.voter_count, self.voter_count+1):
          if self.voters[to].delegate:
          # Because there are not while loops, use recursion to forward the delegation
          # self.delegate(self.voters[to].delegate)
              assert self.voters[to].delegate != msg.sender
              to = self.voters[to].delegate
      self.voters[msg.sender].voted = True
      self.voters[msg.sender].delegate = to
      if self.voters[to].voted:
          # If the delegate already voted,
          # directly add to the number of votes
          self.proposals[self.voters[to].vote].vote_count += self.voters[msg.sender].weight
      else:
          # If the delegate did not vote yet,
          # add to her weight.
          self.voters[to].weight += self.voters[msg.sender].weight

In the method ``delegate``, firstly, we check to see that ``msg.sender`` has not
already voted and secondly, that the target delegate and the ``msg.sender`` are
not the same. A voter shouldn’t be able to delegate a vote to him/herself. We,
then, loop through all the voters to determine whether the person delegate to
had further delegated his/her vote to someone else in order to follow the
chain of delegation. We then mark the ``msg.sender`` as having voted if they
delegated their vote. We increment the proposal’s ``vote_count`` directly if
the delegate had already voted or increase the  delegate’s vote ``weight``
if the delegate has not yet voted.

::

  # Give your vote (including votes delegated to you)
  # to proposal `proposals[proposal].name`.
  def vote(proposal: num):
      assert not self.voters[msg.sender].voted
      self.voters[msg.sender].voted = True
      self.voters[msg.sender].vote = proposal
      # If `proposal` is out of the range of the array,
      # this will throw automatically and revert all
      # changes.
      self.proposals[proposal].vote_count += self.voters[msg.sender].weight


Now, let’s take a look at the logic inside the ``vote()`` method, which is
surprisingly simple. The method takes the key of the proposal in the ``proposals``
dictionary as an argument, check that the method caller had not already voted,
sets the voter’s ``vote`` property to the proposal key, and increments the
proposals ``vote_count`` by the voter’s ``weight``.

With all the basic functionality complete, what’s left is simply returning
the winning proposal. To do this, we have two methods: ``winning_proposal()``,
which returns the key of the proposal, and ``winner_name()``, returning the
name of the proposal. Notice the ``@constant`` decorator on these two methods.
We do this because the two methods only read the blockchain state and not modify
it. Remember, reading the blockchain state is free; modifying the state costs gas.
By having the ``@constant`` decorator, we let the EVM know that this is a
read-only function and we benefit by saving gas fees.

::

  # Computes the winning proposal taking all
  # previous votes into account.
  @constant
  def winning_proposal() -> num:
      winning_vote_count = 0
      for i in range(5):
          if self.proposals[i].vote_count > winning_vote_count:
              winning_vote_count = self.proposals[i].vote_count
              winning_proposal = i
      return winning_proposal


The ``winning_proposal()`` method returns the key of proposal in the ``proposals``
dictionary. We will keep track of greatest number of votes and the winning
proposal with the variables ``winning_vote_count`` and ``winning_proposal``,
respectively by looping through all the proposals.

::

  # Calls winning_proposal() function to get the index
  # of the winner contained in the proposals array and then
  # returns the name of the winner
  @constant
  def winner_name() -> bytes32:
      return self.proposals[self.winning_proposal()].name

And finally, the ``winner_name()`` method returns the name of the proposal by
key’ing into the ``proposals`` dictionary with the return result of the
``winning_proposal()`` method.

And there you have it - a simple voting contract. Of course, there are a few
optimizations that can be made in this contract, but we purposefully kept this
example simple to demonstrate the breadth of functionality available in the
language. Hopefully, this example has provided some insight to the possibilities
of Viper. Currently, many transactions are needed to assign the rights to vote
to all participants. As an exercise, can we try to optimize this?

And of course, no smart contract tutorial is complete without a note on security.
It's always important to keep security in mind when designing a smart contract.
As any application becomes more complex, the greater the potential for
introducing new risks. Thus, it's always good practice to keep contracts as
readable and simple as possible.

Whenever we’re ready, let’s move on to the next example.


.. index:: auction;open, open auction

*************
Simple Open Auction
*************

.. _simple_auction:


In this contract, we will be looking at a simple auction contract where
participants can submit bids during a limited window period. When the auction
period ends, a predetermined beneficiary will receive the amount of the highest
bid.

::

  beneficiary: public(address)
  auction_start: public(timestamp)
  auction_end: public(timestamp)

  highest_bidder: public(address)
  highest_bid: public(wei_value)

  ended: public(bool)

  def __init__(_beneficiary: address, _bidding_time: timedelta):
      self.beneficiary = _beneficiary
      self.auction_start = block.timestamp
      self.auction_end = self.auction_start + _bidding_time

  @payable
  def bid():
      assert block.timestamp < self.auction_end
      assert msg.value > self.highest_bid
      if not self.highest_bid == 0:
          send(self.highest_bidder, self.highest_bid)
      self.highest_bidder = msg.sender
      self.highest_bid = msg.value

  def auction_end():
      assert block.timestamp >= self.auction_end
      assert not self.ended
      self.ended = True
      send(self.beneficiary, self.highest_bid)


As you can see, this example only has a constructor, two methods to call, and
a few variables to manage the contract state. Believe it or not, this is all we
need for a basic implementation of a smart contract.

Let's get started!

::

  # Beneficiary receives money from the highest bidder
  beneficiary: public(address)

  # Limited window period of auction
  auction_start: public(timestamp)
  auction_end: public(timestamp)

  # Current state of auction
  highest_bidder: public(address)
  highest_bid: public(wei_value)

  # Set to true at the end to disallows any change
  ended: public(bool)

We begin by declaring a few variables to keep track of our contract state.
We initialize a global variable ``beneficiary`` by calling ``public`` on the
datatype ``address``. The ``beneficiary`` will be the receiver of money from
the highest bidder. By declaring the variable *public*, the variable is
callable by external contracts. By default, all variables are *private* and
only accessible within the contract.

We also initialize the variables ``auction_start`` and ``auction_end`` with
the datatype ``timestamp`` to manage the open auction period and ``highest_bid``
with datatype ``wei_value``, the smallest denomination of an
ether, to manage auction state. The variable ``ended`` is a boolean to determine
whether the auction is officially over.

Now, the constructor.

::

  # Create a simple auction with `_bidding_time`
  # seconds bidding time on behalf of the
  # beneficiary address `_beneficiary`.
  def __init__(_beneficiary: address, _bidding_time: timedelta):
      self.beneficiary = _beneficiary
      self.auction_start = block.timestamp
      self.auction_end = self.auction_start + _bidding_time

The contract is initialized with two arguments: ``_beneficiary`` of type
``address`` and ``bidding_time`` with type ``timedelta``, the time difference
between the start and end of the auction. We then store these two pieces of
information into the contract variables ``self.beneficiary`` and
``self.auction_end``. Notice that we have access to the current time by
calling ``block.timestamp``. ``block`` is an object available within any Viper
contract and provides information of the block at the time of calling.
Similar to ``block``, another important object available to us within the
contract is ``msg``, which provides information on the method caller as we will
soon see.

With initial setup out of the way, lets look at how our users can make bids.

::

  @payable
  def bid():
      # Check if bidding period is over.
      assert block.timestamp < self.auction_end
      # Check if bid is high enough
      assert msg.value > self.highest_bid
      if not self.highest_bid == 0:
          # Sends money back to the previous highest bidder
          send(self.highest_bidder, self.highest_bid)
      self.highest_bidder = msg.sender
      self.highest_bid = msg.value

The ``@payable`` decorator will require a user to send some ethers to the
contract in order to call the decorated method. In this case, a user wanting
to make a bid would call the ``bid()`` method while sending an amount equal
to their desired bid (not including gas fees). The value sent by the sender
will be available by calling ``msg.value``. Similarly, the address of the sender
can be obtained by calling ``msg.sender``.

Here, we first check whether the current time is before the auction's end time.
We also check to see that the new bid is greater than the highest bid. If
either of these conditions returned false, the bid() method would throw an error
and revert the transaction. If the two ``assert`` statements pass along with a
check that the previous bid is not equal to zero, we can safely conclude that
we have a valid new highest bid. We will send back the previous ``highest_bid``
to the previous ``highest_bidder`` and set our new ``highest_bid`` and
``highest_bidder``.

::

  # End the auction and send the highest bid
  # to the beneficiary.
  def auction_end():
      # It is a good guideline to structure functions that interact
      # with other contracts (i.e. they call functions or send Ether)
      # into three phases:
      # 1. checking conditions
      # 2. performing actions (potentially changing conditions)
      # 3. interacting with other contracts
      # If these phases are mixed up, the other contract could call
      # back into the current contract and modify the state or cause
      # effects (Ether payout) to be performed multiple times.
      # If functions called internally include interaction with external
      # contracts, they also have to be considered interaction with
      # external contracts.

      # 1. Conditions
      # Check if auction end time has been reached
      assert block.timestamp >= self.auction_end
      # Check if this function has already been called
      assert not self.ended

      # 2. Effects
      self.ended = True

      # 3. Interaction
      send(self.beneficiary, self.highest_bid)

With the ``auction_end()`` method, we check whether our current time is past
the ``auction_end`` time we set upon initialization of the contract. We also
check that ``self.ended`` had not previously been set to True. We do this
to prevent any calls to the method if the auction had already ended,
which could potentially be malicious if the check had not been made.
We then officially end the auction by setting ``self.ended`` to ``True``
and sending the highest bid amount to the beneficiary.

And there you have it - an open auction contract. Of course, this is a
simplified example with barebones functionality. As we move on to exploring
more complex examples, we will encounter more design patterns and features of
the Viper language.

Whenever you're ready, let's turn it up a notch in the next example.


.. index:: purchases

*********************
Safe Remote Purchases
*********************

.. _safe_remote_purchases:


In this example, we have an escrow contract implementing a system for a trustless
transaction between a buyer and a seller. In this system, a seller posts an item
for sale and makes a deposit to the contract of twice the item's value. At this
moment, the contract has a balance of 2*value. The seller can reclaim the deposit
and close the sale as long as a buyer had not yet made a purchase. If a buyer
is interested in making a purchase, he/she would make a payment and submit an equal
amount for deposit (totaling 2*value) into the contract and locking the contract
from further modification. At this moment, the contract has a balance of
4*value and the seller would send the item to buyer. Upon the buyer's receipt of
the item, the buyer will mark the item as received in the contract, thereby
returning the buyer's deposit (not payment) and releasing the remaining funds to
the seller - completing the transaction.

There are certainly others ways of designing a secure escrow system with less
overhead for both the buyer and seller, but for the purpose of this example,
we want to explore one way how an escrow system can be implemented trustlessly.

Let's go!

::

  value: public(wei_value) #Value of the item
  seller: public(address)
  buyer: public(address)
  unlocked: public(bool)

  @payable
  def __init__():
      assert (msg.value % 2) == 0
      self.value = msg.value / 2 #Seller initializes contract by posting a safety deposit of 2*value of the item up for sale
      self.seller = msg.sender
      self.unlocked = True

  def abort():
      assert self.unlocked #Is the contract still refundable
      assert msg.sender == self.seller #Only seller can refund his deposit before any buyer purchases the item
      selfdestruct(self.seller) #Refunds seller, deletes contract

  @payable
  def purchase():
      assert self.unlocked #Contract still open (item still up for sale)?
      assert msg.value == (2*self.value) #Is the deposit of correct value?
      self.buyer = msg.sender
      self.unlocked = False

  def received():
      assert not self.unlocked #Is the item already purchased and pending confirmation of buyer
      assert msg.sender == self.buyer
      send(self.buyer, self.value) #Return deposit (=value) to buyer
      selfdestruct(self.seller) #Returns deposit (=2*value) and the purchase price (=value)

This is also a moderately short contract, however a little more complex in
logic. Let's break down this contract bit by bit.

::

  value: public(wei_value) #Value of the item
  seller: public(address)
  buyer: public(address)
  unlocked: public(bool)

Like the other contracts, we begin by declaring our global variables public with
their respective datatypes. Remember that the ``public`` function allows the
variables to be *readable* by an external caller, but not *writeable*.

::

  @payable
  def __init__():
      assert (msg.value % 2) == 0
      self.value = msg.value / 2 #Seller initializes contract by posting a safety deposit of 2*value of the item up for sale
      self.seller = msg.sender
      self.unlocked = True

With a ``@payable`` decorator on the constructor, the contract creator will be
required to make an initial deposit equal to twice the item's sale value to
initialize the contract, which will be later returned. This is in addition to
the gas fees needed to deploy the contract on the blockchain, which is not
returned. We ``assert`` that the deposit is divisible by 2 to ensure that the
seller deposited a valid amount. The constructor stores the item's value
in the contract variable ``self.value`` and saves the contract creator into
``self.seller``. The contract variable ``self.unlocked`` is initialized to True.

::

  def abort():
      assert self.unlocked #Is the contract still refundable?
      assert msg.sender == self.seller #Only seller can refund his deposit before any buyer purchases the item
      selfdestruct(self.seller) #Refunds seller, deletes contract

The ``abort()`` method is a method only callable by the seller and while the
contract is still ``unlocked`` - meaning it callable only prior to any buyer
making a purchase. As we will see in the ``purchase()`` method that as soon as
a buyer calls the ``purchase()`` method and sends a valid amount to the contract,
the contract will be locked and the seller will no longer be able to call
``abort()``.

When the seller calls ``abort()`` and if the ``assert``
statements pass, the contract will call the ``selfdestruct()`` function and
refunds the seller and subsequently destroys the contract.

::

  @payable
  def purchase():
      assert self.unlocked #Contract still open (item still up for sale)?
      assert msg.value == (2*self.value) #Is the deposit of correct value?
      self.buyer = msg.sender
      self.unlocked = False

Like the constructor, the ``purchase()`` method has a ``@payable`` decorator,
meaning it must be called with a payment. For the buyer to make a valid
purchase, we must first ``assert`` that the contract is unlocked and that
the amount sent is equal to twice the item's value. We then set the buyer to
the ``msg.sender`` and lock the contract. At this point, the contract has a
balance equal to 4 times the item value and the seller must send the item
to the buyer.

::

  def received():
      assert not self.unlocked #Is the item already purchased and pending confirmation of buyer
      assert msg.sender == self.buyer
      send(self.buyer, self.value) #Return deposit (=value) to buyer
      selfdestruct(self.seller) #Returns deposit (=2*value) and the purchase price (=value)

Finally, upon the buyer's receipt of the item, the buyer can confirm his/her
receipt by calling the ``received()`` method to distribute the funds as
intended - the seller receives 3/4 of the contract balance and the buyer
receives 1/4.

By calling ``received()``, we begin by ``assert``ing that the contract is indeed
locked, ensuring that a buyer had previously paid. We also ensure that this
method is only callable by the buyer. If these two ``assert`` statements pass,
we refund the buyer his/her initial deposit and send the seller the remain
funds. The contract is finally destroyed and the transaction is complete.
