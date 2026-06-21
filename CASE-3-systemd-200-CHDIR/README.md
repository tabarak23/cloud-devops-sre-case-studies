# Incident Investigation: Diagnosing a systemd 200/CHDIR Service Startup Failure

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/3f31e507940395836a045650c6ce0a0aa7b09e47/images/Gemini_Generated_Image_1p290q1p290q1p29.png)

## Troubleshooting skills are often more valuable than the actual fix.


Modern AI tools can quickly explain Linux error codes and suggest possible solutions. However, effective troubleshooting still depends on understanding how systems work, gathering evidence, validating assumptions, and identifying the actual root cause.

Recently, I investigated a Linux service startup failure where a Flask application became unavailable after a configuration change.

Although the issue itself was simple, it demonstrates an important lesson for anyone working in Linux Administration, Cloud Support, DevOps, Site Reliability Engineering (SRE), or Production Operations.

The objective was not simply to restore the service, but to understand why the failure occurred and validate the solution through evidence-based troubleshooting.

---

# Overview

A Flask application running as a systemd service failed to start after an incorrect working directory was configured.

My objective was to identify the root cause, restore service availability, and verify that the application was functioning normally after recovery.

---

# Initial Symptoms

The application was reported as unavailable following a service restart.

My first step was to check the status of the service.

```bash
sudo systemctl status myapp
```

The output showed:

```text
status=200/CHDIR
```

## Screenshot 1: Service Failure (200/CHDIR)

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/1b92b1c6d869722d6f6a5fbb11af088b3ec086c6/images/Screenshot%20from%202026-06-21%2008-40-02.png)


At this stage, I did not assume the cause of the problem.

The error code provided a useful clue, but error messages should be treated as the starting point of an investigation rather than the final answer.

---

# Investigation Process

## Step 1: Review Service Logs

Before modifying any configuration files, I wanted to gather additional evidence from the service logs.

I reviewed the latest journal entries for the service.

```bash
sudo journalctl -u myapp -n 20 --no-pager
```

The logs indicated that systemd failed while attempting to change into the configured working directory.

Example log entries included messages similar to:

```text
Failed at step CHDIR
Changing to the requested working directory failed
```

## Screenshot 2: Service Log Review

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/1b92b1c6d869722d6f6a5fbb11af088b3ec086c6/images/Screenshot%20from%202026-06-21%2008-40-21.png)


At this point, the investigation suggested that the issue might be related to the service's WorkingDirectory configuration.

However, I wanted to validate that assumption before making any changes.

---

## Step 2: Inspect the Service Configuration

Next, I reviewed the systemd service definition.

```bash
cat /etc/systemd/system/myapp.service
```

Relevant configuration:

```ini
WorkingDirectory=/opt/yourapp
```

## Screenshot 3: Reviewing Service Configuration

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/1b92b1c6d869722d6f6a5fbb11af088b3ec086c6/images/Screenshot%20from%202026-06-21%2008-41-08.png)


The service was configured to start from the following directory:

```text
/opt/yourapp
```

Rather than assuming the directory existed, I decided to verify it directly.

One troubleshooting principle I try to follow is:

**Never trust assumptions. Verify them.**

---

## Step 3: Validate the Configured Working Directory

To confirm whether the configured working directory actually existed on the server, I executed:

```bash
ls -ld /opt/yourapp
```

The output showed:

```text
No such file or directory
```

## Screenshot 4: Verifying Working Directory

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/1b92b1c6d869722d6f6a5fbb11af088b3ec086c6/images/Screenshot%20from%202026-06-21%2008-41-31.png)


This provided strong evidence that the service was configured to use a directory that did not exist on the server.

At this point, the investigation was moving toward a likely root cause.

---

## Step 4: Verify Existing Application Directory

To better understand the environment, I checked the actual application directory.

```bash
ls -ld /opt/myapp
```

The output confirmed that the application files were located under:

```text
/opt/myapp
```

and not:

```text
/opt/yourapp
```

## Screenshot 5: Verifying Actual Application Directory

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/1b92b1c6d869722d6f6a5fbb11af088b3ec086c6/images/Screenshot%20from%202026-06-21%2008-42-24.png)


At this point, multiple pieces of evidence consistently pointed to the same conclusion.

---

# Root Cause Analysis

The root cause was that the systemd service was configured with an invalid working directory.

The service configuration specified:

```ini
WorkingDirectory=/opt/yourapp
```

However, the directory:

```text
/opt/yourapp
```

did not exist on the server.

