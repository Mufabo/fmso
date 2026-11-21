---
title: "How to Write a Textbook with LLMs: A Workflow That Holds Up"
date: 2026-10-31
draft: true
description: "A practical, evidence-based workflow for writing a textbook or study guide with large language models: what to delegate, how to structure each chapter, what changes when the author is a novice, and how to verify every claim."
summary: "One publisher's machine-written book was a celebrated prototype; another was retracted for fake citations. The difference was process. A step-by-step workflow for writing textbooks and study guides with LLMs, a research-backed chapter structure, and the extra rules for authors who aren't experts."
tags: ["LLM", "AI", "textbooks", "study guides", "writing", "education", "learning science", "workflow"]
categories: ["AI"]
---

In 2019, Springer Nature published *Lithium-Ion Batteries: A Machine-Generated Summary of Current Research*. The author was listed as "Beta Writer", an algorithm built with researchers at Goethe University Frankfurt. It was openly labelled as a prototype, freely available, and nobody pretended a human wrote it.

In 2025, the same publisher retracted *Mastering Machine Learning: From Basics to Advanced*, a $169 textbook. Retraction Watch checked 18 of its 46 references and found that about two-thirds either didn't exist or had serious errors. The publisher's own investigation could not verify 25 of the 46. The book carried no AI disclosure, although Springer Nature's policy requires one for anything beyond basic copy editing.

Both books leaned heavily on machines. One was an honest experiment; the other was a liability. The difference was not the technology but the process around it.

This post is a workflow for writing a textbook with large language models (LLMs) that ends up closer to the first case than the second. It covers what LLMs are actually good at, where they reliably fail, a step-by-step process that keeps you in charge of the parts that matter, how the finished book should be structured, what changes when the person writing it is not an expert in the subject, and a worked example: a beginner writing a book on Unreal Engine 5 and C++.

## What LLMs are good at, and what they are not

Before designing a workflow, it helps to know where the evidence points.

### Strengths

**Drafting clear explanations quickly.** This is the obvious one. A model can produce a readable first draft of a section in seconds, and it rarely gets stuck on a blank page.

**Rewriting the same content for different readers.** Google's Learn Your Way experiment (LearnLM Team, 2025) took textbook chapters and rewrote them for a learner's grade level and interests, then added quizzes, audio lessons, slides and mind maps. In a randomised trial, 60 Chicago-area students aged 15–18 spent up to 40 minutes learning about adolescent brain development, either with Learn Your Way or with a standard PDF reader. The Learn Your Way group scored higher right after the session and on a retention test 3–5 days later (78% vs 67%). It was one chapter and one short session, and the authors couldn't say which feature drove the gain, but it is real evidence that LLM-adapted textbook material can help learning.

**Generating practice questions.** In a study across Hong Kong, Singapore, Ireland and the UK, ChatGPT wrote 50 multiple-choice questions for medical graduate exams in about 20 minutes; two experienced examiners took about 3.5 hours for their 50 (Cheung et al., 2023). Blinded international assessors rated the two sets as broadly similar in quality. The AI questions were slightly less relevant and varied more in quality, meaning some were excellent and some were weak.

**Improving on bare text when used as a tutor.** A Harvard randomised trial found that a custom GPT-4 tutor, carefully built around active-learning principles, produced more than twice the learning gains of an in-class active-learning session in an introductory physics course, in less time (Kestin et al., 2025). This was 194 students and two topics, but it suggests that how you package material matters as much as the prose.

### Weaknesses

**Citations.** This is the best-documented failure. When Walters and Wilder (2023) asked GPT-3.5 and GPT-4 to write short literature reviews on 42 topics, 55% of GPT-3.5's citations and 18% of GPT-4's were entirely made up. Among the real ones, 43% and 24% had substantive errors such as wrong volume or page numbers. Newer models do better, but a 2025 test of the free versions of eight chatbots found that only about a quarter of 400 requested references were fully correct, and six of the eight invented some outright (Cabezas-Clavijo & Sidorenko-Bautista, 2025). Never let a model write your bibliography from memory.

**Plausible-but-weak exercise design.** A comparison of ChatGPT-written and human-written language-test items found them broadly comparable overall, but the model was notably worse at writing plausible wrong answers (distractors) (Chun & Barley, 2024). Bad distractors make questions too easy, and too-easy questions teach nothing.

**Long-range structure and synthesis.** Stanford's STORM project (Shao et al., 2024) studied how to get LLMs to write long, grounded, Wikipedia-style articles. Their key finding was that the pre-writing stage, researching the topic and building an outline, determines most of the final quality. Even their improved system showed characteristic problems, which experienced Wikipedia editors flagged: biases in the source material carried into the article, and unrelated facts got linked together as if they were connected. Both are fatal in a textbook.

**Undermining learning if used carelessly.** In a field experiment with nearly 1,000 high school maths students in Turkey, students given plain GPT-4 access during practice did 48% better on practice problems but 17% worse than students with no AI once access was removed (Bastani et al., 2025). A second tutor, which was given the worked solutions and told to give hints rather than answers, largely removed the harm. The lesson for textbook authors: an LLM that does the thinking for the learner makes material feel easier while teaching less.

