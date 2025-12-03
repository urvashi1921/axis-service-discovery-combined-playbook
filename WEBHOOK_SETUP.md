# Webhook Setup Guide

## 📡 Webhook API Integration

This guide explains how to configure and integrate the webhook API endpoint with your service discovery playbooks.

---

## 🎯 Overview

The playbooks send discovery results to a webhook API endpoint via HTTP POST request with JSON payload. No files are saved locally.

**Benefits:**
- ✅ Real-time data delivery
- ✅ No file system dependencies
- ✅ Direct integration with your systems
- ✅ Built-in retry logic
- ✅ Failure notifications

---

## 🔧 AWX Configuration

### Method 1: Environment Variables (Recommended)

Set these in AWX Job Template → Environment Variables:

```bash
WEBHOOK_API_URL=https://your-api.example.com/webhook/discovery
WEBHOOK_TOKEN=your-secret-token-here
```

**Advantages:**
- Secure token management
- Easy to update without playbook changes
- Separate dev/prod configurations

### Method 2: Extra Variables

Set in AWX Job Template → Extra Variables:

```yaml
webhook_api_url: "https://your-api.example.com/webhook/discovery"
webhook_api_token: "your-secret-token"
```

### Method 3: Survey (For User Input)

Create a survey in Job Template with:
- Variable: `webhook_api_url`
- Type: Text
- Default: Your webhook URL
- Required: Yes

---

## 🔐 Security Considerations

### API Token Security

**Best Practices:**
1. Use AWX Credentials feature:
   ```yaml
   webhook_api_token: "{{ lookup('file', '/path/to/token') }}"
   ```

2. Use AWX Custom Credential Types:
   - Create credential type for webhook token
   - Reference in job template
   - Token never exposed in logs

3. Rotate tokens regularly

### Network Security

- Use HTTPS only (validate_certs: yes)
- Whitelist AWX IP at webhook endpoint
- Use API gateway with rate limiting
- Implement request signing for verification

---

## 📨 Webhook Request Format

### Headers

```http
POST /webhook/discovery HTTP/1.1
Host: your-api.example.com
Content-Type: application/json
Authorization: Bearer your-secret-token
X-Job-ID: 12345
X-Timestamp: 2025-12-02T08:00:00Z
Content-Length: 1234
```

### Success Payload Structure

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
        "total_instances": 2,
        "servers": [
          {
            "hostname": "web-server-01",
            "ip_address": "10.0.1.10",
            "os": "Ubuntu 20.04",
            "discovery_timestamp": "2025-12-02T08:00:00Z",
            "discovery": [
              {
                "name": "apache-main",
                "version": "2.4.41",
                "ports": [80, 443],
                "pids": [1234, 1235],
                "bin_path": ["/usr/sbin/apache2"],
                "config_path": ["/etc/apache2/apache2.conf"],
                "type": "MIDDLEWARE"
              }
            ]
          }
        ]
      },
      "nginx": { /* similar structure */ },
      "tomcat": { /* similar structure */ }
    },
    "database": {
      "mysql": {
        "total_servers": 1,
        "total_instances": 1,
        "servers": [
          {
            "hostname": "db-server-01",
            "ip_address": "10.0.2.10",
            "os": "CentOS 7.9",
            "discovery_timestamp": "2025-12-02T08:00:00Z",
            "discovery": [
              {
                "name": "mysql-prod",
                "version": "8.0.32",
                "ports": [3306],
                "pids": [5678],
                "bin_path": ["/usr/sbin/mysqld"],
                "config_path": ["/etc/my.cnf"],
                "data_path": ["/var/lib/mysql"],
                "socket": "/var/lib/mysql/mysql.sock",
                "type": "DATABASE"
              }
            ]
          }
        ]
      },
      "postgresql": { /* similar structure */ },
      "mongodb": { /* similar structure */ }
    }
  },
  "summary": {
    "total_middleware_instances": 3,
    "total_database_instances": 3,
    "total_servers_scanned": 5,
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

### Failure Payload Structure

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
    "task": "Collect MySQL discoveries from all hosts",
    "error_details": {
      "msg": "Connection timeout to managed host",
      "module": "setup",
      "host": "db-server-01"
    }
  }
}
```

---

## 💻 Webhook API Implementation Examples

### Python Flask Example

```python
from flask import Flask, request, jsonify
import logging
from datetime import datetime

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

