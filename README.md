# Identifying-and-removing-access-for-inactive-users-in-Github
This document explains an automated process for identifying and removing access for inactive users from a Github organization using Github Actions. This process helps maintain organizational security and efficiency by ensuring that only active users have access to sensitive resources. [1]



Prerequisites:

Admin access

Personal Access Token

Github CLI



When determining if a user is active or inactive in GitHub, the following activities are considered: [2]

 Repository interactions: Creating, pushing to, or deleting repositories. 

Issue and pull request management: Creating, commenting on, closing, or reopening issues and pull requests. 

Comments and reviews: Leaving comments on commits or pull requests. 

Repository visibility changes: Changing a repository's visibility (public, private). 

User interactions: Watching or starring repositories.

Reading repositories is not counted as an activity since the GitHub Enterprise plan requires the Audit log API. [3]

**Workflow Steps**

**Authentication (Step 0)**

Authenticates with GitHub using a provided access token.

- name: Authenticate GitHub CLI
        run: echo "${{ secrets.ORGANIZATION_ACCESS_TOKEN }}" | gh auth login --with-token

**Variable Initialization (Step 1)**

Sets up the inactivity threshold (default 31 days).

 - name: Initialize Variables
        run: |
          echo "Initializing variables..."
          echo "Inactivity threshold set to ${{ env.INACTIVITY_THRESHOLD_DAYS }} days."

**Repository Checkout (Step 2)**

Checks out the repository to access any necessary files.

 - name: Checkout Repository
        uses: actions/checkout@v4

**Tool Installation (Step 3)**

Installs jq for JSON processing.

- name: Install jq
        run: sudo apt-get install jq -y

**Fetch Organization Members (Step 4)**

Retrieves a list of all members in the organization.

Handles pagination to ensure all members are fetched.

Saves member data to a JSON file.

- name: Fetch Organization Members
        run: |
          echo "Fetching all organization members..."
          page=1
          members_file="org_members.json"
          echo "[]" > $members_file  # Initialize an empty JSON array

          while true; do
            response=$(curl -H "Authorization: Bearer ${{ secrets.ORGANIZATION_ACCESS_TOKEN }}" \
              "https://api.github.com/orgs/virufy/members?per_page=100&page=$page")
            
            # Check if response is empty (no more pages)
            if [ "$(echo "$response" | jq '. | length')" -eq 0 ]; then
              break
            fi

            # Append current page results to the JSON file
            jq -s '.[0] + .[1]' $members_file <(echo "$response") > tmp.json && mv tmp.json $members_file
            
            # Increment page number
            page=$((page + 1))
          done

          echo "Organization members saved to $members_file"

**Generate All Users Report (Step 5)**

Creates a report of all users with their last activity date.

Fetches recent events for each user to determine last activity.

- name: Display All Users with Last Activity Time
        run: |
          echo "Fetching user activity and displaying last active times..."
          
          echo "All Users:" > all_users.txt

          # Process each user in the activity report
          jq -r '.[].login' org_members.json | while read username; do
            
            echo "Fetching events for $username..."
            events=$(curl -H "Authorization: Bearer ${{ secrets.ORGANIZATION_ACCESS_TOKEN }}" \
              "https://api.github.com/users/$username/events")

            last_active=$(echo "$events" | jq -r '.[0].created_at // null')

            if [ "$last_active" != "null" ]; then
              echo "- $username (Last active: $last_active)" >> all_users.txt
            else
              echo "- $username (Last active: Never)" >> all_users.txt
            fi

          done

          cat all_users.txt

**Identify Inactive Users (Step 6)**

Determines which users are inactive based on the threshold.

Handles excluded users to prevent them from being marked as inactive.

Creates a report of inactive users.

- name: Identify Inactive Users Based on Threshold
        run: |
          echo "Identifying inactive users based on threshold (${{ env.INACTIVITY_THRESHOLD_DAYS }} days)..."
          
          # Get current timestamp in UTC
          current_timestamp=$(date -u +%s)
          threshold_seconds=$(( ${{ env.INACTIVITY_THRESHOLD_DAYS }} * 24 * 3600 ))
          threshold_date=$(( current_timestamp - threshold_seconds ))

          echo "Inactive Users (Threshold: $(date -u -d @$threshold_date '+%Y-%m-%dT%H:%M:%SZ')):" > inactive_users.txt
          
          # Read and clean excluded users
          IFS=',' read -ra raw_excluded <<< "${{ env.EXCLUDED_USERS }}"
          declare -a excluded_users
          for user in "${raw_excluded[@]}"; do
            cleaned_user=$(echo "$user" | xargs | tr '[:upper:]' '[:lower:]')  # Trim + lowercase
            excluded_users+=("$cleaned_user")
          done

          jq -r '.[].login' org_members.json | while read username; do
            # Normalize username to lowercase
            normalized_username=$(echo "$username" | tr '[:upper:]' '[:lower:]')
            
            echo "Fetching events for $username..."
            events=$(curl -H "Authorization: Bearer ${{ secrets.ORGANIZATION_ACCESS_TOKEN }}" \
              "https://api.github.com/users/$username/events")

            last_active=$(echo "$events" | jq -r '.[0].created_at // null')

            if [ "$last_active" != "null" ]; then
              # Convert GitHub timestamp to UTC epoch
              last_active_timestamp=$(date -u -d "$last_active" +%s)
              # Calculate difference in seconds
              time_diff=$(( current_timestamp - last_active_timestamp ))
              is_inactive=$(( time_diff > threshold_seconds ))
            else
              is_inactive=1  # Consider inactive if no activity
            fi

            if [ "$is_inactive" -eq 1 ]; then
              # Exact match check with cleaned data
              if [[ ! " ${excluded_users[*]} " =~ " $normalized_username " ]]; then
                echo "- $username (Last active: ${last_active:-Never})" >> inactive_users.txt
              else
                echo "- Skipping excluded user: $username"
              fi
            fi
          done

          cat inactive_users.txt

**Upload Reports as Artifacts (Step 7)**

Uploads both the all-users and inactive-users reports as workflow artifacts.

- name: Upload User Lists as Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: all-user-list-report
          path: all_users.txt
      
      - name: Upload Inactive User List Report as Artifact
        uses: actions/upload-artifact@v4
        with:
          name: inactive-user-list-report
          path: inactive_users.txt



**Reference:**

https://github.com/peter-murray/inactive-users-action/blob/main/README.md about inactive users action using Github Action.

https://github.com/oss-tooling/user-inactivity  about automation which helps monitor and remove inactive members from a Github organization.

https://docs.github.com/en/enterprise-cloud@latest/admin/monitoring-activity-in-your-enterprise/reviewing-audit-logs-for-your-enterprise/using-the-audit-log-api-for-your-enterprise about GitHub Enterprise plan to get audit API.

