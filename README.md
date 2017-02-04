# Online Safe Voting System

This is a draft. At the current state this document doesn't provide a safe voting system. This document is open sourced and allow peer-review.
The goal is not yet to implement such a proposal, but to catch the limits of the current proposal.

---

## Introduction

The aim is to propose a way for people to vote online, from anywhere, safely.
By "safe" we mean that nobody will be able to know who people have voted for. Votes shall remains secrets.

The general idea is to use the blockchain, which will contain every vote every citizen has done and therefore will be able to ensure that every vote has been taken into account.

Each user shall be able to verify that his vote has been taken into account correctly. But still ensure secrecy of whom voted for what.

Each user would have the ability to confirm to the system that his vote has been taken into account and is correct.

By replaying all commits from the blockain, one should be able to verify that the given results match his own results and therefore are correct.

This way, anyone could have the possibility to ensure the public results have not been tampered with. 


## Entities

### Targets

A Target is something a Source vote for.
A Target can represent a human being, an item, an idea, anything. 
One can vote for and against each target, the vote will be applied as a commit with all the details, and will also increment/decrement the votes given to every target.

target:
  resource: <string> - Resource name ('applicant', 'idea', ...)
  votes: <array>
    - value: <int> - given value by the source, can be +/-
    - sourceId: <string> - private source id (nobody but the source can verify its vote cast, but everybody can know that something/someone has casted this vote, without the ability to idenfity it)
    - appliedAt: <UTCDate> - datetime when the vote was applied to the blockchain (not when the Source actually casted its vote, but rather when the queue was applied) 

In order to know what the Target score is, every vote must be calculated, which will give a result of the following:
  positive: <int> - Total of positive votes
  negative: <int> - Total of negative votes
  total: <int> - Total of positive votes minus negative votes


### Source

A Source is something that cast a vote.
A source usually is a human being.

A Source has a private identifier and a public identifier.

Public id is to be used for public records. It is not supposed to be randomly generated but must be given as a way to uniquely indentify a source. 
Every target must have a public id that is unique, and allows to identify the real target identity itself.
  i.e: Use the state security number, or any internal identifier used by the state to identify its citizen. 

Private id is for the Source to identify itself, it is especially useful when one wants to check his vote was taken into account and verify it is correct to what he has voted for.
Private id is randomly generated for each pool of vote and cannot be used to identify the same source across several pools of votes.
  i.e: Presidential election 2017, first run: will generate a private id. The second run (if any) will generate another private id for the same source

The private id is not kept in the database, it must be kept by the source itself. There must be no way for the private and public id to be available at the same time
  i.e: private and public id must never been sent on the same payload

The private id must be generated using both a key and a salt. 
The key could be based on the public id (but must not be the public id on its own). The key will be saved in the database.
  i.e: key could be generated based on the Source public id AND another identifier. Maybe the id of the vote pool. (in order for the private id to be different for the next vote pool)
The salt could be a password the Source has defined, but this salt wouldn't be known/stored in the database.
The private id should be kept by the source as a way of identifying its vote.

The system must be able to regenerate the same private id using the key and the salt.  
  i.e: The source has lost its private id. It gives its salt as input and get its private id back.

If the source loose its salt, there is no possible recovery of the private id. But the vote cast will still be taken into account. Only the ability to verify it was taken into account correctly will be lost.

---

## Operations

### Vote cast process

This process can change a little bit depending on wether we are voting for a single Target (like nowadays elections) or multiple Targets.

If voting for a single Target then the process can be done quite easily, it's a one-time choice.

If voting for multiple Targets then we must decided whether we allow or not for the Source to cast its votes all at once, or if it can be done separately. 
Theorically, both are possible. 

When a Source votes for the first time, the source private id is generated using its salt and visually given to the Source.
The vote cast must not be applied directly. Instead, it must go into a queue (pending vote casts). 
The previously generated private id is given and store in this queue, as well as all vote casts.

The queue will be applied at once at some point. The trigger can be the number of item in the queue, or a delay, or something else.
  i.e: Every 2h, all pending votes will be applied.

This is a security measure to limit the ability for a third party to figure out who voted what. If a vote cast and public results were in real-time, it may be possible to know who has voted what. By applying a queue at once, it is much harder.

When the queue is applied, it creates a commit for each item in the queue. (blockchain commit, for further tracking)
For each item, the Targets are modified, basically adding a new record in the target.votes array. Modifying the array existing records must not be possible. Only adding an item must be possible.


### Vote verification process - For a Source

_This part of the design is probably not properly secured, there may be better alternatives._

Once a Source has casted its votes, and once the queue has been applied, a Source shall have the ability to search for its votes. 

The most important concern is the following: The Source private id itself must not go through the network in any case. (even in HTTPS)
The general idea here, would be to filter all the votes to minimize the amount of data to be loaded. Then to type the private id in a filter, which would filter on the already-fetched data. The private id would only be used locally and wouldn't be tracable. The use of an input that is contained in a secured iframe is required. (like it's done in online paiement website)

