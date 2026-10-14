# Color Review Process – Salesforce metadata

Source for the Color Review (Pink / Red / Blue / Gold / Green) process on `Capture_Strategy__c`, as deployed to the **passportlabs--partial** sandbox on 2026-10-08. Companion to the "How To: Color Review Process" doc.

Deploy with `sf project deploy start --source-dir force-app --target-org <alias>`. Flows deploy active in the partial sandbox; if a target org deploys flows inactive, activate the latest version of each in Setup > Flows.

## Automation

| Flow | Trigger | What it does |
|---|---|---|
| **Sign-Off Stamper** | Capture Strategy, before save | Stamps the date on each sign-off box when first checked. Stamps a color's Completed Date automatically once its required sign-offs and deliverables are all in. Sets Pricing Locked when Blue completes. On a send-back (stage moved to an earlier color) increments Send Back Count and clears that color's and all later colors' sign-offs and Completed Dates (unlocking pricing if Blue is reopened). |
| **Color Review Engine** | Capture Strategy, after save, only when stage / deadline / color stage changes | Computes Target Dates from `Color_Review_Config__mdt` (Target Offset) whenever a bid in Response In Progress has a Submission Deadline. When a color opens, sets its Due Date and Active Due Date from the SLA in the same metadata. Writes those in one update, then notifies on color open, completion, and deadline moves. |
| **Color Review Reminders (Daily)** | Scheduled, 8:00 AM Eastern | Reminder the day before the active color closes, overdue notice the day after. Replaces the retired record-triggered *Color Review Reminders* flow. |
| **Color Review Notify** | Subflow | Posts to the record's Chatter feed and, when `Slack_Channel_Email__c` is set, emails the same text to the bid's Slack channel. Failures never block the save. |
| **Add Business Days** | Subflow | Adds / subtracts business days, skipping weekends and Setup > Holidays (exact-date and yearly fixed-date holidays). |

## Gates (validation rules, all active)

- **No_Skipping_Colors** – a color can only open once the previous one has a Completed Date (Pink can be skipped via *Pink Skipped*). Blue also requires every RFP Requirement to be Validated or have a documented exception.
- **Stage_Exit_Requires_Complete** – a bid cannot move from Response In Progress to Submitted / Shortlisted / Awarded until Color Review Stage is Complete. Cancel and no-bid stages are not blocked.
- **Post_Blue_Pricing_Lock** – after Blue completes, pricing fields need a Change Justification plus Leadership Change Approval.
- **Exception_Notes_Required** – Documented legal exceptions need notes.
- **Sign_Off_<Role>_Only** (9 rules) – only a user holding that role's custom permission can *set* a sign-off box. Anyone can clear one.

`Color_Review_Bypass` (custom permission `Bypass_Color_Review_Gates`) skips every gate above. Admins only.

## Permission sets

- **Color Review Access** – object/field access for the proposal team.
- **Color Review Signer – PropOps / Sales / SE / CS / Leadership / Legal / Marketing / ELT** – one per role; assign to the people who sign in that capacity. Red's Sales/CS box accepts either Sales or CS. ELT is only required on bids of $500k+.

## Required sign-offs for auto-completion

| Color | Sign-offs | Deliverables |
|---|---|---|
| Pink | PropOps, Sales, SE | Win themes, proposal doc link, content owners assigned |
| Red | PropOps, Sales/CS, SE, Legal | Comments resolved, legal exceptions set, all RFP Requirements addressed |
| Blue | PropOps, Sales Leadership | Bid price sheet link, pricing approved |
| Gold | PropOps, Sales/CS Leadership, ELT if $500k+ | Transmittal letter link, risks accepted, executive approval |
| Green | PropOps | Formatting, page limits, file names, links, pricing-matches checks |

CS, Leadership (Pink), Marketing and Non-author boxes are optional. A Completed Date can still be entered manually to override.

## Operational notes

- Add company holidays in Setup > Holidays so SLAs skip them. Only exact-date and yearly fixed-date holidays are honored.
- For Slack, enable *Send emails to this channel* on the bid's channel and paste the address into **Slack Channel Email** on the record.
- `Color_Review_Config__mdt` drives SLA days and Target Offsets per color; the Engine reads both.
- Completed Dates and sign-offs are stamped by automation; a send-back clears them so reviewers re-sign.

## Conditional formatting (native, no package)

Two rulesets in `uiFormatSpecificationSets/` replace the emoji indicators on the record page:

- **Color Review Stage Indicator** on Color Review Stage: gray circle (Not Started), purple (Pink, the palette has no pink), red, blue, orange (Gold, no yellow available), green, and a green thumbs-up at Complete.
- **Color Review Due Status** on Active Due Date: green happy face (on track), orange neutral face (closes today or tomorrow), red sad face (past due). Driven by the `Days_To_Active_Due__c` formula.

Available icons in this org: circle, square, diamond, heart, rocket, thumbs_up, thumbs_down, question_mark, happy_face, neutral_face, sad_face. Colors: red, blue, green, orange, purple, gray.

The page attaches each ruleset through a field-instance property on the FlexiPage, `conditionalFormatRuleset`, whose value is the ruleset API name. Both attachments are in `flexipages/Capture_Strategy_Record_Page1`. Deploy the rulesets together with, or before, the page.