# Your secret token
VALID_TOKEN = "your-secret-token-here"

@app.route('/webhook/discovery', methods=['POST'])
def discovery_webhook():
    # Validate token
    auth_header = request.headers.get('Authorization', '')
    token = auth_header.replace('Bearer ', '')
    
    if token != VALID_TOKEN:
        logging.warning(f"Unauthorized access attempt from {request.remote_addr}")
        return jsonify({"error": "Unauthorized"}), 401
    
    # Get job info from headers
    job_id = request.headers.get('X-Job-ID', 'unknown')
    timestamp = request.headers.get('X-Timestamp', datetime.utcnow().isoformat())
    
    # Parse payload
    data = request.json
    
    if not data:
        return jsonify({"error": "No data received"}), 400
    
    # Log receipt
    logging.info(f"Received discovery from job {job_id}")
    
    # Check if success or failure
    status = data.get('job_info', {}).get('status', 'unknown')
    
    if status == 'success':
        # Process successful discovery
        summary = data.get('summary', {})
        logging.info(f"Discovery completed: {summary.get('total_servers_scanned')} servers scanned")
        logging.info(f"Middleware instances: {summary.get('total_middleware_instances')}")
        logging.info(f"Database instances: {summary.get('total_database_instances')}")
        
        # Save to database
        # save_discovery_results(data)
        
        # Send notifications
        # send_slack_notification(data)
        
        return jsonify({
            "status": "received",
            "message": "Discovery data processed successfully",
            "job_id": job_id,
            "processed_at": datetime.utcnow().isoformat()
        }), 200
    
    elif status == 'failed':
        # Handle failure
        error = data.get('error', {})
        logging.error(f"Discovery failed: {error.get('message')}")
        logging.error(f"Failed task: {error.get('task')}")
        
        # Send alert
        # send_alert_notification(data)
        
        return jsonify({
            "status": "error_received",
            "message": "Failure notification received",
            "job_id": job_id
        }), 200
    
    else:
        logging.warning(f"Unknown status: {status}")
        return jsonify({"error": "Unknown status"}), 400

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, ssl_context='adhoc')
```

### Node.js Express Example

```javascript
const express = require('express');
const app = express();

const VALID_TOKEN = 'your-secret-token-here';

app.use(express.json());

app.post('/webhook/discovery', (req, res) => {
    // Validate token
    const authHeader = req.headers.authorization || '';
    const token = authHeader.replace('Bearer ', '');
    
    if (token !== VALID_TOKEN) {
        console.log(`Unauthorized access from ${req.ip}`);
        return res.status(401).json({ error: 'Unauthorized' });
    }
    
    // Get job info
    const jobId = req.headers['x-job-id'] || 'unknown';
    const data = req.body;
    
    if (!data) {
        return res.status(400).json({ error: 'No data received' });
    }
    
    const status = data.job_info?.status || 'unknown';
    
    if (status === 'success') {
        console.log(`Discovery successful for job ${jobId}`);
        console.log(`Servers scanned: ${data.summary?.total_servers_scanned}`);
        
        // Process data
        // processDiscoveryData(data);
        
        res.status(200).json({
            status: 'received',
            message: 'Discovery data processed',
            job_id: jobId
        });
    } else if (status === 'failed') {
        console.error(`Discovery failed for job ${jobId}`);
        console.error(`Error: ${data.error?.message}`);
        
        // Send alert
        // sendAlert(data);
        
        res.status(200).json({
            status: 'error_received',
            job_id: jobId
        });
    } else {
        res.status(400).json({ error: 'Unknown status' });
    }
});

app.listen(5000, () => {
    console.log('Webhook server running on port 5000');
});
```

---

## 🧪 Testing Your Webhook

### Using webhook.site

1. Go to https://webhook.site/
2. Copy your unique URL (e.g., `https://webhook.site/#!/abc123-def456`)
3. Set in AWX:
   ```yaml
   webhook_api_url: "https://webhook.site/abc123-def456"
   ```
4. Run your discovery job
5. View the received payload on webhook.site

### Using curl (Manual Test)

```bash
curl -X POST https://your-api.example.com/webhook/discovery \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-token" \
  -H "X-Job-ID: test-123" \
  -H "X-Timestamp: 2025-12-02T08:00:00Z" \
  -d '{
    "job_info": {
      "job_id": "test-123",
      "status": "success"
    },
    "summary": {
      "total_servers_scanned": 2,
      "total_middleware_instances": 1,
      "total_database_instances": 1
    }
  }'
```

