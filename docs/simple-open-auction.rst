.. index:: auction;blind, auction;open, blind auction, open auction

*************
Blind Auction
*************

.. _simple_auction:

Simple Open Auction
===================

In this short example, we will be looking at an auction contract where
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


This example is actually shorter and simpler than the previous, but here, we
also introduce some new concepts and features.

Let's begin.

::

  # Auction params
  # Beneficiary receives money from the highest bidder
  beneficiary: public(address)
  auction_start: public(timestamp)
  auction_end: public(timestamp)

  # Current state of auction
  highest_bidder: public(address)
  highest_bid: public(wei_value)

  # Set to true at the end, disallows any change
  ended: public(bool)

We begin by declaring a few parameters to keep track of the contracts state.
We've already seen the datatype 'address' and 'bool' in our previous example,
but here, we initialize auction_start and auction_end with the datatype
'timestamp' and highest_bid with datatype 'wei_value', the smallest denomination
of an ether. The beneficiary will be the receiver of money from the highest
bidder, and we will use auction_start and auction_end to determine whether we
are still within the auction period. 'ended' is a boolean to determine whether
the auction is officially over.

Now, the constructor.

::

  # Create a simple auction with `_bidding_time`
  # seconds bidding time on behalf of the
  # beneficiary address `_beneficiary`.
  def __init__(_beneficiary: address, _bidding_time: timedelta):
      self.beneficiary = _beneficiary
      self.auction_start = block.timestamp
      self.auction_end = self.auction_start + _bidding_time

The contract is initialized with two arguments -  1) the beneficiary's address
and 2) the time difference between the start time and end time of the auction.
Notice that we set our contract variable auction_start to the current time by
setting it to 'block.timestamp'. Similar to 'msg', 'block' is a publicly
available variable within the contract and provides information of the current
block. Also notice that this time, we did not save 'msg.sender' to a variable.
Most of the time we will want to save a reference to the contract creator's
address for some future purpose, however it is not always necessary.

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

The @payable decorator will allow a user to send some ethers to the contract
which will automatically trigger the a function with the payable decorator. In
this case, a user wanting to make a bid would send an amount equal to their
desired bid to the address of this contract's instance and the contract will
call the bid() method. The value sent by the sender will be available by calling
'msg.value'.

Here, we first check whether the current time, retrieved from
'block.timestamp', is before the auction's end time. We also check to see
that the new bid is greater than the highest bid. If either of these conditions
returned false, the bid() method would throw an error and revert the transaction.




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

With the auction_end() method, we check with the 'assert' function that our
current time, which we get with 'block.timestamp' is past the auction's end
time, which we set in the constructor. We also check that contract variable
'self.ended' had not be set to True. We then officially end the auction by
setting 'self.ended' to True and sending the highest bid amount to the
beneficiary.
