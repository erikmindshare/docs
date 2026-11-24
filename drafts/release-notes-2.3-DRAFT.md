DRAFT, not published
Code read: origin/dev f19ade720e (2.3 candidate), origin/release/2.2 3c4b2f44e8 (released 2.2.0.7 source)
Date: 2026-11-23

# AutoSurvey 2.3 release notes (DRAFT)

<!--
Reviewer notes. Remove this block before publish.
- Method: source text only. I read the code with git show, git diff and git grep at the two shas above. I ran no test and read no org.
- Each entry has a src comment: path:line on origin/dev f19ade720e.
- release/2.2 is not an ancestor of dev (it holds 8 build-only commits). All counts are a two-tree diff: git diff origin/release/2.2 origin/dev -- src.
- Claims that the review file (rel23-claims-vs-code.md) marks FALSE or NOT BUILT are not in these notes. Where it marks PARTLY, the entry states the limit.
- The 2.3 scope can still change. These notes describe dev f19ade720e only. See "Open items before the cut" at the end.
-->

## Summary

AutoSurvey 2.3 adds capacity rules, WhatsApp invitations, a scheduled campaign email, a Results tab, logic tools, accessibility checks, brand tools and a new question editor. Most new features start Off. An admin turns each one on in AutoSurvey Setup, on the Features tab.

In 2.2, the Features tab had the name "Optional Capabilities". The cards are the same type of control.

## New features

### 1. Capacity rules

<!-- src: src/classes/AsaCapabilityRegistry.cls:508-516 (card "Capacity rules", defaultOn false); src/classes/AsaCapacityPolicy.cls:41-46 (isOn), :52-79 (counted responses), :163-176 (one-response key); src/classes/AsaCapacityGate.cls:15-28 (limit rules), :60-67 (start check), :112-123 (open time), :160-182 (option limit at submit), :496-505 (email-mode early answer); src/classes/AsaCapacityNotifier.cls:1-23, :40-41; src/classes/AsaPublishShareController.cls:3318-3331 (reopen clears limit); src/lwc/asaSurveySettingsTab/asaSurveySettingsTab.html:114-170; src/lwc/asaLifecycleModal/asaLifecycleModal.html:42-45; src/lwc/asaOptionLimitsPanel/asaOptionLimitsPanel.html:3-5; src/lwc/asaSuitePreviewer/asaSuitePreviewer.html:1128 -->

What it does:
- Set a time when the survey opens. Before that time, the survey link shows "This survey opens on ...". An internal author can still preview the survey.
- Close the survey after a number of counted responses. A counted response is Submitted, and it is not Test, not Ignored and not held by the Fraud & Quality Shield.
- Accept one response from each person. A person is one email address (from an Email address question) or one link value (a link parameter that you name).
- Limit how many people can choose each option of a question. The respondent sees "Full" on an option that has no place left.
- The survey owner gets a notice when a limit gets to 80 percent and to 100 percent. The notice is a Salesforce bell notification and one email. If the owner is not an active user, the active AutoSurvey admins get it.

Where to turn it on: AutoSurvey Setup > Features > card "Capacity rules". Then set the rules on the survey:
- Open time and one response for each person: survey Settings tab, "Responses" card.
- Response limit: the Close dialog, field "or after this many responses".
- Option limits: the question panel, "Limit each option".

Default: Off. With the card Off, no capacity check runs.

Limits:
- AutoSurvey checks the response limit when a response starts and when it completes. A response that started before the limit was full can still complete. Thus the count can go above the limit.
- AutoSurvey checks an option limit when the respondent submits.
- In email mode, the one-response check runs after the respondent enters the email address. The check needs the org key that the Settings save makes. Without that key, no check runs.
- AutoSurvey keeps a hash of the email address or the link value for the one-response rule. In email mode, the email address is also stored as an answer, as before.
- When you change the one-response rule, the rule gets a new version. Each earlier respondent can then respond again.
- When you reopen a survey that is full, AutoSurvey clears the response limit. The audit note records the old limit.
- While a survey has a response limit and the card is On, a direct update that sets a submission to Submitted is refused. This includes a customer Flow or Apex. See "Upgrade notes".

### 2. WhatsApp invitations and reminders (through Twilio)

<!-- src: src/classes/AsaTextSettings.cls:93-108 (5 conditions); src/objects/Setup_State__c/fields/WhatsApp_Enabled__c.field-meta.xml (defaultValue false); src/lwc/asaWhatsAppSetup/asaWhatsAppSetup.html:4, :19-43, :53-54, :164; src/lwc/asaTextMessagingSetup/asaTextMessagingSetup.html:282; src/classes/AsaWhatsAppTemplates.cls:1-5, :15-17; src/classes/AsaTwilioClient.cls:116-122 (send), :224-236 (template list GET); src/classes/AsaTwilioInboundHandler.cls:121-149 (STOP per channel, STOPALL) -->

