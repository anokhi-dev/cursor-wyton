# Wyton — Notifications Audit

Scope: `wyton-api` (backend) + `wyton-web` (frontend). Audit date: 2026-07-17.

**Channels used**
- **Email** — Gmail API (admin OAuth refresh token) via [services/emailService.js](wyton-api/services/emailService.js). No SendGrid/SES/SMTP in active use. Legacy Nodemailer copy + SES helper exist but are dead code.
- **Push** — Firebase Cloud Messaging via [utils/fcmPushNotification.js](wyton-api/utils/fcmPushNotification.js) (`firebase-admin`). Tokens stored in `fmc_token` collection.
- **In-app** — Mongo `in_app_notification` documents written by `userServices.create_in_app_notification` in [services/userServices.js:591](wyton-api/services/userServices.js). Rendered by the web bell + `/notifications` page.
- **SMS/Twilio** — [utils/sms.js](wyton-api/utils/sms.js) exists, not imported anywhere. Sockets.io is NOT used.

**Recipient roles referenced below**: admin, employee, vendor, consultant, invitee (pre-registration), plus per-item participants (responsible person, ball-in-court, member, watcher, tracker, approver, assignee).

---

## 1. Auth / User Account

| # | Trigger | Channel | Recipient | Controller / Service · file:line | Template |
|---|---|---|---|---|---|
| 1 | Login OTP requested | Email | The user logging in | `login` · [controllers/users.js:302](wyton-api/controllers/users.js) → `sendOtpEmail` | "Your Login OTP - Wyton Developers" (6-digit, 10-min TTL) |
| 2 | OTP resent | Email | The user | `resendOtp` · [controllers/users.js:413](wyton-api/controllers/users.js) → `sendOtpEmail` | same |
| 3 | Admin invites new user | Email | Invitee | `inviteUser` · [controllers/users.js:852](wyton-api/controllers/users.js) → `sendInvitationEmail` | "You're Invited to Join Wyton Developers" (3-day link) |
| 4 | Invitation resent | Email | Invitee | `resendInvitedUser` · [controllers/users.js:1256](wyton-api/controllers/users.js) → `sendInvitationEmail` | same, new token |
| 5 | Self-signup completed | Email | New user | `signupAppuser` · [controllers/users.js:2075](wyton-api/controllers/users.js) → `sendSignupApprovedEmail` | "Your Wyton Developers Signup Has Been Approved" |
| 6 | Admin approves pending signup | Email | Approved user | `approveSignup` · [controllers/users.js:2200](wyton-api/controllers/users.js) → `sendSignupApprovedEmail` | same |
| 7 | Test FCM (dev) | Push | Provided token | `testFcmNotification` · [controllers/users.js:1861](wyton-api/controllers/users.js) — route `POST /user/test-fcm-notification` | — |

`sendSignupRequestEmail` in emailService.js:686 has no active caller (dead code).

Web callers: [src/pages/auth/Login.tsx](wyton-web/src/pages/auth/Login.tsx), [VerifyOTP.tsx](wyton-web/src/pages/auth/VerifyOTP.tsx), [ForgotPassword.tsx](wyton-web/src/pages/auth/ForgotPassword.tsx), [ResetPassword.tsx](wyton-web/src/pages/auth/ResetPassword.tsx), [InviteRegister.tsx](wyton-web/src/pages/auth/InviteRegister.tsx), [admin/UserManagement.tsx](wyton-web/src/pages/admin/UserManagement.tsx), [admin/components/InviteUserModal.tsx](wyton-web/src/pages/admin/components/InviteUserModal.tsx), [ApproveSignupModal.tsx](wyton-web/src/pages/admin/components/ApproveSignupModal.tsx).

---

## 2. Chat (Wyton chat platform)

Shared dispatcher `dispatchChatMessagePush` in [controllers/users.js:2470](wyton-api/controllers/users.js) sends **FCM push + in-app** to every channel member except the sender.

| # | Trigger | Channel | Recipient | Controller / Route |
|---|---|---|---|---|
| 8 | Manual dispatch from web after send | Push + In-app | Recipient user IDs from client | `notifyChatMessagePush` · [controllers/users.js](wyton-api/controllers/users.js); route `POST /user/chat/message-push` |

