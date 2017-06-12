# Lottery-driven verification

**Summary** The idea of lotteries on Ethereum as desribed [here](http://www.golemproject.net/doc/GolemNanopayments.pdf) is reused with a different purpose.
Instead of aiming to cut gas cost related to payments, it is used to simplify validation and strengthen the guarantees of Provider's work being correct.

This builds on previous results: [lotteries](http://www.golemproject.net/doc/GolemNanopayments.pdf), [atomic swap](https://github.com/imapp-pl/golem_rd/wiki/Atomic-swap) and remaining work reshuffling.

## Background

Up until now, the main idea for subtask result verification was using redundancy: 
Some portion of the subtasks would be calculated by multiple Providers and their results compared (hashes of).
Discrepancies between results of two such Providers would provide indication towards at least one incorrect result.
Likewise, identical results (hashes of) would indicate correct results.

We suspect that such construct may lead to emergence of "subtask result black market", where Providers would share results of redundant subtasks.
This seems natural, as Providers will favor every solution, which guarantees more effective payout.
Existence of such black market would lead this perceived security of having redundancy to vanish.
Note that Providers will utilize smart-contracts to make their black market robust and "secure".

Also, the Requestor-driven redundancy forced upon the Providers isn't a free mechanism of validation - Providers net it into their pricing.
It makes more sense for the Requestor to do the redundant work himself, instead of paying to have it done.

Note that we are not rejecting redundancy per se, just arguing that attempts to _force_ redundancy by the Requestor may be easily thwarted.
Redundancy might be a viable mechanism, only if it's part of a winning strategy of the Providers, i.e. they "want" to do redundant calculations on their own.

Also note there are [ideas](https://github.com/imapp-pl/golem_rd/wiki/Collusion-problem) to make the black market impossible, namely making the subtask input a toxic secret, which cannot be disclosed without risking being slashed, but implementing these might be hard.

## Key points

1. Provider never gets paid for bad work: Provider that does 100% incorrect work (delivers junk), has 100% chance of getting 0 reward
1. Provider that does 100% correct work, still gets his expected payout in the long run
1. Adding junk into a batch of correct results never increases the expected payout
2. The only redundancy that works from the Requestor's perspective, is the redundancy done by the Reguestor
3. All agents are selfish and rational, i.e. they seek to optimize their net gain.

   for now assume no agent want's to *spend* to hurt the network (no grieving)

## Flow

This is the general flow of the happy path, see below for handling of unhappy paths

1. R publishes task, along with commitment to division into subtasks, initial price and number of winning tickets
2. R provides `sha3(lottery_secret)` and provides timelocked GNT for payout+cushion
3. Providers list is compiled
4. `lottery_seed = sha3(lottery_secret | future_blockhash)`, determines lottery winners. When future block is mined, Requestor knows the winning tickets
5. `assignment_seed = sha3(sha3(lottery_secret) | future_blockhash)` determines assignment of subtasks
7. R calculates results of S1, S2, ... lottery winning subtasks
8. P1... PN commit to their results, revealing hashes of results and cipher texts. This stage includes possible several rounds with reshufling (see below)
9. If no wrong hashes are detected by R, R says so publicly. Otherwise - see subflow below
11. P1...PN reveal KDF-secrets, as in the *atomic swap* mechanism (all of them)
12. R reveals `lottery_secret`, payout happens (otherwise slash and even split of payout+cushion for all Ps, cushion covers larger gas cost)

#### Subflow - bad result hash

1. If above "no wrong hashes are detected by R" doesn't hold - let's say subtask S1 yields a different result for R.
2. R publishes a reject statement, with S1, hash(R-solution), and proof that S1 belonged to P1
3. P1 is dropped from providers list, all of his subtasks are rejected
4. new `lottery_seed2` is produced as in main flow, detemines new winning tickets within _all_ subtasks, equal to the number of rejected winning tickets from P1
    - some of already computed subtasks might be "granted" a winning ticket
    - none but P1 can be deprived of already granted winning tickets

#### Subflow - price bumping

1. After P1 has started work on assigned subtasks, P1 might come to a conclusion, the task is hard/underpriced, i.e. P1 won't get satisfactory payout for the effort
2. P1 notifies R that he's not willing to continue, unless price is raised. This is done publicly
3. If R accepts by setting a bumped price X, he updates the amount of GNT stashed for payout+cushion
4. All Ps with offers higher than X are dropped from provider's list, their subtasks and tickets get rejected. R needs to prove that rejected offers were above X.

Price bumping may happen only once per task, no later than a predetermined moment.
This encourages Ps to send their notifications ASAP and make their best effort to estimate difficulty and manage risk.
    
#### Subflow - reshuffling

This is a slightly modified flow from the previous "Remaining work reshuffling" doc.

1. Initial assignment of subtasks is done like this: `assignment_seed` shuffles the subtasks by determining the assignment matrix `M1`, which assigns equal count of subtasks to `P1`, ..., `Pn` (**NOTE** order matters!), e.g.:

    |           | P1 | P2 | P3 | P4 |
    |-----------|----|----|----|----|
    | subtasks: | 4  | 11 | 12 | 1  |
    |           | 5  | 6  | 7  | 15 |
    |           | 2  | 8  | 14 | 16 |
    |           | 10 | 9  | 3  | 13 |

    (Assuming 4 providers and 16 subtasks)
5. `P1`, ..., `Pn` compute the tasks as fast as possible to get the maximum lottery tickets.
6. First to complete his pool of subtasks, publishes their results' merkle root and by that starts a challenge.
Challenge means other providers must likewise publish their roots, committing to their obtained results.
Those other providers have limited time for this.
Note here that only the results of given provider's _first-most_ subtasks can be published and are payable -- order matters.
7. All subtasks that have not been published this way are brought into a common pool, reshuffled using some `assignment_seed2`, and assigned using matrix `M2`.
If after first challenged `P1`, ..., `P4` published the results for 1, 4, 1, 2 of their subtasks respectively, i.e.:

    |           | P1 | P2 | P3 | P4 |
    |-----------|----|----|----|----|
    | results:  | 4  | 11 | 12 | 1  |
    |           |    | 6  |    | 15 |
    |           |    | 8  |    |    |
    |           |    | 9  |    |    |

    Then the `M2` is:

    |           | P1 | P2 | P3 | P4 |
    |-----------|----|----|----|----|
    | subtasks: | 7  | 5  | 16 | 2  |
    |           | 3  | 10 | 3  | 13 |

    Eight subtasks carried on to round 2, reshuffled and distributed uniformly.

8. Lather, rinse, repeat

Note that the pool of reshuffled subtasks includes rejected tasks, either on grounds of price or verification failure
    
## Q&A

1.  Q: Why R needs to reveal his solution of a rejected duplicate subtask?

    A: Other participants can then recalculate the subtask. Knowing who was "right" in the dispute, they apply that to their reputation system. E.g. if R's rejection is unjust, R's reputation suffers

3.  Q: Why R can't assign and order subtasks himself or set winning tickets/duplicates himself??

    A: He could deal winning subtasks to his P*. It needs to be random and not manipulable. At the same time, R needs to know winning tickets to calculate these subtasks himself

4.  Q: Why is the KDFreveal/payout done in all-at-once fashion?

    A: In order to get non-winning results, R must pay the winning ones. This requirement guarantees that.

5.  Q: Some Provider P delivers X% bad results, what then?

    A: Doing X% bad work doesn't increase payout, only jeopardizes P's (100-X)% good results. Other words: giving extra bad results doesn't increase payout, it only increases chance of getting controlled
    
5.  Q: In particluar, what if only P1 signs up for a task. P1 can then only calculate X% right and get X% of payout on average

    A: less than X%. If he get's caught, all his results are rejected, not only the bad ones. X%-cheat strategy breaks even at verification of 1 subtask (!). If >1 subtask is verified, cheating is inferior to cooperation
  
6.  Q: R might reject the P's with winning ticket subtasks to maneouver these winning tickets into R's colluding P*'s pocket

    A: 1) might end up having only P* deliver results, i.e. R calculates the task himself. Probability that there's substantial amount of never-winning, working Ps is low
       3) working Ps will check the grounds of rejection. Unjust rejection would make them stop contributing
       4) with huge damage to R's reputation

