# Quick Installation Guide

## 🚀 5-Minute Setup

### Step 1: Extract and Upload to Git

```bash
# Extract the zip file
unzip axis-service-discovery-webhook.zip
cd axis-service-discovery-webhook

# Option A: New repository
git init
git add .
git commit -m "Initial commit: Service discovery with webhook integration"
git remote add origin <your-repo-url>
git push -u origin main

# Option B: Existing repository
cp -r * /path/to/your/ansible/repo/
cd /path/to/your/ansible/repo/
git add .
git commit -m "Updated: Webhook-only discovery (no file saving)"
git push
```

### Step 2: ⚠️ IMPORTANT - Replace Placeholder Playbooks

The following playbooks are SIMPLIFIED TEMPLATES and MUST be replaced with your actual full playbooks:

- `Service_discovery/Middleware/Apache/apache_discovery.yml`
- `Service_discovery/Middleware/Nginx/nginx_discovery.yml`
- `Service_discovery/Middleware/Tomcat/tomcat_discovery.yml`
- `Service_discovery/DB/Mongodb/mongodb_discovery.yml`

**What to do:**
1. Copy your actual full discovery playbooks
2. Remove ONLY the file-saving tasks (see below)
3. Keep all discovery logic intact
4. Ensure variable names match: `<service>_discovery_json`

**File-saving tasks to remove:**

```yaml
# Remove these 3 types of tasks from each playbook:

# 1. Remove directory creation
- name: Create output directory on localhost
  file:
    path: "{{ playbook_dir }}/xxx_discovery_output"
    state: directory

# 2. Remove individual file saving
- name: Save discovery results to JSON file
  copy:
    content: "{{ xxx_discovery_json | to_nice_json }}"
    dest: "{{ playbook_dir }}/xxx_discovery_output/..."

# 3. Remove consolidated report saving
- name: Save consolidated JSON report
  copy:
    content: "{{ final_json_output | to_nice_json }}"
    dest: "{{ playbook_dir }}/xxx_discovery_output/consolidated_discovery.json"
```

**Already complete (no changes needed):**
- ✅ `Service_discovery/DB/Mysql/mysql_discovery.yml` - Full implementation
- ✅ `Service_discovery/DB/Postgres/postgres_discovery.yml` - Full implementation

### Step 3: Sync Project in AWX

1. Log into AWX/Tower
2. Navigate to: **Projects** → **Your Project**
3. Click **Sync** button (circular arrow icon)
4. Wait for green status

### Step 4: Configure Webhook in AWX

**Option A: Environment Variables (Recommended)**

In Job Template → Environment Variables:
```bash
WEBHOOK_API_URL=https://your-api.example.com/webhook/discovery
WEBHOOK_TOKEN=your-secret-token-here
```

**Option B: Extra Variables**

In Job Template → Extra Variables:
```yaml
webhook_api_url: "https://your-api.example.com/webhook/discovery"
webhook_api_token: "your-secret-token"
```

### Step 5: Update Job Template

1. Edit your Service Discovery job template
2. **Playbook:** Change to `site_webhook_only.yml`
3. **Save**

### Step 6: Test Run

1. Launch the job template
2. Monitor the output
3. Check webhook receives data (use webhook.site for testing)

---

## 🧪 Quick Test with webhook.site

```bash
# 1. Go to https://webhook.site/ and copy your unique URL

# 2. In AWX Extra Variables:
webhook_api_url: "https://webhook.site/your-unique-id"

# 3. Run the job

# 4. View results in webhook.site
```

---

## 📋 Pre-Flight Checklist

Before running in production:

- [ ] Replaced placeholder playbooks with actual full playbooks
- [ ] Removed all file-saving tasks from playbooks
- [ ] Git repository synced in AWX
- [ ] Webhook URL configured correctly
- [ ] Webhook token set securely
- [ ] Tested webhook endpoint responds (200/201/202)
- [ ] Network connectivity verified (AWX → Webhook)
- [ ] Tested with single discovery service first
- [ ] Reviewed payload format matches your API

---

## 🔍 Verification

After first run, verify:

1. **AWX Job Output:**
   ```
   TASK [Send combined results to webhook API]
   ok: [localhost] => {
       "webhook_response": {
           "status": 200,
           "json": {...}
       }
   }
   ```

2. **No Local Files Created:**
   ```bash
   # SSH to AWX
   ls -la /var/lib/awx/projects/<project>/Service_discovery/
   # Should NOT see any *_output/ directories
   ```

3. **Webhook Received Data:**
   - Check your webhook API logs
   - Verify payload structure
   - Confirm all expected services present

---

## 🆘 Troubleshooting

### "No webhook URL configured"

**Fix:** Set `webhook_api_url` in AWX (see Step 4)

### "Unauthorized" (401)

**Fix:** Verify `webhook_api_token` matches your API

### "Connection timeout"

**Fix:** Check firewall rules, verify AWX can reach webhook URL

### "Variable not found"

**Fix:** Ensure your playbooks set `<service>_discovery_json` variables

---

## 📚 Documentation

- **README.md** - Complete overview and usage
- **WEBHOOK_SETUP.md** - Detailed webhook configuration
- **This file** - Quick installation guide

---

## 🎯 What This Package Provides

✅ **Ready-to-use playbooks** (MySQL, PostgreSQL) - No file saving  
✅ **Template playbooks** (Apache, Nginx, Tomcat, MongoDB) - Replace with your full versions  
✅ **Master orchestration** (site_webhook_only.yml) - Webhook integration  
✅ **Comprehensive documentation** - Setup and troubleshooting  
✅ **Example implementations** - Python Flask, Node.js Express  

---

## 💡 Key Changes from Previous Version

| Before | After |
|--------|-------|
| Saved JSON files to disk | Results in memory only |
| Manual file retrieval | Direct webhook delivery |
| No failure handling | Automatic failure notifications |
| File-based architecture | API-based architecture |
| AWX post-processing needed | Real-time integration |

---

## 📞 Need Help?

1. Review **WEBHOOK_SETUP.md** for detailed configuration
2. Check AWX job output for specific errors
3. Test webhook endpoint independently
4. Verify all playbook variables are set correctly
5. Check network connectivity between AWX and webhook

---

**Ready to go!** Follow the 6 steps above and you'll be up and running. 🚀

**Important:** Don't forget to replace the placeholder playbooks with your actual full versions!
