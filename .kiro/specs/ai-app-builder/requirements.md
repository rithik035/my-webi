# Requirements Document

## Introduction

The AI App Builder is a web application that turns a natural-language prompt into a working full-stack application. A signed-in user describes the app or website in a chat. The Builder first produces a structured plan that the user reviews, edits, and approves. The Builder then generates the project step by step: a frontend with a consistent design system and animations, a backend API, and a database schema with migrations. Before the Builder shows a result, it installs, type-checks, lints, builds, migrates, runs, and smoke-tests the generated project inside an isolated sandbox, and it repairs failures automatically. The user sees the running app in a live preview, can browse and edit the code, can request further changes through chat, and can roll back to any earlier version.

The main difference from existing tools is precision: plan-first generation, a single opinionated target stack, and a mandatory verification and repair loop, so that a version is only marked as working when it has actually built and run.

Product references (public, used for UX and architecture ideas only):

- Lovable (proprietary): observable UX only, such as chat on the left, live preview on the right, code view, version history, a hosted backend and database, and deploy. No requirement in this document involves reverse engineering its private systems.
- stackblitz-labs/bolt.diy (MIT, about 20.0k stars): multiple LLM providers, in-browser preview.
- dyad-sh/dyad (mixed license, about 22.1k stars): local open-source Lovable alternative. The license must be checked before any code is reused.
- OpenHands/OpenHands (MIT, about 92k stars): agent loop and sandbox ideas.
- e2b-dev/fragments (Apache-2.0, about 6.4k stars): app generation with cloud sandboxes for live preview.

Scope labels: every requirement is labeled **[MVP]** or **[Later Phase]**. Technology choices (frameworks, sandbox technology, database engine, hosting) are left to the design phase unless a requirement states otherwise.

Out of scope for this spec: a template marketplace, real-time multi-user collaboration on one project, image or screenshot to app generation, billing and payments, and native mobile app output.

## Glossary

- **Builder**: The AI App Builder web application as a whole, including its user interface and server.
- **Approved_License_List**: The list of license identifiers that the Builder owner has approved as compatible with distributing the Builder, stored in the Builder source repository.
- **User**: A person with a Builder account who creates and edits projects.
- **Workspace_UI**: The Builder's project screen containing the chat panel, the preview panel, and the code panel.
- **Chat_Panel**: The part of the Workspace_UI where the User sends prompts and reads Builder responses.
- **Prompt**: A natural-language message the User sends in the Chat_Panel.
- **Auth_Service**: The Builder component that registers Users, authenticates sessions, and enforces access control.
- **Valid_Email_Address**: An email address of at most 254 characters containing exactly one "@" character, with a non-empty part before the "@" and a part after the "@" that contains at least one "." not at its start or end.
- **Project**: A User-owned container holding chat history, App_Plans, Versions, and settings for one generated application.
- **Project_Manager**: The Builder component that creates, lists, renames, and deletes Projects.
- **Project_Secret**: A secret value that a Generated_Project needs at runtime, such as a database password or a third-party service credential, supplied by the User or created by the Builder for that Project.
- **LLM_Provider**: An external or local large language model service, such as a hosted model API or an OpenAI-compatible endpoint.
- **Provider_Registry**: The Builder component that stores LLM_Provider configurations and routes model requests to the selected LLM_Provider.
- **API_Key**: A secret credential that the User supplies for an LLM_Provider.
- **Planner**: The Builder component that converts a Prompt into an App_Plan.
- **App_Plan**: A structured document describing the application to build: pages and routes, UI components, data entities and fields, API endpoints, authentication needs, design tokens, and an ordered list of Plan_Steps.
- **Plan_Step**: One ordered unit of generation work in an App_Plan, with a title, a description, and the files the step is expected to create or change.
- **Change_Plan**: A structured document that the Planner produces for a chat edit. It lists the App_Plan items to add, modify, or remove and the Generated_Project file paths to create, modify, rename, or delete.
- **Plan_Parser**: The Builder component that converts serialized App_Plan text into an App_Plan object and validates the object against the App_Plan schema.
- **Plan_Printer**: The Builder component that converts an App_Plan object into serialized App_Plan text.
- **Generator**: The Builder component that executes Plan_Steps and chat edits by requesting code from the LLM_Provider and applying File_Operations to the Generated_Project.
- **Target_Stack**: The single set of frontend, backend, database, and tooling technologies that every Generated_Project uses, as fixed in the design document.
- **Generated_Project**: The set of source files, configuration files, migrations, and tests that the Generator produces for a Project.
- **Secret_File**: A Generated_Project file that holds a secret value: an environment variable file with assigned values, a file containing the value of any API_Key stored for the User, or a file containing database credentials, private keys, or access tokens.
- **File_Operation**: One instruction to create, update, rename, or delete a file in the Generated_Project.
- **Edit_Parser**: The Builder component that converts LLM_Provider response text into an ordered list of File_Operations.
- **Edit_Printer**: The Builder component that converts an ordered list of File_Operations into LLM_Provider response format text.
- **Design_System**: The set of design tokens (colors, typography, spacing, radii, shadows, motion durations and easings) and shared UI components that a Generated_Project's frontend uses.
- **Sandbox**: An isolated execution environment in which the Builder installs, builds, migrates, runs, and tests a Generated_Project.
- **Verifier**: The Builder component that runs the Verification_Pipeline in a Sandbox and produces a Verification_Report.
- **Verification_Pipeline**: The ordered checks dependency install, type-check, lint, build, database migration, server start, health check, smoke tests, and generated automated tests.
- **Verification_Report**: The record of each Verification_Pipeline check with its pass or fail result, duration, and log output.
- **Repair_Loop**: The Builder process that sends Verification_Pipeline failures back to the Generator for correction and re-runs the Verifier.
- **Version**: An immutable snapshot of all Generated_Project files at one point in time, with an identifier, a timestamp, a description, an origin (plan generation, chat edit, manual edit, or rollback), and a Verification_Report.
- **Verified_Version**: A Version whose Verification_Report shows that every Verification_Pipeline check passed.
- **Version_Store**: The Builder component that saves, lists, and restores Versions.
- **Preview_Panel**: The part of the Workspace_UI that displays the running Generated_Project from the Sandbox.
- **Code_Panel**: The part of the Workspace_UI that shows the Generated_Project file tree and a code editor.
- **Exporter**: The Builder component that packages a Version as a downloadable archive.
- **GitHub_Integration**: The Builder component that pushes a Version to a GitHub repository.
- **GitHub_Access_Token**: The credential GitHub issues to the Builder when a User connects a GitHub account, used by the GitHub_Integration to create repositories and push commits for that User.
- **Deployer**: The Builder component that publishes a Version to a public hosting environment.
- **Generation_Run**: One execution of the Planner, the Generator, the Verifier, and the Repair_Loop in response to one approved App_Plan or one chat edit request.
- **Reduced_Motion_Preference**: The operating system or browser setting, exposed as the `prefers-reduced-motion` media feature, that requests minimal animation.

## Requirements

### Requirement 1: User Accounts and Authentication [MVP]

**User Story:** As a User, I want to create an account and sign in, so that my Projects and API_Keys are private to me.

#### Acceptance Criteria

1. WHEN a visitor submits a registration form with a Valid_Email_Address and a password of 12 to 128 characters, THE Auth_Service SHALL create a User account and start an authenticated session.
2. IF a visitor submits a registration form with an email address that already belongs to an account, compared without regard to letter case, THEN THE Auth_Service SHALL reject the registration, create no account, and display a message stating that the email address is already registered.
3. WHEN a User submits a registered email address (compared without regard to letter case) together with the password for that account, and the email address is not blocked under criterion 5, THE Auth_Service SHALL start an authenticated session that stays valid until the User signs out or until 7 days have passed since the session's last request, whichever comes first.
4. IF a User submits sign-in credentials where the email address is not registered or the password does not match the account, THEN THE Auth_Service SHALL reject the sign-in and display the same message in both cases, so that the message does not reveal whether the email address exists.
5. IF 5 consecutive failed sign-in attempts occur for one email address within 15 minutes, THEN THE Auth_Service SHALL reject every sign-in attempt for that email address, including attempts with correct credentials, for 15 minutes after the fifth failure. It SHALL apply the same block and the same message to registered and unregistered email addresses, and the message SHALL state that sign-in is temporarily blocked. A successful sign-in resets the consecutive failure count to 0.
6. WHEN a User selects sign out, THE Auth_Service SHALL invalidate that session, so that every later request using that session is rejected as unauthenticated.
7. THE Auth_Service SHALL store User passwords only as salted hashes produced by a password hashing algorithm designed for password storage, and SHALL never store or log a password in plaintext.
8. IF an authenticated request targets a Project, any content of that Project (chat history, App_Plans, Versions, or settings), or an API_Key that the requesting User does not own, THEN THE Auth_Service SHALL reject the request with a not-found response identical to the one returned for a Project or API_Key that does not exist, and SHALL make no change to the targeted data.
9. IF a visitor submits a registration form with an email address that is not a Valid_Email_Address, or a password shorter than 12 or longer than 128 characters, THEN THE Auth_Service SHALL reject the registration, create no account, display a message identifying each invalid field, and keep the entered email address in the form.
10. IF a request that targets a Project, Project content, or an API_Key carries no session, or a session that is invalidated or expired, THEN THE Auth_Service SHALL reject the request as unauthenticated, return no Project or API_Key data, and direct the visitor to the sign-in screen.

