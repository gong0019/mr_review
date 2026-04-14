# MR Review Report Format

## Confirmed Bugs

List bugs with:

- affected file and line number
- root cause
- impact
- suggested fix direction

This section comes first and should be the main focus of the review.

## Risks / Edge Cases

List non-bugs that still have hidden runtime, rollout, or maintenance risks.

Do not mix hypothetical risks into `Confirmed Bugs`.

## Verification

State explicitly:

- what was verified by static review only
- what tests were reviewed or should be run
- what remains unverified and needs runtime or manual checking

## Summary

Give a one-line overall assessment.
