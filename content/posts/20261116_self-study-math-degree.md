---
title: "How to Self-Study a Math Degree"
date: 2026-11-16
draft: true
description: "A practical plan for working through the equivalent of an undergraduate mathematics degree on your own: what a math degree actually contains, two curricula you can follow (OSSU and a textbook path), how many hours it really takes, how to study proof-based maths without a teacher, and how to turn it into a real credential if you want one."
summary: "Everything in an undergraduate maths degree is available for free online: lectures, problem sets, even whole textbooks. What's missing is structure, feedback and a deadline. This post lays out what a degree covers, two concrete curricula, an honest hours estimate, study habits that work for proof-based maths, and part-time distance degrees for when you want the paper too."
tags: ["mathematics", "self-study", "learning", "curriculum", "ossu", "mit-ocw", "proofs", "lean"]
categories: ["Mathematics"]
---

Every piece of an undergraduate mathematics degree is available for free on the internet. MIT publishes its lectures, problem sets and exams. Excellent textbooks are open access. There are communities of people working through the same material who will answer your questions.

And yet most people who start "teaching themselves maths" stall somewhere around the second half of calculus. The material isn't the problem. What's missing is what a university actually sells: a **sequence**, **feedback**, and **deadlines**.

This post is a plan for supplying those three things yourself. It covers:

- what a maths degree actually contains, and in what order;
- two concrete curricula you can follow: the OSSU Math curriculum, and a textbook-based path;
- an honest estimate of the hours involved;
- how to study proof-based maths without a teacher, which is the hard part;
- how to get a recognised degree part-time if you decide you want the credential as well.

## First decide what you want

"Self-study a math degree" can mean three quite different goals, and they lead to different plans.

| Goal | What you need | What you can skip |
| --- | --- | --- |
| **Understanding**: you want to know the mathematics a graduate knows | A curriculum, good books, and a way to check your proofs | Exams, credits, fees |
| **A specific application**: ML, quantitative finance, physics, cryptography | The core plus the relevant electives, done to a depth that lets you read papers | Topics far from your field (some of algebra or topology, say) |
| **A credential**: for a job, a visa, or a master's/PhD application | An accredited programme. Free material can make it much easier, but it can't replace it | Nothing. The institution decides the syllabus |

It's worth being honest about this up front. Free course material gives you the knowledge, but not the paper: MIT OpenCourseWare states plainly that you cannot receive a degree, certificate, or any credit for studying its courses[^1]. If you'll eventually need the degree, it may be cheaper overall to enrol in a part-time programme early and use the free material as support. There's a section on that near the end.

The rest of this post assumes the first goal (understanding), which is also the foundation for the other two.

## What a maths degree actually contains

Undergraduate maths programmes differ in the details, but they share a skeleton. It has roughly four layers.

**1. Computational foundations.** Single and multivariable calculus, an introduction to linear algebra (matrices, determinants, eigenvalues), and ordinary differential equations. These courses are mostly about *calculating*, and engineers and physicists take them too.

**2. The transition to proof.** A course on logic, sets, functions, induction and proof techniques. Usually discrete maths (combinatorics, graphs, number theory) is taught here too, because it's a good playground for proofs. This is the single most important turning point in the degree: everything after it is about proving things, not computing them.

**3. The proof-based core.** Every maths graduate has done these:

- **Real analysis**: limits, continuity, derivatives and integrals rebuilt rigorously from the axioms of the real numbers.
- **Abstract algebra**: groups, rings and fields.
- **Linear algebra, second time**: vector spaces and linear maps, proof-based.
- **Probability** (and usually statistics), built on calculus.

**4. Breadth and depth.** Electives such as complex analysis, topology, number theory, differential geometry, mathematical logic, numerical analysis, PDEs, measure theory, combinatorics. A degree usually requires some breadth across these plus a deeper specialisation in one area, often ending with a project or thesis.

The order matters more than in most subjects. Analysis needs comfort with proofs. Probability needs multivariable calculus and linear algebra. Topology needs analysis and set theory. Skipping ahead is the most common way self-learners get stuck.

## Curriculum 1: OSSU Math

The Open Source Society University (OSSU) maintains a free, community-curated maths curriculum on GitHub. It splits into **Core Mathematics**, which is all required, and **Advanced Topics**, which are electives[^2]. Most of the core is MIT material.

Here's the core, with the durations and weekly effort OSSU lists[^2]:

