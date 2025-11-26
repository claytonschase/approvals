# Approvals – V1 Product Spec

## 1. Overview

Approvals is an internal web app used by the team to manage design/engineering approvals that are linked to jobs in JobTread.

Each **Approval** is attached to a JobTread job, has specifications, files, due dates, statuses, and an email “nudging” schedule that reminds the customer until the approval is completed.

V1 is for **internal team users only** (no customer portal yet). Customers are contacted via email (Mailgun) and any customer-facing view will be added later via magic links.

---

## 2. Goals (V1)

- Provide a **login portal** for team members (Admins + Users).
- Show a **dashboard of active approvals** in a table with sorting/filtering.
- Allow users to **create and edit approvals** that are linked to JobTread jobs.
- Track **versions** of approvals when specs/files change.
- Send **scheduled email reminders** using Mailgun until an approval is completed.
- Maintain a basic **activity log** for each approval.

---

## 3. Non-goals (V1)

- No public customer login or portal.
- No Zapier integration.
- No deep JobTread automation (we only need lookups + basic link for now).
- No complex reporting/analytics (simple table + filters are enough).

---

## 4. User types & permissions

### 4.1 Roles

- **Admin**
  - Manage users.
  - Manage approval statuses.
  - Manage email schedules and templates.
  - Update integration settings (JobTread API key, Mailgun settings).
  - Full CRUD on approvals.

- **User**
  - Login and use the dashboard.
  - Create and edit approvals.
  - Upload files and create revisions.
  - Change approval status (within allowed set).
  - Cannot change global settings or user access.

### 4.2 Authentication

- V1: Email + password login.
- Users are created/invited by an Admin.
- Sessions handled in the app (NextAuth or equivalent).

---

## 5. Core entity: Approval

### 5.1 Fields

- **ID** (internal)
- **Name** – human-readable name of the approval.
- **JobTread link**
  - `jobtreadJobId` (required)
  - `jobtreadJobName` (snapshot)
  - `jobtreadJobNumber` (optional snapshot)
- **Customer info** (for email)
  - `customerName` (optional but recommended)
  - `customerEmail` (optional but recommended; required to send emails)
- **Specifications**
  - Long text field describing the approval details.
  - Maintain a **history** when changed (via versions).
- **Due date**
  - Date (no time is fine for now).
- **Status**
  - Linked to a configurable status (e.g. Draft, Sent to Customer, Approved, Rejected, On Hold, Archived).
- **Version**
  - Current version number (integer).
- **Files**
  - One or more uploaded files per approval/version.
- **Email schedule**
  - Optional reference to a pre-built email schedule.
- **Flags**
  - `isArchived` – archive approvals that are no longer active.

---

## 6. Approval lifecycle

Typical flow:

1. **Create Approval**
   - User selects a JobTread job from a searchable dropdown.
   - Fills in name, specs, due date, status (likely `Draft`), optional schedule.
   - Uploads initial files.
   - Version starts at `1`.

2. **Send to customer**
   - User changes status to e.g. `Sent to Customer`.
   - If an email schedule is selected and customer email is present:
     - Start the schedule (emails will be sent until Approved/Rejected/Archived).

3. **Revision**
   - When specs and/or files are updated:
     - Increment `versionNumber`.
     - Ask user to enter a **revision comment**.
     - Save a new `ApprovalVersion` snapshot (specs + link to files).
     - Log an activity entry.

4. **Approval outcome**
   - Status changes to `Approved` or `Rejected`.
   - Email schedule is automatically **stopped**.
   - Approval can later be **archived** if needed (no longer active).

---

## 7. Dashboard

### 7.1 View

- Default view: **Active approvals**
  - “Active” = approvals not archived (and optionally not Draft).
- Table columns (V1):
  - Approval Name
  - JobTread Job (name + job number)
  - Status
  - Due date
  - Current version
  - Customer name
  - Customer email
  - Last updated

### 7.2 Basic interactions

- Filter approvals by:
  - Status
  - JobTread job
  - Due date range (optional V1)
- Sort by:
  - Due date
  - Last updated
- Row actions:
  - View / edit approval
  - Archive approval (Admin only)

---

## 8. JobTread integration

### 8.1 V1 behaviour

- **Single company-wide API key** stored in settings.
- **Searchable job dropdown** when creating or editing an approval:
  - User types to search JobTread jobs (by name / number).
  - Backed by a JobTread API endpoint.
- For each approval:
  - Store `jobtreadJobId` and snapshot of name/number.
  - Show a clickable link to the JobTread job in the UI.

### 8.2 Future (not in V1, but keep in mind)

- Trigger JobTread actions based on approval status changes.
- Sync approval data back into JobTread custom fields.

---

## 9. Email schedules (Mailgun)

### 9.1 Definitions

