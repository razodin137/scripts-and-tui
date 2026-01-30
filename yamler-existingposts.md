#!/bin/bash
source_directory="/home/mrjohn/Desktop/yamlertest-input"
output_directory="/home/mrjohn/Desktop/yamlertest-output" 

# Function to process a single file
process_file() {
    local file="$1"
    local filename=$(basename "$file")
    
    # Get file's modification date and time with timezone
    mod_date=$(stat -c %y "$file" | cut -d' ' -f1)  # YYYY-MM-DD format
    mod_datetime=$(date -r "$file" +"%Y-%m-%dT%H:%M:%S%z" | sed 's/\([0-9]\{2\}\)$/:\1/') # For the 'date:' field
    
    # Get current date and time for lastmod
    current_datetime=$(date +"%Y-%m-%dT%H:%M:%S%z" | sed 's/\([0-9]\{2\}\)$/:\1/')
    
    # Get original name without extension for title formatting
    name_without_extension="${filename%%.*}"
    
    # Format title: replace dashes/underscores with spaces and capitalize words
    formatted_title=$(echo "$name_without_extension" | 
                     sed -e 's/[-_]/ /g' -e 's/\b\(.\)/\u\1/g')
    
    # Generate clean slug
    slug=$(echo "$name_without_extension" | 
           tr '[:upper:]' '[:lower:]' | 
           tr -cd '[:alnum:] -' | 
           tr ' ' '-' | 
           sed 's/-\+/-/g')  # Replace multiple dashes with single dash
    
    # New filename with modification date prefix
    new_filename="${mod_date}-${slug}.md"
    new_filepath="$output_directory/$new_filename"
    
    # Create temporary file
    temp_file=$(mktemp)
    
    # Create YAML frontmatter and append original content directly
    cat <<EOL > "$temp_file"
---
title: "$formatted_title"
date: $mod_datetime
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

EOL

    # Simply append the original content
    cat "$file" >> "$temp_file"
    
    # Move temporary file to new location
    mv "$temp_file" "$new_filepath"
    
    echo "Processed: $filename → $new_filename"
}

# Validate and create directories
if [ ! -d "$source_directory" ]; then
    echo "Error: Source directory does not exist!"
    exit 1
fi

mkdir -p "$output_directory"

# Process all files in the directory
find "$source_directory" -type f | while read -r file; do
    process_file "$file"
done

echo "Processing complete! Enriched files are in: $output_directory"
