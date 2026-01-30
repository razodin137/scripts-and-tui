#!/bin/bash

# Prompt for source directory
read -p "Enter the source directory path: " SRC_DIR

# Prompt for destination directory
read -p "Enter the destination directory path: " DEST_DIR

# Ensure the destination directory exists
mkdir -p "$DEST_DIR"

# Function to process a single file
process_file() {
    local src_file="$1"
    local dest_file="$2"
    
    # Create destination directory if it doesn't exist
    mkdir -p "$(dirname "$dest_file")"
    
    # Get the filename without extension
    filename=$(basename "$src_file")
    name_without_extension="${filename%.*}"
    
    # Format the title: replace dashes and underscores with spaces and capitalize each word
    formatted_title=$(echo "$name_without_extension" | sed -e 's/[-_]/ /g' -e 's/\b\(.\)/\u\1/g')
    
    # Get creation date (birthtime)
    creation_date=$(stat -c %w "$src_file" 2>/dev/null || stat -f "%SB" -t "%Y-%m-%dT%H:%M:%S%z" "$src_file")
    
    # Get last modification date
    last_modification_date=$(date -r "$src_file" +"%Y-%m-%dT%H:%M:%S%z")
    
    # Generate slug
    slug=$(echo "$name_without_extension" | tr '[:upper:]' '[:lower:]' | sed 's/ /-/g')
    
    # Create the YAML frontmatter
    {
        echo "---"
        echo "title: $formatted_title"
        echo "draft: true"
        echo "date: $creation_date"
        echo "lastmod: $last_modification_date"
        echo "tags: []"
        echo "aliases:"
        echo "  - \"/$slug\""
        echo "---"
        echo
        cat "$src_file"
    } > "$dest_file"
    
    echo "Processed: $src_file -> $dest_file"
}

# Initialize counters
total_files=0
processed_files=0
error_files=0

# Main loop to process all .md files
while IFS= read -r -d '' file; do
    ((total_files++))
    # Construct the destination file path
    dest_file="${file/$SRC_DIR/$DEST_DIR}"
    if process_file "$file" "$dest_file"; then
        ((processed_files++))
    else
        ((error_files++))
    fi
done < <(find "$SRC_DIR" -type f -name "*.md" -print0)

# Final report
echo
echo "Processing complete."
echo "Total files found: $total_files"
echo "Files processed successfully: $processed_files"
echo "Files with errors: $error_files"

# Keep the terminal open
read -p "Press Enter to close this window..."