### Requirement 2: Project Management [MVP]

**User Story:** As a User, I want to create, list, rename, and delete Projects, so that I can manage several apps.

#### Acceptance Criteria

1. WHEN a User submits a first Prompt from the Builder home screen, THE Project_Manager SHALL create a new Project owned by that User, give the Project an initial name of 1 to 100 characters taken from the Prompt, and open the Project in the Workspace_UI with the submitted Prompt shown as the first message in the Chat_Panel.
2. WHEN a User opens the Builder home screen, THE Project_Manager SHALL display only the Projects owned by that User, ordered by last-modified time with the newest first, where last-modified time is the time of the most recent change to the Project's name, chat history, App_Plans, or Versions.
3. WHEN the Project_Manager displays the Project list, THE Project_Manager SHALL show for each Project its name, its last-modified time, and one verification status for its latest Version: "verified" if the latest Version is a Verified_Version, "failed" if any Verification_Pipeline check of the latest Version failed, or "no version" if the Project has no Version.
4. WHEN a User renames a Project with a name that contains 1 to 100 characters after leading and trailing whitespace is removed, THE Project_Manager SHALL save the trimmed name and show the new name in the Project list and the Workspace_UI.
5. WHEN a User requests deletion of a Project, THE Project_Manager SHALL show a confirmation prompt that names the Project, and SHALL leave the Project unchanged if the User cancels the prompt.
6. WHEN a User confirms deletion of a Project, THE Project_Manager SHALL delete the Project's chat history, App_Plans, Versions, and Sandbox resources, and SHALL remove the Project from the User's Project list.
7. WHEN a User reopens a Project, THE Workspace_UI SHALL show the Project's full chat history in the Chat_Panel, the latest Version's files in the Code_Panel, and the latest App_Plan; IF the Project has no Version yet, THEN THE Workspace_UI SHALL show the chat history and the latest App_Plan and leave the Code_Panel file tree empty.
8. IF a User submits a rename with a name that is empty, contains only whitespace, or is longer than 100 characters after trimming, THEN THE Project_Manager SHALL reject the rename, keep the previous name, and show an error message stating the 1 to 100 character limit.
9. IF the Project_Manager fails to create a Project after a User submits a first Prompt, THEN THE Project_Manager SHALL keep the User on the Builder home screen with the Prompt text still in the input field, and show an error message indicating that the Project was not created.
10. IF any part of a confirmed Project deletion fails, THEN THE Project_Manager SHALL show an error message indicating that the deletion did not complete, keep the Project in the User's Project list, and allow the User to request deletion again.

### Requirement 3: LLM Provider Configuration [MVP]

**User Story:** As a User, I want to bring my own API_Key and choose among several LLM_Providers, so that I control model quality and cost.

#### Acceptance Criteria

1. THE Provider_Registry SHALL support at least two hosted LLM_Providers and any LLM_Provider that exposes an OpenAI-compatible chat completions API at a User-supplied base URL of 1 to 2,048 characters.
2. WHEN a User saves an API_Key for an LLM_Provider, THE Provider_Registry SHALL send one test request to that LLM_Provider and, within 15 seconds of the save, show the configuration as valid if the LLM_Provider accepted the API_Key or invalid if it did not.
3. IF the test request for a saved API_Key is rejected by the LLM_Provider, gets no response within 15 seconds, or cannot reach the LLM_Provider, THEN THE Provider_Registry SHALL mark the configuration as invalid and display a message with the failure reason returned by the LLM_Provider, or a message saying the request timed out or the LLM_Provider was unreachable when no reason was returned.
4. WHEN a User selects an LLM_Provider and model for a Project, THE Provider_Registry SHALL send every Planner and Generator request made for that Project from then on to the selected LLM_Provider and model. Requests already in progress at the moment of selection SHALL finish on the previously selected LLM_Provider and model.
5. IF an LLM_Provider returns a rate-limit error or a server error, THEN THE Provider_Registry SHALL retry the request at most 3 times, waiting about 2, 4, and 8 seconds before the first, second, and third retry. If all 3 retries fail, it SHALL stop the active Generation_Run, keep all Generated_Project files as they were before the failed request, and display a message to the User naming the LLM_Provider and the error type.
6. IF a User starts a Generation_Run for a Project whose selected LLM_Provider configuration is not marked valid, or that has no LLM_Provider and model selected, THEN THE Builder SHALL send no request to any LLM_Provider, keep the User's Prompt in the Chat_Panel input, and display a message linking to the LLM_Provider settings.
7. IF a User-supplied base URL is not a well-formed URL using the http or https scheme, THEN THE Provider_Registry SHALL reject the save without sending a test request, keep the previously saved configuration unchanged, and display a message saying the base URL is invalid.
8. IF an LLM_Provider rejects a Planner or Generator request because the API_Key is invalid or lacks permission, THEN THE Provider_Registry SHALL not retry the request, mark the configuration as invalid, and display a message telling the User the API_Key was rejected.

### Requirement 4: Prompt Intake [MVP]

**User Story:** As a User, I want to describe my app in plain language in a chat, so that I can start building without writing code.

#### Acceptance Criteria

1. WHEN a User submits a Prompt whose length is 1 to 10,000 characters after leading and trailing whitespace are removed, THE Chat_Panel SHALL append the Prompt to the Project chat history, display the Prompt in the Chat_Panel within 1 second, clear the Prompt input field, and pass the Prompt to the Planner.
2. IF a User submits a Prompt that is empty or contains only whitespace, or a Prompt longer than 10,000 characters after leading and trailing whitespace are removed, THEN THE Chat_Panel SHALL reject the Prompt without adding it to the Project chat history or passing it to the Planner, keep the entered text in the Prompt input field, and display a message stating the allowed length of 1 to 10,000 characters.
3. WHILE a Generation_Run is in progress for a Project, THE Chat_Panel SHALL disable new Prompt submission for that Project and display a cancel control.
4. THE Builder SHALL persist every Prompt and every Builder response in the Project chat history with a timestamp, so that the Chat_Panel shows all messages in the order they occurred after the User reloads the Workspace_UI or signs in again.
5. WHEN the Planner cannot determine from a Prompt at least one page, the data entities the application stores, or whether the application requires authentication, THE Planner SHALL display 1 to 5 clarifying questions in a single Chat_Panel message, together with a skip control, before producing an App_Plan.
6. WHEN a User activates the skip control for clarifying questions, THE Planner SHALL produce an App_Plan and list in the App_Plan each assumption the Planner made, with each assumption naming the page, data entity, or authentication need it affects.
7. WHEN a User submits a Prompt that answers the displayed clarifying questions, THE Planner SHALL produce an App_Plan that uses the answers, without asking further clarifying questions for that Prompt.
8. IF the Builder fails to persist a submitted Prompt to the Project chat history, THEN THE Chat_Panel SHALL keep the Prompt text in the Prompt input field, not pass the Prompt to the Planner, and display an error message stating that the Prompt was not sent.
9. WHEN a User activates the cancel control during a Generation_Run, THE Chat_Panel SHALL re-enable Prompt submission for that Project within 5 seconds of the activation. The cancellation itself SHALL follow Requirement 22, criteria 2 and 3.

### Requirement 5: Structured App Plan Generation [MVP]

**User Story:** As a User, I want the Builder to produce a clear, step-by-step plan before writing code, so that the result matches what I asked for.

#### Acceptance Criteria

