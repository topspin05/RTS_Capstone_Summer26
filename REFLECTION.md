# Final Reflection

## What I'd do differently

If I were starting over, I'd run every task's worst-case scenario under real timing measurement much earlier. Task D's original insertion sort in App 2 had a WCET (374 ms) that literally exceeded its own 200 ms period, and that went unnoticed until it was pointed out. In hindsight, the utilization math (U = ∑Cᵢ/Tᵢ) is cheap to compute and would have caught this immediately. A single task whose own execution time exceeds its own period is an obvious red flag once you actually do the math, but easy to miss if you're only watching the system run and it happens to "look fine" on a given pass.

I'd also design my background-load workloads with the simulator's limitations in mind from the start, instead of discovering them under time pressure. I lost real hours to a Task-D-style watchdog crash in App 3's load fixture that turned out to be Wokwi's CPU emulation speed varying with browser/host performance. The exact same code passed once and then failed repeatedly with nothing changed. If I'd understood earlier that Wokwi is a functional simulator, not a cycle-accurate hardware model, I would have budgeted real safety margin into my WCET assumptions from the beginning rather than tuning constants reactively after each crash.

Finally, I'd pick my final base application earlier in the process rather than mid-capstone. Switching from an integrated App 2/3/4 build to App 3 alone, once my professor confirmed submitting one app unchanged was allowed, was the right call for the time I had, but it meant re-deriving a demo plan, a script, and a README structure from scratch partway through. Locking in scope earlier would have saved that rework.

## What was harder than expected

The biggest surprise was how much of the actual engineering difficulty lived in figuring out why something wasn't behaving as predicted, not in writing the code itself. I didn't expect my loaded-mode and idle-mode latency numbers to come out nearly identical at first. It turned out to be a real, explainable result, not a bug: my background load's highest-priority task (the only one capable of preempting my bottom-half tasks) had such a small WCET-to-period ratio that the probability of any given button press actually colliding with it was under 1%. Across only 50 presses, catching that collision at all was close to a coin flip. That reframed how I think about "latency under load".

## Most valuable thing learned

The most useful habit I built this term was treating unexpected behavior as a signal to investigate the underlying mechanism, rather than a signal to just patch around it. Every real "bug" I hit this term had a specific, findable, explainable cause once I stopped guessing and instead measured, isolated variables one at a time, and reasoned from what the scheduler was actually doing rather than what I assumed it was doing.

