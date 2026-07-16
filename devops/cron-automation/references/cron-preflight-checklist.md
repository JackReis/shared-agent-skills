# Cron Job Pre-Flight Checklist

## Session Example
**Context**: Email campaign cron job (`74bcc601b138`) scheduled but SMTP untested.

### Pre-Flight Steps
1. **Test SMTP**:
   ```bash
   swaks --to jack@vistadelmar.org --from oursearanchhome@gmail.com --server smtp.gmail.com:465 --auth LOGIN --auth-user oursearanchhome@gmail.com --auth-password "$(security find-generic-password -s "gmail_oursearanchhome" -w)" --ssl
   ```
   Error: `Keychain entry not found`.

2. **Verify Cron Job**:
   ```bash
   cronjob action=list job_id=74bcc601b138
   ```
   Output: Scheduled for 2026-06-22 at 9:00 AM PDT.

3. **Document Fallback**:
   - Draft manual email (`manual_email_draft.md`).
   - Update Keychain:
     ```bash
     security add-generic-password -s "gmail_oursearanchhome" -a "oursearanchhome@gmail.com" -w "YOUR_APP_PASSWORD"
     ```

### Lessons
- Always test SMTP before scheduling.
- Document manual fallback steps.