1. WHEN the Planner receives a Prompt for a Project without an approved App_Plan, THE Planner SHALL produce an App_Plan containing at least 1 page with its route, the UI components used by each page, 0 or more data entities in which every field has a name and a type and every relation names its target data entity and its cardinality (one-to-one, one-to-many, or many-to-many), API endpoints in which each endpoint has a method, a path, a request shape, and a response shape, authentication needs stated either as "none" or as a list of required authentication features, Design_System tokens for colors, typography, spacing, radii, shadows, and motion durations and easings, and an ordered list of 1 to 50 Plan_Steps in which each Plan_Step has a title of 1 to 100 characters, a description of 1 to 1,000 characters, and at least 1 file path the Plan_Step is expected to create or change.
2. THE Planner SHALL order Plan_Steps so that, when the App_Plan defines at least 1 data entity, every Plan_Step that creates or changes the database schema precedes every Plan_Step that creates or changes a backend API endpoint, and every Plan_Step that creates or changes a backend API endpoint precedes every frontend Plan_Step that calls that API endpoint.
3. THE Planner SHALL reference in each API endpoint only data entities defined in the same App_Plan, so that the Plan_Parser reports zero undefined data entity references for every App_Plan the Planner presents to the User.
4. THE Planner SHALL reference in each page only API endpoints and UI components defined in the same App_Plan, so that the Plan_Parser reports zero undefined API endpoint or UI component references for every App_Plan the Planner presents to the User.
5. IF the Plan_Parser rejects the Planner output, THEN THE Planner SHALL send the LLM_Provider a request for a corrected App_Plan that includes each failing field path and reason returned by the Plan_Parser, with at most 2 correction requests per Prompt (3 attempts in total including the original attempt).
6. THE Planner SHALL produce every App_Plan for the Target_Stack, so that no App_Plan names a framework, programming language, database engine, or tooling technology outside the Target_Stack.
7. IF the Plan_Parser rejects all 3 attempts for one Prompt, THEN THE Planner SHALL stop requesting corrections, display a planning failure message in the Chat_Panel that lists the failing field paths from the last attempt, offer a retry action, keep the Prompt and chat history unchanged, and save no App_Plan to the Project.
8. IF the LLM_Provider returns an error or does not respond within 120 seconds during planning, THEN THE Planner SHALL stop the planning attempt, display an error message in the Chat_Panel indicating the failure cause (authentication failure, rate limit, timeout, or unreachable provider), offer a retry action, keep the Prompt and chat history unchanged, and save no App_Plan to the Project.
9. IF a Prompt requests a framework, programming language, database engine, or tooling technology outside the Target_Stack, THEN THE Planner SHALL produce the App_Plan for the Target_Stack and list each replaced technology together with its Target_Stack substitute among the App_Plan's assumptions.

### Requirement 6: Plan Review and Approval [MVP]

**User Story:** As a User, I want to review and edit the plan before generation starts, so that I can correct misunderstandings early.

#### Acceptance Criteria

1. WHEN the Planner produces an App_Plan, THE Workspace_UI SHALL display the App_Plan in separately labeled sections for pages and routes, UI components, data entities, API endpoints, authentication needs, Design_System tokens, Plan_Steps, and assumptions, showing every item of the App_Plan in exactly one section, showing each Plan_Step with its position number, title, description, and expected files in execution order, and omitting the assumptions section when the App_Plan lists no assumptions.
2. WHEN a User completes an edit, addition, removal, or reordering of an item in an unapproved App_Plan in the Workspace_UI, THE Builder SHALL validate the edited App_Plan with the Plan_Parser within 1 second and display each validation error next to the affected item, naming the item and the rule it violates, or above the App_Plan sections if the error does not belong to a single item.
3. WHEN a User completes an edit, addition, removal, or reordering of an item in an unapproved App_Plan, THE Builder SHALL save the edited App_Plan, including any items that fail validation, as the Project's latest App_Plan so that reopening the Project restores the edit.
4. WHEN a User sends a chat message requesting changes to an unapproved App_Plan, THE Planner SHALL produce a revised App_Plan based on the current App_Plan including the User's manual edits, and the Workspace_UI SHALL mark each added, removed, and modified item with a separate indicator for each of the three change types.
5. WHEN a User approves an App_Plan that passes Plan_Parser validation with zero errors, THE Builder SHALL save the approved App_Plan to the Project and start exactly one Generation_Run for that App_Plan.
6. IF a User attempts to approve an App_Plan that has one or more Plan_Parser validation errors, THEN THE Builder SHALL block the approval, start no Generation_Run, list every validation error with the item it affects, and keep the App_Plan and the User's edits unchanged.
7. WHEN the Builder saves an approved App_Plan, THE Workspace_UI SHALL make that App_Plan read-only and disable the approve control for it.
8. IF the Planner reports a planning failure while revising an unapproved App_Plan, THEN THE Builder SHALL keep the previous App_Plan and the User's manual edits unchanged and display a message in the Chat_Panel stating that the revision failed.

### Requirement 7: App Plan Serialization [MVP]

**User Story:** As a developer of the Builder, I want App_Plans to be parsed and printed reliably, so that plans survive storage, editing, and LLM round trips without corruption.

#### Acceptance Criteria

1. WHEN serialized App_Plan text of at most 1,048,576 bytes that conforms to the App_Plan schema is provided, THE Plan_Parser SHALL produce an App_Plan object that contains every page, route, UI component, data entity, field, API endpoint, authentication need, design token, and Plan_Step present in the text, preserving the order of Plan_Steps and of every other list as it appears in the text.
2. IF serialized App_Plan text does not conform to the App_Plan schema, THEN THE Plan_Parser SHALL return an error that lists every failing field path in a single response, each with the reason for the failure, and SHALL NOT produce a partial App_Plan object.
3. WHEN a valid App_Plan object is provided, THE Plan_Printer SHALL convert it into serialized App_Plan text that conforms to the App_Plan schema, and SHALL produce byte-identical text each time it prints App_Plan objects that are equal.
4. FOR ALL valid App_Plan objects, printing with the Plan_Printer and then parsing with the Plan_Parser SHALL produce an App_Plan object equal to the original, where equal means every field has the same value and every list has the same elements in the same order (round-trip property).
5. IF serialized App_Plan text references a data entity, API endpoint, or UI component that the same App_Plan does not define, THEN THE Plan_Parser SHALL return an error that names each undefined reference and the field path where it occurs, and SHALL NOT produce an App_Plan object.
6. IF serialized App_Plan text is not syntactically well-formed or is empty, THEN THE Plan_Parser SHALL return an error that gives the line and column of the first syntax failure, and SHALL NOT produce an App_Plan object.
7. IF serialized App_Plan text is larger than 1,048,576 bytes, THEN THE Plan_Parser SHALL return an error saying the size limit was exceeded, without attempting to parse the text.
8. IF serialized App_Plan text defines two or more pages, routes, UI components, data entities, API endpoints, or Plan_Steps of the same kind with the same identifier, THEN THE Plan_Parser SHALL return an error that names each duplicate identifier and its field paths, and SHALL NOT produce an App_Plan object.

### Requirement 8: Step-by-Step Code Generation [MVP]

**User Story:** As a User, I want the Builder to generate my app one plan step at a time, so that each part is built on a correct foundation.

#### Acceptance Criteria

1. WHEN a Generation_Run starts from an approved App_Plan, THE Generator SHALL execute the Plan_Steps one at a time in the order listed in the App_Plan, starting each Plan_Step only after the File_Operations of the previous Plan_Step have been applied to the Generated_Project.
2. WHEN the Generator executes a Plan_Step, THE Generator SHALL include in the LLM_Provider request the approved App_Plan, the current Plan_Step, the paths of all files currently in the Generated_Project, and the current content of every file listed in the current Plan_Step that already exists in the Generated_Project.
3. THE Generator SHALL produce every file of every Generated_Project using only the frontend, backend, database, and tooling technologies listed in the Target_Stack.
4. WHEN the Generator receives an LLM_Provider response for a Plan_Step and the Edit_Parser accepts the response, THE Generator SHALL apply the resulting File_Operations to the Generated_Project in the order the Edit_Parser produced them and set the Plan_Step status to passed.
5. IF the Edit_Parser rejects an LLM_Provider response for a Plan_Step, THEN THE Generator SHALL send the LLM_Provider a request for a corrected response that includes the Edit_Parser error, up to 2 times, and SHALL set the Plan_Step status to failed if the Edit_Parser rejects the second corrected response.
6. WHEN every Plan_Step of the approved App_Plan has the status passed, THE Generator SHALL pass the Generated_Project to the Verifier.
7. THE Generator SHALL produce a README file in the root directory of the Generated_Project that lists the commands to install dependencies, run database migrations, start the server, and run the generated automated tests, matching the commands that the Verification_Pipeline runs for those checks.
8. THE Generator SHALL produce an environment variable template file in the root directory of the Generated_Project that lists every environment variable the Generated_Project source code reads, with a placeholder value for each variable and no API_Key, password, or other credential value.
9. IF the Provider_Registry reports a failure for a Plan_Step request after its retries, THEN THE Generator SHALL set the Plan_Step status to failed.
10. IF a Plan_Step status is set to failed, THEN THE Generator SHALL end the Generation_Run without executing the remaining Plan_Steps or invoking the Verifier, SHALL set the status of each remaining Plan_Step to not run, SHALL keep the Generated_Project files equal to the latest Version (or empty if the Project has no Version), and SHALL display in the Chat_Panel the failed Plan_Step title and a message indicating the failure reason.

