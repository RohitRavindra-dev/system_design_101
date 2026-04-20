I need you to act like a senior/staff engineer who is helping me become a monster at hld and ace/strong hire for hld interviews and mentor/help me prepare for my system design/hld interviews with faang level companies.
i already spoke to another staff engineer and had him build me a starter list of {scenario,constraints} combinations that i should learn/do deep dives on. 
In this case the scenario is the high level problem we are trying to solve and the constraint is filter/limitation that we are introducing to limit our options.
The way he explained it to me, constraints are one of ordering, consistency, delivery semantics, latency, load, tolerance, scale limitations and real world systems often are made up on multiple sub problems where each sub problem is a base scenario + constraint 1, constraint2,... and knowing what are the pros and tradeoffs of solutions to a given scenario+constraint is what signals between a weak and strong candidate!

Now with this list in hand, i will share with you one scenario,constraint pair from his list and i want you to help me deep dive and thoroughly understand:
1. the base Scenario
-> give me the base scenario, when/why does this happen.
-> what assumptions this system normally relies on

2. the constraint:
-> explain the constraint in relation to the scenario. 
-> what does the constraint imply and when/why does it happen.
-> which assumptions this constraint breaks
-> where/which real world usecases does this constraint show up

3. Good Solutions (ranked best to average...skip all bad fit solutions or ones thats hard to defend/wrong thinking). for each i want you to:
-> explain the solution in depth
-> Why it works
-> Tradeoffs, gotchas
-> Failure modes at scale, what breaks first, when it breaks
-> what technologies (eg redis, kafka, etc) to make use of/propose for implementing the solution naturally.
-> What assumption are we restoring or relaxing?


4. Bad Solutions/anti patterns: these will be those average/poor solutions that i need to be aware of not proposing in the hld interview (maybe because its specious or hard to defend). for each of these please explain:
-> Why its tempting but not right
-> what would it take if one tried to force fit it
-> (basically my idea is, in case the interviewer asks, what if we did this (ie the bad solution) instead, i want to be able to answer its drawbacks and why the good solutions mentioned in point #3) outweigh)
Focus only on realistic interviewer traps (not obviously stupid solutions).
-> Why this fails under broken assumptions


5. Real-world mappings, examples the illustrate the scenario, constraint

6. what other constraints often pair with this current constraint. what complexity do those constraints add which of our solutions (seen above in sections 3,4 are now valid/invalid and why)
-> Which assumptions are now completely impossible to satisfy togethe

makes sense? 
can you help me build this knowledge base? Dont be jejune/handwavy, the aim i repeat again is to help me become a strong hire candidate for hld interviews!

NOTE: i understand that point 3, point 4 seem to have some overlap. so to make it a bit more clearer, let point 3 have the best 1-2 strongest solutions only and point 4 can have the remaining solutions which work but are hard to defend/difficult to fit/come with stronger cons than the pros. Explicitly eliminate invalid approaches based on constraints and mention them after all solutions in point 4!

To be more thorough, im thinking lets do each point (ie 1-6 above, ill be referring to them as sections 1-6 from now onwards) one at a time instead of dumping the entire solution at once. ill cross question, and try to deep dive on each section until i feel 100% locked in and then we can proceed to next section! We can follow a push (from my end) heavy approach instead of you endlessly pulling (by quizes, tangents) after/before each section (keep this minimal, instead teach me everything inside the section properly/correctly so that we don't need pulls/pushes even (ideally)). Optionally after each section you can suggest some followup questions/deepdive points (related to the section(s) seen already and if i feel necessary, ill ask as a followup, else prompt to move on to next section! )

At the end i want you to give me a .md/markdown file that illustrates our entire thread for reference and practice in the future.  

Okay, now to the main part. Todays {scenario, constraint} is: {Read heavy systems, Hot keys / skew (Some data is accessed disproportionately more. eg: celebrity posts, etc)}. 


-----------------------------------
constraint and scenarios can be fetched from scenarios.json and things_learnt.md (for constrain variations)
--------------------------------------

I think i understand section 5 well now, can you please give me everything you covered till now in this section (ie section 5) as a single click to download .md file(ie for section 5). make in comprehensive, almost a 1:1 copy verbatim to your answer and no trimming truncating or using handwavy/slogan like language. Include the q&a we did at the end of the section and optionally any learnings from this section at the end of the .md as well!
-----------
I think i understand section 3 well now, can you please give me everything you covered till now in this section (ie section 3) as a single click to download .md file(ie for section 3). make in comprehensive, almost a 1:1 copy verbatim to your answer and no trimming truncating or using handwavy/slogan like language. Include the questions and answers to all the optional prompts you suggested (short answers, not too detail nor too shallow) at the end of the .md as well please 

-----------------------------

lets first generate a full markdown cheat sheet with all 6 sections. i want it to be comprehensive, not slogan like/handwavy, include examples, analogies and core concepts from every section. don't be parsimonious with explaining stuff, this is gods work we have done in this thread, lets leverage this completely before proceeding. give me the .md file as a click to download file please. make it as elaborate/comprehensive as possible, i don't want to reprompt you after you generate a weak/disappointing v1