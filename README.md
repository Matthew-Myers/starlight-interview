# Starlight reduction in manual process

## Overview

Starlight has a hefty amount of manual processes.  We want to reduce said processes to streamline the application process, which in turn should result in lower cost per application, as well as a higher percentage of applicants reaching the end of the process.

We want to provide a 1 week, 1 month, and 3 month timeline on projects that would reduce the manual processes.

## Manual Processes
Some of the manual processes with some rough estimates with the time it would take a Starlight staff member to execute these tasks assuming a full async workload:

- Data Validation and Extraction: 10-30 minutes 
- Manual Data Re-entry (assuming data is parsed and validated): 5-10 minutes 
- Updating applicants via text (assuming Twilio integration): 90s per status update
- Prompting applicants for proper documents: 5-15 minutes between explaining documents and back and forth communication
- Context Switching between cases: 5-10 minutes
- Head turning between platforms: 5 minutes

## Workflow Thoughts and Baseline Assumptions
For the manual process assumptions above, we're logging around 30-60 minutes of human time per application.

It would not appear that heartshare has an accessible API from my snooping, so let's assume for now that our interface into heartshare will be forced to be a UI.

With the heartshare admin portal, it appears we can see multiple cases at once.  Each case will have an ID, an applicant name, approval or denied status (which appears to not work), as well as an approved amount, and a free form text denial reason.

For the denial reason, let's assume that this is actually freeform text and not a set of precanned statuses.

While having a human in the loop is critical, giving the user automated feedback initially with a combination of OCR and simple LLMS calls could greatly speed up the application process by eliminating the need for Starlight staff to reach out to the user.

Likewise, assuming the text updates are a manual (or templated) message, that could be automated with a webcrawler and simple db to track status of the application.

Let's also make the assumption that there is a internal DB projection of all the jotform submissions to the heartshare application number, as well as associating a point of contact w/ the applicant.  This will help the staff members know which applicants are where in the process.  Since Heartshare has no API, let's assume that the staff members are examining the dashboard each day and updating the status in said system when available.

Given the requirements of the assignment let's first start with the 1 week quick implementation:

# Week 1 Release

## Heartshare webcrawler

An automated crawler through the heartshare ui, running on a standard x hour cron, depending on how frequently heartshare updates cases.

To support our webcrawler, we'll need to create a new DB table/collection called notifications.  Assuming Starlight has no staff portal, we'll simply send out email notifications with a link and direct them to the heartshare case

The webcrawler will: 
- Login
- Naviage to admin panel
- Read the application number
- Create a view of the status
- Check internal DB for updates
- *IF* there is an update to be made which requires user feedback or is a denial, create a record inside of the "Notifications" db / collection
-- Create a simple email to staff member on said case w/ fields that changed as well as a link to the case
- *IF* the case is updated to accepted, send an automated templated message through twilio to update the user, reducing Staff member requirement for the status update
-- still create the notification incase there is additional workflows after the approval of the application

A very light weight data collection system.  Works around the lack of API from heartshare and creates timely notifications to staff members.

Drawbacks and considerations:
1. Assumes there is an assigned team member to the case in each application
2. Notifications are just emails, and can result in a case not being handled if the staff member does not read the email or act on the email.  This is not a replacement for a proper case management system
3. Assumes there is a twilio integration and not just staff members texting off their own phone
4. Not extensible to other savings programs, would have to be recreated per savings program

# 1 Month Release

## Jotform integration + Data Extraction + Initial Data Entry

Jotform allows us to create an API integration w/ a webhook for each time a form is submitted, as seen here: https://api.jotform.com/docs/#post-form-id-webhooks

Using this api and webhook, we can use the jotform fields to start filling out the form with a simple python library like iText.

Basing the fields we'd need and have available to us w/ the standard jotform from loom and the application template from heartshare found here: https://cdn.prod.website-files.com/6789852d1cc9577458580275/688a26f1df3e1acb0a142dc7_Energy%20Share%20Application%202025.pdf

14/22 Form Fields 100% automated from Jotform

## Starlight Staff interface

Building off of our notification system from above, each time a form submission finishes, we'll want to:
- take a first pass at rendering the pdf
- upload to an object store
- store a reference to the object store in our jotform projection db
- assign and send a notification the the starlight staff member of the new application w/ the intial filled out data
- from there the user can validate, edit, and continute to fill out the form