What it does: AutoSurvey sends survey invitations and reminders by WhatsApp, through the WhatsApp sender of your own Twilio account. Each message uses a template that WhatsApp approved. You make the template in Twilio. Then you add it in AutoSurvey and say what each variable holds.

Where to turn it on: AutoSurvey Setup > Text Messaging tab > "WhatsApp messages" toggle. The Features card "Text messages" must also be On.

Default: Off.

WhatsApp sends only when all of these are true: the "Text messages" card is On, the sender is Twilio, the WhatsApp toggle is On, a WhatsApp sender number is set, and a Contact or Lead consent field is set.

Limits:
- A template variable can hold: first name, brand, survey name, survey link or fixed text. Exactly one variable must be the survey link.
- AutoSurvey sends only the Twilio content types "twilio/text" and "twilio/quick-reply".
- An opt-out is for one channel. STOP in WhatsApp does not stop SMS. STOPALL stops both channels.
- The consent field must record that the person agreed to get WhatsApp messages. AutoSurvey does not make that permission.

### 3. Send a campaign survey email later

<!-- src: src/classes/AsaScheduledSendService.cls:1-27 (design and limits), :38 (365 days), :41 (10 each run); src/lwc/asaCampaignDistribution/asaCampaignDistribution.html:595-611; src/classes/AsaScheduledJobs.cls:143 -->

What it does: On the campaign Send step, pick a date and time, then click "Schedule send". The hourly AutoSurvey job sends the email.

Where to turn it on: No Features card. It is on the campaign Send step for a user who can send.

Default: Available.

Limits:
- Email only. Campaign sends only.
- One open schedule for each campaign. A new schedule cancels the open one.
- The email goes out within about one hour after the time that you pick.
- You can pick a time up to 365 days ahead.
- One hourly run starts at most 10 scheduled sends. Other due sends wait for the next run.
- AutoSurvey saves the send request when you schedule it. The audience, the opt-outs and the suppressions are read on the send day.
- The send runs as the owner of the hourly job. The audit record names the admin who scheduled it.

### 4. Results tab

<!-- src: src/lwc/asaResultsPage/asaResultsPage.js:16-34 (layout and 3 tabs); src/lwc/asaResultsPage/asaResultsPage.html:96-131; src/lwc/asaCollectionSection/asaCollectionSection.html:3-6; src/classes/AsaPublishShareController.cls:3933-3946 (ranges and counts), :4066 (getResultsNumbers), :4573 (Test and Ignored excluded) -->

What it does: Each survey has a Results tab. It shows the response numbers (finished, started and low scores), a period control (last 90 days, last 30 days, all time or a month), and three tabs: "Questions", "All responses" and "Trends".

Where to turn it on: No Features card. It is on each survey.

Default: Available.

Limits: Test and Ignored responses are not in the numbers.

### 5. Logic tools

<!-- src: src/classes/AsaCapabilityRegistry.cls:531-539 (card "Logic tools", defaultOn false); src/classes/AsaLogicToolsController.cls:1-26, :32-34; src/classes/LogicPathTester.cls:10-25, :30; src/lwc/asaSuitePreviewer/asaSuitePreviewer.html:192-193 -->

What it does:
- Edit a Show If rule with AND, OR and groups.
- See the survey logic as a map: the start, each screen, each ending and the default completion page.
- Test one set of answers ("What if").
- Test each set of answers that the rules read ("Test all paths"). The test finds a path that gets to no ending.

Where to turn it on: AutoSurvey Setup > Features > card "Logic tools".

Default: Off. The release check runs with the card Off too.

Limits:
- "Test all paths" tests at most 256 answer sets. Above that number, it tests a fixed sample and tells you so.
- A multi-select question takes one option at a time in the test, so that test is not exact.
- The test stops at the CPU, heap or result limit and tells you that it is not complete.

### 6. Accessibility checks

<!-- src: src/classes/AsaCapabilityRegistry.cls:439-453 (card "Accessibility checks", defaultOn false, 2 sub-settings); src/classes/AsaAccessibilityCheck.cls:1-15, :70-79, :168-182 (release stop), :185-193 (2 contrast pairs), :120-131 (alt text only with the setting) -->

What it does: AutoSurvey checks the colour contrast of a survey design and the alt text of image rows. It shows the result on the Design page. It can stop a release that fails.

