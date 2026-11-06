# Test results

All data is fictional. I wrote the expected article and the key fact for every question before running the test.

## Method

`Knowledge Assistant Eval` reads the 15 questions from the `KA Eval Questions` table and sends each one through the real `Knowledge Assistant` workflow as a sub-workflow, including the real Claude call. It then compares the reply with the expected result and saves the totals in `KA Eval Runs`.

| Measure | Counted as correct when |
| --- | --- |
| Retrieval hit | the expected article is among the 4 retrieved chunks |
| Correct citation | the question was answered, every cited chunk was retrieved, and at least one cited chunk is from the expected article |
| Key fact | the answer contains the expected fact (text match, not case-sensitive) |
| Correct refusal | a question the docs do not cover got "I don't know", for a reason other than a failed AI call |
| False refusal | a question the docs do cover got "I don't know" |

## Totals from `KA Eval Runs`

| Run (Europe/Warsaw) | Questions | Retrieval hits | Correct citations | Key fact | Correct refusals | False refusals | By threshold | By AI | Mistakes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-11-06 21:44 | 15 | 10 | 10 | 10 | 5 | 0 | 3 | 2 | none |
| 2026-11-06 21:45 | 15 | 10 | 10 | 10 | 5 | 0 | 3 | 2 | none |

## Per question, first run

| ID | Question | Expected | Top score | Retrieved chunks | Status | Cited |
| --- | --- | --- | --- | --- | --- | --- |
| Q01 | How long does it take for the integration status to turn green after I install the script? | D01, "10 minutes" | 0.671 | D01-3, D01-2, D06-3, D01-1 | answered | D01-3 |
| Q02 | What do I need to set up push notifications for our iPhone app? | D03, ".p8" | 0.496 | D03-3, D02-3, D03-2, D14-3 | answered | D03-2 |
| Q03 | How often are dynamic segments recalculated? | D04, "15 minutes" | 0.655 | D04-1, D04-2, D04-3, D05-3 | answered | D04-1 |
| Q04 | How many custom attributes can one subscriber have? | D05, "50" | 1.000 | D05-3, D04-2, D05-2, D05-1 | answered | D05-2 |
| Q05 | What are the default quiet hours in automations? | D06, "22:00" | 1.000 | D06-2, D12-3, D06-3, D07-2 | answered | D06-2 |
| Q06 | When should the first abandoned cart reminder go out? | D07, "1 hour" | 1.000 | D07-2, D07-3, D07-1, D01-3 | answered | D07-2 |
| Q07 | How is CTR calculated in campaign reports? | D08, "delivered" | 0.625 | D08-1, D08-3, D08-2, D09-1 | answered | D08-1 |
| Q08 | How much is the Growth plan per month? | D10, "49" | 1.000 | D10-1, D04-3, D10-2, D08-3 | answered | D10-1 |
| Q09 | What happens to our data after we cancel the plan? | D11, "30 days" | 0.702 | D11-3, D09-2, D12-2, D10-3 | answered | D11-3 |
| Q10 | Who is allowed to invite new users to our account? | D14, "Admin" | 0.636 | D14-2, D14-1, D11-2, D03-1 | answered | D14-2 |
| N01 | Can NotifyHub send SMS messages to customers? | I don't know | 0.217 | D07-1, D09-1, D14-1, D06-2 | no_match | |
| N02 | Is there a limit on the number of automations? | I don't know | 0.622 | D05-2, D08-1, D08-2, D02-2 | not_covered | |
| N03 | What uptime SLA do you guarantee? | I don't know | 0 | none | no_match | |
| N04 | Can I limit how many notifications a subscriber gets per day? | I don't know | 0.483 | D09-2, D10-2, D10-1, D06-3 | not_covered | |
| N05 | Is NotifyHub HIPAA compliant? | I don't know | 0.134 | D01-1, D03-1, D01-2, D01-3 | no_match | |

N02 and N04 are the tricky questions. Both passed the 0.35 threshold because they share words with the docs ("limit", "number", "automations", "notifications"), so they went to Claude. Claude marked both as not covered. N02 is a good example of why the second layer matters: the best chunk was about attribute limits, which a careless model could have stretched into an answer about automations.

In Q02 the top chunk was about testing on a device (D03-3), but Claude cited the setup section (D03-2), which is the one that actually answers the question.

## Example answers, first run

| ID | Answer |
| --- | --- |
| Q01 | The integration status should turn green within 10 minutes of the first visit to your site after you install the script. If it stays grey after 10 minutes, clear the page cache and visit the site again. |
| Q02 | In your Apple developer account, create an APNs authentication key (.p8 file). In NotifyHub, go to Settings > Channels > iOS, upload the .p8 file, and enter your Team ID, Key ID and bundle ID. Then add the NotifyHub iOS library to your app and request notification permission. |
| Q08 | The Growth plan costs EUR 49 per month for up to 50,000 subscribers. |
| Q10 | Only an Owner or an Admin can invite new users, through Settings > Team > Invite. |
| N02 | I don't know based on the documentation. |

## Guard test: a made-up source

To check the citation guard without waiting for the model to make a mistake, I ran the main workflow once with a fixed fake AI reply that cited `D04-1` and `D09-2`. The search had retrieved `D04-1, D04-2, D03-3, D04-3`, so `D09-2` was not among them. Result: the answer was replaced with "I don't know based on the documentation." and logged as `citation_failed`, with the reason "cited chunks that were not retrieved: D09-2".

## First live call

Before the test runs, the first two chat questions failed at the AI step with "Bad request" because of a temperature setting the model does not accept. Both were answered with "I don't know" and logged as `ai_error`, so a broken AI call never produced a half answer. After removing the setting the same two questions were answered and refused correctly.

## Limits

- 15 questions on 43 chunks that I wrote myself. Real help centers are longer, messier and full of synonyms.
- The threshold was chosen by looking at the scores of these same questions.
- The key-fact check is a text match, and answer quality was read by hand, not scored.