### Requirement 9: Edit Instruction Parsing [MVP]

**User Story:** As a developer of the Builder, I want LLM_Provider output converted into exact file operations, so that code changes are applied precisely and safely.

#### Acceptance Criteria

1. WHEN LLM_Provider response text conforms to the File_Operation format defined in the design document, THE Edit_Parser SHALL produce an ordered list of File_Operations in the same order in which the operations appear in the text. Each File_Operation SHALL contain an operation type (create, update, rename, or delete) and a target path relative to the Generated_Project root directory. Create and update operations SHALL also contain the complete file content, and rename operations SHALL also contain a destination path.
2. IF LLM_Provider response text does not conform to the File_Operation format, including a missing operation type, a missing target path, missing file content for a create or update operation, or a missing destination path for a rename operation, THEN THE Edit_Parser SHALL return an error stating the 1-based line number and column of the first nonconforming element and the reason it does not conform, and SHALL return no File_Operations.
3. IF a File_Operation target path or rename destination path is empty, is absolute, starts with a drive letter, or resolves outside the Generated_Project root directory after its "." and ".." segments are resolved, THEN THE Edit_Parser SHALL reject the entire File_Operation list with a path error that identifies the offending path and the 1-based position of its File_Operation in the list, and SHALL return no File_Operations.
4. THE Edit_Printer SHALL convert any valid ordered list of File_Operations into text that conforms to the File_Operation format. A list is valid when every File_Operation contains the fields that criterion 1 requires for its operation type, every path satisfies the constraints in criterion 3, and the list is within the limits in criterion 8.
5. FOR ALL valid ordered lists of File_Operations, printing with the Edit_Printer and then parsing with the Edit_Parser SHALL produce a list equal to the original (round-trip property). Two lists are equal when they have the same number of File_Operations in the same order and each corresponding pair has the same operation type, the same target path, the same destination path, and byte-identical file content.
6. IF applying any File_Operation in a list fails, THEN THE Generator SHALL restore every Generated_Project file to its exact content and path as they were before the first File_Operation in the list was applied, and SHALL create no Version from that list. Applying a File_Operation fails when an update, rename, or delete targets a path that does not exist at that point in the list, when a create or a rename destination targets a path that already exists at that point in the list, or when a file system write fails.
7. IF applying any File_Operation in a list fails, THEN THE Generator SHALL return an error identifying the 1-based position of the failed File_Operation in the list, its target path, and the failure reason.
8. IF LLM_Provider response text exceeds 5 MB, contains more than 500 File_Operations, or contains a target path or destination path longer than 260 characters, THEN THE Edit_Parser SHALL return an error stating which limit was exceeded and SHALL return no File_Operations.

### Requirement 10: Generated Frontend Quality and Animations [MVP]

**User Story:** As a User, I want the generated frontend to look polished, animate smoothly, and work on any screen size, so that my app feels professional.

#### Acceptance Criteria

1. THE Generator SHALL define all Design_System tokens of a Generated_Project (colors, typography, spacing, radii, shadows, motion durations, and easings) in one source location. Frontend components SHALL reference these tokens instead of hard-coded literal values for those categories.
2. THE Generator SHALL implement animated transitions in the Generated_Project for route changes, element entrance, and the hover, focus, and press states of interactive elements. Each transition SHALL use a duration between 100 and 500 milliseconds and an easing taken from the Design_System motion tokens.
3. WHILE the Reduced_Motion_Preference is set in the viewing browser, THE Generated_Project frontend SHALL replace movement-based animations with either no animation or opacity-only transitions of 150 milliseconds or less, and SHALL still show the final state of every content change that would otherwise have been animated.
4. THE Generator SHALL produce frontend pages that render at viewport widths of 375, 768, and 1280 CSS pixels with no horizontal scrolling and no element extending past the right edge of the viewport.
5. THE Generator SHALL produce frontend pages in which all text has a contrast ratio of at least 4.5:1 against its background color. This includes placeholder text and text shown in hover, focus, and press states, and excludes text in disabled controls.
6. THE Generator SHALL produce frontend pages in which every interactive element can be reached with the Tab and Shift+Tab keys in an order that matches the top-to-bottom, left-to-right visual order. Each element SHALL be operable with Enter or Space. Dialogs and menus SHALL close with Escape. Focus SHALL never be trapped in an element that is not a dialog. The focused element SHALL show a focus indicator at least 2 CSS pixels thick with a contrast ratio of at least 3:1 against adjacent colors.
7. WHEN a page section that displays data from the backend API sends a data request, THE Generated_Project frontend SHALL display a loading state in that section until the request succeeds or fails.
8. WHEN a backend API request for a page section returns zero records, THE Generated_Project frontend SHALL display an empty state in that section with a message indicating that no data exists yet.
9. IF a backend API request for a page section fails or returns no response within 15 seconds, THEN THE Generated_Project frontend SHALL display an error state in that section. The error state SHALL contain a message indicating that the data could not be loaded and a retry control that sends the request again. Other page sections SHALL remain usable.

### Requirement 11: Generated Backend API [MVP]

**User Story:** As a User, I want a real backend API behind my app, so that data is processed and stored on a server instead of only in the browser.

#### Acceptance Criteria

1. THE Generator SHALL implement every API endpoint listed in the approved App_Plan with the HTTP method, path, request shape, and response shape that the App_Plan defines, so that every endpoint in the App_Plan is reachable in the running Generated_Project backend and no endpoint defined in the App_Plan is missing.
2. WHEN a generated API endpoint that creates, updates, or deletes data entities receives a request that passes input validation, THE Generated_Project backend SHALL apply the change to the Generated_Project database before returning a success response, so that a later read through the matching API endpoint returns the changed data, including after a backend restart.
3. IF a request to a generated API endpoint fails input validation against the request shape defined in the approved App_Plan, THEN THE Generated_Project backend SHALL return a 400-class status code and an error body containing a machine-readable error code and a list of invalid fields, each entry naming the field and the reason it failed, and SHALL leave the Generated_Project database unchanged.
4. WHERE the approved App_Plan marks an API endpoint as requiring authentication, IF a request to that endpoint has missing, malformed, expired, or signed-out credentials, THEN THE Generated_Project backend SHALL return a 401 status code without performing the endpoint's operation and without returning any data entity content.
5. WHEN the health endpoint of the Generated_Project backend receives a request and the backend completes a database connectivity check within 5 seconds, THE Generated_Project backend SHALL return a 200-class status code.
6. IF the health endpoint of the Generated_Project backend cannot reach the database within 5 seconds of receiving a request, THEN THE Generated_Project backend SHALL return a 503 status code and an error body indicating that the database is unreachable.
7. IF an error that the endpoint does not handle occurs while a generated API endpoint processes a request, THEN THE Generated_Project backend SHALL return a 500-class status code and an error body containing a machine-readable error code, SHALL exclude stack traces, file paths, and database query text from the response, and SHALL keep serving later requests without a restart.
8. THE Generator SHALL produce, for each API endpoint in the Generated_Project, at least one automated test that checks a valid request returns the response shape defined in the approved App_Plan, and at least one automated test that checks an invalid request returns the validation error defined in criterion 3 for each endpoint that accepts a request body or parameters. These tests run as the generated automated tests check of the Verification_Pipeline.
9. WHERE the approved App_Plan requires end-user authentication, THE Generator SHALL implement sign-up, sign-in, and sign-out endpoints in the Generated_Project that store passwords only as salted hashes, never store or log plaintext passwords, and never return a password or password hash in any API response.
10. WHERE the approved App_Plan requires end-user authentication, WHEN an end user signs out, THE Generated_Project backend SHALL invalidate the credentials used in that session, so that each later request with those credentials to an endpoint requiring authentication returns a 401 status code.

### Requirement 12: Generated Database Schema and Migrations [MVP]

**User Story:** As a User, I want a real database with a schema that matches my app, so that my data persists and can evolve safely.

#### Acceptance Criteria

