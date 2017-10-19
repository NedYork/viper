.. index:: auction;blind, auction;open, blind auction, open auction

*************
Blind Auction
*************

.. _simple_auction:

Simple Open Auction
===================

As an introductory example of a smart contract written in Viper, we will begin
with a simple auction contract. As we dive into the contract, it is important
to remember that all Viper syntax is valid Python3 syntax, however not all
Python3 functionality is available in Viper.

In this contract, we will be looking at an auction contract where
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

And there you have it - a simple open auction contract. Of course, this is an
introductory example with barebones functionality. As we move on to exploring
more complex contracts, we will encounter more design patterns and features of
the Viper language.

And of course, no smart contract tutorial is complete without a note on security.
It's always important to keep security in mind when designing a smart contract.
As any application becomes more complex, the greater the potential for
introducing new risks. Thus, it's always good practice to keep contracts as
readable and simple as possible.

Whenever you're ready, let's turn it up a notch in the next example.