| Course | Source | Duration | Effort | Prerequisites |
| --- | --- | --- | --- | --- |
| Introduction to Mathematical Thinking | Coursera | 10 weeks | 4 h/week | None |
| LaTeX | Overleaf | 1 week | 0.5 h/week | None |
| Calculus 1A: Differentiation | MIT Open Learning Library | 13 weeks | 6–10 h/week | High-school maths |
| Calculus 1B: Integration | MIT Open Learning Library | 13 weeks | 5–10 h/week | Calculus 1A |
| Calculus 1C: Coordinate Systems & Infinite Series | MIT Open Learning Library | 6 weeks | 5–10 h/week | Calculus 1B |
| Multivariable Calculus | MIT OCW | 12 weeks | 6 h/week | Calculus 1C |
| Differential Equations | MIT OCW | 14 weeks | 12 h/week | Calculus 1C |
| Mathematics for Computer Science | MIT OCW | 14 weeks | 6–8 h/week | Calculus 1C |
| Essence of Linear Algebra | YouTube | — | — | High-school maths |
| Linear Algebra | MIT OCW | 14 weeks | 12 h/week | Essence of Linear Algebra |
| Probability | MIT OCW | 14 weeks | 12–16 h/week | Multivariable Calc, Math for CS, Linear Algebra |
| Statistics for Applications | MIT OCW | 14 weeks | 12–16 h/week | Probability |
| Introduction to Analysis | MIT OCW | 14 weeks | 8–10 h/week | Multivariable Calculus |
| Introduction to Abstract Group Theory | NPTEL | 8 weeks | 8–10 h/week | High-school maths |
| Introduction to Rings and Fields | NPTEL | 8 weeks | 8–10 h/week | Group Theory |

For the electives, OSSU's rule is **breadth plus depth**: take one course from each of the six advanced topic areas, then pick one area and take three more courses in it[^2]. The six areas are mathematical logic, geometry and topology, differential equations, probability and statistics, mathematical analysis, and abstract algebra. Some areas have specific course recommendations (forall x for logic, *Topology Without Tears*, MIT's differential geometry), while others only list a scope, such as "real analysis, numerical analysis, complex analysis, optimization theory", and leave the choice of resource to you[^2].

OSSU claims that finishing the curriculum is the equivalent of a full bachelor's degree in mathematics[^2]. That's a fair claim in terms of topics covered. In terms of depth, it depends entirely on how many of the problem sets you actually do.

**Strengths:** free, ordered by prerequisite, mostly lecture-based (good if you learn from video), with a Discord and a Trello board for tracking progress.

**Weaknesses:** the core is weighted toward applied and computational material (three calculus courses, ODEs, statistics) and relatively light on proof-based pure maths: one analysis course and two short algebra courses. If you want a pure maths degree, add more analysis and algebra from the textbook path below.

## Curriculum 2: a textbook path

Many mathematicians will tell you that maths is learned from books and exercises, and lectures are optional. A textbook path is slower to get started but usually goes deeper. Here's a sequence built on well-regarded books, several of them free.

| Stage | Subject | Book(s) |
| --- | --- | --- |
| 0 | Proofs and logic | Richard Hammack, *Book of Proof* (free online)[^3]; or Velleman, *How to Prove It* |
| 1 | Calculus | Stewart or Thomas for computation; Spivak, *Calculus*, if you want a proof-heavy first pass |
| 1 | Linear algebra (computational) | Strang, *Introduction to Linear Algebra* (pairs with MIT 18.06 lectures) |
| 2 | Discrete maths | Lehman, Leighton & Meyer, *Mathematics for Computer Science* (the MIT 6.042 text) |
| 2 | Linear algebra (proof-based) | Sheldon Axler, *Linear Algebra Done Right*, 4th ed. (open access)[^4] |
| 3 | Real analysis | Abbott, *Understanding Analysis*; then Rudin, *Principles of Mathematical Analysis* if you want more |
| 3 | Abstract algebra | Pinter, *A Book of Abstract Algebra* (gentle); then Dummit & Foote as a reference |
| 3 | Probability | Blitzstein & Hwang, *Introduction to Probability* (pairs with Harvard Stat 110) |
| 4 | Complex analysis | Needham, *Visual Complex Analysis*; or Stein & Shakarchi |
| 4 | Topology | Munkres, *Topology* |
| 4 | Measure theory | Axler, *Measure, Integration & Real Analysis* (also open access)[^4] |
| 4 | Number theory, logic, geometry… | Pick by interest |

Axler's *Linear Algebra Done Right* deserves special mention. It's one of the standard second courses in linear algebra, written for maths majors and graduate students, and the fourth edition is free to download from Springer as an open-access book[^4]. The new edition adds over 250 exercises and 70 examples[^5].

