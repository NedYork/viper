###################
Viper by Example
###################

.. index:: voting, ballot


******
Voting
******

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

When we’re ready, let’s move on to the next example.
