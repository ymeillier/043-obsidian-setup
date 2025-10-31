```bash
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


```