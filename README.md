# word ladder: How BFS Turns It Into a Graph Problem, and Why Bidirectional Search Is the Version Interviewers Actually Want

Search "word ladder" and you land in one of two places. One is the 150-year-old puzzle where you connect COLD to WARM by changing one letter at a time. The other — the one most people clicking these results actually need — is LeetCode 127, a hard-rated graph problem that shows up in coding interviews because it tests whether you can spot a graph that isn't drawn as one.

This article walks through the problem as it's actually asked, the BFS approach that works, the bidirectional version that's faster, the messier follow-up (Word Ladder II), and the traps that cost people offers. Then it covers where to practice it if you'd rather not grind it alone.

## What the word ladder problem actually asks

AlgoMonster states it plainly in its own lesson: given a start word, an end word, and a list of dictionary words, find the minimum number of steps to get from the start word to the end word using only words from the dictionary, changing exactly one letter per step.

Their example:


start = "COLD"
end = "WARM"
word_list = ["COLD", "GOLD", "CORD", "SOLD", "CARD", "WARD", "WARM", "TARD"]

COLD → CORD → CARD → WARD → WARM  = 4 steps


LeetCode's version uses `beginWord = "hit"`, `endWord = "cog"`, and `wordList = ["hot","dot","dog","lot","log","cog"]`, returning 5. Same problem, different words.

Two details matter more than they look:

- Both endpoints are known before you start. That single fact is what unlocks the faster algorithm later.
- Every intermediate word must be in the dictionary. You can't invent "CORL" just because it's one letter from COLD.

## Why this is a graph problem wearing a word costume

There's no adjacency list here. The graph is implicit: each word is a node, and an edge exists between two words if they differ by exactly one letter. You have to construct neighbors on the fly, which is where most of the difficulty lives.

Once you see it as a graph, the rest follows from a rule worth memorizing: **BFS finds shortest paths in unweighted graphs.** Every transformation costs 1 step, so the first time BFS reaches the end word, you've found the minimum. Depth-first search would happily find *a* path and then keep you guessing whether a shorter one exists.

## The standard BFS approach

The work is in the neighbor function. AlgoMonster's editorial loops over every position in the current word and substitutes each of the 26 letters, checking each candidate against the dictionary — an approach its own community discussion has argued about at length, with some readers preferring to scan the word list for prefix/suffix matches instead.

Here's the shape of the straightforward version:

python
from collections import deque

def word_ladder(begin, end, word_list):
    words = set(word_list)
    if end not in words:
        return 0                     # early exit, see the trap below

    queue = deque([(begin, 1)])
    while queue:
        word, steps = queue.popleft()
        for i in range(len(word)):
            for c in "abcdefghijklmnopqrstuvwxyz":
                nxt = word[:i] + c + word[i+1:]
                if nxt == end:
                    return steps + 1
                if nxt in words:
                    words.remove(nxt)    # mark visited by deleting
                    queue.append((nxt, steps + 1))
    return 0


Two things in that code do real work:

1. **`words.remove(nxt)`** marks a word as visited. It's cheaper than maintaining a separate `visited` set and it prevents the same word from entering the queue twice.
2. **The early exit** for a missing end word. This is not optional.

### The pre-check that catches everyone

Almost every early solution to Word Ladder on LeetCode failed on this case:


start: "hit"
end: "cog"
wordList: ["hot","dot","dog","lot","log"]


`"cog"` isn't in the dictionary, so no transformation exists — the answer is 0. Solutions that seeded both endpoints without checking would still stitch together a path and return the wrong number. If you write one line of defensive code in this problem, make it `if end not in words: return 0`.

## The complexity argument, and why it's murkier than it looks

Two neighbor-generation strategies get compared constantly, and they behave differently depending on dictionary size:

| Strategy | Neighbor generation | Rough bound |
| --- | --- | --- |
| Scan the whole word list, compare letter by letter | O(N) comparisons per word, O(L) each | O(N² · L) |
| Replace each character with 26 letters, hash-lookup | O(26 · L) per word | O(N · L · 26) |
| Precompute wildcard buckets (`c_t`, `_at`, …) | Graph build is heavier, lookups are O(1) | O(N · L²) to build, then near-linear BFS |

The wildcard trick is the one worth knowing even if you don't ship it: map every pattern with one character blanked to the words that match, and neighbors become lookups instead of scans. It trades O(N·L²) upfront work for cheap traversal.

Which is "best" depends on N and L, and anyone claiming one is universally faster is guessing. For interview purposes, say the character-substitution version is O(26 · N · L) in the worst case and explain the trade-off.

## Bidirectional BFS: same answer, much less ground covered

This is the part that makes interviewers sit up.

Instead of expanding only from the start, run two searches — one forward from the begin word, one backward from the end word — and stop the moment the two frontiers meet. The standard reasoning: **b^(d/2) + b^(d/2) is dramatically smaller than b^d**, where b is the branching factor and d the depth. You're replacing one wide tree with two narrow ones.

The one non-obvious detail: expand the *smaller* frontier each round, not the two sides alternately. Expanding the small side keeps the explored region balanced.

