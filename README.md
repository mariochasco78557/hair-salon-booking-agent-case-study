A voice booking agent for a hair salon: 407 calls and one silent bug
====================================================================

Since September 2026 my AI agent has been answering the phone of a hair salon and booking into its real calendar. By late October: 407 calls, 212 phone bookings. Below is how it works, where each number comes from and the three days in October when it failed in a way my tests didn't catch.

The problem
-----------

The owner is mid-haircut when the phone rings. Either the scissors go down or the booking is lost.

I wanted the phone answered in the owner's voice, with the booking in the calendar the salon already uses.

What the system does
--------------------

1. Phone: a voice agent answers in the owner's cloned voice and books straight into the salon's real calendar.
2. WhatsApp: a second agent on Meta's official API handles the same bookings in writing. It confirms, changes or cancels them and sends reminders.

The assistant follows a script. If the caller asks about something else, it answers briefly and goes back to the booking.

Built with TypeScript, Zod to check every request the model makes, ElevenLabs for the voice and Docker. Each client runs in its own container, so one client's data never sits next to another's.

The design rule: the code decides, the model drafts
---------------------------------------------------

The model is the part that talks: it understands the caller and puts the answer into words. Everything with consequences is decided by normal code that I wrote and tested: who is calling, which slots exist, whether anything gets written to the calendar and which message goes out.

A booking works like this:

1. The model asks my server for free slots.
2. The server checks the real calendar and returns only slots that exist. Each one comes with a signed code that expires soon, like a ticket with a date stamp.
3. To book, the model has to hand that code back. The server checks it again before it writes anything.

So the model can say something clumsy, but it can't book a slot that doesn't exist.

Part of this rule came from a mistake. On 10 October a caller asked for Thursday the 16th, but the 16th was a Friday. The model worked out the date itself, sent Friday to the server and then told the caller Thursday.

That same day I moved dates to the server. The model now passes on only what it heard, the weekday and the day number, and the server works out the date against the calendar. If the two don't match, the agent asks the caller which day they meant.

The numbers and where they come from
------------------------------------

| Number | Value | Where it comes from |
|---|---|---|
| Calls answered, September to late October 2026 | 407 | One count at the end of October |
| Phone bookings in the same period | 212 | The same count |
| First 2 days in production | 63 calls, 34 bookings | My count of those 2 days |
| Call length, first 2 days | About 1.5 minutes | The same count |
| Technical errors, first 2 days | 0 | The same count |
| Calls on the version with the bug, 20 to 22 October | 101 | Call records and raw transcripts |
| Tests on the production system | Over 5,700 | Private repo |
| Tests on the public template | 4,767 + 277, all passing | Public repo, linked below |

Two notes. I haven't re-run the end of October count, so treat 407 and 212 as a snapshot. And 0 technical errors means nothing broke in those 2 days; it says nothing about whether every booking had the right slot.

The failure: three days of silence (20 to 22 October)
-----------------------------------------------------

On 20 October I moved the agent to a new version on a newer model. Over three days it took 101 calls. In some of them the agent said "One moment" and then did nothing. The caller was left in silence on the line.

How I found the cause:

1. My counting script flagged 32 waiting phrases with no action after them. I read the raw transcripts of the 22 calls involved. 3 of the 32 were not failures, because the phrase came after the action. That left 29.
2. In 24 of the 29, the model's turn stopped at exactly 300 tokens. A token is a small piece of text, about part of a word, and 300 was the cap on each reply.
3. The cap came from an older version, built for the previous model. That model used 119 to 149 tokens to write a booking request. The newer one needed 315 to 350 in a simulation with real data. Over 300, the request was cut in half and the voice platform threw it away.

Why did bookings not stop? After a failed attempt, a backup model stepped in a moment later. It made 17 of the 24 bookings in those calls. In my totals the backup looked like a small share of turns. I have no count of my own for that share or for the delay, so I give no figure.

Why my tests missed it: the simulation on 20 October used fake booking codes of 14 characters; the real ones have 91. With short codes, a booking request stayed at 266 to 272 tokens, under the cap, so the tests passed.

The fix was one setting: the cap went from 300 to 1,000. The prompt and the voice stayed as they were. Before going live I tested it:

1. I copied the 18 test cases from 20 October with a fake code of 91 characters, the real size.
2. I added 4 cases that reproduce the failure, such as very short "yes" answers and two bookings in one call.
3. I ran the old version on the same cases, at the same time, to compare.

In the cases both versions ran, waiting phrases with no action went from 1 in 41 to 0 in 41. I also timed replies, but the 1.53 seconds (9 in 10, over 10 scenarios both versions ran) and the 2.49 seconds before come from different sets of scenarios, so I don't compare them. Invented times: 0 in both.

One more thing went into that version. If the calendar does not respond, the agent offers to pass the caller's message on to the owner.

It went live at 21:15 on 22 October, with the older versions kept ready as a way back. The next day it was still taking all the calls.

What you can reuse
------------------

1. If a mistake would cost a booking, take that decision away from the model. Mine heard the weekday and the day number correctly and still got the date wrong.
2. For anything that gets written, give the model options to pick from, never a blank to fill. In the October tests, with slots only from the server, the agent invented 0 times.
3. Test with data the real size. Fake codes of 14 characters against real ones of 91 hid the bug for 101 calls.
4. When you change the model, check every limit you set for the old one. A cap of 300 was fine for a model that used up to 149 tokens per booking and broke one that needed up to 350.
5. Read raw transcripts and watch the backup. The backup made 17 of 24 bookings while my totals made it look like a small share of turns. A backup that answers more than it should is the alarm.

What I have not measured yet
----------------------------

1. Cost per call: I don't have a figure.
2. A baseline: I don't know how many bookings the salon took by phone before the agent, so I can't say how many of the 212 it would have lost.
3. Booking quality: I haven't checked, call by call, that each booking had the right service and slot. The plan is a golden set: anonymised real calls with the expected booking for each, checked by code first and then by a model as judge, compared with a person's verdicts. Planned for January, not built yet.
4. WhatsApp: I only have a partial count, so it is not in the table.
5. Satisfaction: I have not surveyed the owner or the callers.
6. The October tests are text tests, one run per case. They don't measure speech recognition or real silences on the line, so the 0 in 41 is not a real-call figure.

Public template on fake data, same booking design: https://github.com/mariochasco78557/ai-booking-agent-template

Written by Mario Chasco, November 2026. https://www.mariochasco.com/en#sys-booking-agent
