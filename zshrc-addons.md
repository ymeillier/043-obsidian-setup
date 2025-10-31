
```bash

# Obsidian

##alias obsidian='open -a "Obsidian"'
#open -n -a "Obsidian" "$vault_path"


#For permanent vault access or creation.
#Just type obsidian while in your vault directory.IT will either create a new vault or open the existing one.


function obsidian() {
    # --- Dependencies Check ---
    if ! command -v jq &> /dev/null; then
        echo "Error: 'jq' is required for JSON manipulation. Aborting."
        return 1
    fi

    # --- Configuration ---
    local vault_path="$PWD"
    local config_file="$HOME/Library/Application Support/obsidian/obsidian.json"
    # Define your template source path
    local TEMPLATE_CONFIG_PATH="/Users/meillier/Documents/Obsidian/00-Template/.obsidian"
    
    # Check registration (Idempotence)
    local registered_vault_id=$(jq -r --arg path "$vault_path" \
        '.vaults | to_entries[] | select(.value.path == $path) | .key' \
        "$config_file" 2>/dev/null)

    # 1. Registration Block
    if [[ -z "$registered_vault_id" ]]; then
        
        # --- Prompt and Quit Check ---
        echo "A new permanent vault must be registered. This requires closing all open Obsidian windows."
        read -r -p "Is it OK to close all existing Obsidian windows? (y/N) " response
        
        if [[ ! "$response" =~ ^([yY][eE][sS]|[yY])$ ]]; then
            echo "❌ CANCELED: Please create the new vault manually using the Vault Switcher."
            return 1
        fi
        
        # Quit Obsidian (Crucial for forcing config reload) 🛑
        osascript -e 'quit app "Obsidian"' 2>/dev/null
        echo "Closing running Obsidian instances..."
        sleep 2
        
        # 2. Prompt for Vault Name and Generate Metadata
        read -p "Enter a name for the new permanent vault: " vault_name
        if [[ -z "$vault_name" ]]; then
            echo "Vault name cannot be empty. Aborting."
            return 1
        fi
        
        # Generate ID and Timestamp
        local vault_id=$(openssl rand -hex 8)
        local timestamp_ms=$(perl -MTime::HiRes -e 'printf "%.0f\n", Time::HiRes::time * 1000')

        # 3. CRITICAL FIX: Copy Template Configuration 
        if [ ! -d "$vault_path/.obsidian" ]; then
            if [ -d "$TEMPLATE_CONFIG_PATH" ]; then
                # Copy the entire .obsidian contents from the template
                cp -r "$TEMPLATE_CONFIG_PATH" "$vault_path/"
                echo "✅ Copied template configuration to new vault."
            else
                # Fallback: create an empty directory if the template is not found
                mkdir -p "$vault_path/.obsidian"
                echo "⚠️ Template configuration not found. Created empty .obsidian folder."
            fi
        fi

        # 4. Update Global Config JSON
        if [ ! -f "$config_file" ]; then
            mkdir -p "$(dirname "$config_file")"
            echo '{"vaults":{}}' > "$config_file"
        fi
        
        local jq_script='.vaults += {
            ($id): {
                "path": $path,
                "ts": ($ts | tonumber),
                "open": true
            }
        } | .lastOpenVault = $id'

        jq --arg id "$vault_id" \
           --arg path "$vault_path" \
           --arg ts "$timestamp_ms" \
           "$jq_script" \
           "$config_file" > "$config_file.tmp" && mv "$config_file.tmp" "$config_file"

    else
        echo "Vault already registered (ID: $registered_vault_id). Launching directly..."
        vault_id="$registered_vault_id" # Use existing ID for launch
    fi
    
    # 5. Launch Obsidian using the Vault ID in the URI
    
    local encoded_id=$(perl -MURI::Escape -e 'print uri_escape($ARGV[0])' "$vault_id")
    
    echo "Launching permanent vault..."
    open "obsidian://open?vault=$encoded_id"
}









#Function for opening a current folder as a temp vault that always get overwritten for each new ephemeral inspection.

#Type: md_open ./file.md 
# to open a new temp vault for viewing/editing the readme file. 
# Run md_cleanup $PWD to remove references of this vault registration (so that don't have a larege number of registered temp vaults)


function md_open() {
    # 1. Check for required file argument
    if [ -z "$1" ]; then
        echo "Error: You must provide a file path (e.g., README.md)."
        echo "Usage: md_open <file_path>"
        return 1
    fi
    local source_file="$(realpath "$1")"
    
    # --- Configuration and Dependencies ---
    local vault_dir="$(dirname "$source_file")"
    local config_file="$HOME/Library/Application Support/obsidian/obsidian.json"
    local TEMPLATE_CONFIG_PATH="/Users/meillier/Documents/Obsidian/00-Template/.obsidian"
    
    if ! command -v jq &> /dev/null; then echo "Error: 'jq' required. Aborting."; return 1; fi
    if ! perl -MURI::Escape -e '1' 2>/dev/null; then echo "Error: perl module URI::Escape required. Aborting."; return 1; fi

    # 2. Check if the directory is already registered (Idempotence)
    local vault_id=$(jq -r --arg path "$vault_dir" \
        '.vaults | to_entries[] | select(.value.path == $path) | .key' \
        "$config_file" 2>/dev/null)

    # 3. Registration (Modified to use the fixed ID prefix)
    if [[ -z "$vault_id" ]]; then
        # Check for existing .obsidian folder (don't overwrite a persistent vault)
        if [ -d "$vault_dir/.obsidian" ]; then
             echo "Vault config already exists in the repo. Launching directly."
        else
            echo "Creating temporary vault configuration in: $vault_dir"
            
            # --- CRITICAL CHANGE FOR TEMPLATE COPY ---
            if [ -d "$TEMPLATE_CONFIG_PATH" ]; then
                # Copy the entire .obsidian contents from the template
                cp -r "$TEMPLATE_CONFIG_PATH" "$vault_dir/"
                echo "✅ Copied template configuration to new vault."
            else
                # Fallback: create an empty directory if the template is not found
                mkdir -p "$vault_dir/.obsidian"
                echo "⚠️ Template configuration not found. Created empty .obsidian folder."
            fi
            # --- END CRITICAL CHANGE ---
            
            # Generate new metadata
            local vault_name="$(basename "$vault_dir")-TEMP"
            local random_suffix=$(openssl rand -hex 1) # Gets 2 hex characters
            vault_id="99999999999999${random_suffix}" # Unique ID with cleanup marker
            local timestamp_ms=$(perl -MTime::HiRes -e 'printf "%.0f\n", Time::HiRes::time * 1000')
            
            # Ensure config file exists
            if [ ! -f "$config_file" ]; then mkdir -p "$(dirname "$config_file")"; echo '{"vaults":{}}' > "$config_file"; fi
            
            # Register in obsidian.json
            local jq_script='.vaults += { ($id): { "path": $path, "ts": ($ts | tonumber), "open": true } } | .lastOpenVault = $id'
            jq --arg id "$vault_id" --arg path "$vault_dir" --arg ts "$timestamp_ms" "$jq_script" "$config_file" > "$config_file.tmp" && mv "$config_file.tmp" "$config_file"

            # Quit Obsidian (Required for config change)
            osascript -e 'quit app "Obsidian"' 2>/dev/null
            echo "Closing running Obsidian instances to finalize registration..."
            sleep 2
        fi
    fi

    # 4. Launch the specific file in the new repo-vault
    local file_name="$(basename "$source_file")"
    local encoded_id=$(perl -MURI::Escape -e 'print uri_escape($ARGV[0])' "$vault_id")
    local encoded_file=$(perl -MURI::Escape -e 'print uri_escape($ARGV[0])' "$file_name")
    
    echo "Launching file in ephemeral vault: $vault_dir"
    open "obsidian://open?vault=$encoded_id&file=$encoded_file"
    
    echo ""
    echo "🚨 ACTION REQUIRED: When done, manually run: md_cleanup '$vault_dir'"
}






function md_cleanup() {
    # Check for required argument
    if [ -z "$1" ]; then
        echo "Usage: cleanup_obsidian_vault <path_to_repo>"
        return 1
    fi
    
    local vault_dir="$(realpath "$1")"
    local config_file="$HOME/Library/Application Support/obsidian/obsidian.json"
    
    # 1. Look up the Vault ID based on the directory path
    local vault_id_to_remove=$(jq -r --arg path "$vault_dir" \
        '.vaults | to_entries[] | select(.value.path == $path) | .key' \
        "$config_file" 2>/dev/null)

    # 2. File System Cleanup (Removes the marker)
    if [ -d "$vault_dir/.obsidian" ]; then
        rm -rf "$vault_dir/.obsidian"
        echo "✅ File system clean: Removed temporary .obsidian folder from $vault_dir"
    else
        echo "ℹ️ Note: .obsidian folder not found, skipping file system cleanup."
    fi
    
    # 3. JSON Configuration Cleanup (Removes the reference)
    if [[ -n "$vault_id_to_remove" ]]; then
        # Use jq to delete the key associated with the vault ID
        jq "del(.vaults[\"$vault_id_to_remove\"])" \
            "$config_file" > "$config_file.tmp" && mv "$config_file.tmp" "$config_file"
            
        echo "✅ Config clean: Removed vault ID ($vault_id_to_remove) from obsidian.json."
        
        # 4. Quit Obsidian (To force immediate config reload and cleanup of the open window)
        osascript -e 'quit app "Obsidian"' 2>/dev/null
        echo "ℹ️ Obsidian closed to finalize cleanup."
    else
        echo "ℹ️ Vault entry not found in obsidian.json. No config cleanup needed."
    fi
}














function md_cleanup_all() {
    local config_file="$HOME/Library/Application Support/obsidian/obsidian.json"
    local TEMP_ID_PREFIX="99999999999999"

    if ! command -v jq &> /dev/null; then
        echo "Error: 'jq' is required for JSON manipulation. Aborting."
        return 1
    fi

    echo "--- Initiating Cleanup of Ephemeral Vaults ($TEMP_ID_PREFIX*) ---"
    
    # Use jq to select only the vaults whose keys match the temporary prefix
    local temp_vault_data=$(jq -r --arg prefix "$TEMP_ID_PREFIX" \
        '.vaults | to_entries[] | select(.key | startswith($prefix)) | "\(.value.path),\(.key)"' \
        "$config_file" 2>/dev/null)

    if [ -z "$temp_vault_data" ]; then
        echo "✅ No ephemeral vaults found requiring cleanup."
        return 0
    fi

    echo "$temp_vault_data" | while IFS=, read -r vault_dir vault_id_to_remove; do
        
        echo "Processing Vault: $(basename "$vault_dir") (ID: $vault_id_to_remove)"
        
        # 1. File System Cleanup (Removes the marker)
        if [ -d "$vault_dir/.obsidian" ]; then
            rm -rf "$vault_dir/.obsidian"
            echo "  - ✅ Removed temporary .obsidian folder."
        else
            echo "  - ℹ️ .obsidian folder not found (already cleaned)."
        fi
        
        # 2. JSON Configuration Cleanup (Deletes the entry by ID)
        jq "del(.vaults[\"$vault_id_to_remove\"])" \
            "$config_file" > "$config_file.tmp" && mv "$config_file.tmp" "$config_file"
            
        echo "  - ✅ Removed ID from obsidian.json."
    done

    # 3. Quit Obsidian (To force immediate config reload)
    osascript -e 'quit app "Obsidian"' 2>/dev/null
    echo ""
    echo "✨ Complete. All identified ephemeral vaults have been cleaned up and Obsidian closed."
}















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