- **Email schedule**:
  - Has a name (e.g. “Standard”, “Aggressive”).
  - Has one or more schedule items:
    - Offset in days relative to schedule start (e.g. 0, 3, 7).
    - Subject template.
    - Body template.
  - One schedule can be used by many approvals.

- **Schedule start**:
  - Triggered when approval status changes to “Sent to Customer” (or a configured start status).

### 9.2 Behaviour

- A **daily job** runs (via cron or scheduled endpoint):
  - For each approval with an active schedule and not in a final state:
    - Calculate which emails are due based on:
      - Schedule items.
      - When the schedule started.
      - Which emails have already been sent.
    - Send due emails via Mailgun.
    - Write to an `approval_email_logs` table.

- **Schedule stops** when:
  - Approval status changes to a final status (e.g. Approved, Rejected, Archived).

---

## 10. Versioning & history

### 10.1 Versioning

- `currentVersion` on Approval.
- Each time “revision-worthy” changes happen (files/specs):
  - `currentVersion` increments by 1.
  - A new `ApprovalVersion` record is created:
    - Version number.
    - Snapshot of specifications.
    - Revision comment.
    - Timestamp.

- Files:
  - Each file is associated with an approval and a specific version number.
  - Files stored in DigitalOcean Spaces (S3-style); DB stores metadata + key.

### 10.2 Activity log

- Basic `ApprovalActivityLog` entries for:
  - Approval created.
  - Status changed.
  - Specs changed.
  - File uploaded.
  - Email sent (schedule-related).
- Not necessarily surfaced heavily in UI for V1, but data is captured.

---

## 11. Settings / Admin

V1 Settings page for Admins:

- JobTread:
  - JobTread API key.
- Mailgun:
  - API key.
  - From email address.
- Approval statuses:
  - List of statuses (name, sort order, flags e.g. “isFinal”, “isActiveDefault”).
- Email schedules:
  - Create/edit schedules.
  - Add/remove schedule items (offset days + subject/body templates).
- General:
  - Default email schedule for new approvals (optional).

---

## 12. Tech stack (high level)

- **Framework**: Next.js with TypeScript.
- **Database**: Postgres (managed by DigitalOcean).
- **ORM**: Prisma.
- **Auth**: NextAuth (email/password).
- **Email**: Mailgun.
- **File storage**: DigitalOcean Spaces (S3-compatible).
- **UI**: Tailwind CSS + shadcn/ui.

---

## 13. Out of scope for V1

- Customer login / magic links.
- Zapier integration.
- JobTread webhooks & advanced automation.
- Rich reporting/analytics.
- Mobile app.# Approvals – V1 Product Spec

## 1. Overview

Approvals is an internal web app used by the team to manage design/engineering approvals that are linked to jobs in JobTread.

Each **Approval** is attached to a JobTread job, has specifications, files, due dates, statuses, and an email “nudging” schedule that reminds the customer until the approval is completed.

V1 is for **internal team users only** (no customer portal yet). Customers are contacted via email (Mailgun) and any customer-facing view will be added later via magic links.

---

## 2. Goals (V1)

- Provide a **login portal** for team members (Admins + Users).
- Show a **dashboard of active approvals** in a table with sorting/filtering.
- Allow users to **create and edit approvals** that are linked to JobTread jobs.
- Track **versions** of approvals when specs/files change.
- Send **scheduled email reminders** using Mailgun until an approval is completed.
- Maintain a basic **activity log** for each approval.

---

## 3. Non-goals (V1)

- No public customer login or portal.
- No Zapier integration.
- No deep JobTread automation (we only need lookups + basic link for now).
- No complex reporting/analytics (simple table + filters are enough).

---

## 4. User types & permissions

### 4.1 Roles

- **Admin**
  - Manage users.
  - Manage approval statuses.
  - Manage email schedules and templates.
  - Update integration settings (JobTread API key, Mailgun settings).
  - Full CRUD on approvals.

- **User**
  - Login and use the dashboard.
  - Create and edit approvals.
  - Upload files and create revisions.
  - Change approval status (within allowed set).
  - Cannot change global settings or user access.

### 4.2 Authentication

- V1: Email + password login.
- Users are created/invited by an Admin.
- Sessions handled in the app (NextAuth or equivalent).

---

## 5. Core entity: Approval

### 5.1 Fields

- **ID** (internal)
- **Name** – human-readable name of the approval.
- **JobTread link**
  - `jobtreadJobId` (required)
  - `jobtreadJobName` (snapshot)
  - `jobtreadJobNumber` (optional snapshot)
- **Customer info** (for email)
  - `customerName` (optional but recommended)
  - `customerEmail` (optional but recommended; required to send emails)
- **Specifications**
  - Long text field describing the approval details.
  - Maintain a **history** when changed (via versions).
- **Due date**
  - Date (no time is fine for now).
- **Status**
  - Linked to a configurable status (e.g. Draft, Sent to Customer, Approved, Rejected, On Hold, Archived).
- **Version**
  - Current version number (integer).