Before starting the application, systemd attempts to change into the configured working directory.

Because the directory was missing, systemd could not complete the startup process and terminated the service.

As a result, the service failed with:

```text
status=200/CHDIR
```

Although the issue was relatively small, it demonstrates how critical configuration accuracy is when managing Linux services.

Without validating the configured path, it would be easy to spend time troubleshooting unrelated components such as application code, networking, permissions, or Python dependencies.

---

# How This Issue Can Occur

This type of issue commonly occurs during:

* Application migrations
* Server rebuilds
* Directory renaming
* Deployment changes
* Configuration copy/paste mistakes
* Infrastructure automation updates

A single incorrect path in a systemd unit file can prevent an otherwise healthy application from starting.

---

# Resolution

After confirming the root cause, I updated the service definition to reference the correct application directory.

```ini
WorkingDirectory=/opt/myapp
```

## Screenshot 6: Corrected Service Configuration

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/b467c5760fe9b91466d1ed8741669ce8bb978de4/images/Screenshot%20from%202026-06-21%2008-57-31.png)


After applying the change, I reloaded systemd and restarted the service.

```bash
sudo systemctl daemon-reload

sudo systemctl restart myapp
```

---

# Verification

After restarting the service, I verified its operational status.

```bash
sudo systemctl status myapp
```

Result:

```text
Active: active (running)
```

## Screenshot 7: Service Successfully Restored

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/c1d8536ce8182213bef264b955adafa930593bcb/images/Screenshot%20from%202026-06-21%2008-42-53.png)


To confirm that the application itself was functioning correctly, I tested the service endpoint.

```bash
curl localhost:8000
```

Result:

```text
Application Running Successfully
```

## Screenshot 8: Application Health Verification

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/1b92b1c6d869722d6f6a5fbb11af088b3ec086c6/images/Screenshot%20from%202026-06-21%2008-43-21.png)


Normal service operation was restored successfully.

---

# Troubleshooting Workflow

```text
Application Unavailable
          ↓
Review Service Status
          ↓
Observe 200/CHDIR
          ↓
Review Service Logs
          ↓
Inspect Service Configuration
          ↓
Validate Working Directory
          ↓
Directory Not Found
          ↓
Identify Root Cause
          ↓
Apply Configuration Fix
          ↓
Restart Service
          ↓
Verify Recovery
```

---

# How AI Can Help

AI tools can be extremely useful during troubleshooting.

For example, an AI assistant can:

* Explain what 200/CHDIR means
* Suggest investigation steps
* Recommend useful Linux commands
* Explain systemd startup behavior
* Help interpret logs

These capabilities can significantly reduce investigation time.

---

# Why AI Still Cannot Replace Engineers

While AI can explain an error, it cannot automatically determine whether the error is the actual root cause in a production environment.

An engineer must still:

* Collect evidence
* Validate assumptions
* Understand system architecture
* Correlate logs and configuration
* Distinguish symptoms from root causes
* Verify recovery after remediation

In this incident, AI could explain that 200/CHDIR relates to the working directory.

However, an engineer still had to:

* Inspect the service configuration
* Verify whether the directory existed
* Compare expected and actual paths
* Confirm the application location
* Apply the correct fix
* Validate service recovery

The real value comes from investigation and judgment, not simply knowing what an error code means.

---

# Key Takeaways

* Error codes should be treated as clues, not conclusions.
* Always verify configured paths during systemd troubleshooting.
* Logs should be reviewed before modifying configurations.
* Root cause analysis is more valuable than trial-and-error troubleshooting.
* AI can assist investigations, but engineers are still responsible for validating evidence and implementing the correct solution.
* Always verify service health after remediation.

---

# Skills Demonstrated

* Linux Administration
* systemd Troubleshooting
* Log Analysis
* Incident Response
* Root Cause Analysis (RCA)
* Service Recovery
* Cloud Support Operations
* Production Support
* Infrastructure Operations
* Problem Solving and Troubleshooting

---

# Conclusion

This incident involved a simple configuration mistake, but it reinforces an important lesson about troubleshooting Linux services.

The real value was not correcting a path.

The value was understanding how systemd handles working directories, using logs to guide the investigation, validating assumptions through evidence, and confirming recovery after remediation.

Whether working as a Linux Administrator, Cloud Support Engineer, DevOps Engineer, or SRE, the ability to investigate problems methodically remains one of the most valuable engineering skills.

Successful troubleshooting is not about finding the fastest answer.

It is about finding the correct answer based on evidence, understanding, and root cause analysis.