Where to turn it on: AutoSurvey Setup > Features > card "Accessibility checks". Two settings show on the card while it is On:
- "Stop a release that fails the accessibility check" (Off by default).
- "Images need alt text" (Off by default).

Default: Off.

Limits:
- The contrast check reads two pairs only: question text on the background, and answer text on the option card. The minimum is 4.5:1 (WCAG AA for text).
- AutoSurvey checks alt text only when "Images need alt text" is On, and only on image rows.
- AutoSurvey does not check a text label on each picture choice.
- These checks do not make a survey conform to WCAG. They are help for the author.

### 7. Brand tools

<!-- src: src/classes/AsaCapabilityRegistry.cls:455-464 (card "Brand tools", defaultOn false); src/classes/AsaBrandToolsGate.cls:1-27; src/classes/AsaBrandLockPolicy.cls:1-31; src/classes/AsaEmailBrand.cls:1-33 (invitation email, 1,500,000 bytes); src/classes/AsaPdfBrand.cls:1-27 (signed PDF, 150 KB JPEG); src/lwc/asaLogoJpeg/asaLogoJpeg.js:1-25; src/permissionsets/AutoSurvey_Brand_Tools.permissionset-meta.xml:12 -->

What it does:
- Save one brand kit and lock groups of it (colours, fonts, logo). While a group is locked, a survey change to that group must match the kit value.
- Put the survey logo and the button colour on the invitation email.
- Put the survey logo in the header of a signed PDF. With no logo, the header shows the organization name.

Where to turn it on: AutoSurvey Setup > Features > card "Brand tools".

Default: Off. While the card is Off, surveys, emails and PDFs keep the look that they have now.

Limits:
- The lock does not change a survey that exists. It refuses a later change that does not match the kit.
- With the card On and a group locked, a missing or bad kit refuses every change to that group.
- The email logo comes from the release of the invitation, not from the draft. The logo file must be 1,500,000 bytes or less.
- The PDF logo needs a JPEG copy. The Design page makes that copy in the browser when you upload the logo (at most 600 x 200 pixels and 150 KB). If there is no copy, the PDF shows the organization name.
- Only the button and the logo change on the email. The body text stays dark on white.

### 8. One question editor

<!-- src: src/classes/AsaCapabilityRegistry.cls:229-236 (card "One question editor", switch field One_Question_Editor__c); src/objects/Setup_State__c/fields/One_Question_Editor__c.field-meta.xml (defaultValue false); src/lwc/asaQuestionEditor/asaQuestionEditor.js:1-25; src/lwc/asaQePictureTypes/asaQePictureTypes.js:4-9; src/classes/AsaOneQuestionEditorActivation.cls:28 (ACTIVATING_VERSION = null) -->

What it does: The Questions tab shows one editor. The outline is on the left, the live preview is in the centre, and the edit panel is on the right. You can add pictures to the options of a single-choice or multiple-choice question.

Where to turn it on: AutoSurvey Setup > Features > card "One question editor". Reload open tabs after the change.

Default: Off. With the card Off, the Questions tab shows the List and Preview views, as in 2.2.

Limits: A Picklist or a Dropdown question shows no option pictures.

### 9. Check my survey

<!-- src: src/classes/AsaSurveyCheck.cls:1-18, :78-86 (gate); src/lwc/asaSuitePreviewer/asaSuitePreviewer.html:200-202; src/lwc/asaPublishModal/asaPublishModal.html:65 -->

What it does: A button finds problems in a survey before release: routing rule problems, duplicate option labels, a choice question with one or zero options, the same question text two times, a long question, all capitals, two questions in one, many questions and a long time estimate. It uses no AI. The release dialog shows the warnings. They do not stop the release.

Where to turn it on: AutoSurvey Setup > Features > card "AI you control". The user also needs the permission that the card gives.

Default: Off.

Limits: The wording rules read the default language only.

### 10. "AI and your data" table

<!-- src: src/classes/AsaAiTrustController.cls:1-12, :101, :112-121; src/lwc/asaSetupHome/asaSetupHome.html:1310; src/lwc/asaAiTrustTable/asaAiTrustTable.js:60-62; src/lwc/asaAiTrustTable/asaAiTrustTable.html:50-57; src/classes/AsaAiRoutes.cls (0 Shield routes) -->

What it does: AutoSurvey Setup shows one row for each AI route: what it sends, the model and the provider, the credential mode, and On or Off. It also shows the "store prompts and replies" setting, the last 20 AI generation records and recent AI setting changes.

