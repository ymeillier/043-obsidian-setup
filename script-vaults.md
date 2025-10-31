```bash
#!/bin/bash

## Function to list vaults
function vaults() {
    local config_file="$HOME/Library/Application Support/obsidian/obsidian.json"
    
    # Check if the config file exists
    if [ ! -f "$config_file" ]; then
        echo "Error: Obsidian configuration file not found at $config_file"
        return 1
    fi

    # Check if jq is available
    if ! command -v jq &> /dev/null; then
        echo "Error: 'jq' command not found. Please ensure it is installed."
        return 1
    fi

    echo "--- Obsidian Vaults ---"
    
    # The existing jq logic is wrapped in the function
    cat "$config_file" | jq -r '.vaults | to_entries[] | 
      .value.ts = (.value.ts / 1000 | todate) |
      "ID: \(.key) | Path: \(.value.path) | Last Opened: \(.value.ts) | Open: \(.value.open // false)"' |
      column -t -s '|'
}



```