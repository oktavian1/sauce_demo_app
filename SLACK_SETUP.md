# Slack Notifications Setup Guide

Professional Slack integration for E2E test results with industry best practices.

## 🚀 Quick Setup

### **Step 1: Create Slack Incoming Webhook**

1. **Go to Slack App Directory:**
   - Visit: https://api.slack.com/apps
   - Click **"Create New App"** → **"From scratch"**

2. **Name your app:**
   - App Name: `E2E Test Notifications`
   - Workspace: Select your workspace
   - Click **"Create App"**

3. **Enable Incoming Webhooks:**
   - In left sidebar, click **"Incoming Webhooks"**
   - Toggle **"Activate Incoming Webhooks"** to **ON**
   - Click **"Add New Webhook to Workspace"**
   - Select channel (e.g., `#qa-automation`, `#test-results`)
   - Click **"Allow"**

4. **Copy Webhook URL:**
   - You'll see a webhook URL like:
   ```
   https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX
   ```
   - **Copy this URL** (keep it secret!)

---

### **Step 2: Add GitHub Secret**

1. **Go to Repository Settings:**
   - Open: https://github.com/oktavian1/sauce_demo_app/settings/secrets/actions

2. **Add New Secret:**
   - Click **"New repository secret"**
   - Name: `SLACK_WEBHOOK_URL`
   - Value: Paste your webhook URL from Step 1
   - Click **"Add secret"**

---

### **Step 3: Test Notification**

Push any commit to trigger the workflow:

```bash
git commit --allow-empty -m "test: trigger Slack notification"
git push
```

Check your Slack channel for the notification! 🎉

---

## 📊 What You'll Get

### **Rich Formatted Notifications:**

```
✅ E2E Tests - Passed

Repository: oktavian1/sauce_demo_app
Branch: main
Commit: feat: add Allure reporting
Author: Ilham Oktavian

Deploy: success
Tests: success

📊 View Allure Report | 🔗 View Workflow Run

Triggered by push | Run #42
```

### **Color-Coded Status:**
- 🟢 **Green** - All tests passed
- 🔴 **Red** - Tests failed
- ⚠️ **Yellow** - Cancelled/Skipped

---

## 🎯 Industry Best Practices Applied

### **1. Smart Notifications (No Spam)**

```yaml
if: always() && github.event_name != 'pull_request'
```

**What it does:**
- ✅ Notifies on **push to main** (deployments)
- ❌ Does NOT notify on **pull requests** (too noisy)
- ✅ Notifies even if tests fail (`always()`)

**Why:**
- Avoids notification fatigue
- Only alerts on production deployments
- Team stays focused on what matters

---

### **2. State-Based Logic**

```bash
if tests failed OR deploy failed → 🔴 Red alert
elif tests passed AND deploy passed → ✅ Green success
else → ⚠️ Cancelled/Warning
```

**Benefits:**
- Immediate visual status
- Easy to scan in busy channels
- Actionable at a glance

---

### **3. Rich Context (One-Click Access)**

Every notification includes:
- 🔗 **Direct link to Allure report**
- 🔗 **Direct link to workflow run**
- 🔗 **Commit message & author**
- 📊 **Status of each job** (deploy/test)

**Why:**
- No hunting for information
- Faster debugging
- Complete audit trail

---

### **4. Official Action (Most Used)**

Using `slackapi/slack-github-action@v1.27.0`:
- ✅ Official Slack-maintained action
- ✅ 5,000+ stars on GitHub
- ✅ Active maintenance
- ✅ Full Block Kit support

**Alternatives NOT used:**
- ❌ `8398a7/action-slack` (deprecated)
- ❌ Custom curl commands (brittle)

---

## 🔧 Advanced Configuration

### **Notify Only on Failures**

```yaml
- name: Send Slack notification
  if: env.SLACK_WEBHOOK_URL != '' && failure()
  uses: slackapi/slack-github-action@v1.27.0
```

---

### **Notify Only on State Change (Pass → Fail)**

```yaml
- name: Check previous run status
  id: prev
  run: |
    PREV_STATUS=$(gh run list --workflow=deploy-and-test.yml --limit 2 --json conclusion --jq '.[1].conclusion')
    CURR_STATUS="${{ needs.test.result }}"
    
    if [ "$PREV_STATUS" == "success" ] && [ "$CURR_STATUS" == "failure" ]; then
      echo "notify=true" >> $GITHUB_OUTPUT
    elif [ "$PREV_STATUS" == "failure" ] && [ "$CURR_STATUS" == "success" ]; then
      echo "notify=true" >> $GITHUB_OUTPUT
    else
      echo "notify=false" >> $GITHUB_OUTPUT
    fi

- name: Send Slack notification
  if: steps.prev.outputs.notify == 'true'
  uses: slackapi/slack-github-action@v1.27.0
```

---

### **Mention Team on Failure**

```yaml
payload: |
  {
    "text": "${{ steps.status.outputs.status == 'failure' && '<!channel> Tests Failed!' || 'Tests Passed' }}",
    ...
  }
```

**Mention options:**
- `<!channel>` - Notify everyone in channel
- `<!here>` - Notify only active users
- `<@U1234567>` - Mention specific user
- `<!subteam^S1234567>` - Mention user group

---

### **Thread Updates**