Where to turn it on: AutoSurvey Setup > Features > card "AI you control".

Default: Off.

Limits (the table text is not complete in 2.3):
- The table lists "AI follow-ups" under "Features that use no AI". This is not correct. AI follow-ups send the question and the respondent's answer to an Einstein model.
- The Fraud & Quality Shield Einstein review is not in the table. It sends answer text only when AI is On and "AI reads answers" is On (gh#6502, on dev 4f22ae0415). <!-- src: src/classes/ShieldAiReviewer.cls:109-110 aiAllowed() = AsaAiRoutes.aiOn() && AsaAnswerAiGate.isAllowed(), at dev f1540aa8d0 -->
- The table says that with the store setting Off, AutoSurvey does not store prompts and replies. Queued survey generation stores them in all cases.

### 11. Start a survey from a file

<!-- src: src/lwc/asaNewSurveyFlow/fileRoute.js:1-13; src/lwc/asaNewSurveyFlow/asaNewSurveyFlow.js:1283-1295; src/classes/AsaPasteVerbatim.cls:26-28, :102-109, :132-148 -->

What it does: In New survey > Import, the questions file input takes .csv, .json, .txt and .md. A .txt or .md file opens the Paste way with the file text. A .csv or .json file goes to the importer, with no AI.

Where to turn it on: No Features card. It is in the New survey dialog.

Default: Available.

Limits:
- The browser refuses a .txt or .md file over 200 KB, and a .csv or .json file over 500 KB.
- When the "AI you control" path applies, the server refuses pasted text over 50,000 characters or 300 lines. A file under 200 KB can still fail at Build.

### 12. AI picture on the submitted page, and the "Allow AI image generation" setting

<!-- src: src/classes/AsaAiImageService.cls:27 (SETTING_LABEL), :88 (SUBMITTED), :100, :132; src/classes/AsaCapabilityRegistry.cls:368-385 (child of AI features, defaultOn true, no permission set); src/lwc/asaAiImageSettings/asaAiImageSettings.html:1-17; src/lwc/asaSetupHome/asaSetupHome.html:1290 -->

What it does:
- An author can make a picture for the submitted page with AI.
- An admin can turn AI images off with the "Allow AI image generation" setting.

Where to turn it on: AutoSurvey Setup > AI Models & Tiers tab > "Allow AI image generation". It is also the Features card "Allow AI image generation". It needs the "AI features" card On.

Default: On (while "AI features" is On).

Limits: The screen-out page has no AI picture.

### 13. Hosted AI key refresh

<!-- src: src/classes/AiKeyService.cls:15, :325-331 (failed key hash), :542, :571 (control plane request); src/objects/AI_Key_Cache__c/fields/Anthropic_API_Key__c.field-meta.xml; src/objects/AI_Key_Cache__c/AI_Key_Cache__c.object-meta.xml (Protected) -->

What it does: In hosted AI mode, when the hosted Anthropic key has no credit or is not valid, the org asks the AutoSurvey server for a different key and tries again. If every shared key is used up, the error says so.

Where to turn it on: No Features card. Hosted mode only. Your own key (BYOK) does not change.

Default: Active in hosted mode.

Limits: The org keeps the hosted Anthropic key in a protected cache for up to 60 minutes.

### 14. Shield Platform Encryption of Contact Email and Lead Email

<!-- src: src/classes/AsaPersonEmailQuery.cls:1-25 (modes), :36-38 (20 SOSL terms, 20,000 opt-out cap), :42-43 (35 s note); src/classes/AsaRecipientLanguage.cls:123-126 (not through this class) -->

What it does: An org can encrypt Contact.Email and Lead.Email with Shield Platform Encryption. AutoSurvey finds Contacts and Leads by email with dynamic queries, so the Shield field check does not block the encryption. The recipient picker lists and searches Contacts and Leads.

Where to turn it on: No Features card. It applies when you encrypt the field in Salesforce Setup.

Default: Active.

Limits:
- With the probabilistic scheme, Email is not filterable. AutoSurvey then uses the search index. A record saved in the last 35 seconds or so can be missing.
- With the probabilistic scheme, a part of an email address does not find a person.
- With the probabilistic scheme, a send stops when one object has more than 20,000 opted-out people.
- The recipient language lookup by email does not use this method. With the probabilistic scheme, the lookup fails, AutoSurvey logs a warning, and the batch gets no recipient language.

## Changes

<!-- src for each item is in the item comment -->

- The "Optional Capabilities" tab of AutoSurvey Setup is now "Features". <!-- src: src/lwc/asaSetupHome/asaSetupHome.html:1081; src/lwc/asaSetupHome/asaSetupHome.js:127 -->
- The card "Show survey suites" is now "Audiences and rounds". The switch does the same thing. <!-- src: src/classes/AsaCapabilityRegistry.cls:220-227 -->
- The card "AI images" is now the setting "Allow AI image generation". It is a child of "AI features" and assigns no permission set. The permission set "AutoSurvey AI Images" is deprecated. Do not assign it. <!-- src: src/classes/AsaCapabilityRegistry.cls:368-385; src/permissionsets/AutoSurvey_AI_Images.permissionset-meta.xml:15 -->
- The suspension banner says "subscription" and not "license". The rest of the text did not change. <!-- src: src/labels/CustomLabels.labels-meta.xml:33 -->
- CX Analytics shows one survey at a time. It does not pool a score across surveys. <!-- src: src/classes/AsaCxAnalyticsController.cls:14-15; src/lwc/asaCxAnalytics/asaCxAnalytics.html:7, :119 -->
- The survey review is a page in the workspace with three columns: "What you asked for", the questions, and "Where the answers go". <!-- src: src/lwc/asaDraftReviewPage/asaDraftReviewPage.js:1-15 -->
- A survey with no translation shows one language line. The full Languages card shows after the first translation. <!-- src: commit 202903abcb; src/lwc/asaLanguagesCard/asaLanguagesCard.html -->
- In the survey list, a survey with two or more forms shows "N forms". A click opens the form rows. <!-- src: commit 203dc9eaa6; src/lwc/autoSurveySuiteList/autoSurveySuiteList.js -->
- An email with the first question now shows the question text. A 0 to 10 scale shows in two rows of 44-pixel buttons, with the end labels below. <!-- src: src/classes/EmbeddedFirstQuestionRenderer.cls (diff vs 2.2: questionText, renderScoreRows, scoreEndLabels) -->
- "Email people" shows the email body as the email shows it, with a server preview. <!-- src: commit 0d36b0cd34 (gh#6086) -->
- A pasted address with no name gets the first name of the matched Contact or Lead in the text {first name}. <!-- src: commit 2858a98e91 (gh#6083) -->
- The survey question count in the survey list counts a grid as one question and a display row as zero. It agrees with the survey header and Review. <!-- src: commit 545e9a99fb (gh#6195) -->

## Security and data

### Counts (measured)

Command base: `git diff --name-status origin/release/2.2 origin/dev -- src` (1,233 paths: 403 added, 830 changed, 0 deleted).

| Item | 2.2 | 2.3 | New | Command |
|---|---|---|---|---|
| Objects (object-meta added) | | | 5 (4 custom objects, 1 platform event) | `grep -cE '^A.*\.object-meta.xml'` on the name-status list |
| Fields (field-meta added) | | | 72 (37 on the 5 new objects, 35 on objects that exist) | `grep -cE '^A.*/fields/.*field-meta.xml'` |
| Picklist values added to fields that exist | | | 3 | `git diff ... -- <changed field files> \| grep -E '^\+\s+<fullName>'` |
| Permission sets | 51 | 56 | 5 (25 more changed) | `git ls-tree --name-only <ref> src/permissionsets/ \| wc -l` |
| Custom permissions | | | 1 | name-status list |
| Custom labels | 364 | 370 | 6 (1 more changed text) | `git show <ref>:src/labels/CustomLabels.labels-meta.xml \| grep -o '<fullName>[^<]*' \| sort`, then `comm` |
| Named credentials | 11 | 12 | 1 | `git ls-tree --name-only <ref> src/namedCredentials/ \| wc -l` |
| External credentials | 9 | 9 | 0 | same, src/externalCredentials/ |
| Remote site settings | 17 | 17 | 0 | same, src/remoteSiteSettings/ |
| CSP trusted sites | 6 | 6 | 0 | same, src/cspTrustedSites/ |
| Features cards | 22 | 27 | 5 | `out.add(` count in `AsaCapabilityRegistry.rows()` at each ref |
| Apex classes (.cls added) | | | 85 (31 not `_Test`: 30 product classes and 1 test helper) | name-status list |
| Lightning web components (bundles added) | | | 37 | name-status list, `.js-meta.xml` added |
| Apex triggers | | | 2 | name-status list |
| Custom notification types | | | 1 | name-status list |
| Platform cache partitions | | | 1 | name-status list |

### New endpoint

- Named credential `Twilio_Content`, URL `https://content.twilio.com`. It uses the external credential `Twilio_External`, which exists in 2.2. AutoSurvey uses it only to read the list of your WhatsApp templates (HTTP GET). This is the only new endpoint since 2.2. <!-- src: src/namedCredentials/Twilio_Content.namedCredential-meta.xml:7-17; src/classes/AsaTwilioClient.cls:11, :224-236 -->

### New data sent to sub-processors

- Twilio (WhatsApp send, through the named credentials that exist in 2.2): the recipient number, the sender, the template id, and the template variables. A variable can hold the first name, the brand, the survey name, the survey link or fixed text. <!-- src: src/classes/AsaTwilioClient.cls:116-122; src/classes/AsaWhatsAppTemplates.cls:16 -->
- AutoSurvey server (hosted AI mode only): when a hosted key fails, the org sends the SHA-256 hash of the failed key and a failure code. It does not send the key text. <!-- src: src/classes/AiKeyService.cls:325-326, :542, :571 -->

### Guest access (site guest user)

- Two new guest permission sets:
  - "AutoSurvey Guest Request Center": the participant link page class, and Create on the event Participant_Code_Due__e. The "Participant Request Center" card gives it to the guest users of survey sites when On, and removes it when Off. <!-- src: src/permissionsets/AutoSurvey_Guest_Request_Center.permissionset-meta.xml:3, :9, :19-21 -->
  - "AutoSurvey Guest Text Webhook": the Twilio webhook class only (delivery reports and replies). <!-- src: src/permissionsets/AutoSurvey_Guest_Text_Webhook.permissionset-meta.xml:3, :10 -->
- "AutoSurvey Guest Access" loses these two classes. It loses Create on Participant_Code_Due__e. It gets Create and Read on the event Capacity_Notice__e. An org with public surveys only does not expose the webhook or the Request Center to the guest. <!-- src: git diff of src/permissionsets/AutoSurvey_Guest_Access.permissionset-meta.xml; dev lines 3-8, 323-334 -->
- Only a guest user of an active site that holds "AutoSurvey Guest Access" gets a feature guest set. A switch Off removes the set from those guest users only. <!-- src: src/classes/AsaGuestFeatureAccess.cls:10-19 -->
- After Twilio was set up once, the webhook set stays when the text sender goes Off. Twilio still posts STOP replies. <!-- src: src/classes/AsaGuestFeatureAccess.cls:24-31 -->

### New objects

| Object | Type | Sharing | Granted in |
|---|---|---|---|
| Capacity_Lock__c | Custom object | Private | No permission set (system use) |
| Capacity_Notice_Log__c | Custom object | Private | No permission set (system use) |
| Capacity_Notice__e | Platform event, High Volume, Publish After Commit | n/a | AutoSurvey_Admin, AutoSurvey_Guest_Access, Member_Runtime, Runtime_User, Super_User |
| Scheduled_Send__c | Custom object | Private | AutoSurvey_Admin, Super_User |
| Survey_Message_Template__c | Custom object | Private | AutoSurvey_Admin, Super_User |

<!-- src: each object-meta.xml (sharingModel, eventType, publishBehavior); git grep "<object>NAME</object>" origin/dev -- src/permissionsets -->

The capacity notice handler does not trust the event fields. It reads the level, the limit and the count from the lock record. <!-- src: src/classes/AsaCapacityNotifier.cls:16-23, :33-37 -->

### New fields on objects that exist (35)

- AI_Key_Cache__c (protected custom setting): 3 fields for the hosted Anthropic key cache.
- Form_Question__c: Option_Limits__c, Visibility_Rule_Source__c.
- Publish_Release__c: Context_Block__c, First_View_Intro__c.
- Publish_Release_Form__c: Context_Block__c, First_View_Intro__c, Option_Limits__c.
- Setup_State__c: One_Question_Editor__c and 6 WhatsApp fields.
- Suite__c: Context_Block__c, First_View_Intro__c, One_Response_Per__c, Opens_At__c, Response_Limit__c.
- Survey_Draft__c: Review_JSON__c.
- Survey_Phone__c: Opt_Out_Review_Hold__c, WhatsApp_First_Message_Id__c, WhatsApp_User_Id__c.
- Survey_Reminder_Schedule__c: WhatsApp_Template__c.
- Survey_Submission__c: Embedded_Attempt_State__c, Embedded_Attempt_Value__c, Path_Record__c, Response_Key__c, Response_Key_Pending__c.
- Survey_Text_Message__c: Content_Sid__c, External_User_Id__c, Message_Template__c.

Response_Key__c holds an HMAC-SHA256 hash of the one-response value. It does not hold the email address or the link value. <!-- src: src/classes/AsaCapacityPolicy.cls:157-176 -->

New picklist values: AI_Image__c.Purpose__c gets SUBMITTED (restricted picklist). Survey_Text_Message__c.Review_Kind__c gets OptOutUnmatched and OptOutScope.

### New permission sets for admins (3)

The Features card owns each set. It assigns the set to the AutoSurvey admins when On and removes every assignment when Off. Do not assign these sets by hand.
- "AutoSurvey Capacity Rules" (card "Capacity rules"). <!-- src: src/permissionsets/AutoSurvey_Capacity_Rules.permissionset-meta.xml:3 -->
- "AutoSurvey Logic Tools" (card "Logic tools"): the logic tools controller only. <!-- src: src/permissionsets/AutoSurvey_Logic_Tools.permissionset-meta.xml:4, :7 -->
- "AutoSurvey Brand Tools" (card "Brand tools"): the custom permission AutoSurvey_Brand_Tools_Manage and the brand tools controller. No object access. <!-- src: src/permissionsets/AutoSurvey_Brand_Tools.permissionset-meta.xml:5, :10, :12 -->

### Fixes with a data effect

- Send Invitation (Flow action): with more than one email group past the queue limit, the text of one record could go to another recipient. 2.3 sends each group in its own job, with its own sender, subject and body. This fault is also in 2.2.0.7. <!-- src: src/classes/AsaInvitationSendChain.cls:1-15; commit a55de01194 (gh#6370) -->
- A change to the public link of a survey (Invite only, pause, resume) now needs edit access to the survey through sharing, as Close does. <!-- src: commit fe6b2889c0 (gh#6152): AsaSecurity.cls, AsaAccessModeController.cls, AsaPublishShareController.cls -->
- The text consent field pickers do not offer fields that mean the opposite (for example Do Not Call or Email Opt Out). The save refuses them. <!-- src: commit c35efae796 (gh#6081): AsaTextSetupController.cls -->
- A survey delete clears the recipient address on the reminder log records of the survey before it deletes the invitations. <!-- src: commit 9cde7ca4dd (gh#5362): AsaSuiteDeleteService.cls -->

## Upgrade notes

### What the upgrade job does

<!-- src: src/classes/AutoSurvey_PostInstallHandler.cls:384-400 (guest align), :418-431 (editor activation); src/classes/AsaGuestFeatureAccess.cls:44-69, :269-311; src/classes/AsaOneQuestionEditorActivation.cls:28 -->

- The steps of 2.2 run as before.
- New step: if the Participant Request Center is On, or Twilio texts are set up, the upgrade starts a queued job. The job gives each needed guest set to the guest users of active survey sites that hold "AutoSurvey Guest Access". The job only adds. It runs in system mode, because the install user cannot assign permission sets in user mode. If it fails, the job ends Failed and the log names the set.
- New step: the one question editor activation. In this version it writes nothing. The editor stays Off.
- The five new Features cards start Off. The upgrade gives no new permission set to any user.

### What an admin must do after the upgrade

1. Open AutoSurvey Setup and find the check "Feature Guest Access". If it fails, click "Give guest access". Do this if you use text messages through Twilio or the Participant Request Center. Without the webhook guest set, Twilio posts get an error and AutoSurvey does not record STOP replies.
2. Turn on the new features that you want, on the Features tab. Each new card starts Off.
3. For WhatsApp: set the WhatsApp sender, the consent field and the templates on the Text Messaging tab. Then turn on "WhatsApp messages".
4. If a Flow or Apex of your org sets a Survey Submission to Submitted, test it before you set a response limit. With "Capacity rules" On and a response limit set, that direct update is refused.
5. If you translate AutoSurvey labels, translate the 6 new labels: Survey_Already_Responded, Survey_Not_Yet_Open, Survey_Option_Full, Survey_Option_Full_Tag, Survey_Restore_Key_Held, PRC_Panel_FeatureOff.
6. Do not assign the deprecated permission set "AutoSurvey AI Images".
7. If the Fraud & Quality Shield AI review is On in your org, turn ON "AI reads answers" (it starts Off). Until you do, the Shield sends no answers to Einstein: new review events get the status NotRun. Queued events are sent when both switches are On. (gh#6502, on dev 4f22ae0415.)

No action is needed for the Platform Cache partition "QuestionEditor". The code works with no cache room: a miss reads the live data. <!-- src: src/cachePartitions/QuestionEditor.cachePartition-meta.xml:3; src/classes/AsaQuestionEditorSnapshot.cls:23, :143, :157 -->

## Known limits

- Capacity rules: a response that started before the limit was full can complete, so the count can go above the limit. See feature 1.
- Capacity rules: a direct update that sets a submission to Submitted is refused while a limit is set. <!-- src: src/classes/AsaCapacityGate.cls:25-28 -->
- WhatsApp: an opt-out is for one channel. STOP in WhatsApp does not stop SMS. <!-- src: src/classes/AsaTwilioInboundHandler.cls:121-149 -->
- Scheduled send: email and campaign only; at most 10 sends start in one hourly run.
- Accessibility checks: two contrast pairs only; alt text only with "Images need alt text" On.
- "AI and your data" table: three statements are not complete. See feature 10.
- Shield-encrypted Email: the probabilistic scheme has the limits in feature 14.
- The suspension banner says that creating surveys is disabled. The org does not block survey creation. <!-- src: src/labels/CustomLabels.labels-meta.xml:33; measured 0 hits for "EntitlementService" in src/classes/AutoSurveySuiteManagerService.cls and src/classes/AsaSurveyDraftController.cls (review F2). Reviewer: change the label or the code before publish. -->
- Two Survey fields, "New first view" (First_View_Intro__c) and "About the record" (Context_Block__c), are in the package. No AutoSurvey screen sets them. The receipt on the last screen shows only when "New first view" is checked. <!-- src: git grep for First_View_Intro__c|Context_Block__c in src/lwc, src/layouts, src/flexipages: runner only; src/lwc/asaSurveyRunner/asaSurveyRunner.js:6387-6391 -->
- The Features card "Anonymous surveys" is in development. No survey can be marked Anonymous. <!-- src: src/classes/AsaCapabilityRegistry.cls:412-413 -->

## Not in this release

- Interview Mode. It is not in the 2.3 code. <!-- measured: git ls-tree -r --name-only origin/dev src | grep -ic interview = 0 -->
- Bounce tracking. <!-- tracker row R13: "not on dev"; git log origin/dev --grep gh#5291 finds only a test-list commit (ca35a7e55d) -->
- An AI picture for the screen-out page. <!-- src: src/classes/AsaAiImageService.cls:100, :132 (no SCREEN_OUT purpose) -->

## Not verified

These facts are left out of the notes above, because I could not verify them in the code:

1. The behaviour of the Platform Cache partition "QuestionEditor" (1 MB of org cache in the metadata) in a subscriber org. I did not check whether the package gets that capacity or whether the install needs free cache.
2. Whether the upgrade removes the 2.2 assignments of the deprecated set "AutoSurvey AI Images".
3. Whether a logo uploaded before 2.3 has the JPEG copy for the signed PDF, and so whether an admin must upload the logo again.
4. Whether an author (not only an admin) can edit the brand kit record, and whether the logo lock reads Suite__c.Logo_URL__c (review P27 b and c).
5. Whether option limits are checked on import, restore or direct update (review P18), and whether a capacity notice is sent after an import or a restore (review P19).
6. The cap of 100 rows on the low-score list of the Results tab, and the field-access rule for names on that tab.
7. The retention of WhatsApp and text rows (unmatched rows after 7 days, conflict rows after 90 days).
8. That the org sends token counts to the AutoSurvey server (review section 6, item 6). I verified only the failed-key hash.
9. The number of Show If rule kinds in the Logic tools ("9 rule kinds" in the review).
10. That the Design page refuses a colour below AA at save (review P26 says browser only).
11. Whether a .txt or .md file goes to the Paste way when the "AI you control" path does not apply.
12. The sweeper interval for the WhatsApp card ("every 15 minutes", review section 8).
13. The two-business-day review time for a stop sentence, and the daily owner email (review P21).

<!--
Open items before the cut (from the Feature Tracker rec23 and git log; not customer text). Remove before publish.
- 0 commits on origin/dev for: gh#6366 (a new restricted picklist value can be missing after an upgrade; 2.3 adds SUBMITTED on AI_Image__c.Purpose__c), gh#6355 (upgrade on an org with many surveys), gh#6386, gh#6387, gh#6342, gh#6377, gh#6378 (defects in new 2.3 features), gh#6119, gh#6108 (signed, not on dev). Measured: git log --oneline origin/release/2.2..origin/dev --grep="gh#NNNN\b" | wc -l = 0 for each.
- gh#6370 (Send Invitation) IS on dev: a55de01194.
- Upgrade fault A2 (guest STOP can be lost if the align job hides a refused Site read): no ticket in the tracker; no beta upgrade proof yet.
- Decisions open in the tracker: the One question editor ships Off before Erik's own test (board tracker-editor-ships-off-1123).
- If Interview Mode merges before the cut, these notes need a new read of the code.
-->