## The workflow

The process below assumes you are the subject expert, or have one, and want to use LLMs for the heavy lifting of drafting, adapting and generating practice. Each step says what the model does, what you do, and how you check it. If you are learning the subject yourself, read [When the author is a novice](#when-the-author-is-a-novice) first, because several steps depend on judgement you may not have yet.

### Step 1: Decide what the book is for (you, not the model)

Write down, in your own words:

- **Who the reader is.** Background, prerequisites, what they already believe that might be wrong.
- **What they should be able to do afterwards.** Not "understand X" but "derive X", "debug Y", "choose between A and B and justify it".
- **What the book is not.** Topics you will deliberately leave out.

This is the part an LLM is worst at and you are best at. It also becomes the yardstick for every later step: a section that does not serve a stated objective gets cut, however well written it is.

You can use a model here as a sparring partner. Ask it to argue that your objectives are too broad, or to list misconceptions typical readers bring to the subject. But the decisions stay yours.

### Step 2: Build a source base before writing anything

The STORM result is the most important practical lesson in this post: research first, write second.

Collect the material the book will rest on: primary papers, standards, documentation, your lecture notes, datasets, existing open textbooks with compatible licences. Put it somewhere the model can read it: a folder of PDFs and notes, a project knowledge base, or a retrieval system.

Then use the model to explore, not to write:

- "From these sources, list the main concepts and how they depend on each other."
- "What questions would a sceptical expert, a beginner and a practitioner each ask about this topic?" (This is roughly what STORM does to broaden coverage.)
- "Which claims in the outline are not supported by any of the sources provided?"

Keep a reference manager (Zotero or similar) as the single source of truth for citations. Every reference in the final book should come from there, with a DOI or stable URL, never from model output.

### Step 3: Outline as a dependency graph

A textbook is not a list of topics; it is an order in which ideas become understandable. Draft the outline as a set of concepts with prerequisites, then turn that into chapters.

For each chapter, write:

- learning objectives (two to five);
- the prerequisites it assumes;
- the one or two core examples it will build around;
- the misconceptions it must address.

[How to structure the book](#how-to-structure-the-book) below gives a full chapter template.

Ask the model to critique the outline: "Find every place where a concept is used before it is introduced." "Which chapter is overloaded?" Models are good at this kind of structural checking, and it is far cheaper to fix an outline than a finished chapter.

### Step 4: Write a style guide the model sees every time

The biggest practical problem in long LLM-written texts is drift. Notation changes between chapters, the same term gets three names, the voice wanders from chatty to formal. The fix is a short "book bible" that goes into every drafting request:

- audience and reading level;
- voice (second person? first-person plural?), sentence length, British or American spelling;
- notation and symbols, with definitions;
- a glossary of preferred terms and banned synonyms;
- recurring examples or characters that run through the book;
- formatting rules for definitions, worked examples, warnings and exercises;
- things to avoid: unsupported superlatives, filler intros, phrases you dislike.

Keep it to a page or two. Update it whenever you catch the model making the same mistake twice.

### Step 5: Draft one section at a time, grounded in sources

Generate small units: a section of 800–2,000 words, not a chapter. For each one, give the model:

1. the style guide;
2. the book outline, with the current section highlighted;
3. a short summary of the previous section (what has been introduced, which notation is now in use);
4. the relevant source excerpts;
5. the section's objectives and planned examples.

A drafting prompt can look like this:

```text
You are drafting section 4.2 of an undergraduate textbook on numerical methods.
Follow the attached style guide exactly.

Objectives for this section:
- The reader can explain why naive summation loses precision.
- The reader can implement Kahan summation and test it.

Already introduced: floating-point representation, machine epsilon (notation: ε).
Not yet introduced: condition numbers. Do not use them.

Use only the attached sources for factual claims. Where a claim needs a source,
insert [CITE: short description] instead of writing a reference.
Where you are unsure of a fact, insert [CHECK: what to verify].

Structure: motivating example → explanation → worked example → common mistake → 3 exercises.
```

The `[CITE]` and `[CHECK]` placeholders matter. They turn the model's uncertainty into a to-do list instead of confident invention. You then fill citations from your reference manager.

If you use an agentic tool that can read and write files, keep the whole book in a version-controlled repository with one file per section. That makes it easy to rerun a single section, see exactly what changed in a revision, and roll back.

### Step 6: Add the learning, not just the text

A textbook that only explains is a reference book. The techniques with the strongest evidence for durable learning are practice testing and spacing practice over time, with elaborative interrogation, self-explanation and interleaved practice showing more moderate support (Dunlosky et al., 2013). Highlighting and rereading, which plain prose invites, rate poorly.

LLMs are well suited to producing the raw material for these techniques, at volume:

- **Worked examples followed by near-identical problems** to attempt alone.
- **Retrieval questions** at the end of each section and, crucially, questions in later chapters that revisit earlier material (spacing).
- **Mixed problem sets** that require choosing which method applies (interleaving).
- **"Why" and "what if" prompts** that ask the reader to explain a step or predict what changes.
- **Common-misconception items** built from the list you made in Step 3.

Then apply the Bastani lesson. Keep full solutions separate from the main text (an appendix, a separate solutions file, or collapsible blocks), and offer hints in stages. If you build an interactive companion, give the model the worked solution and common mistakes, and instruct it to hint, not answer.

Review exercise quality by hand. Pay particular attention to wrong answer options, which is exactly where the research says models are weakest. A good distractor encodes a real misconception; a bad one is obviously silly.

### Step 7: Verify everything the reader will trust

This is where most LLM-assisted books fail, and where most of your time should go. Treat every model-written sentence as unverified until checked.

**Citations.** Resolve every reference against the DOI or the publisher's page. Check not just that the work exists but that it says what you cite it for. A link is no guarantee: Walters and Wilder found that links were actually more common in fabricated citations than in real ones, and about a third of the links attached to real works led somewhere else.

**Facts.** Run a separate verification pass with a fresh context: give the model one section and its sources, and ask it to list every factual claim with the source passage that supports it, flagging any it cannot match. Then check the flagged ones and spot-check the rest yourself. A fresh context matters; a model checking its own draft in the same conversation tends to agree with itself.

**Maths.** Re-derive key results. For numerical examples, compute them in code rather than trusting the arithmetic in the prose.

**Code.** Every code listing should be extracted from the source files and executed automatically in continuous integration, with the output checked. If a listing can't run in isolation, it shouldn't appear in a textbook.

**Exercises.** Solve each one yourself, or have a second model solve it blind and compare against the answer key. Disagreement means one of them, or the question, is wrong.

### Step 8: Put humans in the loop

No amount of model checking replaces two kinds of human review:

- **A subject expert** who reads for correctness, emphasis and what is missing. Experts notice when the book is technically right but teaches the wrong mental model.
- **Readers from the target audience** who work through a chapter and report where they got lost. This is cheap and catches problems that experts are blind to, such as a step that seems obvious only if you already know the answer.

Run both on a pilot chapter before drafting the rest. Fixing the process after one chapter is far cheaper than after twenty.

### Step 9: Rights, disclosure and honesty

**Disclose AI use.** Most academic publishers now require it, and readers deserve to know. Say what the model did (drafting, exercise generation, editing) and what humans did (design, verification, review).

**Know the copyright position.** The US Copyright Office's January 2025 report concluded that prompts alone do not give a human enough control to count as the author of the output. Human selection, arrangement and substantial modification of AI-generated material can still be protected. In practice, the more of the book's expression comes from your own writing and editing, the stronger your claim. Rules differ by country; if the book is commercial, take proper legal advice.

**Respect source licences.** If you feed open textbooks or papers to a model, check that their licences allow adaptation and attribute them properly.

## How to structure the book

Left to itself, an LLM writes in one shape: an introduction, some explanation, a few bullet points, a summary. That shape is easy to read and poor for learning. Structure is the part of a textbook where learning science gives the clearest instructions, so decide it up front and put it in the style guide. The model will follow a template reliably; it will not invent a good one.

### The whole book

- **Front matter that sets up the reader.** Who the book is for, what they need to know already, and a short diagnostic quiz on the prerequisites with pointers to where to catch up. Then a page on how to use the book: attempt the exercises before reading the solutions, and come back to the review sets.
- **Chapters in dependency order** from the outline in Step 3, with deliberate returns to earlier material. A later chapter should reuse earlier ideas in new problems, not just mention them. That is spacing built into the structure.
- **Cumulative review sets** every few chapters, mixing problem types from across the book (interleaving).
- **Back matter**: a glossary that matches the style guide's terminology, full solutions kept separate from the chapters, and an index.

### Each chapter

This template turns the research above into a repeatable shape:

| Part | What it contains | Why |
| --- | --- | --- |
| 1. Opener | Objectives phrased as things the reader will be able to do; two or three prerequisite questions with page references | Tells readers what to attend to and catches gaps before they cause confusion |
| 2. Key terms first | Short definitions of the new terms the chapter uses | Mayer's pre-training principle: learning the names and parts before the complex explanation lowers the load |
| 3. Motivating problem | A concrete question the chapter will let the reader answer | Gives the explanations a purpose |
| 4. Core sections, one idea each | Explanation → concrete example → worked example → partly worked example → problem to try alone → quick check | Segmenting, worked examples and fading (below) |
| 5. Misconception boxes | The common belief, a clear statement that it is wrong, and the correct explanation | Refutation text (below) |
| 6. Make-it-your-own prompts | Explain a step in your own words, sketch a concept map, predict what changes if a condition changes | Generative learning (below) |
| 7. Mixed practice | Problems from this chapter and earlier ones, unlabelled by section | Retrieval, spacing and interleaving |
| 8. Summary, written by the reader first | A prompt to write the chapter's main points from memory, then the book's own summary to compare | Turns the summary into retrieval practice instead of rereading |
| 9. Looking ahead | One paragraph on where these ideas are used next | Links chapters together |

Four pieces of research shape that template.

**Worked examples, then fade them.** Beginners learn more from studying worked solutions than from solving problems unaided, but the benefit fades as they gain skill and can even reverse: for more experienced learners, step-by-step solutions become redundant and get in the way (the expertise reversal effect; Kalyuga et al., 2003). The recommended bridge is fading: start with a fully worked example, then give examples with later steps left for the reader, until the reader solves problems alone (Renkl & Atkinson, 2003). For a textbook, that means sequences of examples, not single ones, and a "try it first" route for readers who already know the basics. LLMs are good at generating faded variants of a worked example, but check that each removed step is actually solvable from what came before.

**Refute misconceptions directly.** A refutation text names a common wrong belief, says it is wrong, and explains the correct idea. A meta-analysis of 44 comparisons with 3,869 learners found a moderate advantage over standard expository text (g = 0.41) that held across subjects and ages (Schroeder & Kucera, 2022), and a larger pre-registered meta-analysis of 76 studies also found a consistent advantage (Danielson et al., 2025). One caveat: the studies used short passages, the longest about 1,500 words, so whether the effect holds across a whole textbook hasn't been tested. Still, it costs almost nothing to add. The misconception list from Step 1 becomes these boxes, and a model can draft them, but the misconceptions themselves should come from teachers, research or real student errors, not from the model's guess.

**Make readers produce something.** Fiorella and Mayer (2016) reviewed eight activities that reliably improve understanding because the learner has to construct something: summarising, mapping, drawing, imagining, self-testing, self-explaining, teaching and enacting. A textbook can prompt most of these. Each chapter should ask the reader to make at least one thing, not only to read.

**Cut, signal and keep things together.** Mayer's multimedia principles (Mayer, 2020) give three rules that matter for LLM drafts in particular. *Coherence*: leave out interesting but irrelevant material, which models add freely. *Signalling*: use headings, bold key terms and explicit "the key point is" sentences so readers can see the structure. *Contiguity*: put a figure next to the text that explains it, and labels on the figure rather than in a legend.

You can put the template straight into the drafting prompt from Step 5:

```text
Write section 3.2 using this structure exactly:
1. One-paragraph explanation of the idea (no more than 250 words).
2. A concrete example from the running example in the style guide.
3. A fully worked example, with each step justified in one sentence.
4. The same problem type with the last two steps left for the reader.
5. One problem for the reader to solve alone (solution goes to solutions.md, not here).
6. A misconception box: state the belief, say plainly it is wrong, explain why.
   Use only this misconception: [from the misconception list].
7. Two quick-check questions answerable from memory.
Do not add an introduction, a conclusion or a summary.
```

### Study guides are a different shape

A study guide is not a short textbook. Its reader has already met the material and needs to retrieve and practise it, so it should be mostly questions:

- the course objectives, as a checklist;
- key terms with the definitions left blank for the reader to fill in;
- retrieval questions grouped by objective, from recall to application;
- a mixed practice set that doesn't say which topic each question comes from;
- a calibration step: predict your score, take the set without notes, compare;
- a suggested schedule that spreads practice over several days;
- answers in a separate section, with a page reference to the source material for each.

What it should not be is pages of condensed notes. Those invite rereading, which Dunlosky and colleagues rated among the least effective study techniques.

## When the author is a novice

Everything above assumes someone who can tell a correct explanation from a fluent wrong one. Remove that person and the workflow's safety net is gone: the verification steps still run, but nobody can judge what they find. This is the most common way LLM-written study material goes wrong, and it is worth separating into two quite different cases.

### What the evidence says about novices and AI

**People using AI are poor judges of how well they are doing.** In two pre-registered studies, participants solved LSAT logical reasoning problems with ChatGPT. In the first (246 people), their scores improved by about three points over a norm group, but they overestimated their scores by about four. A second study of 452 people with a proper no-AI comparison group found the same pattern. Participants who knew more about AI were more confident but *less* accurate about their own performance (Fernandes et al., 2026). A novice checking an AI-written chapter is exactly in this position.

**Letting the model write can mean not learning it.** In an MIT Media Lab study, 54 participants wrote essays with ChatGPT, with a search engine or unaided, while EEG measured brain activity. The ChatGPT group showed the least connectivity, and most of them could not accurately quote from the essay they had just written (Kosmyna et al., 2025). This is a small preprint and the authors call it preliminary, but it fits the Bastani result above and the generative-learning research: what you produce yourself is what you remember.

Together these point to two separate risks: errors pass straight through to readers, and the author learns less than they think.

### Case 1: a learner making a study guide for themselves

Here the guide matters less than the act of making it. Summarising, mapping and self-testing are exactly the activities that build understanding, so asking a model to "write me a study guide on photosynthesis" outsources the most valuable part of studying. Reverse the roles:

1. **Write first.** Make your own summary, concept map or question list from the course material before opening a chatbot.
2. **Ground the model in your course.** Upload the assigned chapters, slides or lecture notes and tell it to use only those. Your exam is set against that material, not against what the model remembers, and it gives you something to check claims against.
3. **Use the model as an examiner, not an author.** Ask it to find gaps in your summary compared with the source, to quiz you one question at a time and wait for your answer, and to generate practice problems with answers hidden.
4. **Ask for the page.** For every correction the model makes, ask where in your course material that point is covered, and look.
5. **Check your calibration without AI.** Before trusting that you know a topic, answer a mixed set of questions with no notes and no chatbot. The Fernandes result says your confidence while using AI is not a good guide.

### Case 2: a non-expert writing for other people

This covers a teacher assigned a subject outside their field, a student writing revision notes for classmates, or someone building training material for a team. The stakes are higher because readers will trust the material. Six rules:

1. **Adapt, don't originate.** Start from a trusted source, such as an open textbook (OpenStax, LibreTexts), official documentation or the course's set text, and restrict the model to it. Your job becomes adapting a checked source for a particular audience, which is what Learn Your Way did, rather than producing new explanations that only an expert could verify.
2. **Shrink the scope to what you can check.** A study guide keyed to chapters 3–5 of an existing book is safer than an original book on the whole subject.
3. **Get an expert to review. This isn't optional.** Even an hour per chapter from a teacher, lecturer or practitioner catches the errors you can't see: correct-sounding explanations that teach the wrong mental model.
4. **Use checks that don't need expertise.** They catch some errors, not all:
   - every claim matched to a passage in the source (Step 7);
   - every reference taken from a reference manager and opened;
   - calculations and code run, not read;
   - exercises solved independently by a second model, with disagreements flagged for the expert;
   - a pilot with real readers.
5. **Prefer formats where mistakes do less harm.** Practice questions that point back to pages in a trusted book can be checked against it. New explanations cannot.
6. **Label it honestly.** "Revision notes by a student, based on [source], reviewed by [name]" sets the right expectations. "Textbook" does not.

### What a novice can and cannot check

| Check | Possible without expertise? |
| --- | --- |
| Does this reference exist and say this? | Yes: open it |
| Is this claim in the source material? | Yes: find the passage |
| Does the code run and the arithmetic add up? | Yes: run it |
| Do two models agree on this exercise's answer? | Yes, but agreement isn't proof |
| Is this the right way to explain the idea? | No: needs an expert or teacher |
| Is anything important missing? | No: needs an expert |
| Is this misconception one real learners have? | No: needs teaching experience or research |
| Will readers understand it? | Partly: pilot with real readers |

The bottom half of that table is why expert review is the one step a novice can't skip.

## Worked example: a beginner's book on Unreal Engine 5 and C++

To make this concrete, take Sam. Sam did a C++ course at university, has spent a few months making small projects in Unreal Engine 5, and wants to write a book for people one step behind: a first game in Unreal using C++. Sam is both cases at once: still learning the engine, and writing for other people.

Game engines are an especially risky subject for LLM-assisted writing. The APIs change between versions, and the web, which is what models learn from, is full of tutorials written for older ones. A model will happily write Unreal Engine 4 code with complete confidence, and a beginner has no way to tell.

### Step 1: Scope it to what Sam can check

"The complete guide to Unreal Engine 5" is out of reach. This is not:

- **One small game, built across the book.** For example, a top-down game where the player collects items and avoids enemies. Every chapter adds one feature to the same project.
- **One pinned engine version.** At the time of writing, Unreal Engine 5.8 is the current release (the 5.8.4 hotfix shipped on 20 October 2026), and Epic has said it is the last planned major UE5 release. Sam writes "Tested with Unreal Engine 5.8.4" on the cover and in the style guide, and every prompt names that version.
- **One platform and toolchain**: Windows, with a named editor (Visual Studio or Rider) and version.
- **C++ for gameplay code, Blueprints only where the engine expects them** (for example, assigning assets in the editor).

Epic is targeting an early-access release of Unreal Engine 6 at the end of 2027, so the book has a shelf life. Pinning the version makes that honest rather than a surprise.

### Step 2: The source base, and what Sam can't copy

The trusted sources are:

- **Epic's official documentation** for the pinned version, including the API reference.
- **The engine's own C++ templates.** Creating a new C++ project from the Third Person template generates code written by Epic for that exact version. When a model's code and the template disagree, the template wins.
- **The engine headers in Sam's own install**, which are the final word on function names and signatures.
- **The forums and community Q&A**, not as sources of truth but as a record of what beginners actually get stuck on.

One difference from the open-textbook advice above: Epic's documentation is not openly licensed the way OpenStax books are, so Sam can't adapt its text. Sam uses it to check facts and links to it, but writes the explanations independently. Engine source code needs the same care. The Unreal Engine EULA has allowed only short excerpts of engine code, up to 30 lines, to be shared publicly for discussion. Sam's own project code is Sam's to publish, but engine internals shouldn't be pasted into the book. Check the current EULA before publishing.

### Step 3: Know the model's typical mistakes

Ask a general-purpose model for "Unreal C++ character movement" and you may well get something like this, which was standard in Unreal Engine 4:

```cpp
// MyCharacter.h (UE4-era style)
UPROPERTY()
UCameraComponent* Camera;

// MyCharacter.cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    PlayerInputComponent->BindAxis("MoveForward", this, &AMyCharacter::MoveForward);
}
```

It compiles in many setups, and a beginner would have no reason to doubt it. But it reflects two things that have changed:

- **Input.** Unreal now uses the Enhanced Input system by default, with Input Actions and Input Mapping Contexts, and the editor flags the old axis and action mappings as deprecated.
- **Object pointers.** Epic's documentation says `TObjectPtr` should replace raw pointers where possible, and that existing `UPROPERTY` raw pointers should migrate to `UPROPERTY` `TObjectPtr`.

The current style looks more like this (an excerpt; the Third Person C++ template shows the full setup, including where the mapping context is added and the module dependency):

```cpp
// MyCharacter.h
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Camera")
TObjectPtr<UCameraComponent> Camera;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputAction> MoveAction;

void Move(const FInputActionValue& Value);

// MyCharacter.cpp
#include "EnhancedInputComponent.h"

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    if (UEnhancedInputComponent* Input = Cast<UEnhancedInputComponent>(PlayerInputComponent))
    {
        Input->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::Move);
    }
}
```

The point for Sam isn't these two specific changes. It's that the model's confidence says nothing about which era its code comes from, so every listing needs checking against the pinned version.

### Step 4: A verification routine Sam can actually run

None of this needs expertise, only discipline:

1. **Build the game before writing the chapter.** Sam implements each feature by hand in the companion project first. A chapter is never written about code that hasn't run.
2. **Keep the companion project in git**, with a tag per chapter (`ch05-end`). Every code listing in the book is copied from that project, not from a chat window.
3. **Compile with warnings on and treat deprecation warnings as bugs.** If a listing produces a deprecation warning in 5.8.4, it doesn't go in the book.
4. **Rebuild every chapter's tag from a clean checkout** before each draft goes to reviewers, ideally with a scripted command-line build so it can't be skipped.
5. **Check every engine API against the source.** Ask the model to list each engine class, function and macro a chapter uses. Then look each one up in the API reference for 5.8 or search the engine headers. This is lookup work, not judgement, so a beginner can do it.
6. **Compare with the template.** For anything the Third Person template also does (input, camera, movement), Sam's code should follow the same pattern or explain why it doesn't.

### Step 5: Misconceptions from real beginners

The most common beginner questions on Unreal forums are a ready-made misconception list, and every one becomes a refutation box. A few that a reviewer will almost certainly want covered:

- **"UObjects are ordinary C++ objects."** They aren't. They are created with engine functions (`NewObject`, or `CreateDefaultSubobject` for components in a constructor), not `new`, and the engine's garbage collector manages their lifetime.
- **"A pointer member is fine as long as I set it."** If a pointer to a UObject isn't a `UPROPERTY`, the garbage collector doesn't know about the reference, and the object can be destroyed while the pointer still points at it.
- **"The constructor is where gameplay starts."** An actor's constructor also runs to build the class's default object in the editor. Setup that depends on the running game belongs in `BeginPlay`.
- **"UE4 tutorials still apply."** Many do, but input and pointer conventions are two places where they don't.

Sam should confirm each of these with the expert reviewer, and use the reviewer's wording and examples, rather than relying on the model's version or this list.

### Step 6: One chapter, using the template

Here is how the chapter template maps onto Chapter 5, "Making the player move":

| Part | Chapter 5 content |
| --- | --- |
| Opener | Objectives: bind keyboard and gamepad input to movement in C++; explain what an Input Action and an Input Mapping Context each do. Prerequisite check: what is a `UPROPERTY`? Where did we create the camera component? (with chapter references) |
| Key terms first | Input Action, Input Mapping Context, Trigger, Modifier, `ETriggerEvent` |
| Motivating problem | The character from Chapter 4 stands still. By the end of this chapter, WASD and a gamepad stick move it relative to the camera |
| Worked example | `Move()`: every line explained, built and tested in the companion project |
| Partly worked example | `Look()` follows the same pattern, with the last two steps left for the reader |
| Problem to try alone | Add a sprint that increases speed while a key is held. Solution in the back, plus the `ch05-end` tag |
| Misconception box | "My `BindAxis` code from a tutorial should work": why it may compile but is the deprecated system, and what replaced it |
| Make-it-your-own | Draw the path from a key press to `Move()` being called, naming every object in between |
| Mixed practice | Includes a question from Chapter 3: why must `MoveAction` be a `UPROPERTY`? |
| Reader's summary | Write the steps for adding a new input from memory, then compare with the checklist |
| Looking ahead | Chapter 6 uses the same input system to fire projectiles |

### Step 7: Reviewers and honest labelling

Sam needs at least one experienced Unreal C++ developer to review every chapter. The places to look are colleagues, a games lecturer, the Epic Developer Community forums, community Discord servers, or a paid technical reviewer. The request is specific: "Is this the way you would teach it, and what would a beginner misunderstand?" Testing for "does it compile" is Sam's job; the reviewer's job is the mental model.

Sam also pilots each chapter with two or three readers at the book's target level, watching where they get stuck.

Finally, the cover tells the truth: "A first game in Unreal Engine 5 with C++. Tested with Unreal Engine 5.8.4. Written by a recent learner, technically reviewed by [name]. Drafted with AI assistance; all code built and tested by the author." That label makes the book more trustworthy, not less. Readers know exactly what they are getting, and what to check if they use a newer engine.

## Who does what: a summary

| Task | LLM role | Your role | How to check |
| --- | --- | --- | --- |
| Audience and objectives | Sparring partner, list misconceptions | Decide | Does every chapter serve an objective? |
| Research | Map concepts, suggest questions, spot gaps | Choose sources, own the bibliography | Every reference in your reference manager |
| Outline | Critique ordering and load | Design the dependency graph | No concept used before introduced |
| Style guide | Suggest rules from sample text | Write and maintain | Consistency across chapters |
| Drafting | First drafts, section by section | Edit, rewrite, cut | Fresh-context claim check |
| Examples and exercises | Generate in volume | Select, fix distractors, write key items | Solve independently |
| Adapted versions (levels, interests) | Rewrite | Review a sample | Learner testing |
| Citations | None from memory; format only | Supply and verify | Resolve DOI, read the cited passage |
| Code and maths | Draft | Run and re-derive | Automated tests, recomputation |
| Chapter structure | Follow the template | Choose the template, write the misconception list | Every chapter has worked, faded and independent problems |
| Final review | Proofreading, terminology checks | Expert and learner review | Pilot chapter first |

## Common mistakes

- **Generating whole chapters in one request.** Quality and consistency fall apart. Work in sections with a shared style guide.
- **Asking the model for references.** Even good models still invent or garble them. Supply them yourself.
- **Letting fluency stand in for accuracy.** LLM prose sounds authoritative whether or not it's right. The smoother it reads, the more carefully you should check it.
- **Over-explaining.** Models default to covering everything. Textbooks need selection. Cut anything that doesn't serve an objective.
- **Answers next to questions.** Easy access to solutions turns practice into reading. Hide solutions and offer staged hints.
- **Same-model, same-context checking.** Verify with a fresh context, a different model, or a human.
- **Skipping the pilot.** Test one chapter end to end with real readers before scaling up.
- **Letting the model choose the structure.** Its default shape is intro, explanation, bullets, summary. Give it the chapter template instead.
- **Asking for a study guide instead of making one.** For your own studying, the making is the learning. Write first, then use the model to test you.
- **Trusting your own confidence.** AI users overestimate how well they are doing. If you aren't an expert, an expert has to review.

## Should it even be a static book?

The most interesting evidence here points beyond the printed page. Learn Your Way's gains came from combining adapted text with embedded questions and multiple representations; the Harvard tutor worked because it made students do the thinking in small steps. A carefully verified textbook is still the foundation in both cases: Learn Your Way rebuilt its material from existing textbook chapters, and the guarded tutor in the Bastani study worked from solutions written in advance.

So a reasonable target for an LLM-assisted textbook in 2026 is a verified core text, written and checked as described above, plus generated layers on top: versions at different reading levels, extra practice, and an optional tutor that hints rather than answers. The core is where your expertise lives. The layers are where the model earns its keep.

## References

1. Bastani, H., Bastani, O., Sungu, A., Ge, H., Kabakcı, Ö., & Mariman, R. (2025). [Generative AI without guardrails can harm learning: evidence from high school mathematics](https://doi.org/10.1073/pnas.2422633122). *Proceedings of the National Academy of Sciences*, 122(26), e2422633122.
2. Beta Writer. (2019). [*Lithium-Ion Batteries: A Machine-Generated Summary of Current Research*](https://doi.org/10.1007/978-3-030-16800-1). Springer Cham.
3. Cabezas-Clavijo, Á., & Sidorenko-Bautista, P. (2025). [Assessing the performance of 8 AI chatbots in bibliographic reference retrieval: Grok and DeepSeek outperform ChatGPT, but none are fully accurate](https://arxiv.org/abs/2505.18059). arXiv:2505.18059.
4. Cheung, B. H. H., Lau, G. K. K., Wong, G. T. C., Lee, E. Y. P., Kulkarni, D., Seow, C. S., et al. (2023). [ChatGPT versus human in generating medical graduate exam multiple choice questions—a multinational prospective study](https://doi.org/10.1371/journal.pone.0290691). *PLOS ONE*, 18(8), e0290691.
5. Chun, J. Y., & Barley, N. (2024). [A comparative analysis of multiple-choice questions: ChatGPT-generated items vs. human-developed items](https://doi.org/10.31274/isudp.2024.154.08). In C. A. Chapelle, G. H. Beckett, & J. Ranalli (Eds.), *Exploring Artificial Intelligence in Applied Linguistics* (pp. 118–136). Iowa State University Digital Press.
6. Danielson, R. W., Jacobson, N. G., Patall, E. A., Sinatra, G. M., Adesope, O. O., Kennedy, A. A. U., et al. (2025). [The effectiveness of refutation text in confronting scientific misconceptions: a meta-analysis](https://doi.org/10.1080/00461520.2024.2365628). *Educational Psychologist*, 60(1), 23–47.
7. Dunlosky, J., Rawson, K. A., Marsh, E. J., Nathan, M. J., & Willingham, D. T. (2013). [Improving students' learning with effective learning techniques: promising directions from cognitive and educational psychology](https://doi.org/10.1177/1529100612453266). *Psychological Science in the Public Interest*, 14(1), 4–58.
8. Epic Games. (n.d.). [Enhanced Input in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/enhanced-input-in-unreal-engine). Unreal Engine documentation.
9. Epic Games. (n.d.). [Object pointers in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/object-pointers-in-unreal-engine). Unreal Engine documentation.
10. Epic Games. (2026). [State of Unreal 2026: top news from the show](https://www.unrealengine.com/news/state-of-unreal-2026-top-news-from-the-show).
11. Epic Games. (2026, October 20). [5.8.4 hotfix released](https://forums.unrealengine.com/t/5-8-4-hotfix-released/2842775). Epic Developer Community forums.
12. Fernandes, D., Villa, S., Nicholls, S., Haavisto, O., Buschek, D., Schmidt, A., Kosch, T., Shen, C., & Welsch, R. (2026). [AI makes you smarter but none the wiser: the disconnect between performance and metacognition](https://doi.org/10.1016/j.chb.2025.108779). *Computers in Human Behavior*, 175, 108779.
13. Fiorella, L., & Mayer, R. E. (2016). [Eight ways to promote generative learning](https://doi.org/10.1007/s10648-015-9348-9). *Educational Psychology Review*, 28(4), 717–741.
14. GameDev.tv Community. (n.d.). [Axis/Action mappings are now deprecated](https://community.gamedev.tv/t/axis-action-mappings-are-now-deprecated/234078).
15. Kalyuga, S., Ayres, P., Chandler, P., & Sweller, J. (2003). The expertise reversal effect. *Educational Psychologist*, 38(1), 23–31.
16. Kestin, G., Miller, K., Klales, A., Milbourne, T., & Ponti, G. (2025). [AI tutoring outperforms in-class active learning: an RCT introducing a novel research-based design in an authentic educational setting](https://doi.org/10.1038/s41598-025-97652-6). *Scientific Reports*, 15, 17458.
17. Kosmyna, N., Hauptmann, E., Yuan, Y. T., Situ, J., Liao, X.-H., Beresnitzky, A. V., Braunstein, I., & Maes, P. (2025). [Your brain on ChatGPT: accumulation of cognitive debt when using an AI assistant for essay writing task](https://arxiv.org/abs/2506.08872). arXiv:2506.08872 (preprint).
18. LearnLM Team, Google. (2025). [Towards an AI-augmented textbook](https://arxiv.org/abs/2509.13348). arXiv:2509.13348.
19. Mayer, R. E. (2020). *Multimedia Learning* (3rd ed.). Cambridge University Press.
20. Renkl, A., & Atkinson, R. K. (2003). Structuring the transition from example study to problem solving in cognitive skill acquisition: a cognitive load perspective. *Educational Psychologist*, 38(1), 15–22.
21. Retraction Watch. (2025, June 30). [Springer Nature book on machine learning is full of made-up citations](https://retractionwatch.com/2025/06/30/springer-nature-book-on-machine-learning-is-full-of-made-up-citations/).
22. Retraction Watch. (2025, August 4). [Springer Nature retracts book with fake citations](https://retractionwatch.com/2025/08/04/springer-nature-retracts-book-fake-citations-help-us-find-more/).
23. Schroeder, N. L., & Kucera, A. C. (2022). [Refutation text facilitates learning: a meta-analysis of between-subjects experiments](https://doi.org/10.1007/s10648-021-09656-z). *Educational Psychology Review*, 34, 957–987.
24. Shao, Y., Jiang, Y., Kanell, T. A., Xu, P., Khattab, O., & Lam, M. S. (2024). [Assisting in writing Wikipedia-like articles from scratch with large language models](https://arxiv.org/abs/2402.14207). *Proceedings of NAACL 2024*.
25. U.S. Copyright Office. (2025). [*Copyright and Artificial Intelligence, Part 2: Copyrightability*](https://www.copyright.gov/ai/).
26. Walters, W. H., & Wilder, E. I. (2023). [Fabrication and errors in the bibliographic citations generated by ChatGPT](https://doi.org/10.1038/s41598-023-41032-5). *Scientific Reports*, 13, 14045.
