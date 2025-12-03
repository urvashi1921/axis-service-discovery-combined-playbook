# Service Discovery - Webhook Integration (No File Saving)

## 📋 Overview

This package contains updated Ansible playbooks that:
- ✅ **NO file saving** - All results kept in memory
- ✅ **Direct webhook delivery** - Results sent directly to your API
- ✅ **Failure handling** - Sends failure notifications too
- ✅ **AWX optimized** - Works seamlessly with AWX/Tower

---

## 📁 Package Contents

```
axis-service-discovery-webhook/
├── site_webhook_only.yml              # Master playbook with webhook integration
├── Service_discovery/
│   ├── Middleware/
│   │   ├── Apache/
│   │   │   └── apache_discovery.yml   # Apache discovery (no file saving)
│   │   ├── Nginx/
│   │   │   └── nginx_discovery.yml    # Nginx discovery (no file saving)
│   │   └── Tomcat/
│   │       └── tomcat_discovery.yml   # Tomcat discovery (no file saving)
│   └── DB/
│       ├── Mysql/
│       │   └── mysql_discovery.yml    # MySQL discovery (no file saving)
│       ├── Postgres/
│       │   └── postgres_discovery.yml # PostgreSQL discovery (no file saving)
│       └── Mongodb/
│           └── mongodb_discovery.yml  # MongoDB discovery (no file saving)
├── README.md                          # This file
└── WEBHOOK_SETUP.md                   # Webhook configuration guide
```

---

## 🚀 Quick Start

### Step 1: Update Your Git Repository

```bash
# Extract the zip file
unzip axis-service-discovery-webhook.zip

# Copy to your existing repo
cd your-ansible-repo/
cp -r axis-service-discovery-webhook/* .

# IMPORTANT: Replace placeholder playbooks with your actual full playbooks
# Just make sure to remove file-saving tasks from each one

# Commit and push
git add .
git commit -m "Updated to webhook-only discovery (no file saving)"
git push origin main
```

### Step 2: Configure AWX

1. **Sync Project in AWX:**
   - Go to Projects → Your Project
   - Click Sync (refresh icon)
   - Wait for completion

2. **Update Job Template:**
   - Edit your job template
   - **Playbook:** Select `site_webhook_only.yml`
   - **Extra Variables:**
   ```yaml
   webhook_api_url: "https://your-api.example.com/webhook/discovery"
   webhook_api_token: "your-secret-token"
   ```

3. **Run the Job:**
   - Launch the job template
   - Results will be sent directly to your webhook
   - No files will be saved

---

## ⚙️ Configuration

### Environment Variables (Recommended)

In AWX, set these environment variables:

```bash
WEBHOOK_API_URL=https://your-api.example.com/webhook/discovery
WEBHOOK_TOKEN=your-secret-token-here
```

### Extra Variables (Alternative)

Or set in Job Template Extra Variables:

```yaml
webhook_api_url: "https://your-api.example.com/webhook/discovery"
webhook_api_token: "{{ lookup('env', 'WEBHOOK_TOKEN') }}"
```

---

## 📊 Webhook Payload Format

### Success Payload

```json
{
  "job_info": {
    "job_id": "12345",
    "job_name": "Service Discovery",
    "execution_time": "2025-12-02T08:00:00Z",
    "execution_node": "awx-node-1",
    "status": "success"
  },
  "discovery_results": {
    "middleware": {
      "apache": {
        "total_servers": 1,
        "total_instances": 1,
        "servers": [ /* array of server objects */ ]
      },
      "nginx": { /* ... */ },
      "tomcat": { /* ... */ }
    },
    "database": {
      "mysql": {
        "total_servers": 1,
        "total_instances": 1,
        "servers": [ /* array of server objects */ ]
      },
      "postgresql": { /* ... */ },
      "mongodb": { /* ... */ }
    }
  },
  "summary": {
    "total_middleware_instances": 3,
    "total_database_instances": 3,
    "total_servers_scanned": 2,
    "discovery_breakdown": [
      {"service": "Apache", "instances": 1},
      {"service": "Nginx", "instances": 1},
      {"service": "Tomcat", "instances": 1},
      {"service": "MySQL", "instances": 1},
      {"service": "PostgreSQL", "instances": 1},
      {"service": "MongoDB", "instances": 1}
    ]
  }
}
```

### Failure Payload

```json
{
  "job_info": {
    "job_id": "12345",
    "job_name": "Service Discovery",
    "execution_time": "2025-12-02T08:00:00Z",
    "status": "failed"
  },
  "error": {
    "message": "Service discovery failed",
    "task": "Collect MySQL discoveries",
    "error_details": { /* error object */ }
  }
}
```

---

## ⚠️ IMPORTANT: Complete Your Playbooks

The individual discovery playbooks (Apache, Nginx, Tomcat, PostgreSQL, MongoDB) in this package are **simplified placeholders**.

**You MUST replace them with your actual full playbooks**, just removing the file-saving tasks:

### Tasks to Remove from Each Playbook:

