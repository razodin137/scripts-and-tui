#!/bin/bash

DOWNLOADS_FOLDER="$HOME/Downloads"

declare -A CATEGORIES
CATEGORIES=(
    ["Images"]="jpg jpeg png gif bmp tiff"
    ["Documents"]="pdf doc docx txt ppt pptx xls xlsx"
    ["Videos"]="mp4 avi mov mkv"
    ["Music"]="mp3 wav aac flac"
    ["Archives"]="zip rar 7z tar gz"
)

get_category() {
    local extension="${1,,}"
    for category in "${!CATEGORIES[@]}"; do
        if [[ "${CATEGORIES[$category]}" =~ (^|[[:space:]])"$extension"($|[[:space:]]) ]]; then
            echo "$category"
            return
        fi
    done
    echo "Uncategorized"
}

while true; do
    for new_file in "$DOWNLOADS_FOLDER"/*; do
        [ -f "$new_file" ] || continue  # Skip if not a file
        extension="${new_file##*.}"
        category=$(get_category "$extension")

        if [[ "$category" != "Uncategorized" ]]; then
            target_folder="$DOWNLOADS_FOLDER/$category"
            mkdir -p "$target_folder"
            mv "$new_file" "$target_folder/"
            echo "Moved $new_file to $target_folder"
        fi
    done
    sleep 60  # Check every 60 seconds
done

