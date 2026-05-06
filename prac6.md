# Practical 06 — Monitor Resources Using AWS CloudWatch

---

## 📌 Objective

Use AWS CloudWatch to monitor EC2 resources, create alarms that trigger on threshold breaches, configure SNS for email notifications, and test the monitoring setup by simulating high CPU usage.

---

## 🧠 Conceptual Background (Know-How)

### What is AWS CloudWatch?

**Amazon CloudWatch** is a monitoring and observability service that collects metrics, logs, and events from AWS resources and applications. It gives you visibility into your infrastructure's performance and health.

### CloudWatch Core Concepts

```
┌───────────────────────────────────────────────────────────┐
│                    AWS CloudWatch                         │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │   Metrics   │  │    Logs     │  │     Events       │  │
│  │  (Numbers)  │  │  (Text)     │  │  (Notifications) │  │
│  │             │  │             │  │                  │  │
│  │ CPU: 45%    │  │ /var/log/   │  │  EC2 state       │  │
│  │ Memory: 2GB │  │ httpd/      │  │  changed         │  │
│  │ Disk: 8GB   │  │ error.log   │  │  S3 object       │  │
│  └──────┬──────┘  └─────────────┘  │  uploaded        │  │
│         │                          └──────────────────┘  │
│  ┌──────▼──────┐                                          │
│  │   Alarms    │  ← Watch metrics, trigger actions        │
│  │             │                                          │
│  │ CPU > 70%   │──────► SNS → Email/SMS/Lambda            │
│  │ Disk < 10%  │──────► Auto Scaling                      │
│  └─────────────┘                                          │
└───────────────────────────────────────────────────────────┘
```

### CloudWatch Metric Dimensions

Metrics are organized by **namespace**, **metric name**, and **dimensions**:

```
Namespace:   AWS/EC2
Metric:      CPUUtilization
Dimensions:  InstanceId = i-0123456789abcdef0
Period:      5 minutes (default)
Value:       45.2 (%)
```

**Default EC2 Metrics (Free, 5-minute granularity):**
| Metric | Unit | Description |
|---|---|---|
| CPUUtilization | % | CPU usage |
| NetworkIn | Bytes | Incoming network traffic |
| NetworkOut | Bytes | Outgoing network traffic |
| DiskReadOps | Count | Disk read operations |
| DiskWriteOps | Count | Disk write operations |
| StatusCheckFailed | Count | 0=OK, 1=failed |

**Detailed Monitoring ($0.30/metric/month, 1-minute granularity):**
```
EC2 → Instances → Select instance → Actions → Monitor and troubleshoot
→ Enable detailed monitoring → Enable
```

> ⚠️ **Memory and disk space are NOT default CloudWatch metrics** — you must install the CloudWatch Agent to get them.

### What is Amazon SNS?

**Amazon Simple Notification Service (SNS)** is a messaging service that allows you to send notifications to subscribers. CloudWatch Alarms use SNS to send alerts.

```
CloudWatch Alarm triggers → SNS Topic → Subscribers
                                         ├── Email
                                         ├── SMS
                                         ├── Lambda function
                                         ├── SQS queue
                                         └── HTTP endpoint
```

---

## 🛠️ Step-by-Step Guide

### Prerequisites
- A running EC2 instance (from Practical 02)
- AWS Console access
- An email address for notifications

---

### Step A — Set Up CloudWatch Metrics for EC2

#### Navigate to CloudWatch
```
AWS Console → CloudWatch → Metrics → All metrics
```

#### View EC2 Metrics
```
All metrics → AWS/EC2 → Per-Instance Metrics
→ Find your instance ID
→ Click CPUUtilization to graph it
```

**Create a Custom Dashboard:**
1. Go to **CloudWatch → Dashboards → Create dashboard**
2. Dashboard name: `EC2-Monitoring-Dashboard`
3. Click **Create dashboard**
4. Add widget → **Line** → **Metrics**
5. Select: `AWS/EC2 → Per-Instance Metrics → CPUUtilization`
6. Select your instance → **Create widget**
7. Add another widget for `NetworkIn`, `NetworkOut`
8. Save dashboard

---

### Step B — Create a CloudWatch Alarm

1. Go to **CloudWatch → Alarms → All alarms → Create alarm**

**Step 1 — Select metric:**
```
Select metric → EC2 → Per-Instance Metrics → CPUUtilization
Select your instance → Select metric

Statistic:   Average
Period:      5 minutes
```

**Step 2 — Specify conditions:**
```
Threshold type:              Static
Condition:                   Greater/Equal
Than:                        70    (70%)

Additional configuration:
  Datapoints to alarm:       2 out of 3
  (This means 2 consecutive data points must exceed threshold)
  
  Missing data treatment:    Treat missing data as missing
```

**Step 3 — Configure actions:**
```
Alarm state trigger:         In alarm

Send notification to:        Create new SNS topic →
  Create a new topic
  Topic name:                ec2-cpu-alerts
  Email endpoints:           your@email.com
→ Create topic
```

**Step 4 — Name and description:**
```
Alarm name:        EC2-High-CPU-Alarm
Description:       Alert when EC2 CPU exceeds 70% for 10 minutes
```

**Step 5 — Preview and create → Create alarm**

> 📧 **Check your email** — you'll receive a subscription confirmation from AWS SNS. You MUST click **Confirm subscription** or you won't receive alerts!

---

### Step C — Configure SNS Topic for Email Notifications

