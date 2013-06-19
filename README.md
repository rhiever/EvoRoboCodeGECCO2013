#How the gene encoding works
A robot has a genome that is composed of 9 genes; one each for the methods onScannedRobot, onHitByBullet, onHitRobot, onHitWall, onBulletHitBullet, onBulletHit, onBulletMissed plus two that between them compose run (the first is used once at the start of a battle and the second is run inside a while(true) loop). In line with the biologically based style of the project, each gene is a string of hexadecimal numbers one after another; storing this as a string rather than an array of bytes allows for mutations that cause frame shift and change values by reversing the order of nybbles.

The bytes themselves refer to either an instruction, a value or junk rna; most of the advancedRobot methods are in the list of available instructions, the ones that require a parameter are handed the value of a register that is carried through the program. The value of this register can be set explicitly, altered by simple mathematical operations and either of these options can be used in conjunction with event values (e.g. the bearing or energy of a robot that has been scanned). There is also a conditional branching structure that compares the register value with on that is explicitly specified in the gene.

#How gene decoding works
There is an EvolveBot that has all the methods required to translate the previously described genomes; it reads the genome from a text file that is saved into its data directory. In order to have a whole generation of different bots at once, there are 50 robots (EB0-EB49) that simply inherit everything from EvolveBot but by virtue of their different name have a different data directory. These files simply consist of a generated name on the first line, followed by each gene on a separate line; in addition to these fighting genomes, the winner of each generation has its genome archived for future reference along with the same behaviour transcribed into java.

#How breeding works
Breeding is handled largely in the same way as biological sexual reproduction (although with no distinction between male and female); once two robots have been selected to breed, a new genome is constructed from the parents by choosing a "chop place" somewhere on each gene (aligned with the byte frames) and concatenating the bytes before this point from one parent and after from the other.

The process of selecting robots for breeding takes a random bot weighted according to rankings as follows: in the case of n bots, the top ranked bot is n times more likely to be picked than the bottom, the second ranked bot is n-1 times more likely and so on. The first bot is simply picked according to the weighting, while the second bot will be rejected and a replacement found if the pairing is too incestuos; any two robots have a "cousinality" (for siblings it is 0, and for nth cousins it's n) and if this number is below a particular threasold (for 50 bots <=1 works well) then the pairing is rejected.

There is a possibility for a variety of mutations to occur in the breeding process which are split into byte scale and large scale types:
##byte scale
* byte is deleted
* new byte (with a random value) is added
* byte value is changed; a random number with a Gaussian distribution and standard deviation of 20 (this is configurable) is added (it can be positive or negative)
##large scale
* the two pieces from the parents are concatenated in the wrong order
* the piece from a parent can be taken from the wrong gene
* a region can be deleted
* a region can have the order of nybbles reversed
* a region can be moved to another place in the same byte
* a region can be repeated

#How fitness is determined
There are two options for battles, yardstick battles and inter-challenger battles; for most of the runs so far, only yardstick battles have been used. A number of yardstick bots are named and each EB fights each of the yardstick bots in a one on one battle consisting of several rounds (this typically varied around 3-10). The scores from these battles are challenger_score+1/(sum_of_scores + 1) __randy: this probably ought to be foirmatted nicer__ to give a modified percentage; adding one to the top and bottom is does to make reducing the opponents score advantageous even if a bot doesn't score any of its own points. These points are averaged and a ranked list of bots is returned; although it is only this ranked list that is used for breeding, these average scores are reported in the log as percentages.

#Modifications for restricted code length
One unusual requirement for the evorobocode competition was that robots were restricted to the nanobot class; with the quite close relationship between rna instructions and what I suspect complied robots look like, it was decided to equate this to a genome limit of ~500 nybbles. The first step to evolving bots with this size was to add a fitness penalty to any genome larger than this; this was achieved by taking away n% of a robots points when its genome was n% too large (i.e. a genome of 750 nybbles would lose half its scored points). A limit was set that a robot more than twice the required length would score zero, rather than a negative score. While this showed some promise, it appeared that only the simplest of behaviours could evolve as there was no room to manuevure.

In order to provide a more forgiving environment for behaviours to develop, while still generating bots that could be entered into the competition, the idea of feast/famine periods was introduced. This was a system whereby there would be feast periods, during which the genome length was allowed to be much larger (a value of 5000 seemed to work), and famine periods, where length was restriced to 500. The transition between these two regemes was linear and gradual; this allows the most useful behaviours evolved during feast to survive, while less helpful behaviours and junk rna to be preferentially stripped.
