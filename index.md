# RBENV vs RVM

Despite the claims of either of the two major opponents, the battle for best Ruby version manager still rages on. Both tools have distinctive approaches:

Rbenv allows you to automatically or manually change your Ruby version and leaves Ruby installation and gem management to other tools. The rbenv community respects these alternate tools and values the singular focus of rbenv. Its simplicity enables it to be lightweight and makes debugging easier.

Rvm allows you to automatically or manually change your Ruby version, installs Ruby for you, and gives you the option of specifying which gems to use with which Ruby versions. The rvm community respects its comprehensiveness and trusts rvm to synthesize these disparate tasks into one robust platform. Its complexity makes debugging harder. 

This explanation dives into some of the inner workings of each solution. If you understand their underpinnings, you can make your own judgments about their respective complexities and decide which approach seems more intuitive to you. 

## Installation

### RBENV

You can use Homebrew to install rbenv and ruby-build. Rbenv lets you manage which Ruby version you want to use. Ruby-build simplifies the process of installing new Ruby versions:

    $ brew update
    $ brew install rbenv ruby-build

*~/.bash_profile:*

    eval "$(rbenv init -)"

[Rbenv init](https://github.com/sstephenson/rbenv/blob/master/libexec/rbenv-init) enables autocompletion of all the commands that come along with rbenv. It also prepends the `~/.rbenv/shims` directory to your `$PATH` and automatically "rehashes" those shims. Shims allow rbenv to determine which Ruby version you need before running Ruby programs, and rehashing them keeps them up to date.

###RVM

    $ curl -L https://get.rvm.io | bash -s stable

"get.rvm.io" links to the [rvm installer](https://github.com/rvm/rvm/blob/master/binscripts/rvm-installer) script. It creates a new `.rvm` directory, adds the [rvm binaries](https://github.com/rvm/rvm/tree/master/bin) to your path, and loads the function "rvm" into your shell session. You can opt to run rvm as a [binary](https://github.com/rvm/rvm/blob/master/bin/rvm), but it's recommended to run it [as a function](https://rvm.io/workflow/scripting).

*~/.bash_profile:*

    [[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm" # Load RVM into a shell session *as a function*

    export PATH="$PATH:$HOME/.rvm/bin" # Add RVM to PATH for scripting


The rvm-installer [unpacks the rvm TAR file](https://github.com/rvm/rvm/blob/master/binscripts/rvm-installer#L302) and runs another installer script, which adds the code above to your `.bash_profile`:

*~/.rvm/scripts/functions/installer:*

    # $profile_file (.bash_profile, for example) and $local_rvm_path ($HOME/.rvm) have been set 

    # Insert quoted text into .bash_profile
    printf "%b" "
    export PATH=\"\$PATH:$local_rvm_path/bin\" # Add RVM to PATH for scripting
    " >> "$profile_file"

    # Insert quoted text into .bash_profile
    printf "%b" "
    [[ -s \"$local_rvm_path/scripts/rvm\" ]] && source \"$local_rvm_path/scripts/rvm\" #     Load RVM into a shell session *as a function*
    " >> "$profile_file"

 Rvm also has an [Autolibs](https://rvm.io/rvm/autolibs) install option for managing dependencies. Autolibs can be set to various modes. These modes permit rvm to install a package manager (like Homebrew) and dependencies, to just install dependencies, to just fail when a dependency is missing, and more.

## How rbenv/rvm determines your Ruby version
</n>

Both solutions work by adding the selected Ruby version's executable file to your `PATH` environment variable. This causes any program requesting to be run in Ruby to find that version first and use it. 

###RBENV

Shims are executables that intercept both the `ruby` command and commands that have executables with the [shebang](https://github.com/rspec/rspec-core/blob/master/exe/rspec#L1) `#!/usr/bin/env ruby`, like `rspec`.

Say you run: 

    $ rspec spec/features/edit.rb

Because shims are at the begining of your path, your system finds the shim named "rspec" and runs it. In the shim, the command (ie `rspec`) and any arguments (ie `spec/features/edit.rb`) are passed to `rbenv exec`:

*~/.rbenv/shims/rspec:*

    #!/usr/bin/env bash

    ...

    # Set the first argument ("rspec") to a variable called program
    program="${0##*/}"

    ...

    # Pass in program and "spec/features/edit.rb" to rbenv exec
    exec "/usr/local/Cellar/rbenv/0.4.0/libexec/rbenv" exec "$program" "$@"

`rbenv exec` determines which Ruby version you want to use (in [this](https://github.com/sstephenson/rbenv#choosing-the-ruby-version) order), ensures that the command (`rspec`)  will use that version by prepending the version's executable to `PATH`, and then runs it. The order in which rbenv returns the Ruby version to be prepended is numbered below:

*~/usr/local/Cellar/rbenv/0.4.0/libexec/rbenv-exec:*[^1]

    # Set RBENV_VERSION to "rbenv-version-name". See "rbenv-version-name" script below. 
    export RBENV_VERSION="$(rbenv-version-name)"

    ...

    # Rspec executable file. Ie "~/.rbenv/versions/1.9.3-p547/bin/rspec"
    RBENV_COMMAND_PATH="$(rbenv-which "$RBENV_COMMAND")"

    # Directory containing Ruby binary file. Ie "~/.rbenv/versions/1.9.3-p547/bin"
    RBENV_BIN_PATH="${RBENV_COMMAND_PATH%/*}"

    # Prepend the proper Ruby version to PATH, so RSpec shebang finds that first
    export PATH="${RBENV_BIN_PATH}:${PATH}" 

    ...

    # Run the proper Rspec script with argument you passed
    exec -a "$RBENV_COMMAND" "$RBENV_COMMAND_PATH" "$@"

*~/usr/local/Cellar/rbenv/0.4.0/libexec/rbenv-version-name:*

    # 1) If you have set an RBENV_VERSION environment variable and you have downloaded that version of ruby, use that
    if version_exists "$RBENV_VERSION"; then
        echo "$RBENV_VERSION"

      ...

    # If you have not set RBENV_VERSION, read the "rbenv-version-file"
    if [ -z "$RBENV_VERSION" ]; then
      RBENV_VERSION_FILE="$(rbenv-version-file)"
      RBENV_VERSION="$(rbenv-version-file-read "$RBENV_VERSION_FILE" || true)"
    fi

    # 4) If you have not set RBENV_VERSION, and if the "rbenv-version-file" outputs nothing, use the "system" Ruby
    if [ -z "$RBENV_VERSION" ] || [ "$RBENV_VERSION" = "system" ]; then
        echo "system"
      exit
    fi

*~/usr/local/Cellar/rbenv/0.4.0/libexec/rbenv-version-file:*

    # 2) Look in current directory for ".ruby-version" or ".rbenv-version" file. If file is there, use that Ruby version. If file is not there, move up a directory and look again. Etc. 
       find_local_version_file() {
      local root="$1"
      while [ -n "$root" ]; do
        if [ -e "${root}/.ruby-version" ]; then
          echo "${root}/.ruby-version"
          exit
        elif [ -e "${root}/.rbenv-version" ]; then
          echo "${root}/.rbenv-version"
          exit
        fi
        root="${root%/*}"
      done
    }

    # Run "find_local_version_file" function with $RBENV_DIR (your working directory) as an arg
    find_local_version_file "$RBENV_DIR"

    # If there is no local version file, see if a global version file is set
    global_version_file="${RBENV_ROOT}/version"

    ...

    # 3) And use that Ruby version
    echo "$global_version_file"

### RVM

Instead of hijacking your Ruby commands to get your Ruby version, rvm hijacks your "cd" command. When you cd into a directory, rvm searches certain files for the places where the Ruby version may be defined.

The first file it looks in is the `.rvmrc` file in your *project* directory, which might instruct rvm to use a specific Ruby version. (This is not to be confused with [system or user `.rvmrc` files](https://rvm.io/workflow/rvmrc)). The next files it looks in are (in order): `.versions.conf`, `.ruby-version`, `.rbfu-version`, `.rbenv-version`, `Gemfile`. 

When `cd` is called, rvm uses this process to locate the file: 

*~/.rvm/scripts/cd:*

    # Make functions in "rvmrc_project" file available to this file
    source "${rvm_scripts_path}/functions/rvmrc_project"

    ...

    # Run __rvm_project_rvmrc function
    __rvm_project_rvmrc >&2 || true

*~/.rvm/scripts/functions/rvmrc_project:*        

    __rvm_project_rvmrc() {

        # Set working_dir to the first argument or the pwd.
        working_dir="${1:-"$PWD"}"    

        ...

        # If you are not at your root directory, call "__rvm_project_dir_check" to get the first file to look in
        if
          [[ -z "$working_dir" || "$HOME" == "$working_dir" 

        ...


        else
            if
            __rvm_project_dir_check "$working_dir" found_file
    }

    __rvm_project_dir_check()
    {
        ...

        # Set the first argument ($working_dir) to a variable called "path_to_check"
        path_to_check="$1"  

        # Set the second argument (found_file) to a variable called "variable"
        variable="${2:-}"

        _valid_files=(
          "$path_to_check"
          "$path_to_check/.rvmrc" "$path_to_check/.versions.conf" "$path_to_check/.ruby-version"
          "$path_to_check/.rbfu-version" "$path_to_check/.rbenv-version" "$path_to_check/Gemfile"
          )

          # Call "__rvm_find_first_file", passing in a variable name and the "_valid_files" above
          __rvm_find_first_file _found_file "${_valid_files[@]}" || true

          ...

          # Set "variable" to "_found_file"
          if [[ -n "$variable" ]]
            then eval "$variable=\"\${_found_file:-$variable_default}\""
          fi
    }

*~/.rvm/scripts/functions/utility:*    

    __rvm_find_first_file() {
        ...

        # Set "_variable_first_file" to "_found_file"
        _variable_first_file="$1"

        # Loop through "_valid_files" in order. If one exists, set "_variable_first_file" to that and exit, returning 0
        for __file_enum in "$@"
          do
          if
              [[ -f "$__file_enum" ]]
          then
              eval "$_variable_first_file=\"\$__file_enum\""
              return 0
          fi
          done
          eval "$_variable_first_file=\"\""
          return 1
    }

If any of the files in `_valid_files` are in your working directory, the `_variable_first_file` (set to `_found_file` set to `found_file`) will be returned. At this point, rvm still has to look in this file to find where the Ruby version is defined. For the sake of this explanation, let's pretend that `found_file` is `.ruby-version` and see what happens at the end of the `__rvm_project_rvmrc` function: 

*~/.rvm/scripts/functions/rvmrc_project:*    

    __rvm_project_rvmrc()
    {
        ...

        # Run "_rvm_load_project_config", passing in "found_file"
        __rvm_conditionally_do_with_env __rvm_load_project_config "${found_file}"
    }    

    __rvm_load_project_config()
    {
        ...

        # depending on what "found_file" is...
        case "$1" in
          (*/.rvmrc)
              ...
          (*/.versions.conf)
              ...
          (*/Gemfile)
              ...
          (*/.ruby-version|*/.rbfu-version|*/.rbenv-version)
              ...

              # Take output of the file and set it to "rvm_ruby_string" variable   
              rvm_ruby_string="$( \command \tr -d '\r' <"$1" )"

    }

Once rvm has determined which version to use, it still needs to instruct any programs that run in Ruby to use this version by adding the appropriate ruby version's executable file to your PATH:

*~/.rvm/scripts/functions/rvmrc_project:*    


    __rvm_load_project_config()
    {    
        ...

        # After setting "rvm_ruby_string", call "_rvm_use"
        rvm_create_flag=1 __rvm_use || return 3
    }

*~/.rvm/scripts/functions/selector:*

    __rvm_use(){

      ...

      # Call "__rvm_use_"
      __rvm_use_ || return $?  
    }

    __rvm_use_(){
        ...

        # Add the following to __path_prefix (which later gets added to PATH):
        # GEM_HOME, the folder where rvm stores gem executables for Ruby version, ie: ~/.rvm/gems/ruby-2.2.2
        # "rvm_ruby_binary", which uses "rvm_ruby_string" to locate the file path where rvm stores Ruby binary for that version, ie: ~/.rvm/rubies/ruby-2.2.2/bin/ruby
        # "rvm_bin_path," the folder that stores all of rvm's own executables
        __path_prefix="$GEM_HOME/bin:${rvm_ruby_binary%/*}:${rvm_bin_path}"
    }

Now the proper Ruby version and any gems grouped under that version are added to `PATH`!

## Changing your Ruby version

### RBENV

In order of precedence:

1. `rbenv shell {version}` changes the `RBENV_VERSION` environment variable. 
2. `rbenv local {version}` creates/modifies a `.ruby-version` file in your current directory. You can also just create/modify one yourself.
3. `rbenv global {version}` creates/modifies the `~/.rbenv/version` file. You can also just create/modify one yourself.

If you'll always want to use a specific Ruby version in a specific directory (ie a Rails app), using a `.ruby-version` file is the best way to go. 

###RVM

As you saw earlier, to enable rvm to automatically switch Ruby versions, you can set the version in any of [these files](https://rvm.io/workflow/projects).

If you want to change the Ruby version you are currently using, you can run `rvm use {version}` at any point. If you want to change the Ruby version rvm will always use, you can run `rvm --default use {version}` at any point. Both of these functions ultimately run `__rvm_use`[^2] and are defined in rvm's "command line interface" script:

*~/.rvm/scripts/cli:*

    rvm()
    {
        ...

        # Calls "_rvm_parse_args" function
         __rvm_parse_args "$@"

        ...

        # Case statement that runs "__rvm_use" if "rvm_action" is "use"
        case "${rvm_action:=help}" in use)
          if rvm_is_a_shell_function
          then __rvm_use && __rvm_use_ruby_warnings
          fi 
    }

    __rvm_parse_args()
    {
        ...

        # Loop through arguments to "rvm" function. Set "rvm_token" to the first argument passed to __rvm_parse_args, then set it to the second argument passed, etc...
        while
          [[ -n "$next_token" ]]
          do
          rvm_token="$next_token"
          if
              (( $# > 0 ))
          then
              next_token="$1"
              shift
        else
            next_token=""
        fi


        case "$rvm_token" in

          # If the argument is "use" ...
          use)
              # Set "rvm_action" to "use"
            rvm_action="$rvm_token"

            ...

            # Set "rvm_ruby_string" (which __rvm_use needs!) to the next argument/actual version, i.e. 2.2.2
               rvm_ruby_string="$next_token"

          # If the argument is "--default" ...
          --head|--static|--self|--gem|--reconfigure|--default

              ...

              # Set ruby version flag. Rvm's "initialize" script checks for and uses this in next shell session. The next argument is "use", so rvm will keep moving through the loop defined above.
            rvm_token=${rvm_token#--}
            rvm_token=${rvm_token//-/_}
            export "rvm_${rvm_token}_flag"=1
            ;;
    }

## Installing Ruby versions

### RBENV

Use [ruby-build](https://github.com/sstephenson/ruby-build) with rbenv to install and list versions of Ruby. 

    $ rbenv install {version}

Ruby-build installs them to `~/.rbenv/versions`. If you open, for example, `~/.rbenv/versions/1.9.3-p547/bin/ruby` you can see the Ruby binary file.

###RVM 

    $ rvm install {version}

Say your version was 2.2.2. Rvm creates a folder called `~/.rvm/gems/ruby-2.2.2/bin`. In this folder it stores a script that wraps the Ruby binary[^3], which is located here: `~/.rvm/rubies/ruby-2.2.2/bin/ruby`. RubyGems also installs gem wrapper scripts here.

## Installing and managing gems
</br>
Rbenv's core functionality is to manage Ruby versions, not gems. It relies on [Bundler](http://bundler.io/) for gem management. Rvm also provides the option of utilizing Bundler, in addition to or as an alternative to its own gem management feature.

To set up your system (regardless of Ruby version manager), from your project directory: 

    $ gem install bundler
    $ bundle install

### RBENV

####Where gems are installed

The `bundle-install` man file (`~/.rbenv/versions/2.2.2/lib/ruby/gems/2.2.0/gems/bundler-1.10.5/lib/bundler/man/bundle-install`) indicates that: "The location to install the specified gems [...] defaults to Rubygems setting." Because `bundle install`  behaves like `gem install` in this regard, we can see how `gem install` determines where to install gems for the sake of this explanation:

    $ gem install rspec 

When `gem` is called, the respective rbenv shim is invoked. As you read earlier, the shim passes the command and its arguments to `rbenv exec`. `Rbenv exec` locates the proper version of Ruby and prepends that version's binary to your `$PATH`.[^4] 

When `rbenv-exec` runs your `gem install rspec` command, RubyGems looks for the directory where your Ruby version is installed (specifically `{version}/bin`), finding it in your $PATH. There, it creates script called a [Binstub](https://github.com/sstephenson/rbenv/wiki/Understanding-binstubs). Binstubs prepare the `$LOAD_PATH` with the gem's dependencies before the gem executable is called: 

*~/.rbenv/versions/1.9.3-p547/bin/rspec:*

    #!/usr/bin/env ruby
    # This file was generated by RubyGems.
    # The application 'rspec-core' is installed as part of a gem, and this file is here to facilitate running it.

    require 'rubygems'

    version = ">= 0"

    ...

    # prepare the $LOAD_PATH with gem's dependencies. Same as "require", except you can specify a version
    gem 'rspec-core', version

    # load gem's executable file
    load Gem.bin_path('rspec-core', 'rspec', version)

RubyGems also generates the gem's actual executable file, which lives here: `~/.rbenv/versions/2.2.2/lib/ruby/gems/2.2.0/gems/rspec-core-2.14.8/exe/rspec`

That's it! Your gem is installed! Now, whenever you run `rspec spec/features/edit.rb`, the shim, which calls the binstub[^5], which calls the executable, is called. 

####Managing installed gems 

Rbenv delegates gem management to Bundler. How Bundler manages gem versions and dependencies is beyond the scope of this explanation, but there are plenty of [resources](http://patshaughnessy.net/2011/9/24/how-does-bundler-bundle) available if you are curious. 

####Rehashing gems

Whenever you install/uninstall a gem with an executable, run `rbenv rehash`. This adds/removes shims for that executable. It is also a best practice to run this when you install new Ruby versions, in case those versions have additional commands that have not been shimmed.


###RVM

####Where gems are installed 

Both rbenv and rvm direct RubyGems to the directory in which to install gems by adding the a directory ending in `{version}/bin` to `PATH`[^6]. Rvm's gems are installed in `.rvm/gems/{version}/bin`.  

####Managing installed gems

Because the appropriate version of Ruby and the appropriate gem directory will be added to your `PATH`, you can delegate gem management to Bundler with rvm. Integrating Bundler and rvm should in theory [not involve doing anything special](https://rvm.io/integration/bundler).  

Rvm also utilizes a concept called [gemsets](https://rvm.io/gemsets/basics), directories that contain different versions of Ruby and the gems to use with them. Rvm exposes this mechanism to the user, so that you can create your own gemsets:

    $ rvm use 2.2.2
    $ rvm gemset create rails410

    # Start using your "rails410" gemset and installing gems to it
    $ rvm gemset use rails410
    $ gem install rails -v 4.1.0

The downside is that creating a gemset involves manually specifying which gems to use with each version of Ruby, and using a gemset requires you to remember to run `rvm gemset use` command every time. Bundler automates this process with project based Gemfiles, taking the burden off of you.

##Summary
</n>

Both tools have similar ways of enabling you to manually and automatically switch your Ruby version (using their commands to set a version or saving the version in a file). A core difference is the way rvm (vs. ruby-build and pure Bundler) installs Ruby versions and manages gems. 

Consider which approach seems more intuitive to you overall and if the complexity of rvm justifies the benefits it has, if any, over rbenv. If you can't find any benefits, rbenv is the way to go :)


[^1]: This assumes you have installed Homebrew.
[^2]: See `_rvm_use` function in `~/.rvm/scripts/functions/selector` file above.
[^3]: See `rvm_ruby_binary` in `~/.rvm/scripts/functions/selector` file above.
[^4]: See `RBENV_BIN_PATH` variable in `rbenv-exec` file above.
[^5]: See last line of `rbenv-exec` file above.
[^6]: See `GEM_HOME` in `~/.rvm/scripts/functions/selector`.