Let's explore SNS in more detail.

#### View the SNS Topic
```
AWS Console → SNS → Topics → ec2-cpu-alerts
→ View subscriptions, ARN, access policy
```

#### Add Additional Subscribers
```
SNS → Topics → ec2-cpu-alerts → Create subscription
  Protocol: Email
  Endpoint: another@email.com
→ Create subscription
(The new subscriber must confirm via email)
```

#### Test SNS Directly
```
SNS → Topics → ec2-cpu-alerts → Publish message
  Subject:   Test Notification
  Message:   This is a test message from AWS SNS
→ Publish message
```

You should receive the test email immediately.

#### SNS Topic Policy (Access Control)
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudwatch.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:us-east-1:ACCOUNT_ID:ec2-cpu-alerts",
      "Condition": {
        "StringEquals": {
          "AWS:SourceAccount": "ACCOUNT_ID"
        }
      }
    }
  ]
}
```

---

### Step D — Test by Simulating High CPU Usage

#### SSH into your EC2 instance
```bash
ssh -i my-ec2-keypair.pem ec2-user@<public-ip>
```

#### Method 1 — Using `stress`
```bash
# Install stress utility
sudo yum install -y stress      # Amazon Linux
# or
sudo apt install -y stress      # Ubuntu

# Stress all CPUs for 15 minutes
sudo stress --cpu $(nproc) --timeout 900

# Check CPU usage in real time
top
# Press 'q' to quit top
```

#### Method 2 — Using `dd` (disk + CPU)
```bash
# This stresses both CPU and I/O
dd if=/dev/zero of=/dev/null bs=1M count=100000
```

#### Method 3 — Using a Bash Loop
```bash
# Pure CPU stress with bash
while true; do :; done &
while true; do :; done &
while true; do :; done &
while true; do :; done

# To stop:
kill $(jobs -p)
```

#### Method 4 — Install and Run the CloudWatch Agent for Memory Metrics

The default CloudWatch metrics don't include memory. Install the agent:

```bash
# Download the agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm

# Install
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Run the setup wizard
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start the agent
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

---

## 📊 Alarm States and Transitions

```
┌─────────────────────────────────────────────────────────┐
│              CloudWatch Alarm States                    │
│                                                         │
│  ┌──────────┐  CPU > 70%   ┌─────────┐                  │
│  │   OK     │─────────────▶│  ALARM  │────► SNS Alert   │
│  │          │◀─────────────│         │                  │
│  └──────────┘  CPU < 70%   └────┬────┘                  │
│                                 │                       │
│  ┌──────────────┐               │                       │
│  │INSUFFICIENT  │◀──────────────┘                       │
│  │    DATA      │  No data received                     │
│  └──────────────┘                                       │
│                                                         │
│  State transitions generate CloudWatch Events           │
└─────────────────────────────────────────────────────────┘
```

---

## 📋 CloudWatch Logs (Bonus)

Send application logs to CloudWatch:

```bash
# Install and configure CloudWatch agent for logs
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Manual log group creation
aws logs create-log-group --log-group-name /ec2/apache/access-logs

# Send Apache logs to CloudWatch
# Add to /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json:
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/httpd/access_log",
            "log_group_name": "/ec2/apache/access-logs",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

**Create Metric Filter from Logs (count 404 errors):**
```
CloudWatch → Log groups → /ec2/apache/access-logs →
Metric filters → Create metric filter
  Filter pattern: [ip, id, user, timestamp, request, status_code=404, bytes]
  Metric name:    404-errors
  Metric value:   1
→ Create metric filter

Then create an alarm on this custom metric!
```

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Navigate the CloudWatch console and view EC2 metrics
- [ ] Create a CloudWatch Dashboard for custom metric views
- [ ] Configure a CloudWatch Alarm with appropriate thresholds
- [ ] Create an SNS Topic and add email subscribers
- [ ] Confirm SNS subscriptions and test notifications
- [ ] Simulate high CPU usage on an EC2 instance
- [ ] Observe alarm state transitions (OK → ALARM)
- [ ] Explain the difference between default and custom metrics
- [ ] Set up the CloudWatch Agent for memory/disk metrics

---

## 💰 Cost Awareness

```
CloudWatch Pricing:
  Basic monitoring (5-min data):    FREE for EC2
  Detailed monitoring (1-min data): $0.30/metric/month
  Custom metrics:                   $0.30/metric/month (first 10,000)
  Alarms:                           $0.10/alarm/month
  Dashboards:                       $3.00/dashboard/month (after 3 free)
  Logs ingestion:                   $0.50/GB
  
SNS Pricing:
  First 1 million notifications:    FREE
  Email/SMS:                        $0.75/100,000 emails
```

---

## 🔐 Security Best Practices

1. Use **IAM roles** for EC2 instances instead of hardcoded credentials when sending metrics to CloudWatch
2. **Encrypt CloudWatch Logs** using KMS
3. Set alarms for **security events**: failed logins (`/var/log/secure`), unusual API calls via CloudTrail
4. Create a **billing alarm** to catch unexpected costs early

---

## 📚 Further Reading

- [CloudWatch User Guide](https://docs.aws.amazon.com/cloudwatch/)
- [CloudWatch Agent Setup](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [Amazon SNS Documentation](https://docs.aws.amazon.com/sns/)
- [CloudWatch Pricing](https://aws.amazon.com/cloudwatch/pricing/)