**Hybrid approach.** The two curricula combine well. Use OSSU for the computational foundations, where MIT lectures shine, then switch to books for stages 2 to 4, using lecture videos (MIT OCW, YouTube) only where you get stuck.

## How long does it take?

Let's do the arithmetic. Taking the middle of OSSU's effort range for each core course and multiplying by its duration gives roughly **1,450 hours** for the core. The nine electives (six for breadth, three for depth) at around 150 hours each add another **1,350 hours** or so. Total: about **2,800 hours** of focused work.

For comparison, a European bachelor's degree is usually 180 ECTS credits, and one ECTS credit is defined as 25 to 30 hours of student work. That puts a real degree at **4,500 to 5,400 hours**, including exams, a thesis, and some non-maths modules. So the OSSU estimate is optimistic, and that's before counting the time you'll spend stuck, revisiting prerequisites, or redoing problems. A realistic figure for genuinely learning the material is **3,000 to 4,000 hours**.

What that means in calendar time:

| Weekly hours | Hours per year (48 weeks) | Years for ~3,500 hours |
| --- | --- | --- |
| 5 | 240 | ~15 |
| 10 | 480 | ~7 |
| 15 | 720 | ~5 |
| 20 | 960 | ~3.5 |
| 40 (full-time) | 1,920 | ~2 |

If you're working full-time, **10 to 15 hours a week over 5 to 7 years** is realistic. That's in line with part-time degrees: the Open University's part-time maths degrees, for example, take about six years at 60 credits a year[^6].

That's a long time, which is the main argument for having a clear plan. It also argues for front-loading the material that pays off soonest for your goal.

### A sample plan at ~12 hours a week

| Year | Focus |
| --- | --- |
| 1 | Proofs (Hammack or Intro to Mathematical Thinking), LaTeX, single-variable calculus |
| 2 | Multivariable calculus, computational linear algebra, discrete maths |
| 3 | Proof-based linear algebra (Axler), real analysis part 1, ODEs |
| 4 | Real analysis part 2, abstract algebra, probability |
| 5 | Statistics, complex analysis, topology |
| 6 | Depth electives in your chosen area, plus a capstone project |

Notice that proofs come **first**, before calculus. Most curricula put them later because schools teach computation first, but if you're self-studying there's no reason to wait. Being comfortable with proofs early makes every later course easier, including calculus. (If you want a gentle start, my earlier post on mathematical proofs is built around a single example: why every four-digit palindrome is divisible by 11.)

## How to study proof-based maths alone

This is where self-study succeeds or fails. Watching lectures feels productive but builds very little. What builds mathematical ability is **solving problems and writing proofs**. Some habits that help:

### Do the exercises. Most of them.

A rough rule: for every hour of reading, spend two or three hours on exercises. If a book has 40 exercises per chapter, you don't need all 40, but you need enough that the chapter's ideas feel like tools rather than facts. If you can't solve any of the exercises, you haven't understood the chapter yet, however clear it seemed while reading.

### Read actively

Read with a pencil. Before reading a proof, cover it and try to prove the theorem yourself for five or ten minutes. Then read the proof and compare. For each definition, write down an example and a non-example. For each theorem, ask what happens if you drop a hypothesis.

### Solve the feedback problem

This is the biggest disadvantage of self-study. At university, someone marks your proofs. Alone, you can write a proof with a gap in it and never find out. Some ways to get feedback:

- **Solution manuals and solved problem sets.** MIT OCW courses often include solutions. Compare your argument line by line, not just the final answer.
- **Proof assistants.** Writing a proof in Lean gives you a strict, tireless marker: if there's a gap, it won't compile. The *Natural Number Game* is a fun introduction, and *Mathematics in Lean* covers undergraduate material. It's too slow to formalise everything, but formalising a few proofs per course sharpens your sense of what counts as a complete argument.
- **Math Stack Exchange.** Post your proof with the tag *proof-verification* (now called *solution-verification*) and ask specifically what's wrong.
- **Study partners.** The OSSU Discord, local maths meetups, or a friend working through the same book. Explaining a proof to someone else is the fastest way to find the holes in it.
- **LLMs, carefully.** They can be useful for asking "is this step justified?" or for getting a hint without the full solution. But they also confidently approve wrong proofs and produce plausible-looking wrong ones. Treat them as a study partner, not as a marker, and verify everything they say.

### Write in LaTeX

Write up your solutions properly, at least for the important exercises. It forces you to finish your arguments, gives you a record to come back to, and LaTeX is a skill you'll need anyway. OSSU includes a short Overleaf course for exactly this reason[^2].

### Use spaced repetition for definitions and theorems

