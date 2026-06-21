# Incident Investigation: Diagnosing a systemd 203/EXEC Service Startup Failure

## Troubleshooting skills are often more valuable than the actual fix.

Modern AI tools can explain Linux error codes, suggest commands, and provide troubleshooting guidance. However, effective troubleshooting still depends on understanding Linux fundamentals, analyzing logs, validating assumptions, and identifying the actual root cause.

Recently, I investigated a Linux service startup failure where a Flask application became unavailable after a configuration change.

Although the issue itself was relatively simple to fix, I decided to document the investigation because it demonstrates an important lesson for anyone working in Linux Administration, Cloud Support, DevOps, SRE, or Production Operations.

The objective was not simply to restore the service, but to understand why the failure occurred and validate the solution through evidence-based troubleshooting.

---

# Overview

A Flask application running as a systemd service failed to start after an incorrect application file was configured in the service definition.

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
status=203/EXEC
```

## Screenshot 1: Service Failure (203/EXEC)

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-43-05.png)


At this stage, I did not immediately assume the cause of the problem.

The error code provided a useful clue, but error messages should be treated as starting points for investigation rather than final answers.

---

# Investigation Process

## Step 1: Review Service Logs

Before modifying any configuration files, I wanted to collect additional evidence from the service logs.

I reviewed the most recent journal entries for the service.

```bash
sudo journalctl -u myapp -n 20 --no-pager
```

The logs indicated that systemd was encountering an issue during the execution phase of the startup process.

Example log entries included:

```text
Failed at step EXEC
No such file or directory
```

## Screenshot 2: Service Log Review

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-43-23.png)


At this point, the investigation suggested that the failure might be related to the command configured within the service definition.

However, I wanted to validate that assumption before making any changes.

---

## Step 2: Inspect the Service Configuration

Next, I reviewed the systemd service definition.

```bash
cat /etc/systemd/system/myapp.service
```

Relevant configuration:

```ini
ExecStart=/usr/bin/python3 /opt/myapp/web.py
```

## Screenshot 3: Reviewing Service Configuration

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-43-47.png)


The service was configured to start the following application file:

```text
/opt/myapp/web.py
```

Rather than assuming the file existed, I decided to verify it directly.

One troubleshooting principle I try to follow is:

**Never trust assumptions. Verify them.**

---

## Step 3: Validate the Configured Application File

To confirm whether the configured application file actually existed on the server, I executed:

```bash
ls -l /opt/myapp/web.py
```

The output showed:

```text
No such file or directory
```

## Screenshot 4: Verifying Application File

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-44-02.png)


This provided strong evidence that the service was attempting to execute a file that did not exist on the server.

---

## Root Cause Analysis

The root cause was that the systemd service was configured to execute a Python file that did not exist.

The service definition contained:

```ini
ExecStart=/usr/bin/python3 /opt/myapp/web.py
```

However, the actual application file present on the server was:

```text
/opt/myapp/app.py
```

and not:

```text
/opt/myapp/web.py
```

When systemd attempted to execute the configured command, it could not locate the specified file.

As a result, the service startup failed with:

```text
status=203/EXEC
```

Although the issue was relatively small, it demonstrates how critical configuration accuracy is when managing Linux services.

Without validating the configured execution path, it would be easy to spend time troubleshooting unrelated areas such as networking, permissions, Python packages, or application code.

---

# How This Issue Can Occur

This type of issue commonly occurs during:

* Application migrations
* Server rebuilds
* File renaming
* Deployment changes
* Configuration copy/paste mistakes
* Infrastructure automation updates
* Manual configuration modifications

A single incorrect file path in the ExecStart directive can prevent an otherwise healthy application from starting.

---

# Resolution

After confirming the root cause, I updated the service definition to reference the correct application file.

```ini
ExecStart=/usr/bin/python3 /opt/myapp/app.py
```

## Screenshot 5: Corrected Service Configuration

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-46-02.png)


After applying the change, I reloaded systemd and restarted the service.

```bash
sudo systemctl daemon-reload

sudo systemctl restart myapp
```

## Screenshot 6: Reloading systemd and Restarting Service

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-46-28.png)


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

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-46-45.png)


To confirm that the application itself was functioning correctly, I tested the service endpoint.

```bash
curl localhost:8000
```

Result:

```text
Application Running Successfully
```

## Screenshot 8: Application Health Verification

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/82c6722ba6e30919a8ba4d1beaffd171cea15e07/images/Screenshot%20from%202026-06-21%2009-47-02.png)


Normal service operation was restored successfully.

---

# Troubleshooting Workflow

```text
Application Unavailable
          ↓
Review Service Status
          ↓
Observe 203/EXEC
          ↓
Review Service Logs
          ↓
Inspect Service Configuration
          ↓
Validate Application File
          ↓
File Not Found
          ↓
Identify Root Cause
          ↓
Apply Configuration Fix
          ↓
Reload systemd
          ↓
Restart Service
          ↓
Verify Recovery
```

---

# How AI Can Help

AI tools can be extremely useful during troubleshooting.

For example, an AI assistant can:

* Explain what 203/EXEC means
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

In this incident, AI could explain that 203/EXEC relates to command execution.

However, an engineer still had to:

* Inspect the service configuration
* Verify whether the configured file existed
* Compare expected and actual application paths
* Identify the incorrect file reference
* Apply the correct fix
* Validate service recovery

The real value comes from investigation and judgment, not simply knowing what an error code means.

---

# Key Takeaways

* Error codes should be treated as clues, not conclusions.
* Always validate configured execution paths during systemd troubleshooting.
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

The real value was not correcting the file path.

The value was understanding how systemd executes startup commands, using logs to guide the investigation, validating assumptions through evidence, and confirming recovery after remediation.

Whether working as a Linux Administrator, Cloud Support Engineer, DevOps Engineer, or SRE, the ability to investigate problems methodically remains one of the most valuable engineering skills.

Successful troubleshooting is not about finding the fastest answer.

It is about finding the correct answer based on evidence, understanding, and root cause analysis.
