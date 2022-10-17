# multiple-accounts

Notes on handling multiple github accounts

## Initial Notes

### Goal:
I dont like remembering things, so want to just be easily able to switch between github accounts when im coding locally regardless of where i am and what im doing.

### Needs:
- want to just use my password manager for ssh keys
- want the profile switch to work for both git and gh together

## Password Manager

> If you dont have a password manager, I highly recommend getting/using one. If your password manager does not have assistance with ssh integration, an alternative is is to use [direnv](https://direnv.net/) but that requires manual additions for every directory you're in.

I use [1Password](https://1password.com/) which [has a way](https://developer.1password.com/docs/ssh/get-started) to integrate with the ssh setup while still allowing the items to be defined by each user.

1. Make a new ssh key directly in the 1password ui. This creates a private key on start and then autogenerates the remaining needed items (pub, etc).

<img width="378" alt="Screen Shot 2022-10-16 at 5 19 36 PM" src="https://user-images.githubusercontent.com/114540173/196066196-702c6d3a-98f8-4bc3-ae81-6e487220ac4a.png">

2. Add the created ssh private key to the same github account's profile.

<img width="582" alt="Screen Shot 2022-10-16 at 5 29 29 PM" src="https://user-images.githubusercontent.com/114540173/196066636-d5e3f6e5-c41c-4b8a-ac23-9be82ecff855.png">

2.5 Repeat steps 1 and 2 for all github accounts you need

We're going to be doing this for github `username_a` and `username_b` respectively for this example.

3. Then you need to make your local setup use that info.

Run the following to help with ssh setup (1pass files can be in odd locations) by creating a `~/.1password` folder for easy access (linux/mac).
From the green tip section of [instructions](https://developer.1password.com/docs/ssh/get-started#step-4-configure-your-ssh-or-git-client).
````
mkdir -p ~/.1password && ln -s ~/Library/Group\ Containers/2BUA8C4S2C.com.1password/t/agent.sock ~/.1password/agent.sock
````

The local setup boils down to the following edits being added to the user info:

`~/.bashrc`
````
source ~/.secrets
source ~/.alias
````
`~/.secrets`
````
# 1Password
# https://developer.1password.com/docs/ssh/get-started#step-4-configure-your-ssh-or-git-client
# this is needed bc [not every setup](https://developer.1password.com/docs/ssh/agent/compatibility) allows use of IdentityFile in the config
export SSH_AUTH_SOCK=~/.1password/agent.sock
````
`~/.ssh/config`
````
Host username_a_host
  HostName github.com
  User git
  IdentityFile ~/.ssh/username_a_ssh_key.pub
  IdentitiesOnly yes
 
Host username_b_host
  HostName github.com
  User git
  IdentityFile ~/.ssh/username_b_ssh_key.pub
  IdentitiesOnly yes
````
`~/.ssh/username_a_ssh_key.pub`
> :warning: make sure the file has [chmod 600](https://unix.stackexchange.com/questions/257590/ssh-key-permissions-chmod-settings)
````
copied down version from 1pass of the public key for username_a
````
`~/.ssh/username_b_ssh_key.pub`
> :warning: make sure the file has [chmod 600](https://unix.stackexchange.com/questions/257590/ssh-key-permissions-chmod-settings)
````
copied down version from 1pass of the public key for username_b
````

4. Turn on 1pass ssh agent

[Instructions](https://developer.1password.com/docs/ssh/get-started#step-3-turn-on-the-1password-ssh-agent) specify the following settings for use:

-------->INSERT PHOTOS HERE<-------

## Switch between github accounts on gh

1. Download gh (github cli) - [instructions](https://cli.github.com/manual/installation)

2. login to a specific gh account
> This is needed so we can download a `gh` extension.

```
gh auth login username_a
```

and then go through the appropriate steps until you're fully logged in as said user.

that is, when you run `gh auth status` it should give something like the following:

````
github.com
  ✓ Logged in to github.com as username_a (oauth_token)
  ✓ Git operations for github.com configured to use ssh protocol.
  ✓ Token: *******************
````

:warning: DO NOT LOGOUT YET

3. Install the [gh profile](https://github.com/gabe565/gh-profile) extension and set each up for `username_a` and `username_b`.
> Alternatively you can do this manually with [yermulnik's workaround](https://gist.github.com/yermulnik/017837c01879ed3c7489cc7cf749ae47).

**Do the following for each profile, we'll be doing this for username_a and username_b**.

BEGIN_FOR_EACH_USERNAME_LOOP

3.1 Create the profile

````
gh profile create username_blah
gh profile switch username_blah
````

What this does is create the following file structure in your `~/.configs/gh` directory

````
config.yml
hosts.yml
profiles
 |
  - username_blah
    |
    - hosts.yml
````

:warning: This might log you out of `gh auth`. Dont worry as that is expected since we are changing the focus from the original `gh/hosts.yml` file to a profile-specific one (aliasing `gh/hosts.yml` in the process). What this does is it sets up the `gh profile` extension, which sym points to a specific gh profile's hosts.yml file instead of the direct `gh auth login`'s singular hosts.yml file.

3.2 Fill out the profile

Since you're on the correct profile, login to gh
> If you do this and then you usually redirect to the web, make sure that you are also logged into the correct github profile at github.com first as it'll just auth to whatever is already there.

````
gh auth login
````

3.3 Check that you logged in for the right profile

aka check that `gh auth status` shows the information that matches your current `gh profile switch

Alternatively, you can just manually check that the `~/.configs/gh/profiles/username_blah/hosts.yml` file has info in it that matches the `username_blah` profile.

Example: `username_blah` logged in with `ssh`
> Note - this and the oauth_token should be filled out for you from the `gh auth login`. Do not fill this out manually.
````
github.com:
    user: username_blah
    oauth_token: gho_THETOKEN
    git_protocol: ssh
````

:warning: if you want to login with one username at ssh && http depending on the repo, it's best to make two separate profiles so you can call them directly.
:bangbang: Once you setup your `gh profiles` you dont need to use `gh auth login` or `gh auth logout` anymore. If you'd prefer to use those to let others login, make sure to switch to the default profile first or `gh profile create random_user` so your other profiles dont get overwritten if theyre the profile that's set when the login happens.

END_FOR_EACH_USERNAME_LOOP

## Connect git config and gh

1. Cleanup the git config

If you've used git config before to create a global user.name and user.email, make sure to delete them from your config file

You can do so by running:
````
git config --global --edit
````

OR much simpler:
````
git config --global --unset user.name
git config --global --unset user.email
````

2. Fixing repo auth for ssh.

Since 1pass is our ssh auth, this technically means we have a different host; that is, everytime you clone a new repo you just need to update the host from git@github.com to the 1pass host in your configs.

Simple way to do that is with the following aliases:
> The `gh auth setup-git` in the alias links your `git config` profile to whatever is your `gh auth` profile.
> I like keeping `ghswitch` and `sshswitch` separate, but might be easiest just to have them as one command
`~/.alias`
````
alias ghswitchparams='echo -e "1: user.name"'
alias ghswitch='function _blah(){ gh profile switch $1; gh auth setup-git; };_blah'

alias sshswitchparams='echo -e "1: user.name\n2: <org-OR-username>/<repo-name>"'
alias sshswitch='function _blah(){ git remote set-url origin "$1_host:$2.git"; echo "Using 1pass-ssh. remote origin set to $1_host:$2.git instead of regular git@github.com:$2.git";};_blah'
````

## Complete :partying_face:

Should now be able to switch easily between github accounts locally and keep git and gh in sync however you want.
