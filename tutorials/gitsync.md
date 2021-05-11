As a developer, the temptation to spend three hours automating a task that takes thirty seconds is constantly looming and ever-alluring. So, having giving into that temptation for synchronizing git repositories, I'm going to help you out a little.

```
url=$(git remote get-url origin)
token=$(cat $path/to/your/.git-token)

before="https://"
after="https://$token@"

url="${url/$before/$after}"

git pull origin main

git add .

git commit -m "Automated commit on `date`"

git push $url
```

This is a script I wrote to automate git commits. It will commit all changes in a repository and push it to the remote, with a commit message dated to that commit.
