.. index:: purchases

*********************
Safe Remote Purchases
*********************

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
