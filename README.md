# Color Review Process – Salesforce metadata

Source for the Color Review (Pink / Red / Blue / Gold / Green) process on `Capture_Strategy__c`, as deployed to the **passportlabs--partial** sandbox on 2026-10-08. Companion to the "How To: Color Review Process" doc.

Deploy with `sf project deploy start --source-dir force-app --target-org <alias>`. Flows deploy active in the partial sandbox; if a target org deploys flows inactive, activate the latest version of each in Setup > Flows.

## Automation

| Flow | Trigger | What it does |
|---|---|---|
| **Sign-Off Stamper** | Capture Strategy, before save | For each color's milestone checkbox, stamps Approved By (the saving user), Approved Date and the color's Completed Date when checked; clears all three when unchecked. Locks pricing while Blue is complete. Counts backward stage moves in Send Back Count. |
| **Color Review Engine** | Capture Strategy, after save, only when stage / deadline / color stage changes | Computes Target Dates from `Color_Review_Config__mdt` (Target Offset) for bids in Response In Progress with a Submission Deadline. When a color opens, sets its Due Date and Active Due Date from the SLA. Writes those in one update, then notifies on color open, completion, and deadline moves. |
| **Color Review Reminders (Daily)** | Scheduled, 8:00 AM Eastern | Reminder the day before the active color closes, overdue notice the day after. |
| **Color Review Notify** | Subflow | Emails the bid's Slack channel (`Slack_Channel_Email__c`) and posts to Chatter as a fallback until Chatter is retired. Failures never block the save. |
| **Add Business Days** | Subflow | Adds / subtracts business days, skipping weekends and Setup > Holidays. |

## Milestones (Oct 8 Prop Ops decision: no per-person sign-offs)

| Color | Checkpoint field | Stamps |
|---|---|---|
| Pink | Strategy Aligned | Pink_Strategy_Aligned_By / _Date, Pink Completed Date |
| Red | Narrative Approved | Red_Narrative_Approved_By / _Date, Red Completed Date |
| Blue | Pricing Approved | Blue_Pricing_Approved_By / _Date, Blue Completed Date, Pricing Locked |
| Gold | Executive Approval | Gold_Executive_Approval_By / _Date, Gold Completed Date |
| Green | Final QC Complete | Green_QC_Complete_By / _Date, Green Completed Date |

Colors can be opened in any order and run in tandem. A color the bid moved past with no Completed Date is listed in **Skipped Colors**. The 36 legacy `*_SO_*` sign-off fields are no longer on the page or used by automation; delete them once the PIN team signs off.

## Gates (validation rules)

- **Stage_Exit_Requires_Complete** (active) – a bid cannot move from Response In Progress to Submitted / Shortlisted / Awarded until Pricing Approved and Final QC Complete are both checked. Cancel and no-bid stages are not blocked.
- **Slack_Email_Required_For_Review** (active) – a color cannot be opened until Slack Channel Email is set.
- **Post_Blue_Pricing_Lock** (active) – after Blue completes, pricing fields need a Change Justification plus Leadership Change Approval.
- **Exception_Notes_Required** (active) – Documented legal exceptions need notes.
- **No_Skipping_Colors** (inactive, kept for reference) – the strict order gate the team chose not to use.

`Color_Review_Bypass` (custom permission `Bypass_Color_Review_Gates`) skips every active gate. Admins only.

## Permission sets and reporting

- **Color Review Access** – object/field access for the proposal team. Milestone boxes are editable; the By / Date stamps are read-only and written only by the flow.
- **Color Review Reports / Color Review - Active Bids** – every bid in Response In Progress with stage, due status, skipped colors and who approved each milestone and when. Subscribe leadership to it weekly.

## Operational notes

- Add company holidays in Setup > Holidays so SLAs skip them. Only exact-date and yearly fixed-date holidays are honored.
- For Slack, enable *Send emails to this channel* on the bid's channel and paste the address into **Slack Channel Email** on the record. Opening a color requires it.
- `Color_Review_Config__mdt` drives SLA days and Target Offsets per color.

## Conditional formatting (native, no package)

Two rulesets in `uiFormatSpecificationSets/` replace the emoji indicators on the record page:

- **Color Review Stage Indicator** on Color Review Stage: gray circle (Not Started), purple (Pink, the palette has no pink), red, blue, orange (Gold, no yellow available), green, and a green thumbs-up at Complete.
- **Color Review Due Status** on Active Due Date: green happy face (on track), orange neutral face (closes today or tomorrow), red sad face (past due). Driven by the `Days_To_Active_Due__c` formula.

Available icons in this org: circle, square, diamond, heart, rocket, thumbs_up, thumbs_down, question_mark, happy_face, neutral_face, sad_face. Colors: red, blue, green, orange, purple, gray.

The page attaches each ruleset through a field-instance property on the FlexiPage, `conditionalFormatRuleset`, whose value is the ruleset API name. Both attachments are in `flexipages/Capture_Strategy_Record_Page1`. Deploy the rulesets together with, or before, the page.
