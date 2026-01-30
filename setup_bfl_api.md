#!/bin/bash

# Define base directory
BASE_DIR="$HOME/Desktop/bfl-api"
REQUEST_SCRIPT="$BASE_DIR/create_request.sh"
POLL_SCRIPT="$BASE_DIR/poll_result.sh"
PAYLOAD_FILE="$BASE_DIR/request_payload.json"

# Create the directory structure
mkdir -p "$BASE_DIR"

# Create the payload file
cat <<EOF > "$PAYLOAD_FILE"
{
  "prompt": "A cat on its back legs running like a human is holding a big silver fish with its arms. The cat is running away from the shop owner and has a panicked look on his face. The scene is situated in a crowded market.",
  "width": 1024,
  "height": 768
}
EOF

# Create the create_request.sh script
cat <<'EOF' > "$REQUEST_SCRIPT"
#!/bin/bash

API_ENDPOINT="https://api.bfl.ml/v1/flux-pro-1.1"
API_KEY="12345-ABCDE-67890"  # Replace this with your actual API key
PAYLOAD_FILE="request_payload.json"

if [ ! -f "$PAYLOAD_FILE" ]; then
  echo "Error: $PAYLOAD_FILE not found!"
  exit 1
fi

# Make the POST request
RESPONSE=$(curl -s "$API_ENDPOINT" \
  --request POST \
  --header "accept: application/json" \
  --header "x-key: $API_KEY" \
  --header "Content-Type: application/json" \
  --data @"$PAYLOAD_FILE")

# Extract the request ID
REQUEST_ID=$(echo "$RESPONSE" | jq -r '.id')

# Check if request ID was received
if [ -z "$REQUEST_ID" ] || [ "$REQUEST_ID" == "null" ]; then
  echo "Error: Failed to create request. Response: $RESPONSE"
  exit 1
fi

echo "Request created successfully. Request ID: $REQUEST_ID"
echo "$REQUEST_ID" > request_id.txt  # Save the request ID for polling
EOF

# Create the poll_result.sh script
cat <<'EOF' > "$POLL_SCRIPT"
#!/bin/bash

API_ENDPOINT="https://api.bfl.ml/v1/get_result"
API_KEY="12345-ABCDE-67890"  # Replace this with your actual API key
REQUEST_ID=$(cat request_id.txt)

if [ -z "$REQUEST_ID" ]; then
  echo "Error: Request ID not found. Run create_request.sh first."
  exit 1
fi

# Poll the API for the result
while true; do
  RESPONSE=$(curl -s "$API_ENDPOINT" \
    --request GET \
    --header "accept: application/json" \
    --header "x-key: $API_KEY" \
    --get --data-urlencode "id=$REQUEST_ID")

  STATUS=$(echo "$RESPONSE" | jq -r '.status')

  if [ "$STATUS" == "Ready" ]; then
    RESULT_URL=$(echo "$RESPONSE" | jq -r '.result.sample')
    echo "Result is ready: $RESULT_URL"
    echo "Downloading the image..."
    curl -s -o generated_image.jpeg "$RESULT_URL"
    echo "Image downloaded as generated_image.jpeg"
    break
  else
    echo "Status: $STATUS"
  fi

  sleep 0.5  # Wait before polling again
done
EOF

# Make the scripts executable
chmod +x "$REQUEST_SCRIPT" "$POLL_SCRIPT"

# Display setup information
echo "Setup completed!"
echo "Navigate to the directory $BASE_DIR and run the following commands:"
echo "1. Create a request: ./create_request.sh"
echo "2. Poll for result: ./poll_result.sh"
echo "All files are saved in $BASE_DIR."