### Using Postman

1. Create new request: POST
2. URL: Your webhook URL
3. Headers:
   - `Content-Type: application/json`
   - `Authorization: Bearer your-token`
   - `X-Job-ID: test-123`
4. Body: Raw JSON (use example payload above)
5. Send and check response

---

## 🔍 Troubleshooting

### Issue: Webhook not receiving data

**Checks:**
1. Verify URL is correct:
   ```bash
   curl -I https://your-api.example.com/webhook/discovery
   ```

2. Check AWX can reach webhook:
   ```bash
   # From AWX server
   curl -v https://your-api.example.com/webhook/discovery
   ```

3. Verify token is set:
   ```yaml
   # In AWX job output, look for:
   # "Configure WEBHOOK_API_URL in AWX"
   ```

4. Check firewall rules

5. Review webhook API logs

### Issue: 401 Unauthorized

**Solutions:**
- Verify token matches exactly (no extra spaces)
- Check Authorization header format: `Bearer <token>`
- Ensure token is not expired
- Review webhook API authentication logic

### Issue: 500 Server Error

**Solutions:**
- Check webhook API logs for errors
- Verify payload format matches expected structure
- Check for JSON parsing errors
- Ensure all required fields are present

### Issue: Timeout

**Solutions:**
- Increase timeout in playbook (default 60s):
  ```yaml
  timeout: 120
  ```
- Check webhook API response time
- Ensure webhook API is responsive
- Check network latency

### Debugging Playbook

Add debug task before webhook call:

```yaml
- name: Debug webhook configuration
  debug:
    msg: |
      Webhook URL: {{ webhook_api_url }}
      Token Set: {{ 'YES' if webhook_api_token else 'NO' }}
      Payload Size: {{ combined_payload | to_json | length }} bytes
```

---

## 📊 Monitoring & Logging

### AWX Side

Monitor in AWX Job Output:
- Look for "Send combined results to webhook API" task
- Check HTTP response status
- Review retry attempts if any

### Webhook Side

Implement logging:
```python
logging.info(f"Received {len(data)} bytes from job {job_id}")
logging.info(f"Processing time: {processing_time}ms")
logging.info(f"Servers: {servers_count}, Instances: {instances_count}")
```

### Metrics to Track

- Request count per day
- Success vs failure ratio
- Average payload size
- Processing time
- Error rates
- Retry attempts

---

## 🔄 Retry Logic

The playbook includes built-in retry:

```yaml
until: webhook_response.status in [200, 201, 202]
retries: 3
delay: 5
```

**Behavior:**
- Tries up to 3 times
- Waits 5 seconds between attempts
- Succeeds if status code is 200, 201, or 202
- Fails after 3 failed attempts

**To adjust:**
```yaml
retries: 5        # More attempts
delay: 10         # Longer wait
```

---

## 🚨 Failure Handling

### In Playbook

The playbook automatically sends failure notifications:

```yaml
rescue:
  - name: Send failure notification to webhook
    uri:
      url: "{{ webhook_api_url }}"
      method: POST
      body: "{{ failure_payload }}"
```

### In Webhook API

Handle failures appropriately:

```python
if status == 'failed':
    # Send urgent alert
    send_pagerduty_alert(data)
    
    # Log for analysis
    log_failure_to_database(data)
    
    # Create ticket
    create_jira_ticket(data)
```

---

## 💡 Best Practices

1. **Always use HTTPS** in production
2. **Implement rate limiting** on webhook endpoint
3. **Validate payload structure** before processing
4. **Log all requests** for audit trail
5. **Use idempotency keys** (job_id) to prevent duplicate processing
6. **Implement request signing** for additional security
7. **Set up monitoring alerts** for webhook failures
8. **Use async processing** for large payloads
9. **Implement retry with exponential backoff** on webhook side
10. **Store raw payloads** for debugging and replay

---

## 📞 Support

If you encounter issues:
1. Check AWX job output logs
2. Review webhook API logs
3. Test webhook endpoint independently
4. Verify network connectivity
5. Check payload format matches expected structure

---

**Version:** 1.0  
**Last Updated:** December 2025
