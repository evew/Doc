# Use git

## Set Up Git

Tell Git your name so your commits will be properly labeled. 

`$ git config --global user.name "evew"`

Tell Git the email address that will be associated with your Git commits.The email you specify should be the same one found in your email settings.To keep you email address hidden.see "Keeping you email address private."

`$ git config -- global user.email "MY EMAIL ADDRESS"`

## Clone with SSH

Generating a new SSH key

    $ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    # Creates a new ssh key, using the provided email as la label
    Generating public/private rsa key pair.

    Enter a file in which to save the key (/Users/you/.ssh/id_rsa): [Press]

    Enter passphrase (empter for no passphrase):[Type a passphrase]
    Enter same passphrase again: [Type passphrase again]

Adding your SSH key to the ssh-agent