- **Files**
  - One or more uploaded files per approval/version.
- **Email schedule**
  - Optional reference to a pre-built email schedule.
- **Flags**
  - `isArchived` – archive approvals that are no longer active.

---

## 6. Approval lifecycle

Typical flow:

1. **Create Approval**
   - User selects a JobTread job from a searchable dropdown.
   - Fills in name, specs, due date, status (likely `Draft`), optional schedule.
   - Uploads initial files.
   - Version starts at `1`.

2. **Send to customer**
   - User changes status to e.g. `Sent to Customer`.
   - If an email schedule is selected and customer email is present:
     - Start the schedule (emails will be sent until Approved/Rejected/Archived).

3. **Revision**
   - When specs and/or files are updated:
     - Increment `versionNumber`.
     - Ask user to enter a **revision comment**.
     - Save a new `ApprovalVersion` snapshot (specs + link to files).
     - Log an activity entry.

4. **Approval outcome**
   - Status changes to `Approved` or `Rejected`.
   - Email schedule is automatically **stopped**.
   - Approval can later be **archived** if needed (no longer active).

---

## 7. Dashboard

### 7.1 View

- Default view: **Active approvals**
  - “Active” = approvals not archived (and optionally not Draft).
- Table columns (V1):
  - Approval Name
  - JobTread Job (name + job number)
  - Status
  - Due date
  - Current version
  - Customer name
  - Customer email
  - Last updated

### 7.2 Basic interactions

- Filter approvals by:
  - Status
  - JobTread job
  - Due date range (optional V1)
- Sort by:
  - Due date
  - Last updated
- Row actions:
  - View / edit approval
  - Archive approval (Admin only)

---

## 8. JobTread integration

### 8.1 V1 behaviour

- **Single company-wide API key** stored in settings.
- **Searchable job dropdown** when creating or editing an approval:
  - User types to search JobTread jobs (by name / number).
  - Backed by a JobTread API endpoint.
- For each approval:
  - Store `jobtreadJobId` and snapshot of name/number.
  - Show a clickable link to the JobTread job in the UI.

### 8.2 Future (not in V1, but keep in mind)

- Trigger JobTread actions based on approval status changes.
- Sync approval data back into JobTread custom fields.

---

## 9. Email schedules (Mailgun)

### 9.1 Definitions

- **Email schedule**:
  - Has a name (e.g. “Standard”, “Aggressive”).
  - Has one or more schedule items:
    - Offset in days relative to schedule start (e.g. 0, 3, 7).
    - Subject template.
    - Body template.
  - One schedule can be used by many approvals.

- **Schedule start**:
  - Triggered when approval status changes to “Sent to Customer” (or a configured start status).

### 9.2 Behaviour

- A **daily job** runs (via cron or scheduled endpoint):
  - For each approval with an active schedule and not in a final state:
    - Calculate which emails are due based on:
      - Schedule items.
      - When the schedule started.
      - Which emails have already been sent.
    - Send due emails via Mailgun.
    - Write to an `approval_email_logs` table.

- **Schedule stops** when:
  - Approval status changes to a final status (e.g. Approved, Rejected, Archived).

---

## 10. Versioning & history

### 10.1 Versioning

- `currentVersion` on Approval.
- Each time “revision-worthy” changes happen (files/specs):
  - `currentVersion` increments by 1.
  - A new `ApprovalVersion` record is created:
    - Version number.
    - Snapshot of specifications.
    - Revision comment.
    - Timestamp.

- Files:
  - Each file is associated with an approval and a specific version number.
  - Files stored in DigitalOcean Spaces (S3-style); DB stores metadata + key.

### 10.2 Activity log

- Basic `ApprovalActivityLog` entries for:
  - Approval created.
  - Status changed.
  - Specs changed.
  - File uploaded.
  - Email sent (schedule-related).
- Not necessarily surfaced heavily in UI for V1, but data is captured.

---

## 11. Settings / Admin

V1 Settings page for Admins:

- JobTread:
  - JobTread API key.
- Mailgun:
  - API key.
  - From email address.
- Approval statuses:
  - List of statuses (name, sort order, flags e.g. “isFinal”, “isActiveDefault”).
- Email schedules:
  - Create/edit schedules.
  - Add/remove schedule items (offset days + subject/body templates).
- General:
  - Default email schedule for new approvals (optional).

---

## 12. Tech stack (high level)

- **Framework**: Next.js with TypeScript.
- **Database**: Postgres (managed by DigitalOcean).
- **ORM**: Prisma.
- **Auth**: NextAuth (email/password).
- **Email**: Mailgun.
- **File storage**: DigitalOcean Spaces (S3-compatible).
- **UI**: Tailwind CSS + shadcn/ui.

---

## 13. Out of scope for V1

- Customer login / magic links.
- Zapier integration.
- JobTread webhooks & advanced automation.
- Rich reporting/analytics.
- Mobile app.