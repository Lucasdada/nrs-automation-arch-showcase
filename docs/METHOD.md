# Method

Every candidate followed the same loop. The loop keeps changes small, verified,
and easy to review.

1. **Facts.** Read the code and write down what is actually there: the call
   sites, the duplicated rules, the measured size. No guessing. Facts that need
   digging are delegated, not assumed.
2. **Design interview.** Put the open decisions to the owner in one short,
   numbered round. Each question carries a recommendation. Questions that
   depend on an earlier answer wait for the next round.
3. **Implement.** Make the change in one behaviour-preserving step.
4. **Verify.** Run the type checks and the test suites for both the server and
   the client. Add tests for any new rule.
5. **Commit and push.** One commit per candidate, with a message that states the
   problem, the change, and any deferred work.

## Rules of thumb

- A module should have a **small interface and a lot of behaviour behind it**.
- **One owner per concern.** A request shape, a cache key, and a refresh have
  one place that defines them.
- **The interface is the test surface.** If a rule can be tested without a
  browser or a server, write it so it can be.
- **Measure.** A candidate reports a real number (lines, call sites, tests), not
  an opinion.
- **Keep behaviour.** The user sees the same product before and after.

## Verification used

- Server type check and server tests (Vitest).
- Client type check and client tests (Vitest).
- Prettier for formatting, run by the pre-commit hook.
