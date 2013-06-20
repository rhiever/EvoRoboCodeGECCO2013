#RoboNucleicAcid: A platform to evolve RoboCode bots
###[Mike Worth](http://www.mike-worth.com), University of Oxford
###[Randal S. Olson](http://www.randalolson.com), Michigan State University

In this document, we briefly describe the RoboNucleicAcid platform that we developed in Java to evolve RoboCode bots for the EvoRoboCode competition at GECCO '13. The RoboNucleicAcid platform is composed of two core components: (1) a genetic algorithm that evolves byte strings that can be transcribed into RoboCode bot behaviour, and (2) an interface that passes the evolved RoboCode bot behaviours to the RoboCode platform for fitness evaluation, then receives the fitness scores of the bots back to use in the genetic algorithm. Below, we describe the components of the genetic algorithm that we implemented to make it possible to evolve RoboCode bots.

The RoboNucleicAcid code is maintained online at the [RoboNucleicAcid Github repository](https://www.github.com/MikeWorth/RoboNucleicAcid).

##Gene encoding
Each bot has a genome that is composed of 9 genes, one for each of the AdvancedBot event methods `onScannedbot`, `onHitByBullet`, `onHitBot`, `onHitWall`, `onBulletHitBullet`, `onBulletHit`, `onBulletMissed`. The last two genes describe the `run` method, the first gene describing the behaviour at the start of a battle and the second gene describing the behaviour in the `while(true)` loop. As an abstraction of biological genomes, each gene is composed of a string of hexadecimal numbers. Storing the genome as a string rather than an array of bytes allows for mutations that cause frame shift and change values by reversing the order of nybbles.

The bytes themselves are transcribed into either a RoboCode bot instruction (e.g., `fire()` or `forward()`), a numerical value or junk RNA. Most of the AdvancedBot control methods are encoded by a specific byte value. Additionally, the instructions that require a parameter (such as `fire()`) are provided the value of a register that is maintained through the program. The value of this register can be assigned explicitly or altered by simple mathematical operations encoded in the genome. Further, either of these operations can be used in conjunction with event values, e.g., the register can be assigned to the current bearing of the bot, or multiplied by the energy of an enemy bot that has been scanned. There is also a conditional branching structure (also encoded in the genome) that compares the current register value with a value that is explicitly specified in the gene and can conditionally skip entire contiguous sections of instructions depending on the result of the comparison.

##Gene decoding
There is an EvolveBot class that contains all of the methods required to transcribe the previously described genomes into a RoboCode bot. Just before each battle, an EvolveBot reads the genome from a text file that is saved into its data directory. In order to have a whole generation of different bots at once, there are 50 bots (EB0-EB49) that inherit everything from EvolveBot but by virtue of their different name have different data directories. These data files consist of a generated name on the first line followed by the byte strings composing each gene on separate lines. In addition to these genomes, the winner of each generation has its genome archived for future reference. If a bot is saved or archived during an evolutionary run, we also transcribe the bot's genome into its corresponding Java code. The bot we included in this submission is thus a result of transcribing one of the best EvolveBots from an evolutionary run into its corresponding Java code.

##Crossover
Crossover is handled similar to biological sexual reproduction (although with no distinction between male and female, and not quite as messy): Once two bots have been selected to breed, a new offspring genome is constructed from the parents by choosing a "chop place" somewhere on each gene (aligned with the byte frames) and concatenating the bytes before this point from one parent and after from the other.

The process of selecting bots for breeding chooses a random bot weighted according to rankings as follows. In the case of `n` bots, the top ranked bot is `n` times more likely to be picked than the bottom bot, the second ranked bot is `n-1` times more likely and so on. The first parent is simply picked according to the weighting, while the second parent will be rejected and replaced if the pair are too closely related. We determine relatedness between two bots by looking at their "cousinality," e.g., for siblings it is 0, and for `nth` cousins it is `n`. If this number is below a particular threshold (for 50 bots <=1 works well), the pairing is rejected.

##Mutations
There is a probability for a variety of mutations to occur in the breeding process, which are split into byte scale and large scale mutations:

###byte scale mutations
* byte is deleted
* new byte (with a random value) is added
* byte value is changed; a random number from a Gaussian distribution and standard deviation of 20 is added

###large scale mutations
* the two pieces from the parents are concatenated in the wrong order
* the piece from a parent can be taken from the wrong gene
* a region can be deleted
* a region can have the order of nybbles reversed
* a region can be moved to another place in the same byte
* a region can be duplicated

##Fitness evaluation
There are two options for battles, yardstick battles and inter-challenger battles. For most of the evolutionary runs so far, we have only used yardstick battles. With yardstick battles, at the beginning of each evolutionary run, we define one or more pre-coded bots to use as yardstick bots, e.g., sample.SpinBot. Each EvolveBot fights each of the yardstick bots in a 1v1 battle consisting of several rounds (typically 3-10). For inter-challenger battles, all of the 50 bots in a population for a given generation enter into 1v1 battles with every other bot in that the population at that generation.

We calculate the fitness for the bot by calculating a value based off of the bot's score. Specifically, we define the fitness from the battle(s), `fitness_challenger`, as

`fitness_challenger` = (`challenger_score` + 1.0) / (`sum_of_both_bots_scores` + 1.0)

, where `challenger_score` is the score of the bot being evaluated and `sum_of_both_bots_scores` is the sum of the scores by the bot being evaluated *and* the opponent bot. If the bot competes against multiple bots in a generation (in separate battles), we define the bot's fitness as the average of `fitness_challenger` from all battles. We add `1.0` to the numerator and denominator to make reducing the opponent's score advantageous even if a bot doesn't score any points itself. After all of the bots have been evaluated, a ranked list of bots is returned. Although it is only this ranked list that is used for breeding, the average scores are reported in the log for the evolutionary run.

##Modifications for restricted code length
One requirement for the EvoRoboCode competition is that bots are restricted to the nanobot class. Since there is a near-linear relationship between the number of RNA instructions and what nanobot class bots look like, we decided that a nanobot class EvolveBot could have a maximum genome size of 500 nybbles. The first step to evolving bots with this size limit was to add a fitness penalty to any genome larger than 500 nybbles. We implemented the fitness penalty by taking away `n%` of a bot's points when its genome was `n%` too large, e.g.. a genome of 750 nybbles would lose half of its scored points. A limit was set such that a bot more than twice the allowed genome length would score zero, rather than a negative score. While this technique showed some promise of evolving bots in the nanobot class, it appeared that only simple combat behaviours could evolve due to the restrictive maximum genome size.

In order to increase the evolvability of the bots (while still evolving bots that could be entered into the competition), we implemented periods of "feast" and "famine" for genome size. With this feature enabled, there would be feast periods near the beginning of the evolutionary runs, during which the maximum genome length was allowed to be much larger than the normal size limit, e.g., we increased the limit to 5,000. Additionally, there would be famine periods near the end of the evolutionary runs, where the maximum genome length was restricted back to 500. The transition from the initial feast period to the final famine period was linear and gradual, such that the most useful behaviours evolved during the feast period to survive the yardstick battles, while the less helpful behaviours and junk RNA were preferentially mutated away in the famine period.