1. **Remove "Create output directory" task:**
```yaml
- name: Create output directory on localhost
  file:
    path: "{{ playbook_dir }}/xxx_discovery_output"
    state: directory
```

2. **Remove "Save discovery results" task:**
```yaml
- name: Save discovery results to JSON file
  copy:
    content: "{{ xxx_discovery_json | to_nice_json }}"
    dest: "{{ playbook_dir }}/xxx_discovery_output/..."
```

3. **Remove "Save consolidated report" task:**
```yaml
- name: Save consolidated JSON report
  copy:
    content: "{{ final_json_output | to_nice_json }}"
    dest: "{{ playbook_dir }}/xxx_discovery_output/consolidated_discovery.json"
```

### What to Keep:

✅ Keep all discovery logic
✅ Keep the `set_fact` tasks that create `xxx_discovery_json` variables
✅ Keep the debug tasks for console output

---

## 🔧 Webhook API Requirements

Your webhook API endpoint should:

1. **Accept POST requests** with JSON body
2. **Respond with status codes**: 200, 201, or 202
3. **Handle these headers:**
   - `Content-Type: application/json`
   - `Authorization: Bearer <token>`
   - `X-Job-ID: <awx_job_id>`
   - `X-Timestamp: <iso8601_timestamp>`

Example API endpoint (Python Flask):

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/webhook/discovery', methods=['POST'])
def discovery_webhook():
    token = request.headers.get('Authorization', '').replace('Bearer ', '')
    
    # Validate token
    if token != 'your-secret-token':
        return jsonify({"error": "Unauthorized"}), 401
    
    # Get payload
    data = request.json
    
    # Process discovery results
    print(f"Received discovery from job: {data['job_info']['job_id']}")
    print(f"Total instances: {data['summary']['total_middleware_instances']}")
    
    # Save to database, send notifications, etc.
    
    return jsonify({"status": "received", "message": "Discovery data processed"}), 200
```

---

## 🧪 Testing

### Test Without Webhook (Dry Run)

```bash
# Don't set webhook URL - will skip sending
ansible-playbook site_webhook_only.yml -e "webhook_api_url=''"
```

### Test With Mock Webhook

Use https://webhook.site/ for testing:

1. Go to https://webhook.site/
2. Copy your unique URL
3. Set in AWX:
```yaml
webhook_api_url: "https://webhook.site/#!/your-unique-id"
```
4. Run job and view results in webhook.site

---

## 🔍 Troubleshooting

### Issue: Webhook not receiving data

**Check:**
- ✅ Is `webhook_api_url` set correctly?
- ✅ Is `webhook_api_token` set?
- ✅ Can AWX reach the webhook URL? (firewall, DNS)
- ✅ Is webhook API returning 200/201/202?

**Debug:**
```yaml
# Add to site_webhook_only.yml before uri task:
- name: Debug webhook config
  debug:
    msg: |
      URL: {{ webhook_api_url }}
      Token: {{ 'SET' if webhook_api_token else 'NOT SET' }}
```

### Issue: "No webhook URL configured" message

The default URL is a placeholder. Set your actual URL:
```bash
# In AWX Extra Variables:
webhook_api_url: "https://your-actual-api.com/webhook"
```

### Issue: Discovery playbook variables not found

Make sure your individual discovery playbooks set these variables:
- `apache_discovery_json`
- `nginx_discovery_json`
- `tomcat_discovery_json`
- `mysql_discovery_json`
- `postgres_discovery_json`
- `mongodb_discovery_json`

---

## 📞 Support

For questions or issues:
1. Check WEBHOOK_SETUP.md for detailed configuration
2. Review AWX job output for error messages
3. Test webhook endpoint independently first
4. Verify all playbook variables are set correctly

---

## ✅ Checklist

Before running in production:

- [ ] Replaced placeholder playbooks with actual full playbooks
- [ ] Removed all file-saving tasks from playbooks
- [ ] Set webhook_api_url in AWX
- [ ] Set webhook_api_token securely
- [ ] Tested webhook endpoint responds correctly
- [ ] Verified firewall/network allows AWX → Webhook communication
- [ ] Tested with a single discovery service first
- [ ] Verified payload format matches your API expectations
- [ ] Set up error handling in webhook API
- [ ] Configured AWX notifications for job failures

---

## 🔄 Migration from File-Based Approach

If you're migrating from the previous file-saving approach:

1. **Backup your current playbooks**
2. **Update each playbook:**
   - Remove file-saving tasks
   - Keep all `set_fact` tasks
   - Ensure variable names match
3. **Update master playbook** to `site_webhook_only.yml`
4. **Configure webhook** in AWX
5. **Test thoroughly** before production use

---

## 📝 Notes

- All discovery data is kept in memory during execution
- No local files are created or saved
- Results are sent only to webhook API
- Failures are also reported to webhook
- Retries are built-in (3 attempts with 5 second delay)
- Suitable for AWX/Tower environments
- Can be run manually with ansible-playbook command

---

**Version:** 1.0  
**Last Updated:** December 2025  
**Compatible with:** AWX 21+, Ansible 2.9+
