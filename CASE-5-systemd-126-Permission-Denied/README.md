# Incident Investigation: Diagnosing a systemd 126 Permission Denied Service Startup Failure


![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/9a374f76133fc9dffaff731326495e85ac4427bb/images/Gemini_Generated_Image_crrgdtcrrgdtcrrg.png)


## Troubleshooting skills are often more valuable than the actual fix.

Modern AI tools can explain Linux errors and suggest possible solutions. However, effective troubleshooting still depends on understanding Linux fundamentals, analyzing logs, validating assumptions, and identifying the actual root cause.

Recently, I investigated a Linux service startup failure where a Flask application became unavailable after a service restart.

Although the issue itself was relatively simple to fix, I decided to document the investigation because it demonstrates an important lesson for anyone working in Linux Administration, Cloud Support, DevOps, SRE, or Production Operations.

The objective was not simply to restore the service, but to understand why the failure occurred and validate the solution through evidence-based troubleshooting.

---

# Overview

A Flask application running as a systemd service failed to start because the configured startup script did not have executable permissions.

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
status=126
```

## Screenshot 1: Service Failure (126 Permission Denied)

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/df87cabeef996dc9b62eed3fb068031e53f53b82/images/Screenshot%20from%202026-06-22%2012-00-15.png)


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

The logs indicated that systemd was encountering a permission-related issue during the execution phase of the startup process.

Example log entries included:

```text
Permission denied
Failed at step EXEC
```

## Screenshot 2: Service Log Review

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/df87cabeef996dc9b62eed3fb068031e53f53b82/images/Screenshot%20from%202026-06-22%2012-00-34.png)


At this point, the investigation suggested that the issue might be related to execution permissions.

However, I wanted to validate that assumption before making any changes.

---

## Step 2: Inspect the Service Configuration

Next, I reviewed the systemd service definition.

```bash
cat /etc/systemd/system/myapp.service
```

Relevant configuration:

```ini
ExecStart=/opt/myapp/start.sh
```

## Screenshot 3: Reviewing Service Configuration

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/df87cabeef996dc9b62eed3fb068031e53f53b82/images/Screenshot%20from%202026-06-22%2012-00-58.png)


The service was configured to execute a shell script named:

```text
/opt/myapp/start.sh
```

Rather than assuming the file was properly configured, I decided to inspect its permissions directly.

One troubleshooting principle I try to follow is:

**Never trust assumptions. Verify them.**

---

## Step 3: Validate File Permissions

To verify the permissions assigned to the startup script, I executed:

```bash
ls -l /opt/myapp/start.sh
```

The output showed:

```text
-rw-r--r--
```

## Screenshot 4: Verifying Script Permissions

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/df87cabeef996dc9b62eed3fb068031e53f53b82/images/Screenshot%20from%202026-06-22%2012-01-17.png)


The file existed on the server.

However, it did not contain the execute permission bit.

This meant Linux could read the file but was not allowed to execute it as a program.

At this point, multiple pieces of evidence consistently pointed to the same conclusion.

---

# Root Cause Analysis

The root cause was that the startup script configured in the systemd service existed on the server but did not have executable permissions.

The service definition contained:

```ini
ExecStart=/opt/myapp/start.sh
```

The file itself was present.

However, its permissions were:

```text
-rw-r--r--
```

which did not include execute permissions.

When systemd attempted to execute the script, Linux denied the operation and terminated the startup process.

As a result, the service failed with:

```text
status=126
```

Although the issue was relatively small, it demonstrates how Linux file permissions directly impact service startup behavior.

Without validating file permissions, it would be easy to spend time troubleshooting unrelated components such as application code, networking, or systemd configuration.

---

# How This Issue Can Occur

This type of issue commonly occurs during:

* Application deployments
* File transfers between servers
* Git checkouts
* Script creation by developers
* Manual file modifications
* CI/CD pipeline changes
* Infrastructure automation updates

A script can exist in the correct location and still fail to execute if the required permissions are missing.

---

# Resolution

After confirming the root cause, I added execute permissions to the startup script.

```bash
chmod +x /opt/myapp/start.sh
```

I then verified the updated permissions.

```bash
ls -l /opt/myapp/start.sh
```

Result:

```text
-rwxr-xr-x
```

## Screenshot 5: Corrected Script Permissions

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/df87cabeef996dc9b62eed3fb068031e53f53b82/images/Screenshot%20from%202026-06-22%2012-03-43.png)


After applying the fix, I reloaded systemd and restarted the service.

```bash
sudo systemctl daemon-reload

sudo systemctl restart myapp
```

## Screenshot 6: Reloading systemd and Restarting Service

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/df87cabeef996dc9b62eed3fb068031e53f53b82/images/Screenshot%20from%202026-06-22%2012-04-03.png)


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

![image alt](https://github.com/tabarak23/cloud-devops-sre-case-studies/blob/df87cabeef996dc9b62eed3fb068031e53f53b82/images/Screenshot%20from%202026-06-22%2012-04-20.png)


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
Observe 126 Error
          ↓
Review Service Logs
          ↓
Inspect Service Configuration
          ↓
Verify Script Permissions
          ↓
Execute Permission Missing
          ↓
Identify Root Cause
          ↓
Apply Permission Fix
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

* Explain what a 126 error means
* Suggest investigation steps
* Recommend useful Linux commands
* Explain Linux permission models
* Help interpret logs

These capabilities can significantly reduce investigation time.

---

# Why AI Still Cannot Replace Engineers

While AI can explain that a 126 error usually indicates a permission problem, it cannot automatically determine the exact root cause within a production environment.

An engineer must still:

* Collect evidence
* Validate assumptions
* Understand Linux permissions
* Correlate logs and configuration
* Distinguish symptoms from root causes
* Verify recovery after remediation

In this incident, AI could explain that the error was permission-related.

However, an engineer still had to:

* Review the service configuration
* Verify that the script existed
* Inspect file permissions
* Confirm that execute permissions were missing
* Apply the correct fix
* Validate service recovery

The real value comes from investigation and judgment, not simply knowing what an error code means.

---

# Key Takeaways

* Error codes should be treated as clues, not conclusions.
* Always verify file permissions when troubleshooting execution failures.
* Logs should be reviewed before modifying configurations.
* Root cause analysis is more valuable than trial-and-error troubleshooting.
* AI can assist investigations, but engineers are still responsible for validating evidence and implementing the correct solution.
* Always verify service health after remediation.

---

# Skills Demonstrated

* Linux Administration
* Linux File Permissions
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

This incident involved a simple permission issue, but it reinforces an important lesson about troubleshooting Linux services.

The real value was not adding execute permissions to a file.

The value was understanding how Linux permissions affect service execution, using logs to guide the investigation, validating assumptions through evidence, and confirming recovery after remediation.

Whether working as a Linux Administrator, Cloud Support Engineer, DevOps Engineer, or SRE, the ability to investigate problems methodically remains one of the most valuable engineering skills.

Successful troubleshooting is not about finding the fastest answer.

It is about finding the correct answer based on evidence, understanding, and root cause analysis.