7.  Q: Why rejected winning tickets can go to already commited-to tasks?

    A: 1) to have the chance of winning distributed uniformly across subtasks, regardless of subtask's history
       2) to provide a small incentive for Providers to check their peers (ones they suspect are cheating) and report to R. Such Provider-driven redundancy is incentivized by chances of getting extra winning-tickets
       3) to prevent R from maneouvering winning tickets to R's pocket

8.  Q: P1 (or his Sybils) might push bad results to hasten overal solution, when better paid work is to be done

    A: P1 won't do that, as its better to just commit to correct task and do the more expensive work. Besides that better paid work might be also more difficult, so it requires some involvment on behalf of P1 first.
    
9.  Q: R retries task creation many times until his P*s get the winning tickets without having to do much work

    A: spends gas on Task broadcasts, wastes GNT deposited for payout+cushion

8. Q: Why is it that the winning ticket subtasks are verified, not others?

    A: Suppose lottery functions normally, but R picks verified subtasks himself - what breaks then?
    Then P1 can get paid for doing no work. The winning ticket subtasks is the narrowest set of subtasks R needs to verify to guarantee "0 payout for junk" rule
    
    TODO: Does this need to be forced on R?
        - 3 options: R is free to choose, R _must_ verify winning-tickets (favourite), R _must_ verify _only_ winning-tickets
        
