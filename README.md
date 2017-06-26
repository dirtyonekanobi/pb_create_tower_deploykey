# pb_create_tower_deploykey
Ansible Playbook to Create a Tower Deploy Key

## INTRODUCTION

We use Ansible Tower with deploy keys in order to `push` to a GitHub repository.  By default, we only pull/clone from GitHub

However, in the case of data archiving or state updates, it is useful to upload data from Tower to GitHub.

Deploy Keys ensure we will only push data relevant to the repository in question, and not risk damaging any other repos.

Deploy keys are great for security, especially in this use case.  But.. they create a problem:
GitHub will only allow a deploy key to be used on one repository.  

Unfortunately - When attempting to deploy/push to multiple repositories, there is no way to easily select which key to use.

So, any time we need to push to a repository from Ansible Tower - we have to hack it. Fortunately, we have the technology.

We will use a Host feature of ssh to 'trick' the git subsystem to use a particular key.
____

## STEP 1: CREATE a New Deploy Key

*This assumes you have created a template in Ansible Tower, pointing to the `main.yaml` in this repo, with a survey, using a `{{ REPO_SHORTNAME }}` variable*

Find and launch the "CREATE_TOWER_DEPLOY_KEY" template. The survey will ask you to enter a 'Repo_Name.' 

This is a shortname/alias for the repo you would like to write to. Keep this handy for later.

Example: "post-deploy-config-archive" (it is easy to line this up w/your GH repository name, but not required.

After the playbook completes, it will print the output of the new PUBLIC key to the screen.  

Copy this key (beginning with `ssh-rsa`), or download the play output - keep the pubic key on hand for later step.

____

##STEP 2: Add New Deploy Key to GitHub 

Navigate to the repo in question and select `Settings`.  (you must own or have write/admin access to this repository)

Navigate to `Deploy Keys`.  Add a new key and give this a name.  Copy the output of the ssh key to the text area.  Save and step away.
____

## STEP 3: Using this in your playbook 

On the backend, that playbook created an alias for 'post-deploy-config-archive' to point `github.com` (or your enterprise GH URL)

Now, when calling `post-deploy-config-archive` it will redirect to GitHub... 

To use these new fancy spices, we will only need to change our push command.

Instead of using `git push origin [BRANCH]` we will need to call our alias. 
`origin` is an alias for `git@github.com:[USER]/[REPO].git`;

Using the above example we would call `git push git@github.com:[USER/ORG]/post-deploy-config-archive.git` 

This would fail via Tower, because git wouldn't know what ssh key to use.

To utilize the alias, issue the command: `git push git@post-deploy-config-archive:[USER/ORG]/post-deploy-config-archive.git [BRANCH]`

Notice the difference?  Instead of calling our GitHub url, we call the repo name we entered in the 1st step.

If we are using our playbook on Tower, and our local computer, we would want to setup a conditional - to ensure the alias doesn't get attempted locally.

An easy check is to use the lookup module `current_user: {{ lookup('env', 'USER') }}'

This will return the environment variable for the current user.  You can then use a `when` conditional.

`when current_user == 'awx'` (The Tower default runner is named 'awx')

Example: [GITHUB LINK](https://github.com/dirtyonekanobi/pb_create_tower_deploykey/git_commit.yaml)

This way git will use your *normal* key when on your local computer, and the aliased deploy key on Tower.
____

##HOW DOES THIS WORK 

One of the steps in the *DEPLOY_KEY* playbook adds a block of text to Tower's `$HOME/.ssh/config` file.

```
### EXAMPLE ###

# Configured By Ansible
Host post-deploy-config-archive
HostName github.build.ge.com
IdentityFile $HOME/.ssh/post-deploy-config-archive

### END EXAMPLE ###
```

By doing this, anytime SSH invokes w/a url of `post-deploy-config-archive` it is redirected to GitHub, and uses the key indentified in the IdentityFile line.

### BOOM (*drops mic*)
![](https://media.giphy.com/media/w7mLEAMcpjrpe/giphy.gif)