1. WHEN the Generator executes the Plan_Steps of an approved App_Plan, THE Generator SHALL produce a database schema in the Generated_Project with exactly one table or collection for each data entity in the App_Plan. Each table or collection SHALL have a primary key and the fields, field types, required-field constraints, uniqueness constraints, and relations that the App_Plan defines for that entity.
2. THE Generator SHALL give every migration file in the Generated_Project a sequence identifier that is unique within the Generated_Project and higher than the identifiers of all earlier migration files. For every Version, applying all of the Version's migration files in ascending sequence order to an empty database SHALL complete with a success exit status and produce the full schema described in criterion 1.
3. WHEN the Generated_Project's migration command runs against a database where all of the Generated_Project's migrations were already applied, THE Generated_Project SHALL finish with a success exit status and leave the schema and all stored records unchanged (idempotence property).
4. WHEN a chat edit changes the data model by adding, removing, or renaming a data entity or field, or by changing a field type, constraint, or relation, THE Generator SHALL add exactly one new migration file that turns the previous Version's schema into the changed schema, and SHALL leave the content of every previously generated migration file unchanged.
5. WHEN a chat edit does not change the data model, THE Generator SHALL leave the set of migration files in the Generated_Project unchanged.
6. WHEN the new migration file from criterion 4 is applied to a database holding the previous Version's schema and records, THE Generated_Project SHALL keep every stored record in tables, collections, and fields that the change does not remove, and SHALL keep the stored values of renamed tables, collections, and fields under their new names.
7. IF a migration file added by a chat edit drops a table, a collection, or a field, THEN THE Generator SHALL list each dropped table, collection, and field in the Chat_Panel response for that chat edit.
8. THE Generator SHALL produce seed data in the Generated_Project with between 3 and 20 records for each data entity, so that the Preview_Panel displays populated pages. Every seed record SHALL satisfy its entity's field types, required-field constraints, uniqueness constraints, and relations, and every relation SHALL reference an existing seed record. Applying the seed data after all migrations on an empty database SHALL complete with a success exit status.

### Requirement 13: Verification Pipeline [MVP]

**User Story:** As a User, I want the Builder to prove that my app builds and runs before showing it to me, so that I only see working results.

#### Acceptance Criteria

1. WHEN the Generator finishes a Generation_Run's file changes, THE Verifier SHALL run the Verification_Pipeline checks against those Generated_Project files in this order: dependency install, type-check, lint, build, database migration, server start, health check, smoke tests, and generated automated tests. Each run SHALL start at the dependency install check.
2. WHEN a Verification_Pipeline check fails, THE Verifier SHALL stop the Verification_Pipeline. THE Verifier SHALL record the failing check, its exit status, and its log output in the Verification_Report, and SHALL mark every later check as "not run".
3. THE Verifier SHALL include in the smoke tests a page load of every page route and a request to every API endpoint in the approved App_Plan. For routes that take parameters, THE Verifier SHALL use identifiers from the Generated_Project seed data. THE Verifier SHALL count each of the following as a smoke test failure: a server error status, no response within 30 seconds, or an uncaught frontend runtime error during page load. THE Verification_Report SHALL name each failing route or endpoint. A client error status, such as the rejection of an unauthenticated request, SHALL NOT count as a failure.
4. IF a single Verification_Pipeline check runs longer than the per-command time limit in the Builder configuration (default 10 minutes), THEN THE Verifier SHALL terminate that check's processes, mark the check as failed, and record a timeout in the Verification_Report.
5. WHEN every Verification_Pipeline check passes, THE Version_Store SHALL save the Generated_Project as a Verified_Version. The saved Verification_Report SHALL list each check with its pass result and duration in seconds.
6. WHEN a User opens the Verification_Report of the current Version, THE Workspace_UI SHALL display each check's result (passed, failed, or not run), its duration, and its log output. THE Verifier SHALL keep at least the last 5,000 lines of log output per check, and THE Workspace_UI SHALL label a check's log output as truncated when earlier lines were dropped.
7. THE Verifier SHALL decide each check's result by these rules: dependency install, type-check, lint, build, database migration, and generated automated tests pass when the check's command exits with status zero, and lint warnings alone SHALL NOT fail the lint check; database migration runs every migration against an empty database in the Sandbox; server start passes when the Generated_Project server accepts connections within 60 seconds of launch; health check passes when a request to the Generated_Project's health endpoint returns a success status within 10 seconds.
8. WHILE the Verification_Pipeline is running, THE Workspace_UI SHALL show each check's status (pending, in progress, passed, failed, not run, or cancelled) and SHALL update a check's displayed status within 2 seconds of that status changing.
9. IF the Sandbox fails to start, or the Verifier cannot run a check for a reason outside the Generated_Project, THEN THE Verifier SHALL record an infrastructure error and its cause in the Verification_Report. In that case, THE Verifier SHALL NOT start the Repair_Loop and SHALL NOT save a Verified_Version. THE Builder SHALL keep the Generated_Project files unchanged, and THE Workspace_UI SHALL show an error message with a retry option.

### Requirement 14: Automatic Error Repair [MVP]

**User Story:** As a User, I want the Builder to fix its own build and runtime errors, so that I do not have to debug generated code.

#### Acceptance Criteria

1. WHEN a Verification_Pipeline check fails during a Generation_Run, THE Repair_Loop SHALL send the Generator a fix request. The request SHALL contain the failing check name, the last 200 lines of that check's log output, and the current content of up to 20 Generated_Project files, chosen as the first 20 distinct file paths that appear in that log output. The request SHALL ask for the fix as an ordered list of File_Operations.
2. WHEN the Generator applies the File_Operations of a repair attempt to the Generated_Project, THE Repair_Loop SHALL re-run the full Verification_Pipeline, starting from the dependency install check.
3. THE Repair_Loop SHALL perform no more repair attempts per set of repair attempts than the repair attempt limit in the Project settings. The limit is an integer from 0 to 5 with a default of 3. A limit of 0 means a failed check ends the Generation_Run without any repair attempt.
4. IF the Repair_Loop reaches the repair attempt limit without a passing Verification_Pipeline, THEN THE Builder SHALL mark the Generation_Run as failed, save the Generated_Project as an unverified Version with its final Verification_Report, and display that final Verification_Report in the Chat_Panel. The Preview_Panel SHALL stay on the last Verified_Version. If the Project has no Verified_Version, the Preview_Panel SHALL display a message stating that no Verified_Version exists.
5. WHEN a User selects retry on a failed Generation_Run, THE Repair_Loop SHALL start a new set of repair attempts. The attempt count SHALL restart at zero, and the limit SHALL be the current repair attempt limit in the Project settings. The first attempt SHALL use that Generation_Run's unverified Version files and final Verification_Report as its input.
6. WHEN a repair attempt ends, THE Chat_Panel SHALL display an entry for that attempt. The entry SHALL show the attempt number and the repair attempt limit, the name of the failing check that the attempt targeted, and the file path and operation type (create, update, rename, or delete) of each File_Operation applied. It SHALL also show the attempt result: passed, or failed with the name of the check that failed.
7. IF a User enters a repair attempt limit that is not an integer from 0 to 5, THEN THE Builder SHALL reject the value, keep the previously saved limit, and display an error message stating the allowed range.
8. IF the Edit_Parser cannot parse the Generator's repair response, or the response yields zero File_Operations, THEN THE Repair_Loop SHALL count the repair attempt as used, leave the Generated_Project files unchanged for that attempt, record the attempt as failed in the Chat_Panel, and start the next repair attempt with the same failing check if the repair attempt limit has not been reached.
9. IF the Provider_Registry reports an LLM_Provider failure during a repair attempt after its retries are exhausted, THEN THE Builder SHALL end the current set of repair attempts and save the Generated_Project as an unverified Version with the most recent Verification_Report. The Preview_Panel SHALL stay on its current content. The Chat_Panel SHALL display an error message stating that the LLM_Provider failed and offer a retry option.

### Requirement 15: Sandboxed Execution [MVP]

**User Story:** As a User, I want generated code to run in isolation, so that it cannot harm the Builder, my account, or other Users.

#### Acceptance Criteria