Module: `CHAT_MESSAGE`. Android payload is data-only (for Notifee), iOS uses full APNs alert. Deep-links to `/chat-room?channelUrl=…`.

Web callers: [src/services/chat.ts](wyton-web/src/services/chat.ts) `notifyChatMessagePushRecipients`, [ChatRoom.tsx](wyton-web/src/pages/ChatRoom/ChatRoom.tsx).

---

## 3. Task Room

Shared dispatcher `sendTaskNotifications(users, options)` in [services/taskServices.js:2861](wyton-api/services/taskServices.js) always sends **Email + In-app**, and **Push** when `sendPushNotification=true`.

| # | Trigger | Recipient | Controller · file:line | Module / Type |
|---|---|---|---|---|
| 10 | Task created — responsible person | Responsible person | `create_task` · [controllers/tasks.js:536](wyton-api/controllers/tasks.js); `create_time_task` · [1302](wyton-api/controllers/tasks.js) | `RESPONSIBLE_PERSON_FOR_TASK` |
| 11 | Task created — ball-in-court | BIC user(s) | [controllers/tasks.js:559, 1324](wyton-api/controllers/tasks.js) | `BALL_IN_COURT_PERSON_FOR_TASK` |
| 12 | Task created — member | Members | [controllers/tasks.js:582, 1346](wyton-api/controllers/tasks.js) | `MEMBER_FOR_TASK` |
| 13 | Task updated — new responsible person | New RP | [controllers/tasks.js:2183](wyton-api/controllers/tasks.js) | `RESPONSIBLE_PERSON_FOR_TASK` |
| 14 | Task updated — newly added BIC | New BIC | [controllers/tasks.js:2227](wyton-api/controllers/tasks.js) | `BALL_IN_COURT_PERSON_FOR_TASK` |
| 15 | Task updated — newly added members | New members | [controllers/tasks.js:2264](wyton-api/controllers/tasks.js) | `MEMBER_FOR_TASK` |
| 16 | Add single member | New member | [controllers/tasks.js:2502](wyton-api/controllers/tasks.js) | `MEMBER_FOR_TASK` |
| 17 | Bulk task update — new RP | New RP per task | [controllers/tasks.js:821](wyton-api/controllers/tasks.js) (throttled 10/750ms) | `RESPONSIBLE_PERSON_FOR_TASK` |
| 18 | Bulk task update — new members | New members per task | [controllers/tasks.js:869](wyton-api/controllers/tasks.js) | `MEMBER_FOR_TASK` |
| 19 | Task comment added | BIC + RP + all members + mentioned users (minus commenter) | [controllers/tasks.js:2802](wyton-api/controllers/tasks.js) | `TASK_COMMENT` — **Email + In-app + Push**. Reply-To routes replies via Lambda |
| 20 | Task reply via email ingress | Same set as comment | `add_task_comment_reply_from_email` · [controllers/tasks.js:1701](wyton-api/controllers/tasks.js) | `TASK_COMMENT` |
| 21 | **Cron** — task due today/tomorrow | RP, BIC, members | `run_task_due_reminder_cron` · [services/taskServices.js:3948](wyton-api/services/taskServices.js); scheduled by [jobs/recurringTaskCron.js](wyton-api/jobs/recurringTaskCron.js) 09:00 daily | Email only, type "Task Due Reminder" |

Web callers: [TaskModal.tsx](wyton-web/src/pages/TaskManagement/TaskModal.tsx), [UpdateTaskModal.tsx](wyton-web/src/pages/TaskManagement/UpdateTaskModal.tsx), [DependedTaskModal.tsx](wyton-web/src/pages/TaskManagement/DependedTaskModal.tsx).

---

## 4. Action Plan

