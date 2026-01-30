#!/bin/bash

# Define the specific directory
directory="/media/mrjohn/cinnamonpartext4/mrjohn/inbox"

# Prompt the user for a filename
read -p "Enter the filename (e.g., my-new-post.md): " filename

full_path="$directory/$filename"

# Remove file extension from filename
name_without_extension="${filename%%.*}"

# Format the title: replace dashes with spaces and capitalize each word
formatted_title=$(echo "$name_without_extension" | sed -e 's/-/ /g' -e 's/\b\(.\)/\u\1/g')

# Get current date and time in the format YYYY-MM-DDTHH:MM:SS-XX:XX
current_datetime=$(date +"%Y-%m-%dT%H:%M:%S%z")

# Get last modification date in the same format as current date
last_modification_date=$(date +"%Y-%m-%dT%H:%M:%S%z")

# Get current date in YYYY-MM-DD format
current_date=$(date +"%Y-%m-%d")

# Generate slug from the filename (using lowercase and dashes)
slug=$(echo "$name_without_extension" | tr '[:upper:]' '[:lower:]' | sed 's/ /-/g')

# Create the YAML frontmatter
cat <<EOL > "$full_path"
---
title: "$formatted_title"
date: $current_datetime
lastmod: $current_datetime
slug: "$slug"
aliases:
  - "/$slug"
  - "$formatted_title"
type: post
draft: true
categories:
  - Uncategorized
tags: []
---

$current_date

# 

EOL

echo "File $full_path created with the following frontmatter:"
cat "$full_path"

# Open the file in nano editor
xed "$full_path"

