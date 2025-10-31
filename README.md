
This repo describes the custom bash scripts used to customize our workflow of using Obsidian as our markdown editor.

I personally needed to use Obsidian to open md files without necessarily creating a new Vault every time.

For example if I was to clone the kubernetes-the-hard-way repo, from 
```
https://github.com/kelseyhightower/kubernetes-the-hard-way
```

![[file-20251031161558455.png]]

and would like to review the md files at hte root and within the docs/ folder, 
![[file-20251031161647806.png]]

we would need to open that folder as a vault:
![[file-20251031161809279.png]]
However, this is just one repo and example. When doing so every time we need to open an md file, we will end up with lots of vaults. 



I also  do have a number of Vaults for various projects but want to limit the number of "permanent" vaults managed by Obsidian.  

My first objectives was to create tools to manage temporary vaults.


## vaults()

Existing Vaults are listed in the left pane of the Vault switcher: 
![[file-20251031160555170.png]]
or also from the vault switcher within an already opened Obsidian Vault window:
![[file-20251031160642770.png]]

When vaults are created, Obsidian will assign a unique ID to the Vault and keep track of the existing Vault in its obsidian.json file in 
```
~/Library/Application Support/obsidian/obsidian.json
```

![[file-20251031160754069.png]]

We created a first bash function to list the vaults from the CLI (vaults.sh in repo):

```bash
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

The vaults() function is placed in our ~/.zshrc file so that it can be executed invoking the ```vaults``` command.



## md_open() & md_cleanup()

The next function allows creating a vault from any folder but a vault that will be using a vault ID starting with 14 '9s' so that they can easily be identified as a temporary Vault. 

There is an associated md_cleanup() function used to delete that vault when editing is complete so that we don't keep creating new vaults for each new file we open in Obsidian. 

For example, using the example of cloning another repo which contains a markdown file i'd like to open in obsidian. Or say i have a new md file that craeted in a tmp folder that i would like to edit with obsidian, but not keep as a vault. 

![[file-20251031162324826.png]]

we use the open_md() function to open that directory as vault:
![[file-20251031162534890.png]]

this will open Obsidian:
![[file-20251031162508252.png]]
we edit hte file in obsidian
![[file-20251031162632281.png]]
close the window manually, and validate the md file changes were saved:

![[file-20251031162738276.png]]

if run md_open() again, we are back to being able to edit the file via Obsidian.

The Vault is available in the Vault switcher:
![[file-20251031163020416.png]]

Now if we run vaults(), we see our new vault with its Vault ID starting with 14 9s:
![[file-20251031162911847.png]]

We now run the md_cleanup() function to cleanup the registration of the folder as a Vault in Obsidian and can validate with the vaults() function that our vault no longer exist. 
![[file-20251031163214640.png]]

Only the OBsidian vault was deleted, the files are intact:
![[file-20251031163252506.png]]

## md_cleanup_all()

Sometimes you might forget running the cleanup function and over time you might have a number of vaults that were meant to be temporary. 
We had those being created with the same identifier starting with 14 9s and the md_cleanup_all() function will clean any vault starting with that ID prefix

For example We are actually currently editing this exact md file that you are reading in Obsidian using such a temporary Vault. 
![[file-20251031163518761.png]]

When we run the md_cleanup_all() function, our obsidian window will close and its vault will be deleted:
![[file-20251031173254750.png]]


however we can open our folder as a vault again and resume editing:
![[file-20251031173343313.png]]




## obsidian()

Now the net function we defined is obsidian().

This one is used when wanting to either:
1 - create a new Vault for a folder
2 - Open an existing vault from the cli.

For example, if we navigate to "/Users/meillier/Documents/Obsidian/Google-v2/Google-v2", this is a Vault, we know that because it has the .obsidian directory:
![[file-20251031173410415.png]]

if we just type "obsidian", it will open the vault in another Obsidian window.

We could also create a new Permanent Vault using this same function. The Vault ID will not use the 999x prefix.


## Obsidian Template

One great benefits of Obsidian is its rich ecosystem of community plugins that extend its functionality.

I have a couple favorites that I always want loaded with my Obsidian Vault: Colored Text, Image Toolkit to zoom on screenshots, Quiet Outline ...
![[file-20251031173439703.png]]

Plugin configurations for a Vault are managed via json files in the .obsidian file:
![[file-20251031173505250.png]]

Each plugin code and optionally configurations are saved as json files within each plugin folder:

![[file-20251031173522050.png]]

Each new Vault created will however come up without these plugins since Obsidian leaves it to the user to have its different Vaults configured differently if needed.

We needed a way to ensure that each new Vault created via the utilities we created would also copy the configs of our reference/template Vault.

When creating new vaults, our scripts thus copy those configs.

![[file-20251031173546140.png]]

For example we create a new directory and md file:

and md_open(). It will ask if we trust the author of the vault:
![[file-20251031173608144.png]]



and then we can validate that this new vault comes configured with our plugins:
![[file-20251031173627157.png]]
Note: Plugins are accesses via the settings icon in the bottom left:
![[file-20251031173646206.png]]
One of hte most important configuration of our Template Vault is how screenshots are saved in our Vault.
They should be saved in a subfolder under assets/ named after our .md:

![[file-20251031173706641.png]]This is set by the Files and Links config:
![[file-20251031173732712.png]]

added to a subdirectory of the folder because it was not configured properly:
![[file-20251031173745795.png]]
This is what the customer attachment location plugin is for using these configurations for making sure the screenshots are saved an assets folder 

![[file-20251031173811798.png]]


## md_open error

If you run into this error:
![[file-20251031172243728.png]]

![[file-20251031172253713.png]]



it could be that you removed the vault from the list via the GUI which does not delete the .obsidian folder but cleansup the vault ID from the obsidian.json file queries with vaults()



## .zshrc and .bash_profile

make sure .zshrc is sourced by your bash profile:

```
# Execute the main configuration file for interactive shells
source ~/.zshrc

# If the shell is interactive, run the zshrc file
[[ -r ~/.zshrc ]] && source ~/.zshrc
```