| # | Trigger | Channel | Recipient | Controller · file:line | Module |
|---|---|---|---|---|---|
| 22 | Update project action group — RP changed | Email + In-app | New RP | `updateProjectGroupStatus` · [controllers/actionPlan.js:517 (email) / 553 (in-app)](wyton-api/controllers/actionPlan.js) | `RESPONSIBLE_PERSON_FOR_ACTION_PLAN` |
| 23 | Update project action group — BIC changed | Email + In-app | New BIC | [controllers/actionPlan.js:569 / 606](wyton-api/controllers/actionPlan.js) | `BALL_IN_COURT_PERSON_FOR_ACTION_PLAN` |
| 24 | Update project action group — tracked field change | Email + In-app | Watchers on `notificationWatchers`; primary skipped | `emitTrackedActionPlanItemUpdateNotification` · [controllers/actionPlan.js:51, 131 (in-app), 182 (email)](wyton-api/controllers/actionPlan.js) | `TRACKED_ACTION_PLAN_ITEM_UPDATED` |
| 25 | Bulk update — RP added | Email + In-app | New RP(s) | `bulkUpdateProjectGroupStatus` · [controllers/actionPlan.js:1684 / 1717](wyton-api/controllers/actionPlan.js) | `RESPONSIBLE_PERSON_FOR_ACTION_PLAN` |
| 26 | Bulk update — BIC added | Email + In-app | New BIC | [controllers/actionPlan.js:1733 / 1766](wyton-api/controllers/actionPlan.js) | `BALL_IN_COURT_PERSON_FOR_ACTION_PLAN` |
| 27 | Bulk update — tracked change | Email + In-app | Watchers | [controllers/actionPlan.js:1632](wyton-api/controllers/actionPlan.js) | `TRACKED_ACTION_PLAN_ITEM_UPDATED` |

Web callers: [ProjectActionPlan.tsx](wyton-web/src/pages/ProjectActionPlan.tsx), [CoTool/ProjectCoToolPage.tsx](wyton-web/src/pages/CoTool/ProjectCoToolPage.tsx).

---

## 5. Bidding — Trades

Notifications in [services/activityLogService.js](wyton-api/services/activityLogService.js), invoked from trade updates in [services/projectTradeServices.js](wyton-api/services/projectTradeServices.js), [controllers/projects.js](wyton-api/controllers/projects.js), and [controllers/tasks.js](wyton-api/controllers/tasks.js).

| # | Trigger | Channel | Recipient | Service · file:line | Module |
|---|---|---|---|---|---|
| 28 | RP set/changed on trade | Email + In-app | New RP | `updateResponsiblePerson` case · [services/activityLogService.js:906 (email) / 942 (in-app)](wyton-api/services/activityLogService.js) | `RESPONSIBLE_PERSON_FOR_BIDDING` |
| 29 | BIC set/changed on trade | Email + In-app | New BIC | `updateBallInCourtPerson` case · [services/activityLogService.js:1008 / 1044](wyton-api/services/activityLogService.js) | `BALL_IN_COURT_PERSON_FOR_BIDDING` |
| 30 | Tracked bidding item field update | In-app | Watchers (via `notificationWatchers`); primary skipped | `emitTrackedBiddingItemUpdateNotification` · [controllers/projects.js:96, 119](wyton-api/controllers/projects.js) | `TRACKED_BIDDING_ITEM_UPDATED` |
| 31 | Vendor RFP invitation (create/resend) | Email + PDF attachment | Vendor primary + contact-person emails | `sendRfpInvitation` · [services/projectServices.js:1136](wyton-api/services/projectServices.js); bulk `sendRfpInvitations` · [758](wyton-api/services/projectServices.js) → `sendRfpInvitationEmail` · [services/emailService.js:569](wyton-api/services/emailService.js) | Template `RFP_INVITATION` (per-project override) |
| 32 | **Cron** — project trade due today/tomorrow | Email | RP + BIC on the trade | `run_project_trade_due_reminder_cron` · [services/projectTradeServices.js:190](wyton-api/services/projectTradeServices.js) (routed through combined-due cron) | "Due Today/Tomorrow | Bidding" |

Web callers: [BiddingDetail.tsx](wyton-web/src/pages/BiddingDetail.tsx), [ProjectDetailBidding.tsx](wyton-web/src/pages/ProjectDetailBidding.tsx), [BidEmailDrawer.tsx](wyton-web/src/components/common/BidEmailDrawer.tsx), [AwardBidModal.tsx](wyton-web/src/components/common/AwardBidModal.tsx), [InviteRegistration.tsx](wyton-web/src/pages/InviteRegistration.tsx).

---

## 6. Bidding — Consultants (Project Contractors)

| # | Trigger | Channel | Recipient | Controller · file:line |
|---|---|---|---|---|
| 33 | Add project contractor bid — RFP invite | Email | Consultant primary + contact emails | `addProjectContractorBid` · [controllers/projectContractors.js:198](wyton-api/controllers/projectContractors.js) → `sendEmail` (Gmail API); 7-day token, custom HTML |
| 34 | Resend consultant invitation | (token rotation only, no email) | — | `resendProjectContractorInvitation` · [controllers/projectContractors.js:264](wyton-api/controllers/projectContractors.js) |

