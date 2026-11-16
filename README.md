# A booking agent for a hair salon

The problem: the owner is mid-haircut when the phone rings. Either the scissors go down or the booking is lost.

What it does: since September 2026 my agent has been answering the salon's phone in the owner's cloned voice and booking into its real calendar. On WhatsApp, a second agent confirms or changes bookings and sends reminders.

Code decides; the model reads and drafts. The model talks to the caller, but it cannot invent a slot. My server only offers real slots, each with a signed code that expires soon, and it checks that code again before anything is written to the calendar.

The numbers: by late October 2026, 407 calls and 212 phone bookings. The first 2 days: 63 calls and 34 bookings, about 1.5 minutes per call, 0 technical errors.

One real failure: from 20 to 22 October I ran a new version on a newer model. In 101 calls, the agent sometimes said "One moment" then did nothing, and the caller was left in silence.

The cause was a cap on how long each reply could be: 300 tokens, kept from a version built for another model. The newer model needed more room for the booking request, so it was cut off. In 24 of the 29 silent moments, the reply stopped at exactly 300. My simulation missed it because its fake booking codes had 14 characters; the real ones have 91.

I raised the limit to 1,000, gave the test cases booking codes the real size and added four that reproduce the failure. In the same tests, the silent moments went from 1 in 41 to 0 in 41, and the fix went live on the evening of 22 October. What I kept from it: I test with data the real size, and I read the raw transcripts, not only the counters.

What I have not measured yet: I have no cost per call. I do not know how many bookings the salon took by phone before the agent, so I cannot say how many of the 212 it would have lost. Zero technical errors means nothing broke, not that every booking had the right slot. The 0 in 41 comes from text tests, not real calls.

What I would do next:
1. Build a golden set from real calls, anonymised: cases with the expected booking, checked by code first and then by a model as judge, compared with a person's verdicts.
2. Measure the cost per call.
3. Agree a baseline with the owner, so the 212 can be compared with something.

Public template, on fake data (4,767 + 277 tests, all passing): https://github.com/mariochasco78557/ai-booking-agent-template
All my systems: https://www.mariochasco.com