## Additional time savings

We can update the jotform form and extend it to explicitly ask for the con ed information to be entered by the user manually. There are pros and cons to making this change however.

Pros:
- 100% automated fields Heartshare application, meaning staff member just needs to validate user passed documents
Cons:
- Slightly more involved application process which may disuade some users
- Workflows are unique per program / application

At the end of the day, a solid A/B test would hopefully give us the information we would need to make a more educated decision.

## Estimated time savings:
5 minutes w/ no form changes
10 minutes w/ form changes
user still needs to validate documents are correct, which is the biggest time sink

## Milestones / checklist
Week 1
- [ ] Plan Jotform Webhook integration + Subsequent Jotform API calls
- [ ] Integrate service w/ application db/service
- [ ] Define and implement JotForm Schema + db
- [ ] Define and deploy Base Application Object Store
- [ ] Define and deploy Application File store
Week 2
- [ ] Initilize Jotform DB w/ references to program
- [ ] Store base application PDF in Object Store
- [ ] Map Jotform fields to PDF fields / text boxes
- [ ] Test and Identify frameworks to edit PDF, prioritizing ease of adding new program pdfs
Week 3
- [ ] Implement PDF editor service
- [ ] Update Application File Store w/ user application
- [ ] Store Uploaded files to Jotform in our Application File Store
- [ ] Design new notification email
Week 4
- [ ] E2E tests with cleaned real world examples (no PII)
- [ ] Create metrics + visibility dashboard (cloudwatch, grafana, etc.)
- [ ] Go live

# 3 Month Release

## OCR data extraction + Omni Model LLM verification + ID confirmation platform

When users are filling out the information on the Jotform, there is no guarantee that the documents the user is uploading are correct in both format and contents.

The back and forth between a starlight member reading the documents and comparing against the jotform data as well as reaching out to the applicant to get the updated forms can be time consuming.

A combination of OCR to extract data and or LLM calls can help us understand and determine the contents, form type, and retrieve fields in a way that is extensible to multiple programs or format types.

An LLM w/ tools to interact w/ a pdf can allow us to again take in the application form details and query against the provided documents and fields to automatically fill the pdfs, as well as submit the application through a web tool.

Events here can be keyed into to automatically prompt the applicant to send updated information, or describe the reason for the application's denial.  This is especially handy if the denial reasons are free text and non standard.

This would allow us to fairly confidently add in new programs and forms quite quickly, with the drawback being LLMs are prone to hallucination and can be error prone.  A human should still be in the loop and able to monitor the applicantion process and communication.

## Milestones / checklist
Month 1: Data extraction
- [ ] Design accuracy tests with existing data, user data, account numbers, balances, rent, mortgage payments, etc.  Both accepted and denied
- [ ] Identify OCR models and run tests w/ real world data for accuracy
- [ ] Identify LLM models and run tests to get answers / user data for accuracy and price
- [ ] Extend existing PDF initial form filler w/ OCR + LLM, creating two pdfs for Starlight Staff to give feedback on
Month 2: Data validation
- [ ] Design accuracy tests with existing data for approval and denials.  Including cases of missing user provided docs, mismatched IDs, invalid docs, etc.
- [ ] Design "Requirements" prompt to be extensible and follow a workflow through a JSON defined object so we can continue
- [ ] Test various LLMS for accuracy w/ approval / denial.  False positives are better than false negatives for us currently.
- [ ] Provide insights to Starlight Staff during the notification email, get feedback from staff on accuracy in the real world
- [ ] Assuming feedback is positive, continue to the Twilio design, else consider a redesign or using a 3rd party for things like ID match
- [ ] Cut over to PDFs generated on OCR + Jotform data as default
Month 3: Twilio workflows
- [ ] Design workflows for reprompting users for data, using OCR + LLM for validation.
- [ ] Call for a Starlight Staff if the documents don't pass validation again
Stretch: LLM Based Application PDF parsing
- [ ] Design flow to parse PDFs into requirements
- [ ] Given the OCR + LLM extracted data, step through requirements filling in fields
- [ ] Design cache to prevent repeat parsing of application PDFs and store locations of field mappings inside the PDFs to prevent future redundant