1. THE Builder SHALL run every Generated_Project dependency install, type-check, lint, build, database migration, server, and test command inside a Sandbox dedicated to exactly one Project, and SHALL run no Generated_Project command on the Builder's own servers outside a Sandbox.
2. THE Sandbox SHALL prevent Generated_Project processes from reading Builder secrets, including API_Keys, Builder credentials, and Builder environment variables, and SHALL block every network connection from Generated_Project processes to the Builder's databases, the Builder's internal services, and other Projects' Sandboxes.
3. THE Sandbox SHALL enforce the CPU, memory, disk, and process count limits defined in the Builder configuration, with defaults of 2 CPU cores, 2 GiB of memory, 5 GiB of disk, and 256 concurrent processes per Sandbox.
4. WHEN 30 minutes have passed since the last User activity in the Workspace_UI for a Project, and no Generation_Run is in progress for that Project, THE Builder SHALL stop that Project's Sandbox and keep every saved Version in the Version_Store unchanged. User activity means sending a Prompt, editing a file in the Code_Panel, interacting with the Preview_Panel, or opening the Project.
5. WHEN a User opens a Project whose Sandbox is stopped, THE Builder SHALL start a Sandbox containing the files of the Project's most recent Version and show the current start-up stage in the Preview_Panel. The stages are Sandbox start, file restore, dependency install, and server start. The Preview_Panel SHALL show the running Generated_Project once the health check passes.
6. IF a Generated_Project process exceeds a CPU, memory, disk, or process count limit of its Sandbox, THEN THE Builder SHALL stop that process, keep the Sandbox and its files, record the exceeded limit and the stopped command in the Verification_Report of the current Generation_Run (or, when no Generation_Run is in progress, show a message in the Preview_Panel naming the exceeded limit), and mark the affected Verification_Pipeline check as failed.
7. IF a Sandbox does not pass the health check within 180 seconds of the start request, THEN THE Builder SHALL stop the start attempt, show an error message in the Preview_Panel indicating the failed start-up stage, offer a retry action, and keep every Version in the Version_Store unchanged.
8. IF a dependency install, type-check, lint, build, database migration, or test command runs longer than the per-command time limit stated in Requirement 13, criterion 4, THEN THE Builder SHALL stop the command, record a timeout result with the command's log output in the Verification_Report, and mark that Verification_Pipeline check as failed.

### Requirement 16: Live Preview [MVP]

**User Story:** As a User, I want to see and click through my running app next to the chat, so that I can judge the result immediately.

#### Acceptance Criteria

1. WHEN the Version_Store saves a Verified_Version, THE Preview_Panel SHALL display the running Generated_Project of that Version at its root route within 10 seconds of the save, without the User reloading the Workspace_UI.
2. THE Preview_Panel SHALL provide viewport presets for mobile at 375 CSS pixels wide, tablet at 768 CSS pixels wide, and desktop at the full panel width, SHALL use the desktop preset by default, SHALL mark the active preset, and SHALL switch presets without reloading the running Generated_Project or changing its current route.
3. WHEN a User selects reload, THE Preview_Panel SHALL reload the running Generated_Project at the current route, including its query string, without restarting the Generated_Project server and without changing the displayed Version.
4. THE Preview_Panel SHALL display, in a log area that is collapsed by default and shows the count of entries while collapsed, every frontend console error, uncaught frontend exception, and backend server error output from the Sandbox for the displayed Version, with each entry showing its source (frontend or backend), timestamp, and message, keeping at most the 500 most recent entries and clearing the entries when a different Version is displayed.
5. WHEN a User selects send error to chat on a displayed log entry, THE Chat_Panel SHALL post a fix request containing that entry's source, message, and stack trace (if present), truncated to 4,000 characters, and SHALL start a chat edit Generation_Run for the Project.
6. WHEN a User selects open in new tab, THE Builder SHALL open the running Generated_Project of the displayed Version in a new browser tab at the current route, at a URL accessible only to the authenticated owner of the Project.
7. IF the Generated_Project server process exits, or the Preview_Panel cannot load the running Generated_Project within 10 seconds of a Verified_Version being saved, THEN THE Preview_Panel SHALL display an error state indicating that the preview failed to load, keep the log area entries, and offer the reload control.
8. IF a request to the preview URL of a Project comes from an unauthenticated person or from a User who does not own the Project, THEN THE Builder SHALL deny the request without returning any Generated_Project content and SHALL display a message indicating that access is denied.
9. WHILE a Generation_Run is in progress for the Project, THE Preview_Panel SHALL keep displaying the last Verified_Version, SHALL show an indicator that a Generation_Run is in progress, and SHALL disable the send error to chat control.

### Requirement 17: Code View and Manual Editing [MVP]

**User Story:** As a User, I want to browse and edit the generated code, so that I can understand and fine-tune my app.

#### Acceptance Criteria

1. WHEN a User selects a file in the Code_Panel file tree, THE Code_Panel SHALL display the content of that file with syntax highlighting within 1 second for files up to 1 MB. The file tree SHALL list folders before files, each group sorted alphabetically. Binary files and files larger than 1 MB SHALL appear as non-editable, with a notice that names the reason.
2. WHEN a User selects a file in a Version's change list, THE Code_Panel SHALL display a line-by-line diff that marks added and removed lines. The diff SHALL compare the file in that Version with the same file path in the Version immediately before it in the Project's Version list. A file added in that Version SHALL appear with all lines marked added. A file deleted in that Version SHALL appear with all lines marked removed. The Project's first Version SHALL appear with all lines marked added.
3. WHEN a User selects save in the Code_Panel while at least one file has content that differs from the latest Version, THE Builder SHALL apply all unsaved edits as one manual edit to the Generated_Project and run the Verifier. Unsaved edits are changes typed in the Code_Panel editor that have not yet been saved.
4. WHEN the Verifier finishes for a manual edit, THE Version_Store SHALL save a Version. The Version SHALL have the origin manual edit, a description that lists the paths of the changed files, and the Verification_Report. The Version SHALL be marked verified only if every Verification_Pipeline check passed.
5. WHILE a Generation_Run is in progress or the Verifier is running for a manual edit, THE Code_Panel SHALL display files as read-only and disable the save action.
6. IF the Verification_Report for a manual edit contains at least one failed check, THEN THE Builder SHALL save the result as an unverified Version without starting the Repair_Loop, keep the Preview_Panel on the last Verified_Version, and display the failed check names and their log output to the User.
7. IF the Builder cannot apply a manual edit to the Generated_Project or cannot start the Verifier, THEN THE Builder SHALL display an error message naming the failed step, save no Version, and keep the unsaved edits in the Code_Panel editor.
8. IF a User with unsaved edits selects another file, closes the Workspace_UI, or starts a Generation_Run, THEN THE Code_Panel SHALL ask the User to save or discard the unsaved edits before it continues the action.

### Requirement 18: Iterative Chat Edits [MVP]

**User Story:** As a User, I want to ask for changes to my existing app in the chat, so that I can refine it step by step without starting over.

#### Acceptance Criteria

1. WHEN a User submits a Prompt in a Project that has an approved App_Plan, THE Planner SHALL produce a Change_Plan that lists each App_Plan item to add, modify, or remove and each Generated_Project file path to create, modify, rename, or delete, where every referenced App_Plan item exists in the approved App_Plan or is added by the same Change_Plan.
2. WHEN the Planner produces a Change_Plan that adds, modifies, or removes at least one data entity, API endpoint, page, or authentication need, THE Workspace_UI SHALL display the Change_Plan with approve and reject controls and SHALL NOT start the Generator until the User selects approve.
3. WHEN the Planner produces a Change_Plan that adds, modifies, or removes no data entity, API endpoint, page, or authentication need, THE Generator SHALL start the chat edit Generation_Run without an approval step.
4. WHEN a chat edit Generation_Run saves a Version, whether verified or unverified, THE Builder SHALL update the approved App_Plan to include every App_Plan item addition, modification, and removal listed in the Change_Plan, and SHALL save that Version with the origin chat edit.
5. IF, outside the Repair_Loop, the Edit_Parser output for a chat edit contains a File_Operation that targets a file path not listed in the Change_Plan, THEN THE Generator SHALL apply none of the File_Operations in that LLM_Provider response and SHALL request a corrected response up to 2 times before marking the chat edit Generation_Run as failed with the Generated_Project files kept equal to the latest Version.
6. WHEN a chat edit Generation_Run saves a Version, THE Chat_Panel SHALL display a summary that lists each changed file path with its operation type (created, modified, renamed, or deleted), including files changed by the Repair_Loop, the overall Verification_Report result (passed or failed), and the Version identifier.
7. IF the User selects reject on a displayed Change_Plan, THEN THE Builder SHALL leave the Generated_Project files, the approved App_Plan, and the latest Version unchanged, record the rejection in the Project chat history, and re-enable Prompt submission.
8. IF the Planner output for a chat edit fails Change_Plan validation after 2 correction requests to the LLM_Provider, THEN THE Builder SHALL display a planning failure message in the Chat_Panel and leave the Generated_Project files, the approved App_Plan, and the latest Version unchanged.

### Requirement 19: Version History and Rollback [MVP]

**User Story:** As a User, I want every change saved as a version I can return to, so that I can experiment without fear of losing a working app.

#### Acceptance Criteria

