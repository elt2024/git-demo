# git-demo
#!/bin/bash

# Variables
INACTIVE_DAYS=90  # Number of days of inactivity to flag
OUTPUT_FILE="/tmp/inactive_users_report.txt"
EMAIL="your_email@example.com"

# Header for the report
echo "Inactive User Report - $(date)" > "$OUTPUT_FILE"
echo "---------------------------------------------" >> "$OUTPUT_FILE"

# Find inactive user accounts
echo "Checking for users inactive for more than $INACTIVE_DAYS days..." >> "$OUTPUT_FILE"

# Iterate through system users
while IFS=: read -r username _ uid _ _ home shell; do
    if [[ $uid -ge 1000 && $shell != "/usr/sbin/nologin" && $shell != "/bin/false" ]]; then
        last_login=$(lastlog -u "$username" | awk 'NR==2 {print $4, $5, $6}')
        if [[ "$last_login" == "**Never logged in**" ]]; then
            echo -e "\nUser: $username (UID: $uid)" >> "$OUTPUT_FILE"
            echo "Status: Never logged in" >> "$OUTPUT_FILE"
            echo "Home Directory: $home" >> "$OUTPUT_FILE"
            echo "Shell: $shell" >> "$OUTPUT_FILE"
        else
            # Calculate inactivity
            last_login_epoch=$(date -d "$last_login" +%s 2>/dev/null || echo 0)
            current_epoch=$(date +%s)
            if [[ $(( (current_epoch - last_login_epoch) / 86400 )) -ge $INACTIVE_DAYS ]]; then
                days_inactive=$(( (current_epoch - last_login_epoch) / 86400 ))
                echo -e "\nUser: $username (UID: $uid)" >> "$OUTPUT_FILE"
                echo "Last Login: $last_login ($days_inactive days ago)" >> "$OUTPUT_FILE"
                echo "Home Directory: $home" >> "$OUTPUT_FILE"
                echo "Shell: $shell" >> "$OUTPUT_FILE"
            fi
        fi
    fi
done < /etc/passwd

# Email the report (Optional: Requires a configured MTA like sendmail or postfix)
if [[ -s "$OUTPUT_FILE" ]]; then
    echo "Inactive user report generated. Sending email to $EMAIL..."
    mail -s "Inactive User Report" "$EMAIL" < "$OUTPUT_FILE"
else
    echo "No inactive users found." >> "$OUTPUT_FILE"
fi

# Output results
echo "Report saved to $OUTPUT_FILE"
cat "$OUTPUT_FILE"
How It Works:
User Iteration:
Reads /etc/passwd to get a list of system users.
Filters for users with UID >= 1000 (typical non-system users) and valid login shells.
Inactivity Check:
Uses the lastlog command to check the user's last login.
Flags users who have never logged in or have been inactive for more than INACTIVE_DAYS.
User Details:
Collects the username, UID, last login date, home directory, and shell information.
Report:
Saves the details to a text file.
Optionally sends the report via email using the mail command.
Prerequisites:
Ensure the mail utility is installed and configured for sending emails (sudo apt install mailutils on Debian-based systems).
The script should be executed with appropriate permissions to access user account details (sudo).
Usage:
Save the script as inactive_user_report.sh.
Make it executable:
bash
Copy code
chmod +x inactive_user_report.sh
Run the script:
bash
Copy code
sudo ./inactive_user_report.sh
You can modify the INACTIVE_DAYS and EMAIL variables to suit your needs.






You said:
now using that script send it as a slack message for the admin group
ChatGPT said:
ChatGPT
To send the report as a Slack message, you can integrate the Slack API into the script. This requires a Slack Webhook URL that you can generate for a specific Slack channel (e.g., an admin group). Follow these steps to modify the script to send the report to Slack:

Updated Script: inactive_user_report_slack.sh
bash
Copy code
#!/bin/bash

# Variables
INACTIVE_DAYS=90  # Number of days of inactivity to flag
OUTPUT_FILE="/tmp/inactive_users_report.txt"
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/XXXXXXXXX/YYYYYYYYY/ZZZZZZZZZZZZZZZZZZZZZ"
CHANNEL_NAME="#admin-group"  # Slack channel or group name

# Header for the report
echo "Inactive User Report - $(date)" > "$OUTPUT_FILE"
echo "---------------------------------------------" >> "$OUTPUT_FILE"

# Find inactive user accounts
echo "Checking for users inactive for more than $INACTIVE_DAYS days..." >> "$OUTPUT_FILE"

# Iterate through system users
while IFS=: read -r username _ uid _ _ home shell; do
    if [[ $uid -ge 1000 && $shell != "/usr/sbin/nologin" && $shell != "/bin/false" ]]; then
        last_login=$(lastlog -u "$username" | awk 'NR==2 {print $4, $5, $6}')
        if [[ "$last_login" == "**Never logged in**" ]]; then
            echo -e "\nUser: $username (UID: $uid)" >> "$OUTPUT_FILE"
            echo "Status: Never logged in" >> "$OUTPUT_FILE"
            echo "Home Directory: $home" >> "$OUTPUT_FILE"
            echo "Shell: $shell" >> "$OUTPUT_FILE"
        else
            # Calculate inactivity
            last_login_epoch=$(date -d "$last_login" +%s 2>/dev/null || echo 0)
            current_epoch=$(date +%s)
            if [[ $(( (current_epoch - last_login_epoch) / 86400 )) -ge $INACTIVE_DAYS ]]; then
                days_inactive=$(( (current_epoch - last_login_epoch) / 86400 ))
                echo -e "\nUser: $username (UID: $uid)" >> "$OUTPUT_FILE"
                echo "Last Login: $last_login ($days_inactive days ago)" >> "$OUTPUT_FILE"
                echo "Home Directory: $home" >> "$OUTPUT_FILE"
                echo "Shell: $shell" >> "$OUTPUT_FILE"
            fi
        fi
    fi
done < /etc/passwd

# Convert the report to a single Slack message
if [[ -s "$OUTPUT_FILE" ]]; then
    REPORT_CONTENT=$(cat "$OUTPUT_FILE" | sed ':a;N;$!ba;s/\n/\\n/g')  # Escape newlines for Slack
    PAYLOAD=$(cat <<EOF
{
    "channel": "$CHANNEL_NAME",
    "username": "Inactive User Report Bot",
    "text": "Inactive User Report - $(date):\\n$REPORT_CONTENT"
}
EOF
)
    # Send the message to Slack
    curl -X POST -H 'Content-type: application/json' --data "$PAYLOAD" "$SLACK_WEBHOOK_URL"
    echo "Report sent to Slack channel $CHANNEL_NAME."
else
    echo "No inactive users found. Nothing to report."
fi

# Output results
echo "Report saved to $OUTPUT_FILE"
cat "$OUTPUT_FILE"