Analysis and algebra introduce a lot of definitions, and you can't prove anything about a "normal subgroup" if you have to look up what one is. Flashcards (Anki or similar) for **definitions and theorem statements** work well. Don't use them for proofs. Proofs should be understood and reconstructed, not memorised.

### Make your own deadlines

Without exams, it's easy to drift. Some things that help:

- Follow a real course's schedule. Pick an MIT OCW course and do its problem sets in the weeks it assigned them.
- Take the past exams under exam conditions: timed, closed book.
- Track your progress publicly: a blog, a GitHub repo of solutions, the OSSU Trello board.
- Fix your weekly hours in your calendar, like a class you've paid for.

## Common pitfalls

- **Watching instead of doing.** Ten hours of lectures with no problems builds far less than three hours of lectures and seven of problems.
- **Endless prerequisite review.** You don't need to master all of calculus before starting proofs, or all of algebra before analysis. Go forward and backfill when you hit a gap.
- **Collecting resources instead of using them.** Choose one main book per subject and finish it. Use others only as references.
- **Skipping the transition to proof.** If analysis feels impossible, the problem is almost always proofs, not analysis. Go back to Hammack or Velleman.
- **Perfectionism.** You won't do every exercise, and you'll forget things. Real graduates forget a lot too. What sticks is the way of thinking.
- **Going silent.** Self-learners who never discuss maths with anyone tend to stall. Find at least one person or community.

## When you want the credential too

If you decide you want an actual degree, there are good part-time distance programmes designed for working adults, and your self-study will make them much easier. Two examples:

- **The Open University (UK).** Its BSc (Honours) Mathematics (Q31) covers pure and applied maths to graduate level by distance learning[^7]. Its part-time degrees take about six years at 60 credits a year, can be done full-time in three, and have a time limit of 16 years[^6].
- **FernUniversität in Hagen (Germany).** A state distance university offering a 180-ECTS Bachelor of Science in Mathematics, six semesters full-time and correspondingly longer part-time, with starts in both the summer and winter semesters[^8]. Total fees are around €2,100 for full-time study, plus a small fee for each additional semester part-time[^8]. Teaching is in German.

If you already have some university credits, both may count prior study toward the degree[^6]. Many conventional universities also offer part-time or evening study, and a few accept strong self-taught applicants into master's programmes if they can show the background, usually through entrance exams or by taking a few undergraduate modules first.

## Bottom line

- **All the material is free.** What you have to supply yourself is the sequence, the feedback, and the deadlines.
- **Know your goal.** Understanding, an application area, or a credential each need a different plan. Free courses give knowledge but no credit.
- **A maths degree has four layers:** computational foundations, the transition to proof, the proof-based core (analysis, algebra, proof-based linear algebra, probability), and electives for breadth and depth.
- **Follow a curriculum.** OSSU Math is a good free spine. Add textbooks such as Axler, Abbott and Pinter for more depth in pure maths.
- **Budget 3,000 to 4,000 hours.** At 10 to 15 hours a week, that's 5 to 7 years, about the same as a part-time degree.
- **Learn proofs first.** Everything after calculus depends on them.
- **Do the problems and get feedback.** Solutions, Lean, Math Stack Exchange and study partners all help. Watching lectures alone doesn't build mathematical ability.
- **If you need the paper,** part-time distance degrees such as the Open University or FernUniversität in Hagen are built for people who already work.

## References

[^1]: [MIT OpenCourseWare: MIT Curriculum Guide](https://mitocw.ups.edu.ec/courses/mit-curriculum-guide/)
[^2]: [OSSU Math: Path to a free self-taught education in Mathematics (GitHub)](https://github.com/ossu/math)
[^3]: [Richard Hammack, *Book of Proof*](https://richardhammack.github.io/BookOfProof/)
[^4]: [Sheldon Axler, *Linear Algebra Done Right*, 4th ed. (Springer, open access)](https://link.springer.com/book/9783031410253)
[^5]: [Linear Algebra Done Right: author's site](https://linear.axler.net/)
[^6]: [The Open University: BSc (Honours) Mathematics and Physics (Q77)](https://www.open.ac.uk/courses/science/degrees/bsc-mathematics-and-physics-q77)
[^7]: [The Open University: BSc (Honours) Mathematics (Q31)](https://www.open.ac.uk/courses/qualifications/q31)
[^8]: [JKU Zentrum für Fernstudien: Bachelor Mathematik (FernUniversität in Hagen)](https://www.jku.at/zentrum-fuer-fernstudien/fernstudium-hagen/studienangebot/bachelorstudiengaenge/bachelor-mathematik/)