1. WHEN a Generation_Run finishes, whether the final Verifier run passed or the Repair_Loop stopped with failures, THE Version_Store SHALL save a new Version that contains all Generated_Project files, the origin plan generation or chat edit matching the Generation_Run's trigger, a description of 1 to 200 characters summarizing the change, and the Verification_Report from the final Verifier run.
2. WHEN the Verifier finishes for a manual edit saved in the Code_Panel (Requirement 17, criteria 3 and 4), THE Version_Store SHALL save a new Version that contains all Generated_Project files, the origin manual edit, a description of 1 to 200 characters that names the changed file paths, and the Verification_Report from that Verifier run, without running the Repair_Loop.
3. THE Workspace_UI SHALL list the Project's Versions newest first, 50 Versions at a time with a control that loads the next 50. Each entry SHALL show the Version's description, its timestamp as a date and time to the second in the User's local time zone, its origin, and the status "verified" if the Version is a Verified_Version or "unverified" otherwise. A Version with no Verification_Report counts as unverified.
4. WHEN a User selects a Version from the list, THE Preview_Panel SHALL display that Version running in the Sandbox within 60 seconds, together with a label naming the selected Version's identifier and a control that returns the Preview_Panel to the latest Version. The latest Version SHALL stay unchanged.
5. WHEN a User confirms rollback to a Version in a confirmation dialog that states the Sandbox database will be reset and its current data discarded, THE Version_Store SHALL save a new Version with the origin rollback, a description that contains the selected Version's identifier, and files identical to the selected Version's files.
6. FOR ALL Versions V, rolling back to V SHALL produce a Generated_Project whose set of file paths and file contents equals those of V. Files that were added after V SHALL be absent (invariant property).
7. WHEN a rollback Version is saved, THE Builder SHALL reset the Sandbox database, apply the rollback Version's migrations and seed data, run the Verifier, and attach the resulting Verification_Report to the rollback Version without running the Repair_Loop.
8. THE Version_Store SHALL NOT modify or delete the files, description, timestamp, origin, or Verification_Report of any existing Version when it saves a new Version or performs a rollback.
9. IF saving a Version fails, THEN THE Version_Store SHALL leave the existing Versions and the current Generated_Project files unchanged, and THE Workspace_UI SHALL display an error message stating that the Version was not saved.
10. WHILE a Generation_Run is in progress for the Project, THE Workspace_UI SHALL disable the rollback control and display a message stating that rollback is available after the Generation_Run finishes.
11. IF the selected Version does not start in the Sandbox within 60 seconds, THEN THE Preview_Panel SHALL display an error message stating that the Version could not be previewed, and the latest Version SHALL stay unchanged.

### Requirement 20: Code Export [MVP]

**User Story:** As a User, I want to download my project's code, so that I own it and can run it anywhere.

#### Acceptance Criteria

1. WHEN a User requests export of a Version of a Project that the User owns, THE Exporter SHALL produce a ZIP archive of that Version's Generated_Project files, named with the Project name and the Version identifier, and SHALL start the download within 30 seconds for a Version whose included files total at most 100 MB.
2. THE Exporter SHALL exclude from the archive every Secret_File and every dependency directory and build output directory defined for the Target_Stack in the design document.
3. FOR ALL Versions V, extracting the exported archive of V SHALL produce exactly the file paths, relative to the Generated_Project root, and byte-identical file contents of V's files minus the files excluded by criterion 2, with no additional files (round-trip property).
4. IF a User requests export of a Version that is not a Verified_Version, THEN THE Exporter SHALL display a warning before the download starts. The warning SHALL name each failed Verification_Pipeline check, or state that verification did not complete. THE Exporter SHALL start the download only after the User confirms the warning, and SHALL produce no download if the User cancels.
5. IF a User requests export of a Version of a Project that the User does not own, THEN THE Builder SHALL reject the request without producing an archive and SHALL display an error message indicating that access is denied.
6. IF archive production fails or does not finish within 120 seconds, THEN THE Exporter SHALL stop the export, deliver no partial archive, leave the Version unchanged, and display an error message indicating that the export failed and offering a retry.
7. WHEN the Exporter excludes one or more Secret_Files from an archive, THE Exporter SHALL show the User the path of each excluded Secret_File before the download starts.

### Requirement 21: Workspace User Interface [MVP]

**User Story:** As a User, I want a clean, responsive, and animated builder interface, so that building apps is fast and pleasant.

#### Acceptance Criteria

1. WHILE the browser viewport is at least 1024 CSS pixels wide, THE Workspace_UI SHALL display the Chat_Panel on the left and a tabbed Preview_Panel and Code_Panel on the right, separated by a divider that the User can move by pointer drag and by arrow keys when the divider has keyboard focus, and that cannot be moved to a position where either side is narrower than 320 CSS pixels.
2. WHILE the browser viewport is narrower than 1024 CSS pixels and at least 320 CSS pixels wide, THE Workspace_UI SHALL display exactly one of the Chat_Panel, the Preview_Panel, or the Code_Panel at a time, SHALL keep a switch control for those three panels visible without scrolling, and SHALL not require horizontal page scrolling outside the code editor and the Preview_Panel content.
3. WHEN the User selects the light theme or the dark theme, THE Workspace_UI SHALL apply the selected theme to all Workspace_UI panels within 100 milliseconds without a page reload, SHALL leave the Generated_Project content shown in the Preview_Panel unchanged, and SHALL apply the same theme in the User's later sessions.
4. WHEN the Builder receives a chunk of a streamed LLM_Provider response, THE Chat_Panel SHALL append the chunk text to the displayed message within 100 milliseconds of receipt, without waiting for the full response, and THE Workspace_UI SHALL animate each new message entrance and each panel or tab switch with a transition lasting 300 milliseconds or less.
5. WHILE the Reduced_Motion_Preference is set, THE Workspace_UI SHALL replace movement-based animations, including message entrance and panel transitions, with no animation or opacity-only transitions of 150 milliseconds or less.
6. THE Workspace_UI SHALL meet WCAG 2.1 level AA success criteria in both the light theme and the dark theme for color contrast (at least 4.5:1 for normal text, at least 3:1 for large text, UI component boundaries, and focus indicators), keyboard operation (every control, including the divider and the panel switch control, operable by keyboard alone with no keyboard trap), and visible focus (a visible focus indicator on every control that has keyboard focus), excluding the Generated_Project content shown in the Preview_Panel.
7. WHEN a User activates a control in the Workspace_UI by pointer click, tap, or the Enter or Space key, THE Workspace_UI SHALL show a visible change in that control's state (pressed, selected, or loading) within 100 milliseconds, and IF the triggered action has not completed within 1 second, THEN THE Workspace_UI SHALL show a progress indicator until the action completes or fails.
8. WHEN the browser viewport width crosses 1024 CSS pixels in either direction, THE Workspace_UI SHALL switch between the layouts in criteria 1 and 2 without a page reload and SHALL keep the unsent Prompt text, the file open in the Code_Panel, and the running state of the Preview_Panel.
9. IF a streamed LLM_Provider response ends with an error or no chunk arrives for 30 seconds, THEN THE Chat_Panel SHALL keep the text received so far, SHALL mark the message as incomplete with an error indication, and SHALL show a control to retry the request.
10. IF the User has not selected a theme, THEN THE Workspace_UI SHALL use the dark theme when the browser reports a dark color scheme preference and the light theme otherwise.

### Requirement 22: Generation Progress, Cancellation, and Usage [MVP]

**User Story:** As a User, I want to see what the Builder is doing and stop it when needed, so that I stay in control of time and cost.

#### Acceptance Criteria

1. WHILE a Generation_Run is in progress, THE Workspace_UI SHALL display each Plan_Step and each Verification_Pipeline check with exactly one status from pending, in progress, passed, failed, not run, or cancelled, and SHALL show a status change within 2 seconds of the Builder recording it.
2. WHEN a User selects cancel during a Generation_Run, including while the Generation_Run is paused under criterion 5, THE Builder SHALL, within 5 seconds, abort all in-flight LLM_Provider requests, send no new LLM_Provider requests, and terminate all running Sandbox commands for that Generation_Run.
3. WHEN a Generation_Run is cancelled, THE Builder SHALL discard every File_Operation that the Generation_Run applied, keep the Generated_Project files byte-identical to the latest Version (or to an empty Generated_Project if the Project has no Version), create no new Version, mark every Plan_Step and Verification_Pipeline check that was pending or in progress as cancelled, and add a chat history entry that records the cancellation with its timestamp.
4. WHEN a Generation_Run ends by completion, failure, or cancellation, THE Chat_Panel SHALL display the total input tokens and total output tokens that the LLM_Provider reported for every request in the Generation_Run, and IF the LLM_Provider did not report token counts for one or more requests, THEN THE Chat_Panel SHALL label the displayed totals as incomplete.
5. IF the sum of input tokens and output tokens consumed by a Generation_Run reaches or exceeds the per-run token limit in the Project settings after any LLM_Provider response, THEN THE Builder SHALL pause the Generation_Run, send no further LLM_Provider requests or Sandbox commands while paused, and display a prompt in the Chat_Panel that shows the tokens consumed and offers continue and cancel actions.
6. IF a User enters a per-run token limit that is not a whole number from 1,000 to 10,000,000, THEN THE Builder SHALL reject the value, keep the previously saved per-run token limit, and display an error message indicating the allowed range.
7. WHEN a User selects continue on a paused Generation_Run, THE Builder SHALL resume the Generation_Run from the Plan_Step or Verification_Pipeline check where it paused and SHALL allow additional consumption equal to one more per-run token limit before pausing again under criterion 5.
8. IF a User selects neither continue nor cancel within 30 minutes after a Generation_Run pauses under criterion 5, THEN THE Builder SHALL cancel the Generation_Run as specified in criteria 2 and 3.

