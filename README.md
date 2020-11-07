## Systemd User PAM SSH script

This script has a rather specific use case. If you fit the following demographic
then this script might just be for you!

 * You use systemd
 * You login at the linux VT using a getty
 * You have a `systemd --user` service called `ssh-agent.service` that starts
   your ssh agent.
 * You have to type your passphrase after logging in in order to
   decrypt your SSH key.

This script allows you to only type your password once. When logging in, your
SSH key will be decrypted and added to your ssh-agent for you.

## Installation

(1) Set up your ssh-agent systemd user service with the proper
    environment using lingering to start it at boot

    # setup module
    echo '[Unit]
    Description=SSH key agent

    [Service]
    Type=simple
    Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
    ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

    [Install]
    WantedBy=default.target' \
      > ~/.config/systemd/user/ssh-agent.service

    # setup environment
    echo 'SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/ssh-agent.socket"' \
      > ~/.pam_environment

    # enable now and at boot
    systemctl --user start ssh-agent
    systemctl --user enable ssh-agent

    # enable lingering
    loginctl enable-linger $(whoami)

(2) Install the script to a well-known location (You can modify `/usr/lib`)

    sudo curl -o /usr/lib/systemd/systemd-user-pam-ssh \
    https://raw.githubusercontent.com/capocasa/systemd-user-pam-ssh/master/systemd-user-pam-ssh

    chmod +x /usr/lib/systemd/systemd-user-pam-ssh

(3) Configure pam

    echo "auth  optional  pam_exec.so  expose_authtok /usr/lib/systemd/systemd-user-pam-ssh" \
    | sudo tee -a /etc/pam.d/login

(4a) Use your system password as your private key passphrase (not recommended)

    ssh-keygen -p -f ~/.ssh/id_rsa
    # type your system password


(4b) Encrypt a passphrase with your system password and a heavy derivation function (recommended)

    ## Change your passphrase (optional)
    
    ssh-keygen -p -f ~/.ssh/id_rsa
    # type your passphrase

    ## Save your passphrase encrypted with your system password

    read -s PASSWORD
    # type your system password

    read -s PASSPHRASE
    # type your passphrase

    echo $PASSPHRASE | openssl enc -pbkdf2 -in - -out ~/.ssh/passphrase -e -aes256 -k "$PASSWORD"
    
    unset PASSWORD
    unset PASSPHRASE

---
This is an extended version of the [original script by EvanPurkhiser](https://github.com/EvanPurkhiser/systemd-user-pam-ssh)