RP-changed / BIC-changed on consultants reuse the same handlers as trades (case #28–29) with `projectContractorId` instead of `projectTradeId`.

---

## 7. Contract Management (Vendor / Wyton)

All in [services/contractManagementService.js](wyton-api/services/contractManagementService.js). Vendor-flow updates deliberately produce **only in-app** unless noted.

| # | Trigger | Channel | Recipient | file:line | Module |
|---|---|---|---|---|---|
| 35 | `send_email_to_user` — vendor already registered | Email | Existing vendor user | `sendNotificationEmailToUser` · line 524 | — |
| 36 | `send_email_to_user` — invite already pending | Email | Same target | line 541 | — |
| 37 | `send_email_to_user` — brand-new user | Email | New invitee | line 574 `sendInvitationEmail` (3-day link); marks contract `INVITE_SENT` | — |
| 38 | Contract sent for signature | In-app | Vendor **and** all admin/employee users | line 1869 | `CONTRACT_SENT_FOR_SIGNATURE` |
| 39 | Vendor submits insurance module | In-app | All admin/employee users | line 2576 | `INSURANCE_SENT_FOR_REVIEW` |
| 40 | Vendor requests insurance waiver | In-app | All admin/employee users | line 2683 | `INSURANCE_SENT_FOR_REVIEW` |
| 41 | Vendor marks contract signed | In-app | All admin/employee users | line 3274 | `CONTRACT_SIGNED` |
| 42 | Phase status → REQUESTED (vendor → approval) | In-app | Admin + employee (except actor) | line 4082 `update_phase_status` | `PHASE_SENT_FOR_APPROVAL` |
| 43 | Phase status → APPROVED | In-app | Vendor of the contract | line 4106 | `PHASE_APPROVED` |
| 44 | Phase status → EDITED_BY_VENDOR | In-app | Admin + employee | line 4148 | `PHASE_EDITED_BY_VENDOR` |
| 45 | Phase status → VENDOR_APPROVED | In-app | Admin + employee | line 4191 | `PHASE_VENDOR_APPROVED` |
| 46 | **Cron** — insurance expiring in 15 days | In-app + Email | Trade RP(s) + vendor user on contract | `run_insurance_policy_exp_reminder_cron` · line 4744 (in-app), 4754 (email); scheduled 09:30 daily via [jobs/recurringTaskCron.js:90](wyton-api/jobs/recurringTaskCron.js) | `INSURANCE_POLICY_EXP_FIFTEEN_DAY_REMINDER` |

Web callers: [ContractManagement.tsx](wyton-web/src/pages/ContractManagement.tsx), [CreatePaymentTerms.tsx](wyton-web/src/pages/CreatePaymentTerms.tsx), [SignContractSection.tsx](wyton-web/src/components/SignContractSection.tsx), [InsuranceUpload.tsx](wyton-web/src/components/InsuranceUpload.tsx).

---

## 8. Phase Invoices

| # | Trigger | Channel | Recipient | Controller · file:line | Module |
|---|---|---|---|---|---|
| 47 | Vendor raises phase invoice | In-app + Push | All BIC + RP on the project trade (except creator) | `raiseInvoice` · [controllers/phaseInvoice.js:151 (in-app), 169 (FCM)](wyton-api/controllers/phaseInvoice.js) | `INVOICE_RAISED` — deep-link `/commitments/{cmId}/phases` |

Web callers: [VendorCommitments.tsx](wyton-web/src/pages/VendorCommitments.tsx), [VendorCommitmentPhases.tsx](wyton-web/src/pages/VendorCommitmentPhases.tsx), [CreateInvoiceVendor.tsx](wyton-web/src/pages/CreateInvoiceVendor.tsx).

QuickBooks event publisher fires on invoice approve/reject/paid (~line 618+) — this is NOT a user-facing notification, only a webhook.

---

## 9. Change Events

All in [services/changeEventServices.js](wyton-api/services/changeEventServices.js) — in-app only.

| # | Trigger | Recipient | file:line | Module |
|---|---|---|---|---|
| 48 | Vendor creates change event | All admin/employee users | `create_change_event` · line 402 | `CHANGE_EVENT_REQUESTED` |
| 49 | Admin/employee creates change event | All vendors on the project (via ContractManagement.userId) | line 418 | `CHANGE_EVENT_APPROVED` |
| 50 | Vendor re-submits rejected change event | All admin/employee users | `update_change_event` · line 602 | `CHANGE_EVENT_REQUESTED` |
| 51 | Change event → APPROVED | Creator (if not the actor) | `update_change_event_status` · line 677 | `CHANGE_EVENT_APPROVED` |
| 52 | Change event → REJECTED | Creator (if not the actor) | line 677 | `CHANGE_EVENT_REJECTED` |

Web callers: [ChangeEvent.tsx](wyton-web/src/pages/ChangeEvent.tsx), [ChangeEventCreate.tsx](wyton-web/src/pages/ChangeEventCreate.tsx).

---

## 10. Correspondence Tracking (Permit / Violation / RFI)

Shared helper `emitTrackedCorrespondenceItemUpdateNotification` · [services/correspondenceTrackingRowActionServices.js:109](wyton-api/services/correspondenceTrackingRowActionServices.js). Fires on priority toggle, due-date nudge, follow-up, or any tracked-field update.

| # | Trigger | Channel | Recipient | file:line | Module |
|---|---|---|---|---|---|
| 53 | Permit item updated / priority / nudge / follow-up | In-app + Email | Watchers on `notificationWatchers`; primary skipped | line 161 (in-app), 219 (email) | `TRACKED_PERMIT_ITEM_UPDATED` |
| 54 | Violation item updated / priority / nudge / follow-up | In-app + Email | Watchers | same helper | `TRACKED_VIOLATION_ITEM_UPDATED` |
| 55 | RFI item updated / priority / nudge / follow-up | In-app + Email | Watchers | same helper | `TRACKED_RFI_ITEM_UPDATED` |

Web callers: [TrackingRowActionsMenu.tsx](wyton-web/src/components/correspondence/TrackingRowActionsMenu.tsx), [PermitTracking.tsx](wyton-web/src/pages/PermitTracking.tsx), [ViolationTracking.tsx](wyton-web/src/pages/ViolationTracking.tsx), [CorrespondencePermitCreate.tsx](wyton-web/src/pages/CorrespondencePermitCreate.tsx), [CorrespondenceViolationCreate.tsx](wyton-web/src/pages/CorrespondenceViolationCreate.tsx).

---

## 11. Scheduling

| # | Trigger | Channel | Recipient | Service · file:line | Module |
|---|---|---|---|---|---|
| 56 | Schedule activity created — new assignee | In-app + Email | Assignee (excluding actor) | `notifySchedulingAssignments` · [services/schedulingServices.js:246 (in-app), 270 (email)](wyton-api/services/schedulingServices.js); called from `create_task` line 711 | `SCHEDULING_ASSIGNEE` |
| 57 | Schedule activity created — new approver | In-app + Email | Approver (excluding actor) | same helper, `role="approver"` | `SCHEDULING_APPROVER` |
| 58 | Schedule activity updated — newly added assignees/approvers | In-app + Email | New assignees/approvers | `update_task` · [services/schedulingServices.js:816](wyton-api/services/schedulingServices.js) | same modules |

Web callers: [src/pages/Scheduling.tsx](wyton-web/src/pages/Scheduling.tsx), `src/components/scheduling/*`.

---

## 12. Documents (Google Drive tracking)

All in [services/projectDocumentService.js](wyton-api/services/projectDocumentService.js). Shared `notifyTrackedUsers` (line 55) sends **In-App + Push + Email** to all tracked users. `notifyTrackingUsersByDriveItem` (line 153) climbs the parent-folder chain.

| # | Trigger | Recipient | file:line | Module |
|---|---|---|---|---|
| 59 | Folder created inside tracked ancestor | Trackers | `createFolder` · line 1495 | `DOCUMENT_TRACKING_UPDATED` ("Tracked Item Created") |
| 60 | Drive item moved into/out of tracked ancestor | Trackers | `moveDriveItem` · line 1281 | `DOCUMENT_TRACKING_UPDATED` |
| 61 | Drive item deleted (Trash) | Trackers | `deleteDriveItem` · line 708 | `DOCUMENT_TRACKING_UPDATED` |
| 62 | Drive item copied | Trackers | `copyDriveItem` · line 1425 | `DOCUMENT_TRACKING_UPDATED` ("Tracked Item Copied") |
| 63 | File uploaded into tracked folder | Trackers | `uploadFile` · line 2381 | `DOCUMENT_TRACKING_UPDATED` |
| 64 | File new version uploaded | Trackers | `uploadFileVersion` · line 2233 | `DOCUMENT_TRACKING_UPDATED` ("Tracked Item Edited") |
| 65 | User added as tracker | Newly-added trackers only (diff) | `addDocumentTracking` · line 2747 | `DOCUMENT_TRACKING_ADDED` |

All three channels for every event above.

---

## 13. Meetings

| # | Trigger | Channel | Recipient | Service · file:line |
|---|---|---|---|---|
| 66 | Distribute meeting agenda | Email (+ attachments) | Meeting attendees (or admin-selected subset) | `distribute_agenda` · [services/meetingServices.js:1559](wyton-api/services/meetingServices.js) → `sendEmail`; subject `Meeting Agenda: {name} (#{n})` |
| 67 | Export meeting to ZIP | Email (+ ZIP) | Requesting user only | `buildZipAndEmail` · [services/meetingServices.js:441](wyton-api/services/meetingServices.js) (called from `export_meeting` line 1213 when `exportType != "pdf"`); subject `Meeting #{n} export is ready` |

Web callers: [MeetingForm.tsx](wyton-web/src/pages/MeetingForm.tsx) (Distribute Agenda + Share Google Meet).

No push or in-app for meetings.

---

## 14. Read-only surfaces (no notifications sent)

- **Overview / Portfolio badges** — `services/userPortfolioSummaryServices.js` reads `in_app_notification` counts only.
- **Notification API** — `get_user_notifications`, `count_user_notifications`, `mark_all_notifications_as_read`, `mark_notification_as_read` in [services/userServices.js](wyton-api/services/userServices.js). Routes `GET/PATCH /user/notifications*`, `GET /user/unread-notifications-count`.

---

## 15. Notification Infrastructure

### 15.1 Backend (wyton-api)
- **Email sender** — [services/emailService.js](wyton-api/services/emailService.js). Gmail API with admin OAuth refresh token. Exports: `sendEmail`, `sendInvitationEmail`, `sendRfpInvitationEmail`, `sendOtpEmail`, `sendSignupRequestEmail` (dead), `sendSignupApprovedEmail`, `sendNotificationEmail` (generic templated — used by nearly every in-product event), `sendCombinedDueReminderEmail`.
- **Push sender** — [utils/fcmPushNotification.js](wyton-api/utils/fcmPushNotification.js). Service account at `config/wyton-notification-firebase-adminsdk.json`. Exports `sendNotificationToUser` (unused), `sendNotificationToMultipleUsers` (primary fanout), `sendTestNotification`. Auto-purges invalid tokens.
- **FCM token storage** — [model/fmc-token-model.js](wyton-api/model/fmc-token-model.js), collection `fmc_token`. Written by `userServices.manage_fmc_token` (called from `verifyOtp` and `setupFcmToken`).
- **In-app writer** — [services/userServices.js:591 `create_in_app_notification`](wyton-api/services/userServices.js) + `fanOutBiddingNotificationWatchers` (line 21) to fan out to every user in a related item's `notificationWatchers` array.
- **In-app model** — [model/in-app-notification-model.js](wyton-api/model/in-app-notification-model.js).
- **Module catalogue** — [lib/constant.js:586 `notification_module`](wyton-api/lib/constant.js) (all 30+ module identifiers).
- **Per-project email templates** — [model/emailTemplate.js](wyton-api/model/emailTemplate.js) + [services/emailTemplateService.js](wyton-api/services/emailTemplateService.js) (`RFP_INVITATION`).
- **Combined due-reminder digest** — [services/combinedDueReminderService.js](wyton-api/services/combinedDueReminderService.js).
- **Cron scheduler** — [jobs/recurringTaskCron.js](wyton-api/jobs/recurringTaskCron.js): recurring task gen (09:00), combined due reminder emails (09:30), insurance-exp reminder (09:30).
- **Frontend link builders** — [lib/frontendLinks.js](wyton-api/lib/frontendLinks.js).
- **Dead code** — [services/emailService copy.js](wyton-api/services/emailService%20copy.js) (legacy Nodemailer), [utils/sendEmail.js](wyton-api/utils/sendEmail.js) (SES "TennisPAL" leftover), [utils/sms.js](wyton-api/utils/sms.js) (Twilio, unused).

### 15.2 Frontend (wyton-web)
- **FCM web push init** — [src/config/firebase.ts](wyton-web/src/config/firebase.ts), [src/hooks/useFCM.ts](wyton-web/src/hooks/useFCM.ts), [src/scripts/inject-firebase-config.js](wyton-web/src/scripts/inject-firebase-config.js).
- **Service worker** — [public/firebase-messaging-sw.js](wyton-web/public/firebase-messaging-sw.js).
- **PWA manifest** — [public/manifest.json](wyton-web/public/manifest.json).
- **Deep-link resolver** — [src/lib/notificationNavigation.ts](wyton-web/src/lib/notificationNavigation.ts).
- **In-app bell state** — [src/store/slices/notificationSlice.ts](wyton-web/src/store/slices/notificationSlice.ts). Endpoints: `GET /user/notifications`, `GET /user/unread-notifications-count`, `POST /user/notifications/:id/mark-read`, `POST /user/notifications/mark-all-read`, `DELETE /user/notifications/:id`.
- **Bell popup** — [src/components/common/notifications/NotificationPopup.tsx](wyton-web/src/components/common/notifications/NotificationPopup.tsx), card [NotificationCard.tsx](wyton-web/src/components/common/notifications/NotificationCard.tsx).
- **Full page** — [src/pages/Notifications.tsx](wyton-web/src/pages/Notifications.tsx) (route `/notifications`).
- **Module → icon** — [src/constants/notificationIcons.ts](wyton-web/src/constants/notificationIcons.ts), constants [notificationModules.ts](wyton-web/src/constants/notificationModules.ts).
- **Bell mount + 60s polling** — [NewNavbar.tsx](wyton-web/src/components/layout/NewNavbar.tsx) (employees), [VendorNavbar.tsx](wyton-web/src/components/layout/VendorNavbar.tsx) (vendors), legacy [Navbar.tsx](wyton-web/src/components/layout/Navbar.tsx) still present.
- **Custom UI toast system** — [src/store/slices/uiSlice.ts](wyton-web/src/store/slices/uiSlice.ts), [src/components/common/NotificationSystem.tsx](wyton-web/src/components/common/NotificationSystem.tsx), mounted at [src/App.tsx:111](wyton-web/src/App.tsx). Auto-fires from `handleApiError`/`handleApiSuccess` in [src/services/api.ts](wyton-web/src/services/api.ts).
- **Chat push bridge** — [src/services/chat.ts](wyton-web/src/services/chat.ts) `notifyChatMessagePushRecipients` → `POST /user/chat/message-push`.

---

## 16. Latent findings (worth fixing separately)

1. **react-toastify has no root `<ToastContainer/>`.** The only mount is at [MeetingForm.tsx:1289](wyton-web/src/pages/MeetingForm.tsx). All `toast.*` calls in ChatRoom (30+), ChatInput, ChatAddItemPopover, CameraCaptureOverlay, useDuplicateAgendaModal silently no-op unless the meeting form happens to be on-screen. Users lose chat feedback.
2. **SW notification icon `/src/assets/svgs/mLogo.svg`** ([firebase-messaging-sw.js:45-46](wyton-web/public/firebase-messaging-sw.js), [useFCM.ts:63](wyton-web/src/hooks/useFCM.ts)) is not a `public/` path — it 404s in production.
3. **Legacy [Navbar.tsx](wyton-web/src/components/layout/Navbar.tsx) still ships** with duplicate FCM init, bell, and polling — confirm it is dead-code-eliminated by App routing.
4. **In-app notification cache in localStorage** ([notificationSlice.ts:30-52](wyton-web/src/store/slices/notificationSlice.ts)) persists per browser — could leak notifications between accounts on shared devices.
5. **`sendSignupRequestEmail`** in [emailService.js:686](wyton-api/services/emailService.js), **`utils/sendEmail.js`** (SES/TennisPAL), and **`utils/sms.js`** (Twilio) are dead code — safe to remove.