### Requirement 23: Secrets and Data Protection [MVP]

**User Story:** As a User, I want my API_Keys and data protected, so that nobody else can use or read them.

#### Acceptance Criteria

1. THE Provider_Registry SHALL store each API_Key only in encrypted form, using an encryption key held outside the Builder database, so that no Builder database record or database backup contains the plaintext API_Key.
2. WHEN the Workspace_UI requests a saved API_Key for display, THE Provider_Registry SHALL return a masked value that contains only the last 4 characters of an API_Key longer than 8 characters, contains no characters of an API_Key of 8 characters or fewer, and never contains the full API_Key.
3. THE Builder SHALL replace every occurrence of a saved API_Key, a User password, a session token, or a Project_Secret value with a fixed redaction placeholder before writing to application logs, chat history, or Verification_Reports, and before sending request content to an LLM_Provider. The one exception is the selected LLM_Provider's API_Key, which the Builder sends to that LLM_Provider only as the request's authentication credential.
4. THE Generator SHALL write Project_Secret values only to Sandbox environment variables and to no Generated_Project file, Version, or archive produced by the Exporter.
5. THE Builder SHALL transmit all traffic between the browser and the Builder server over HTTPS. IF the browser sends a request over unencrypted HTTP, THEN THE Builder SHALL redirect to the HTTPS address of the same resource without processing the request.
6. IF a request reads, updates, uses, or deletes an API_Key that the requesting User did not save, THEN THE Provider_Registry SHALL reject the request with a not-found response and leave the API_Key unchanged.
7. IF the Provider_Registry cannot access the encryption key, THEN THE Provider_Registry SHALL reject every API_Key save and every LLM_Provider request that needs a saved API_Key, display an error message stating that saved credentials are temporarily unavailable, and store no plaintext API_Key.
8. IF a File_Operation produced by the Generator contains a Project_Secret value or a saved API_Key, THEN THE Generator SHALL, before applying the File_Operation, replace the Project_Secret value with a reference to that Project_Secret's Sandbox environment variable name and replace the API_Key with the fixed redaction placeholder.

### Requirement 24: Third-Party Code Licensing [MVP]

**User Story:** As the Builder owner, I want reused open-source code tracked with its license, so that the Builder can be distributed legally.

#### Acceptance Criteria

1. THE Builder source repository SHALL contain a third-party notices file with one entry for each third-party component included in the Builder: each direct dependency, each transitive dependency, and each piece of code copied or adapted from a reference project. Each entry SHALL record the component name, source URL, version or commit identifier, and license identifier.
2. THE Builder source repository SHALL contain an Approved_License_List. Its initial contents SHALL be MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, and ISC.
3. THE Builder source repository SHALL include code copied or adapted from a reference project only when all three conditions hold: the license that applies to the copied files is on the Approved_License_List, the copied files keep the original copyright and license notice, and the third-party notices file has an entry for the copied code.
4. WHEN the Builder build process runs, THE Builder build process SHALL check the license of every direct and transitive dependency against the Approved_License_List. IF any license is not on the list, THEN the build SHALL fail and report the name, version, and license of each component that does not comply.
5. IF the Builder build process cannot determine the license of a dependency, THEN THE Builder build process SHALL fail the build and report the name and version of each dependency whose license is unknown.
6. IF a dependency included in the Builder build has no entry in the third-party notices file, or its entry shows a version that differs from the version used in the build, THEN THE Builder build process SHALL fail the build and report the name and version of each dependency that is missing or mismatched.

### Requirement 25: Push to GitHub [Later Phase]

**User Story:** As a User, I want to push my project to a GitHub repository, so that I can version and share the code with standard tools.

#### Acceptance Criteria

1. WHEN a User connects a GitHub account, THE GitHub_Integration SHALL request only the permissions to create repositories and to write commits to repository contents, and SHALL NOT request permission to delete repositories, manage organization settings, or change account settings.
2. WHEN a User pushes a Version to a new or existing repository, THE GitHub_Integration SHALL create exactly one commit on that repository's default branch whose file paths and file contents equal the files the Exporter includes in that Version's archive. The commit SHALL delete files that are on the branch but not in that set, SHALL have a message containing the Version identifier and the Version description, and SHALL produce a link to the created commit shown in the Workspace_UI. For a new repository, THE GitHub_Integration SHALL create it with private visibility unless the User selects public visibility.
3. IF a push to GitHub fails, whether GitHub returns an error, the network connection fails, or no response arrives within 120 seconds, THEN THE GitHub_Integration SHALL display the error returned by GitHub, or an error message indicating a network failure or timeout. It SHALL NOT display a success link and SHALL leave the Project's Versions unchanged.
4. WHEN a User disconnects the GitHub account, THE GitHub_Integration SHALL delete the stored GitHub_Access_Token, show the GitHub account as disconnected in the Workspace_UI, and reject push requests until the User connects a GitHub account again.
5. THE Builder SHALL store GitHub_Access_Tokens encrypted at rest with a key held outside the Builder database, and SHALL redact GitHub_Access_Tokens from application logs, chat history, Verification_Reports, LLM_Provider requests, and Generated_Project files.
6. IF GitHub rejects the stored GitHub_Access_Token as expired or revoked, THEN THE GitHub_Integration SHALL delete the stored GitHub_Access_Token, show the GitHub account as disconnected, display a message asking the User to reconnect, and leave the Project's Versions unchanged.
7. IF the Version a User selects for a push is not a Verified_Version, THEN THE GitHub_Integration SHALL display a warning naming the failed Verification_Pipeline check and start the push only after the User confirms.

### Requirement 26: One-Click Deployment [Later Phase]

**User Story:** As a User, I want to publish my app to a public URL with one action, so that others can use it.

#### Acceptance Criteria

1. WHEN a User selects deploy on a Verified_Version, THE Deployer SHALL provision the frontend, backend, and a production database for the Project, apply the Version's migrations to the production database in their defined order, and publish the app within 15 minutes of the deploy selection at the Project's public HTTPS URL, which stays the same across all later deployments of the same Project.
2. IF a User selects deploy on a Version that is not a Verified_Version, THEN THE Deployer SHALL block the deployment without provisioning any resource or changing the production database, and SHALL display the name and log output of each failed Verification_Pipeline check, or a message stating that the Version has no Verification_Report if verification has not run.
3. WHEN the Deployer publishes a Version, THE Deployer SHALL run the health check against the public URL up to 3 times at 10-second intervals and display a pass result if any attempt passes within 60 seconds of publishing, or a fail result otherwise.
4. IF provisioning, migration, publishing, or the health check fails, or the deployment exceeds the 15-minute limit, THEN THE Deployer SHALL mark the deployment as failed, return the production database to its state from before the deployment started, keep the previously deployed Version serving traffic at the public URL (or serve no app if no Version was deployed before), and display the failed stage and its log output.
5. IF a deployment would apply a migration that drops a table, a collection, or a field holding at least one record in the production database, THEN THE Deployer SHALL pause the deployment before applying any migration, list each affected table, collection, and field with its record count, and require explicit User confirmation before applying the migration.
6. IF the User declines the confirmation required in criterion 5, or gives no response within 30 minutes, THEN THE Deployer SHALL cancel the deployment without applying any migration, keep the previously deployed Version serving traffic, and display a message stating that the deployment was cancelled.
7. IF a User selects deploy while another deployment of the same Project is in progress, THEN THE Deployer SHALL reject the new deployment request and display a message stating that a deployment is already in progress.
8. WHEN a deployment passes the health check, THE Deployer SHALL display the deployed Version's identifier, the deployment timestamp, and the public URL in the Workspace_UI.
