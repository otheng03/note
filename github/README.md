# SSH key management

**Generate SSH key**

```
ssh-keygen -t ed25519 -C "07c00h@gmail.com"
```

**Add SSH key to agent**

```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

**Add SSH key to GitHub**

```
cat ~/.ssh/id_ed25519.pub
# Then select and copy the contents of the id_ed25519.pub file
# displayed in the terminal to your clipboard
```

**Add SSH key to GitHub**

In the upper-right corner of any page on GitHub, click your profile photo, then click  Settings.

In the "Access" section of the sidebar, click  SSH and GPG keys.

Click New SSH key or Add SSH key.