Since a limited amount of the votes have already been fetched, it's a local filter that is going to only display the relevant Source votes. The private id won't be send through the network, and the use of an iframe should ensure it is safe. 

The idea is to limit the fetched data so they can be filtered without too much difficulties. But we also need to send a lot of data to make sure that even if something was sniffing the network, the actual votes the source had made would be lost within tons of non-related votes. The system must be set up in such a way there are always at least 1000-10000 records sent. (doesn't imply they must be all displayed at once)

Then, with the private id filter, we only display the rights votes to the Source.

This way, we have a two-ways operation that should ensure privacy.

**Note**: The input within an iframe is extemelly important, because it is the only way to ensure that third-parties software like plugins/addons on the browser wouldn't be able to read the given value. It wouldn't secure against keylogger though. Maybe a more secure way is not to type but, like in e-banking systems, to select the numbers on by one, with a random position of the numbers on the screen.

**Note**: The limitations of the fetched data can be done using different filters, the time could be one. So we could fetch data for all votes casted between 2pm and 11pm on the 28th of the month, for example.


#### Complaints

If the Source doesn't agree with the results, it can fill a complaint, which will lead to an investigation. (But this is another part really)



### Blockchain verification

To ensure all votes have correctly been applied, the blockchain should make it's own calculations.

Basically, it should rebuild the Targets votes values from scratch, by applying each commit and compare the final result to the official result. If it totally match for every Target, then it means we have a 100% validation result.
If it doesn't, then it shall calculate the differential between the official results and the blockchain results in order to know how much the difference is. 

The votes in the queue shall not be taken into account, since they haven't been applied yet to the official results.

---

# Blockchain in depths _(Source: followmyvote.com)_

## Definition of Blockchain

> A blockchain is an audit trail for a database which is managed by a network of computers where no single computer is responsible for storing or maintaining the database, and any computer may enter or leave this network at any time without jeopardizing the integrity or availability of the database. Any computer can rebuild the database from scratch by downloading the blockchain and processing the audit trail.

## Motivations

> Traditional databases are maintained by a single organization, and that organization has complete control of the database, including the ability to tamper with the stored data, to censor otherwise valid changes to the data, or to add data fraudulently. For most use cases, this is not a problem since the organization which maintains the database does so for its own benefit, and therefore has no motive to falsify the database’s contents; however, there are other use cases, such as a financial network, where the data being stored is too sensitive and the motive to manipulate it is too enticing to allow any single organization to have total control over the database. Even if it could be guaranteed that the responsible organization would never enact a fraudulent change to the database (an assumption which, for many people, is already too much to ask), there is still the possibility that a hacker could break in and manipulate the database to their own ends.
> The most obvious way to ensure that no single entity can manipulate the database is to make the database public, and allow anyone to store a redundant copy of the database. In this way, everyone can be assured that their copy of the database is intact, simply by comparing it with everyone else’s. This is sufficient as long as the database is static; however, if changes must be made to the database after it has been distributed, a problem of consensus arises: which of the entities keeping a copy of the database decides which changes are allowed and what order those changes occurred in? If any of the entities can make changes at any time, the redundant copies of the database will quickly get out of sync, and there will be no consensus as to which copy is correct. If all of the entities agree on a certain one who makes changes first, and the others all copy from it, then that one has the power to censor changes it doesn’t like. Furthermore, if that one entity disappears, the database is stuck until all of the others can organize to choose a replacement. All of the entities may agree to take turns making changes and all the others copy changes from the one whose turn it is, but this opens the question of who decides who gets a turn when.

## Blockchain and online voting

> Another application for blockchain technology is voting. By casting votes as transactions, we can create a blockchain which keeps track of the tallies of the votes. This way, everyone can agree on the final count because they can count the votes themselves, and because of the blockchain audit trail, they can verify that no votes were changed or removed, and no illegitimate votes were added.

![Blockchain and online voting - followmyvote](https://followmyvote.com/wp-content/uploads/2015/06/Blockchain-Technology-Infographic-FMV.png)

---

# Why not use followmyvote directly

That is a possible implementation. But maybe we will want our tool not to be open source. Or maybe we'll want to fork it, or maybe it won't be good enough. A much deeper look into the source code is needed to take such decision.

---

# External links

- https://followmyvote.com/online-voting-technology/blockchain-technology/ (source of the quotes)
- http://www.forbes.com/sites/realspin/2016/08/30/block-the-vote-could-blockchain-technology-cybersecure-elections/#353b6abb2eca
- http://www.the-blockchain.com/2017/01/25/1880525-votes-cast-blockchain-largest-ever/
- https://followmyvote.com/ (good example of open-source and free implementation of blockchain)
- https://github.com/FollowMyVote (source code)
