# Web Automation Guide: Core Concepts & Approach

## Overview
This guide explains how the Oracle CPU tracker works and teaches you the fundamental concepts for building similar automation scripts.

---

## The Core Problem

**Goal:** Automatically detect when Oracle releases new security patches without manually checking their website.

**Solution Pattern:** Compare current website state vs. previously stored state → Alert if changed

---

## Key Concepts

### 1. **Web Scraping Basics**

**What it is:** Automatically downloading and parsing web pages to extract information.

```powershell
# Fetch a web page
$response = Invoke-WebRequest -Uri "https://example.com" -UseBasicParsing

# The response contains:
# - $response.Content    = Raw HTML text
# - $response.StatusCode = HTTP status (200 = success)
# - $response.Links      = All hyperlinks on the page
```

**Key parameters:**
- `-UseBasicParsing` = Faster, doesn't require IE engine
- `-TimeoutSec 30` = Fail if page doesn't load in 30 seconds

---

### 2. **Regular Expressions (Regex)**

**What it is:** Pattern matching to find specific text in large documents.

#### Example from our script:
```powershell
$pattern = 'Critical\s+Patch\s+Update\s+-\s+(January|April|July|October)\s+(\d{4})'
```

**Breaking it down:**
- `Critical\s+Patch\s+Update` = Literal text with `\s+` (one or more spaces)
- `\s+-\s+` = Spaces, then hyphen, then spaces
- `(January|April|July|October)` = Match ANY of these months
- `\s+` = Space(s)
- `(\d{4})` = Exactly 4 digits (the year)

**Finding all matches:**
```powershell
$allMatches = [regex]::Matches($content, $pattern)

# Access them:
$allMatches[0].Value  # First match
$allMatches.Count     # How many found
```

#### Common regex patterns:
- `\d` = Any digit (0-9)
- `\d{4}` = Exactly 4 digits
- `\w` = Any word character (a-z, A-Z, 0-9, _)
- `\s` = Any whitespace (space, tab, newline)
- `+` = One or more of previous
- `*` = Zero or more of previous
- `(option1|option2)` = Either option1 OR option2
- `.` = Any character
- `\.` = Literal dot (escape special characters with \)

**Pro tip:** Test regex patterns at [regex101.com](https://regex101.com)

---

### 3. **State Management**

**The Pattern:**
1. **Read** previous state from file
2. **Get** current state from website
3. **Compare** them
4. **Update** state file if changed
5. **Alert** user if changed

```powershell
# Read previous state
if (Test-Path $StateFile) {
    $lastCPU = Get-Content $StateFile -Raw
}

# Get current state
$currentCPU = Get-OracleCPUInfo

# Compare
if ($currentCPU -ne $lastCPU.Trim()) {
    # State changed! Alert user
    Show-Alert
    
    # Update state file
    $currentCPU | Out-File -FilePath $StateFile -Force
}
```

**Why this works:**
- Simple text file stores last known value
- Script only alerts when value changes
- No database needed for simple tracking

---

### 4. **Logging**

**Why log everything:**
- Debugging when things break
- Audit trail of what happened
- Proof the script is running

```powershell
function Write-Log {
    param($Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$timestamp - $Message" | Out-File -FilePath $LogFile -Append
    Write-Host $Message  # Also show in console
}
```

**Best practices:**
- Log when script starts/ends
- Log every major action
- Log errors with details
- Include timestamps

---

### 5. **Windows Notifications**

```powershell
# Create a toast notification
Add-Type -AssemblyName System.Windows.Forms

$notification = New-Object System.Windows.Forms.NotifyIcon
$notification.Icon = [System.Drawing.SystemIcons]::Information
$notification.BalloonTipTitle = "Title Here"
$notification.BalloonTipText = "Message Here"
$notification.Visible = $true
$notification.ShowBalloonTip(30000)  # Show for 30 seconds
```

---

## How I Approached This Problem

### Step 1: Understand the Data Source
- Visit the Oracle security page
- Identify what information I need: "Critical Patch Update - Month Year"
- Note where it appears (top of table, first entry)
- Observe the pattern/format

### Step 2: Extract the Data
1. Download the page with `Invoke-WebRequest`
2. Build a regex pattern that matches the format
3. Test to ensure it finds the right information

### Step 3: Store & Compare
1. Save the current value to a file
2. Next run: compare new value vs. saved value
3. Alert if different

### Step 4: Make it User-Friendly
1. Add logging for troubleshooting
2. Add notifications so you don't miss alerts
3. Create formatted alert files

### Step 5: Automate
Schedule with Task Scheduler to run weekly

---

## Building Your Own Web Automation

### Template Approach:

```powershell
# 1. Configuration
$TargetURL = "https://example.com/updates"
$StateFile = "$PSScriptRoot\state.txt"
$LogFile = "$PSScriptRoot\log.txt"

# 2. Function to get current data
function Get-CurrentData {
    $response = Invoke-WebRequest -Uri $TargetURL -UseBasicParsing
    
    # Extract what you need using regex
    if ($response.Content -match 'YourPattern(\w+)') {
        return $matches[1]
    }
    return $null
}

# 3. Function to log
function Write-Log {
    param($Message)
    "$((Get-Date)) - $Message" | Out-File -FilePath $LogFile -Append
}

# 4. Main logic
$current = Get-CurrentData

if (Test-Path $StateFile) {
    $previous = Get-Content $StateFile
    
    if ($current -ne $previous) {
        Write-Log "CHANGE DETECTED: $previous -> $current"
        # Alert user here
        $current | Out-File $StateFile -Force
    }
} else {
    # First run
    $current | Out-File $StateFile -Force
}
```

---

## Common Use Cases

You can adapt this pattern for:

1. **Monitor software releases:**
   - Check vendor site for new version numbers
   - Alert when new release appears

2. **Track product availability:**
   - Monitor "Out of Stock" → "In Stock"
   - Alert when product becomes available

3. **Monitor price changes:**
   - Check product page daily
   - Alert when price drops

4. **Track job postings:**
   - Monitor careers page
   - Alert when new positions posted

5. **Security advisories:**
   - Check CVE databases
   - Alert on new vulnerabilities in your software

---

## Troubleshooting Tips

### Script finds nothing:
1. Check if website structure changed
2. View the raw HTML: `$response.Content | Out-File test.html`
3. Open test.html and find your target text
4. Adjust regex pattern accordingly

### Script always alerts:
- Check for extra whitespace: use `.Trim()` when comparing
- Check date formats (they might include timestamps)
- Log both old and new values to see differences

### Website blocks you:
Some sites block automated requests. Solutions:
- Add `-UserAgent` to mimic a browser:
  ```powershell
  -UserAgent "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
  ```
- Add delays between requests
- Check site's `robots.txt` for scraping policies

---

## Advanced Concepts to Learn Next

1. **JSON APIs** - Many sites offer JSON endpoints (easier than HTML scraping)
2. **Error Handling** - Try/catch blocks for robust scripts
3. **Rate Limiting** - Don't hammer websites too frequently
4. **Authentication** - Handling login-required pages
5. **Parallel Processing** - Monitor multiple sites simultaneously

---

## Tools for Learning

- **Regex Testing:** [regex101.com](https://regex101.com)
- **PowerShell ISE:** Built into Windows for script development
- **VS Code:** Better editor with PowerShell extension
- **Postman:** Test web requests before scripting

---

## Key Takeaway

**The Universal Pattern:**
1. Fetch data from source (web, file, API)
2. Extract what you need (regex, parsing)
3. Compare with previous state
4. Alert on changes
5. Update state

This pattern applies to 90% of automation tasks!