For long-running tests, send initial notification then update in thread:

```yaml
# Step 1: Send initial message
- name: Post initial notification
  id: slack
  uses: slackapi/slack-github-action@v1.27.0
  with:
    payload: '{"text": "⏳ Tests running..."}'

# Step 2: Update in thread
- name: Update with results
  uses: slackapi/slack-github-action@v1.27.0
  with:
    update-ts: ${{ steps.slack.outputs.ts }}
    payload: '{"text": "✅ Tests passed!"}'
```

---

### **Schedule-Based Notifications**

Only notify during business hours:

```yaml
- name: Check time
  id: time
  run: |
    HOUR=$(TZ='Asia/Jakarta' date +%H)
    if [ $HOUR -ge 9 ] && [ $HOUR -le 18 ]; then
      echo "notify=true" >> $GITHUB_OUTPUT
    else
      echo "notify=false" >> $GITHUB_OUTPUT
    fi

- name: Send notification
  if: steps.time.outputs.notify == 'true'
  uses: slackapi/slack-github-action@v1.27.0
```

---

## 🎨 Message Customization

### **Add Test Summary Stats**

Enhance message with test counts:

```yaml
- name: Parse test results
  id: tests
  run: |
    PASSED=$(cat automation/allure-results/*.json | jq '[.[] | select(.status=="passed")] | length')
    FAILED=$(cat automation/allure-results/*.json | jq '[.[] | select(.status=="failed")] | length')
    echo "passed=$PASSED" >> $GITHUB_OUTPUT
    echo "failed=$FAILED" >> $GITHUB_OUTPUT

- name: Send notification
  uses: slackapi/slack-github-action@v1.27.0
  with:
    payload: |
      {
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "✅ *${{ steps.tests.outputs.passed }}* passed | 🔴 *${{ steps.tests.outputs.failed }}* failed"
            }
          }
        ]
      }
```

---

### **Add Duration**

```yaml
- name: Calculate duration
  id: duration
  run: |
    START=${{ github.event.workflow_run.created_at }}
    END=$(date +%s)
    DURATION=$((END - $(date -d "$START" +%s)))
    MINUTES=$((DURATION / 60))
    echo "minutes=$MINUTES" >> $GITHUB_OUTPUT

- name: Send notification
  with:
    payload: |
      {
        "text": "Tests completed in ${{ steps.duration.outputs.minutes }} minutes"
      }
```

---

### **Add Screenshots on Failure**

Upload failure screenshot to Slack:

```yaml
- name: Upload failure screenshot
  if: failure()
  uses: MeilCli/slack-upload-file@v3
  with:
    slack_token: ${{ secrets.SLACK_BOT_TOKEN }}
    channel_id: ${{ secrets.SLACK_CHANNEL_ID }}
    file_path: 'automation/test-results/**/trace.png'
    initial_comment: 'Test failure screenshot'
```

---

## 🚦 Multi-Environment Notifications

Different channels for different environments:

```yaml
env:
  SLACK_WEBHOOK_URL: ${{ 
    github.ref == 'refs/heads/main' && secrets.SLACK_WEBHOOK_PROD ||
    github.ref == 'refs/heads/staging' && secrets.SLACK_WEBHOOK_STAGING ||
    secrets.SLACK_WEBHOOK_DEV
  }}
```

Setup secrets:
- `SLACK_WEBHOOK_PROD` → `#prod-alerts`
- `SLACK_WEBHOOK_STAGING` → `#staging-tests`
- `SLACK_WEBHOOK_DEV` → `#dev-tests`

---

## 📈 Metrics & Analytics

Track notification effectiveness:

```yaml
- name: Log notification
  run: |
    echo "Notification sent at $(date)" >> notifications.log
    gh api repos/${{ github.repository }}/dispatches \
      -f event_type=notification_sent \
      -F client_payload[status]=${{ steps.status.outputs.status }}
```

---

## 🐛 Troubleshooting

### **No notifications received?**

1. **Check secret exists:**
   ```bash
   gh secret list
   # Should show SLACK_WEBHOOK_URL
   ```

2. **Verify webhook URL:**
   - Go to Slack App settings
   - Check webhook is active
   - Test with curl:
   ```bash
   curl -X POST -H 'Content-type: application/json' \
     --data '{"text":"Test message"}' \
     YOUR_WEBHOOK_URL
   ```

3. **Check workflow conditions:**
   - Is event type `pull_request`? (Won't notify)
   - Is `SLACK_WEBHOOK_URL` empty? (Won't notify)

---

### **Notification sent but not formatted?**

- Slack may have formatting disabled
- Check channel settings → Preferences → Display

---

### **Rate limited?**

Slack has limits:
- **1 message per second** per webhook
- Solution: Batch notifications or add delays

---

## 📚 Resources

- **Slack Block Kit Builder:** https://app.slack.com/block-kit-builder
- **Action Docs:** https://github.com/slackapi/slack-github-action
- **Slack API:** https://api.slack.com/messaging/webhooks

---

## ✅ Checklist

- [ ] Created Slack webhook
- [ ] Added `SLACK_WEBHOOK_URL` secret
- [ ] Pushed commit to trigger notification
- [ ] Verified notification received
- [ ] Customized message (optional)
- [ ] Set up alerts for failures (optional)

---

**Happy Testing! 🎉**
