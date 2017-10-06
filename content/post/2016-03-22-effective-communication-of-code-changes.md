+++
date = "2016-03-22T00:00:00-08:00"
lastmod = "2016-03-22T00:00:00-08:00"
title = "Effective Communication of Code Changes"
type = "slides"
draft = false
+++

# Agenda

### 1. What's the problem?

### 2. Story Time

### 3. My Proposed Solution

.footnote[<sup>\*</sup>Press "p" for presenter's notes, and "c" for split view.]

???

# Agenda

 - Give a brief introduction of the topics that will be discussed.

---

# Code is our Enemy

<br />

```
> Code is bad. It rots. It requires periodic maintenance. It has bugs that
> need to be found. New features mean old code has to be adapted.
>
> The more code you have, the more places there are for bugs to hide. The
> longer checkouts or compiles take. The longer it takes a new employee to
> make sense of your system. If you have to refactor there's more stuff to
> move around.
```

.footnote[<sup>\*</sup>http://www.skrenta.com/2007/05/code_is_our_enemy.html]

???

# Code is our Enemy

 - Set the theme for the talk

 - Code is not friendly

 - If we desire high quality code:

    - We must treat code with caution.

    - We must review code with the assumption that it is wrong!

---

# Communication

<br />

## 1. New features mean old code has to be adapted.

<br />

## 2. [Code] has bugs that need to be found.

???

# Communication

 - Expand on few key points from previous quote.

## 1. New features mean old code has to be adapted

 1. To responsibly _adapt_, a developer making the changes must
    understand the code being modified.

 2. Often, _what_ the code does is insufficient to really understand it;
    _why_ the code does it, is essential.

 3. To understand _why_:

    1. What is the historical context? The lineage?

    2. How did the code get here?

       1. Written once, and never changed?

       2. Gone through many different versions over time?

    3. What were the past decisions and why were they made?

 4. Old code needs to communicate to current developers making these
    adaptations.

 5. Without this information, we can modify old code, but no way to know
    if our changes are _correct_.

 6. New code needs to communicate to future developers

## 2. [Code] has bugs that need to be found.

 1. What is a bug?

    1. Define "bug" to mean "defective", "failure, "wrong behavior"

 2. To determine "right" vs. "wrong", need to know:

    1. How is it _supposed_ to work? i.e. what's the _expected_ behavior?

    2. Then, how _does_ it work? i.e. what's the _actual_ behavior?

 3. _Why_ is the expected behavior so?

    1. As new features are introduced, the _why_ might change.

    2. Maybe, as a result of some other change in the system, we don't
       want an existing system to behave the same way anymore.

 4. If the code doesn't communicate these answers, we cannot tell "bug"
    from "feature"

    1. We cannot know _how_ to adapt the old "bugs" and "features" to
       fit with these new changes.

---

background-image: url(story-time.jpg)
background-size: 75%

# Story Time

???

# Story Time

 - With all that said, I want to tell a story.

 - A story about some development that I did about 6 months ago.

 - A story in which I needed answers to the previous questions, about
   some old code I encountered, and the answers were nowhere to be found.

---

# New Feature

<br />

```
commit 04bca5f9bad93642b6df6c7303dbc064f7a318db
Author: Prakash Surya <prakash.surya@delphix.com>
Date:   Fri May 29 23:12:11 2015 -0700

    DLPX-37794 Transformations Milestone 2: ...

```

???

# New Feature

## 1. Introduction

 - At the end of May last year, I committed the change shown here.

 - This was part of work I was doing for a new features.

 - The details of the project and feature are irrelevant.

## 2. Details

 - What is important to know, is that I was adding a new API

 - An API to allow the creation of a new object

 - The API takes various input parameters, and then creates and persists
   a new object.

## 3. More Details

 - Additionally, this API relies on our API validation layer to ensure
   the parameters are valid.

 - It expects any inputs not conforming to the schema will be rejected.

 - As a result, additional input validation isn't required.

## 4. Summarize

 - All good so far, right?

 - We have an API validation layer, so it makes sense to use it, right?

---

# Bug Found

<br />

```
> I created a new test, which attempts to create a transformation with a
> number of different invalid inputs for the 'operations' property. When
> setting the operations property to Python None (converted to null in
> json translation), we see the below DFE, which triggers the fatal
> fault logic and core dump on the server...
```

.footnote[<sup>\*</sup>http://jira.delphix.com/browse/DLPX-38039]

???

# Bug Found

## 1. Introduction

 - My change goes in, all is well...

 - Until a month later...

 - A colleague starts writing some additional tests.

## 2. Initial Testing

 - Prior to landing my change, I did _some_ testing.

 - Maily positive teseting.. i.e. I verified the things that should
   work, did work.

## 3. Additional Testing

 - QA team started writing some additional negative tests.. i.e. to
   verify invalid parameters are properly handled.

 - What do ya' know.. the additional tests triggered a crash (fatal
   exception) in my code.

 - WTF!? OK, time to start digging.. let's figure out what's going on.

---

# Investigation

<br />

```
checkArgument(params.getOperations() != null, "A list of operations must be “
    “provided, this can be an empty list but must not be null; found 'null'.");
```

???

# Investigation

 - Here’s where the failure occurred.

 - The `operations` field of the input should never be `NULL`. That’s
   specified by the JSON schema.

 - But, here we are; obviously, the `operations` passed in were `NULL`.

 - If the schema disallows `NULL`, how is this possible?

---

# Huh? Interesting...

<br />

```
"operations": {
  "type": "array",
  "description": "...",
  "create": "required",
  "items": {
    "type": "object",
    "description": "...",
    "$ref": "/delphix-source-operation.json"
  }
}
```

???

# Huh? Interesting...

 - Maybe I messed up the schema definition?

 - The `operations` are specified as being `required`

 - Additionally, and more importantly, they're specified as being of
   type: `array`

 - `NULL` is not an `array`

 - The API validation should not have let this request through.

 - Tangent:

    - Remember, I said "no need for input validation" due to using the
      API validation layer?

    - Turns out, I'm generally paranoid when I write code.

    - I don't trust myself to write code correctly.

    - As a result, I program defensively, defending against my own self.

    - This defensive posture turned an implicitly non-fatal failure,
      into an explicit fatal failure, by causing the fatal exception.

    - Without the defensive check, who knows what the failure mode would
      have been! Or how long it would have taken to debug it!

    - OK, back on topic...

 - So, what do we do now?

 - Obviously the API validation layer is letting an unexpected value
   through.

 - It's letting `NULL` through, when it should only allow an array.

 - What's going on here?

---

# Root Cause

<br />

```
commit bb9352d401d89e506056271bbee4c2ec82f5811a
Author: xxxxxxxxx xxx <xxx@delphix.com>
Date:   Wed Nov 14 16:21:06 2012 -0500

    21172 New World Network Configuration
    10533 backend restarts stack even ...
    12306 Enhancement: updating network ...
    12687 NWS should expose light-weight ...
    14509 have a UI control to set jumbo ...
    14783 Changing number of interface ...
    18029 remove code dealing with link ...
    18259 Server Setup > Network: Click ...
    18722 Race condition in ...
    19636 notify the user when network ...
    20616 GUI should not do reverse name ...
```

???

# Root Cause

 - With help from a colleague, we root caused the issue.

 - Turns out, over 3 years ago.. this change was made.

---

# WAT

<br />

```
--- a/ValidatorFactoryImpl.java
+++ b/ValidatorFactoryImpl.java

+    if (type == JsonType.ARRAY)
+        acceptedTypes.add(JsonType.NULL);
```

???

# WAT

 - In that change, there's the addition of these two lines

 - Basically, this says, if the schema requires an array, we also accept
   `NULL`

 - What the ...!? Well, that explains things!

 - Now the question is... is this a "bug"? or "feature"?

 - _Why_ was this change made? Was it intentional, or accidental?

 - The change clearly breaks conformance with the JSON schema standards,
   so it must be a bug! ...right?

 - Or, do we intentially break conformance for a reason?

 - If I change the code to match _my_ expectations, what will I break
   that expects the current behavior?

 - Is the current code _correct_?

---

# Code is our Enemy

<br />

```
--- a/ValidatorFactoryImpl.java
+++ b/ValidatorFactoryImpl.java

+    if (type == JsonType.ARRAY)
+        acceptedTypes.add(JsonType.NULL);
```

<br />

.center[```
147 files changes
3841 insertions(+)
8653 deletions(-)
```]

???

# Code is our Enemy

## 1. Theme

 - Circle back to theme of "code is our enemy"

 - These two lines of change, are amidst a 12 thousand line diff...

 - Which spans 147 files...

 - All of which, was written 3 years ago...

 - And this is causing my failure...

 - Sigh... :(

## 2. Just Ask the Author?

 - Luckily, the author is still available for questioning...

 - I can simply ask them, and they'll remember why these two lines exist
   in the massive diff.. right?

 - There's no way they'd forget why these two lines were added.. right?

 - They'll be able to tell if they should be removed, to conform to the
   schema spec, or if they're required by some existing feature.. right?

 - C'mon.. I can't even remember the code I wrote yesterday

 - It's unreasonable to expect somebody to remember two lines written
   over 3 years ago.

 - And even if they could, in general, we can't expect all previous
   devleopers to be available for questioning.

 - At some point, we simply cannot rely on "oral tradition".

 - We cannot expect to answer these questions using anything more than
   the code (and documentation) at hand.

## 3. The Five W's

 - If I'm going to change these lines, I need to thoroughly understand
   them.

 - I need to understand their intentions.. their motivations.. and the
   expectations that are place on them.

 - In short, I need to know "The Five W's" about this two line addition
   made over 3 years ago.

---

# Information Gathering

<br />

```
> The Five Ws are questions whose answers are considered basic in
> information-gathering or problem-solving.
```

.footnote[<sup>\*</sup>https://en.wikipedia.org/wiki/Five_Ws]

???

# Information Gathering

 - "The Five W's" are:

    1. Who?

    2. What?

    3. When?

    4. Where?

    5. Why?

 - Lets look at each of these, in the context of this issue at hand:

    - Who?

       - Who authored this commit?

       - This is part of the commit.. the "author" and/or "committer"
         fields

    - What?

       - What is the change?

       - Again, part of the commit.. the diff

    - When?

       - When was the change made?

       - Also part of the commit.. the "author data" and/or "committed
         date"

    - Where?

       - Where was this change made?

       - I sense a pattern here...

       - You guessed it! Again, part of the commit.. the repository.

    - And finally, we have Why?

       - Why was this change made to begin with? Why does it do what it
         does?

       - Unfortunately, this answer is nowhere to be found.

       - It's not in Bugzilla...

          - Bugzilla doesn't even exit anymore; been replace by JIRA

       - It's not in JIRA...

          - Which of the 11 tickets do I look at?

          - Doesn't matter, none contain any information.

       - It's not in ReviewBoard...

          - The code review can't even be found.

          - But even if it could, would there have been useful info to
            be found?

       - Sigh... :(

---

class: center, middle

# Commit messages... Let's use them.

???

# Commit messages... Let's use them.

 - If you couldn't already tell wehre I'm going with this...

 - The `git` commit message is the "right" place to answer "why?"

 - Not only are all other "four W's" there...

 - But it was the only piece of data that wasn't readily accessible for
   such an old change.

 - Bugzilla has since been replace with JIRA...

 - JIRA didn't have any details...

 - ReviewBoard didn't even contain the review anymore...

 - As we grow, we're bound to have more instances like my previous
   example.

 - We need to start taking advantage of commit messages.

---

# Baby Steps

<br />

```
commit a8f6344fa0921599e1f4511e41b5f9a25c38c0f9
Author: Eli Rosenthal <eli.rosenthal@delphix.com>
Date:   Wed Mar 2 14:51:48 2016 -0800

    6672 arc_reclaim_thread() should use gethrtime() instead of ddi_get_lbolt()
    6673 want a macro to convert seconds to nanoseconds and vice-versa
    Reviewed by: Matthew Ahrens <mahrens@delphix.com>
    Reviewed by: Prakash Surya <prakash.surya@delphix.com>
    Reviewed by: Josef 'Jeff' Sipek <jeffpc@josefsipek.net>
    Reviewed by: Robert Mustacchi <rm@joyent.com>
    Approved by: Dan McDonald <danmcd@omniti.com>
```

???

# Baby Steps

 - As a way to drive change _now_, I propose we follow in the footsteps
   of illumos.

 - I propose we make use of "Reviewed by" and "Approved by" metadata in
   the commit message.

 - This isn't a replacement for full-fledged commit messages, but it is
   a step in the right direction.

 - Who reviewed a change, and who GK'd the change is valuable
   information

    - Sometimes the author is no longer available (e.g. leaves the
      company or project), but the reviewers and/or GK is still around
      for questioning; i.e. just like this commit, where the author was
      an intern.

 - Additionally, it reinforces who is knowledgeable in a given area, and
   makes this information more accessible.

 - It would hopefull also build pride in one's reviews and GK approvals.

    - I'm sure nobody wants to have their name marked as a reviewer or
      GK of defective code, and/or a commit that is reverted.

 - More importantly, this is a small, actionable change, where we can
   drive utility and value _now_ &ndash; _today_.

 - Obviously I hope this change would lead to a broader shift to using
   full-fledged commit messages, that explain the "why?" of a commit...

 - But if if we didn't go "all the way", this would still be useful on
   its own.
