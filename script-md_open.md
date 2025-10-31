```bash
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


```