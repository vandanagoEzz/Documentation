# Access Checklist: New Hires and Departures

Last updated: 2026-10-25. Owner: Vandana Mishra (founder).

## Purpose and the one rule

Every person gets only the access their role needs, and loses all of it on their last day. Vandana (the founder) approves every request and owns this list. Use it for employees and contractors alike.

**The rule:** access follows the role. Give the minimum, review it when the role changes, and revoke it in the order below on departure.

## Access each role needs

Start from the row for the role, add nothing without Vandana's approval, and turn on two-step sign-in for every account.

| Role | Gets | Does not get |
| --- | --- | --- |
| Everyone | goEasy email (goeasy.kids), Teams, the shared folders for their team | Admin access to any system |
| Operations Manager | goEasy admin (Ops Manager role), driver pipeline tracker, Jira, phone and text lines for drivers and parents | Payment settings, AWS, database |
| Rides Support | goEasy admin (Rides Support role), the shared support number and mailbox | Driver payouts, billing |
| Accounts | goEasy admin (Accounts role), payments and invoicing tools | Driver onboarding, AWS, database |
| Marketing and social | Meta Business Suite (Facebook and Instagram), Google Analytics (view), Mailchimp, the Socials and marketing folders | Admin on the goEasy system, payments |
| Developer | Jira, Confluence, code repositories, AWS through Identity Center (developer permission set), test environments | Production database write access and the root account, unless Vandana approves |
| Founder (Super Admin) | Everything, including billing, root and the break-glass account | n/a |

Contractors get the same row as an employee in the same role, plus an end date set when the access is created.

## Onboarding steps

Do these in order, before the person's first day.

1. Write down the person's name, role, manager, start date, and for contractors the end date.
2. Vandana approves the role and the access row from the table above.
3. Create their goEasy email account and add them to Teams.
4. Turn on two-step sign-in for the email account before anything else.
5. Add them to each tool for their role, using their own named account. Do not share one login between people.
6. For developers: add them to the developer group in AWS Identity Center, not as a separate IAM user.
7. Share only the folders and files their role needs.
8. Give them the role guide or procedure, and have them sign in to each tool once to confirm it works.
9. Add a line to the access log: person, role, each system, date given, who approved.

## Offboarding steps, in order

Do steps 1 to 5 at the moment the person's access should end, the same hour. The order goes from the highest risk to the lowest, so the most damaging access closes first. Do the rest the same day.

1. **Cut the main sign-in.** Reset the goEasy email password, sign the account out of all devices, and remove their two-step sign-in. This stops most other sign-ins and password resets.
2. **Cloud and code.** Remove them from AWS Identity Center, delete any separate IAM user, and deactivate and then delete their access keys. Remove them from the code repositories and from Jira and Confluence admin rights. If they ever had the root account or database passwords, change those now.
3. **goEasy admin and customer data.** Disable their login to the goEasy admin and any database or reporting access.
4. **Money.** Remove them from payment, invoicing, bank and driver-payout tools.
5. **Public and marketing accounts.** Remove them from Meta Business Suite (Facebook and Instagram), Google Ads, Google Analytics, Mailchimp, the LinkedIn company page and any other social or ad account.
6. **Team tools.** Remove them from Teams, Zoom, Calendly, Otter and any other shared tool.
7. **Shared passwords and keys.** Change every shared password, API key or token they knew, such as the company social logins and service keys. Update the apps that use them.
8. **Files.** Move their files and folders to their manager, and remove them from shared folders in Dropbox and Google Drive.
9. **Equipment and phone.** Collect the laptop, phone, cameras and badges. Forward the phone line or number they used.
10. **Mailbox.** Set up forwarding to their manager and an auto-reply, then keep the mailbox for 90 days before deleting it.
11. **Log it.** Record the date and time each system was closed, and who did it.

## Records and follow-up checks

- **Access log.** Keep one list of who has access to what, with the date given, the approver, and for contractors the end date. Update it at every start, role change and departure.
- **Check after a departure.** A week later, look at sign-in activity for the person's accounts (AWS, email, admin) to confirm nothing was used after the last day.
- **Review every quarter.** Vandana compares the access log with who actually works at goEasy and removes anyone who should not be there.
- **Role changes.** When someone changes role, add the new access and remove the old access the same week.

## Systems and details to confirm

The system list was built from tools goEasy uses today. Please confirm or correct:

- **Missing or extra systems.** Payment and invoicing tools, the bank, the driver payout tool, the code repositories, and the Vercel, Canva and Peachjar accounts are not confirmed here.
- **Admin roles.** The goEasy admin roles (Super Admin, Manager, Ops Manager, Rides Support, Accounts) are being built under ticket UAD-1338. Until they exist, use the closest current permission and re-check afterward.
- **Password manager.** If goEasy uses one, add it to both lists so shared logins can be changed in one place.
- **Who approves and who acts.** The list says Vandana approves. Name who does the steps when she is away.
- **Mailbox retention.** 90 days is used here. Change it if you want a different period.