11. Q: Why doesn't R reveal his `lottery_secret`? He could sell this knowledge to P*, allowing P* to optimize his payout unfairly
    
    A: that would render R's assurance of correctness void.
    
    TODO: not 100% sure about this, rethink! In case it's not enough guarantee, revealing the `lottery_secret` might slash the R's deposit
    
    See also Known Problems, this is sever if P*'s are R's Sybil identities

12. Q: Why can't only winning ticket subtasks by assigned randomly to Providers, and the rest of assignment be R-driven
    
    A: R would then assign _easiest_ tasks to colluding P*. Assignment randomness has its own merit
    
14. Q: doesn't the protocol rely too much on the Reputation system?

    A: The reputation system is mentioned in the protocol, it is however auxiliary to the security mechanisms. The execution of the protocol provides a data feed for the rep system to base on, because it makes a trace of rejected computations be left behind
    
15. Q: What if some P*s collude and request price bumping at the end of tasks lifetime, taking results hostage?

    A: Taking results hostage can happen always anyway. In this approach P*s risk their work being expropriated and given to other nodes. Also opportunity to request bumping is limited.
    
16. Q: Can't price-based P-rejection be used to maneouver winning tickets into R's hands?

    A: No, Requestor needs to prove P's price offer was higher than the new price threshold set after bumping. All P's over the threshold are dropped

17. Q: What if R can't calculate even a single redundant subtask (R is a fridge)?

    A: There would need to be a mechanism allowing R to connect to a slave Golem node fully in R's control. Bit similar to a "full node & light client" interaction
    
### Known problems

1. R can put some own P\*s on his task, and only make them deliver anything if P\* gets an excesive amount of winning-tickets. These winning-tickets should appear on the front of P* assigned subtasks to be cheap to reach. When successful, this optimizes R's cost unfairly, as non-colluding Ps get less than expected in the long run. This is an open threat, but:
    - on large scale this requires an army of P* Sybil identities, with effective anti-sybil protection it would be hard to pull off
    - those winning-tickets were also a correctness guarantee. If R deals winning-tickets to himself, he loses assurance. Argument applies only if "R _must_ verify _only_ winning-tickets"
    - mitigation1: 2-tiered lottery, Requestor knows only tier-1 (verified tasks). From within tier-1 tickets, paying tickets are drawn without R's knowledge
    - mitigation2: winning-tickets become known to R & verification starts after all (50%? 75%?) tasks are completed, after the laggards had been dropped
    
    
    
    
    
    
    
    
    
    
    