python
def ladder_length(begin, end, word_list):
    words = set(word_list)
    if end not in words:
        return 0

    front, back = {begin}, {end}
    steps = 1

    while front and back:
        if len(front) > len(back):
            front, back = back, front        # always grow the smaller side
        nxt_level = set()
        for word in front:
            for i in range(len(word)):
                for c in "abcdefghijklmnopqrstuvwxyz":
                    candidate = word[:i] + c + word[i+1:]
                    if candidate in back:
                        return steps + 1
                    if candidate in words:
                        nxt_level.add(candidate)
                        words.remove(candidate)
        front = nxt_level
        steps += 1

    return 0


Note the `steps` initialization: the two frontiers start at 1 step each, so the first collision returns `steps + 1`. Off-by-one errors here are common — trace the COLD/WARM example by hand before you trust your index.

Also note the behavioral difference from the one-directional version. In bidirectional BFS both endpoints sit in the queues from the beginning, which means a missing end word wouldn't naturally stop the search. That's exactly why the `end not in words` guard moves from "nice to have" to "load-bearing."

## Word Ladder II is a different animal

LeetCode 126 asks for *all* shortest transformation sequences, and it's a harder problem dressed in the same words. The standard approach is two phases: BFS to build a predecessor map, then DFS backtracking from the end word to collect every shortest path. (Building successors and going forward tends to blow up — recording predecessors is the version that passes.)

The subtle part is what you record. Each word must only be enqueued once per level, but you still need *all* the edges at that level, because two different words can both be one letter from the same successor and both belong in the answer. That requires a buffer — a `currentLevelVisited` map that holds the level's discoveries and only merges into the global visited set once the level is finished. Skip the buffer and you'll silently drop valid paths.

## Mistakes that show up again and again

- **Forgetting the end-word check.** Covered above; it's the single most common failure.
- **Using DFS.** It doesn't give you shortest paths. Full stop.
- **Not marking words visited on discovery.** You'll revisit the same words across levels and either time out or return a number that's too large.
- **Expanding the wrong frontier** in the bidirectional version.
- **Returning the count of words instead of steps.** COLD → CORD → CARD → WARD → WARM is 4 transformations but 5 words in the chain. Decide which one the problem wants before you write `return`.

A clean word ladder solution doesn't require cleverness. It requires correctly identifying an implicit graph, choosing BFS, and not tripping over the bookkeeping.

## Practicing it without re-deriving the pattern from scratch

Word Ladder sits inside a pattern rather than standing alone: BFS on an implicit graph, where "neighbors" are generated by mutation instead of read from an adjacency list. Recognize the pattern and problems like minimum genetic mutation, open the lock, and similar shortest-transformation questions stop looking new.

AlgoMonster files Word Ladder under Graph and structures its lessons that way — pattern first, then the specific problem, with a template you adapt rather than a wall of explanation. Its free tier covers a limited subset of patterns and problems, which is enough to test whether the text-and-diagram format works for how you learn. Everything else sits behind Pro, and the Pro checkout page at the time of writing showed:

- **Pro – Yearly:** $5.84/month, billed annually at $69.99/year
- **Pro – Lifetime:** $89, billed once

That includes all patterns, lessons and problems, unlimited AI assistant usage, Speedrun mode, company-tagged questions updated monthly, and 500+ lessons across 6 courses. Plenty of comparison blogs quote other figures — $99/$189, $119/$299, $459, whatever — and the ones I checked were inconsistent with each other and with the checkout page. Treat the live page as the source of truth, since AlgoMonster's promotional pricing moves around.

One claim worth flagging: several third-party reviews describe a money-back "job guarantee." I couldn't confirm that on AlgoMonster's own pages, so don't let it drive your decision.

## Full plan comparison

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to evaluate the format | $0 | — | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, company questions updated monthly, 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, member discount on live bootcamps | $69.99/year ($5.84/mo) | Annual, cancel or upgrade anytime | [See current yearly pricing](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates and free access to future content | $89 once | One-time | [Check the lifetime option](https://algo.monster?fpr=li69) |

No monthly plan exists — annual or lifetime, nothing in between.

## Annual or lifetime, for someone studying word ladder problems

The math is unusually easy here because the one-time price is small. At $89 lifetime versus $69.99 for a year, the two are within about $20 of each other. If you're anywhere near a second round of interview prep in the next two years, lifetime pays for itself almost immediately.

The case for annual: you're prepping for one specific loop in the next couple of months and you're confident you won't touch structured prep again afterward. Even then, saving roughly $19 to give up permanent access is a thin argument.

One thing to plan around — there are no video walkthroughs. It's text, diagrams, and an in-browser editor. If you learn by watching someone else's screen, this format will frustrate you regardless of price, and no plan tier fixes that.

## Where word ladder practice should fit in your prep

Word Ladder is worth studying because of the pattern, not the specific words. Learn BFS on an implicit graph properly and you're solving a family of problems. Memorize the COLD/WARM chain and you've learned one.

If you're working through it solo, the order that holds up: write the plain BFS version from scratch, verify it against the missing-end-word case, then rewrite it bidirectionally and trace the step counter by hand. Do Word Ladder II only after 127 is automatic — the predecessor-map bookkeeping is much harder to reason about if the base BFS is still shaky.

If you'd rather follow a structured path through the pattern, 👉 [see what's currently unlocked on AlgoMonster](https://algo.monster?fpr=li69), or start with the free tier and decide after you've worked through a few graph lessons. Either way, don't pay for a plan until you've confirmed you can learn from diagrams instead of video. That's the actual deciding factor here, more than the $20 gap between annual and lifetime.
